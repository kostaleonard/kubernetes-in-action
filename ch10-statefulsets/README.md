# Chapter 10: StatefulSets: deploying replicated stateful applications

## Replicating stateful pods

Suppose each pod keeps its own state in your application (i.e., not a RESTful application, but a stateful application). You will likely need to allow each pod to keep its own volume. But, if the pod template uses a PersistentVolumeClaim and pods are managed by a ReplicationController or ReplicaSet, all of them will share the same claim and, therefore, volume.

### Running multiple replicas with separate storage for each

There are ways to create separate storage for replicated pods, but they are cumbersome and error-prone.

### Providing a stable identity for each pod

Pods are ephemeral and can be moved at will. Sometimes you need a stable identity (e.g., network address) for each pod. This is common in distributed stateful applications. An ugly solution is to have a service for every single pod. This will work because service IPs are stable, but pods won't be able to know through which service they are exposed.

## Understanding StatefulSets

StatefulSets were designed to solve the above problems. They allow the user to treat instances of the app as non-fungible entities, each having a stable name and state.

### Comparing StatefulSets with ReplicaSets

Stateful apps are like pets and stateless apps are like cattle (fungibility). StatefulSets manage stateful apps, while ReplicaSets and ReplicationControllers manage stateless apps.

**Specifically, pods created by StatefulSets each have their own set of volumes (i.e., persistent state) that differentiates them from their peers, and a predictable, stable network identity in the form of pod name and hostname.**

### Providing a stable network identity

Kubernetes gives a stable network identity to each of the pods by making you create a headless "governing" service. Through the headless service, each pod will have its own DNS entry. When a pod backed by a StatefulSet is rescheduled, it acquires the same name and hostname as the pod that disappeared.

### Providing stable dedicated storage to each stateful instance

As discussed before, each pod in a StatefulSet needs its own PersistentVolumeClaim (in a ReplicaSet, all pods share a single PersistentVolumeClaim and PersistentVolume). StatefulSets therefore have, in addition to a pod template, one or more volume claim templates to create new PersistentVolumeClaims along with each pod. When a StatefulSet is scaled down, only the pod is deleted, leaving any claim(s) to be reattached to a new pod if the StatefulSet is scaled up. The PersistentVolumeClaim(s) must be deleted manually, since that may not be what is intended.

### Understanding StatefulSet guarantees

Kubernetes guarantees not only stable, persistent storage and network identity for each pod, but also that **at most one** instance of each pod is running. This is to prevent collisions in network identity and storage.

## Using a StatefulSet

### Creating the app and container image

The app we are using is a modified version of the original kubia app that stores POST request content in a file, `/var/data/kubia.txt`. The image is available at `docker.io/luksa/kubia-pet`.

### Deploying the app through a StatefulSet

To deploy the app, you need 3 different kinds of objects:

1. PersistentVolumes for storing data files (you won't need to create these if your cluster supports dynamic provisioning of PersistentVolumes).
1. A governing Service required by the StatefulSet.
1. The StatefulSet itself.

We will dynamically provision the PersistentVolumes using the k8s.io/minikube-hostpath provisioner that comes in the standard StorageClass on Minikube.

The headless service can be created from a yaml file. From `kubia-service-headless.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  clusterIP: None
  selector:
    app: kubia
  ports:
  - name: http
    port: 80
```

Lastly, the StatefulSet is defined as follows. From `kubia-statefulset.yaml`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kubia
spec:
  serviceName: kubia
  replicas: 2
  selector:
    matchLabels:
      app: kubia
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: luksa/kubia-pet
        ports:
        - name: http
          containerPort: 8080
        volumeMounts:
        - name: data
          mountPath: /var/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      resources:
        requests:
          storage: 1Mi
      accessModes:
      - ReadWriteOnce
```

Create both the Service and the StatefulSet.

### Playing with your pods

Before, we've connected to pods by running an exec inside another pod in the cluster, using port-forwarding, connecting to services, etc. Now, we're going to use the API server as a proxy to the pods. First, use `kubectl proxy` to have Kubernetes handle authentication with the API server. Then, connect to the pods.

This command blocks:

```bash
kubectl proxy
```

The trailing slash is necessary unless you use `-L` to follow redirects.

```bash
curl localhost:8001/api/v1/namespaces/default/pods/kubia-0/proxy/
```

Now we'll send a POST request to the `kubia-0` pod.

```bash
curl -X POST -d "Hey there! This greeting was submitted to kubia-0." localhost:8001/api/v1/namespaces/default/pods/kubia-0/proxy/
```

If you send the GET request again, you will see that your data has been stored in the `kubia-0` pod's volume. You can verify by sending a GET request to the `kubia-1` pod and seeing that no data has been stored.

If you delete the `kubia-0` pod directly (`kubectl delete pod kubia-0`), the StatefulSet will create a new pod also named `kubia-0` to replace the old one. If you send the GET request again to the new pod, you will see that the storage persisted correctly.

Scaling down a StatefulSet will leave the PersistentVolumeClaims intact, as we have said. So if the StatefulSet is scaled up again, the new pod will have the same state as the original unless the PersistentVolumeClaim is manually deleted.

Now we'll add a (regular) Service for clients to connect to the pods. From `kubia-service-public.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia-public
spec:
  selector:
    app: kubia
  ports:
  - port: 80
    targetPort: 8080
```

This service is accessible from inside the cluster. You can use some of the methods previously described to hit the service, or you can access it through the proxy:

```bash
curl localhost:8001/api/v1/namespaces/default/services/kubia-public/proxy/
```

You'll notice that you get a random node each time, which is not always what you want--we will improve on this later.
