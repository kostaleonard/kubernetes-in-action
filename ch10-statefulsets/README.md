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

## Discovering peers in a StatefulSet

Peer discovery is an important part of StatefulSets (consider distributed databases). Kubernetes uses DNS, specifically SRV records, to enable peer discovery for StatefulSets.

To list the SRV records for your stateful pods, you can create a temporary pod and run `dig` in it:

```bash
kubectl run -it srvlookup --image=tutum/dnsutils --rm --restart=Never -- dig SRV kubia.default.svc.cluster.local
```

You will see 2 SRV records indicating the FQDN of your stateful pods, as well as 2 A records indicating their IP addresses.

### Implementing peer discovery through DNS

We've updated the app (available at `docker.io/luksa/kubia-pet-peers`) so that a GET request to the service root (i.e., `GET kubia-public/`) causes the app to return the data across the entire StatefulSet cluster. The app will perform a DNS SRV lookup to get the hostnames and IP addresses of all the pods backing the service, then will send GET requests to each pod's `/data` endpoint to get the data on that pod.

### Updating a StatefulSet

Since the StatefulSet is running, we'll update the template with `kubectl edit`. You could also use commands like `patch` and `apply`. Depending on your Kubernetes version, you may need to delete the pods manually so that they update to the new image.

### Trying out your clustered data store

First, you can add some data to random nodes in the cluster with the following. Make sure the proxy is running.

```bash
curl -X POST -d "Parallel Lives" localhost:8001/api/v1/namespaces/default/services/kubia-public/proxy/
```

Now you can read the stored data with:

```bash
curl localhost:8001/api/v1/namespaces/default/services/kubia-public/proxy/
```

## Understanding how StatefulSets deal with node failures

StatefulSets guarantee that there will never be two pods running with the same identity and storage. When a node appears to fail (it might just be that the kubelet on that node failed and all of the pods are healthy), a StatefulSet will not create a replacement pod. It will only create a new pod when the cluster administrator deletes the pod or the whole node.

### Simulating a node's disconnection from the network

**Note: This exercise cannot be completed on Minikube since it requires multiple nodes.**

To simulate a node disconnecting from the network, ssh into a node and turn off its network interface. Kubernetes will mark the node as "NotReady", which you can see by running `kubectl get nodes`. The status of all pods running on that node will become "Unknown", as seen in the output of `kubectl get pods`.

### Deleting the pod manually

Pods with "Unknown" status are automatically deleted after some configurable time. You can also delete the pod manually with `kubectl delete pod kubia-0`. Either way, the StatefulSet will not create a new pod because it can't get in contact with the node's Kubelet to get confirmation that the pod has been deleted. So, you can forcibly delete the pod with `kubectl delete pod kubia-0 --force --grace-period 0`.

**Note: Don't delete stateful pods forcibly unless you know that the node is no longer running or is unreachable, and will remain so forever.**
