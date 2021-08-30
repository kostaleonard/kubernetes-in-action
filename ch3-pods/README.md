# Chapter 3: Pods

## About pods

Pods are the basic building block of Kubernetes. Pods wrap containers or groups of containers that need to work closely with each other. It is very common for pods to wrap only a single container. Containers provide isolation for single processes from each other; pods group containers so that they can share some, but not all, resources--that is, pods reduce some of the isolation between related containers. Some of these shared resources include, by default:

* Network resources: ports, interfaces.
* Hostname
* IPC namespace

Notably, containers in the same pod do not share filesystems or PID namespaces by default.

Containers in the same pod will always run on the same physical machine; the pod is an atomic unit.

Each pod has its own static IP address that is routable to every other pod in the cluster as if they all share the same flat LAN, regardless of physical network topology.

Think of pods as separate machines where each contains only a single app (which may have related subcomponents).

## Creating pods from YAML and JSON

### Anatomy of a YAML

The YAML for a Kubernetes object almost always contains 3 parts:

1. `metadata`: Object metadata, e.g., name.
1. `spec`: Object specifications and contents.
1. `status`: Object status--read only.

You only need to define the first 2 when writing a YAML.

### Examining a YAML descriptor of an existing pod

To view the YAML definition of an existing pod, run:

```
kubectl get pod kubia -o yaml
```

### Writing a YAML

Use `kubectl explain <resource>` to see which attributes that resource (e.g., pod) supports. These can be defined in the YAML. You can also examine the attributes themselves with `kubectl explain pods.spec`. Alternatively, see the [Kubernetes API reference documentation](https://kubernetes.io/docs/concepts/overview/kubernetes-api/) for help preparing manifests of specific objects.

From `kubia-manual.yaml`:

```
apiVersion: v1
kind: Pod
metadata:
  name: kubia-manual
spec:
  containers:
  - image: kostaleonard/kubia
    name: kubia
    ports:
    - containerPort: 8080
      protocol: TCP
```

### Creating a pod from a YAML

To create a resource (here, pod) from a YAML, use:

```
kubectl create -f kubia-manual.yaml
```

## Viewing application logs

Containerized applications usually write logs to stdout and stderr instead of writing to files. Docker automatically redirects those streams to files so they can be viewed later.

If you have the container ID, you can run `docker logs <container-id>` to see the logs. You can do this if you `ssh` into the machine that has the pod you're looking for and run `docker ps` to get the container ID. Using `minikube`, that might look like:

```
minikube ssh
docker ps
docker logs <container-id>
```

Or, you can just use `kubectl logs <pod-name>` to get the logs for the containers on the pod (this is part of the reason why it is preferrable to have one container per pod if possible).

```
kubectl logs kubia-manual
```

If your pod has multiple containers, you can get the logs for just one container with:

```
kubectl logs kubia-manual -c kubia
```

## Sending requests to the pod with port  forwarding

Apart from Services, there are other ways to interact with deployed pods. Port forwarding is one that is usually done for debugging.

```
kubectl port-forward kubia-manual 8888:8080
```

## Organizing pods with labels

### About labels

Pods and other Kubernetes resources can be organized with labels. Operations can then be applied to all resources with a given label.

Labels are key-value pairs, e.g., `app: ui` or `rel: stable`. Resources can have many labels, but only one value per key. Labels are usually added at resource creation time.

### Creating pods with labels

From `kubia-manual-with-labels.yaml`:

```
apiVersion: v1
kind: Pod
metadata:
  name: kubia-manual-v2
  labels:
    creation_method: manual
    env: prod
spec:
  containers:
  - image: kostaleonard/kubia
    name: kubia
    ports:
    - containerPort: 8080
      protocol: TCP
```

Deploy with `kubectl create -f kubia-manual-with-labels.yaml` and then view pods and labels with `kubectl get pods --show-labels` or `kubectl get pods -L creation_method,env`.

### Modifying labels of existing pods

Use the `kubectl label` command.

```
kubectl label pod kubia-manual creation_method=manual
```

When you overwrite a label, use the `--overwrite` flag.

```
kubectl label pod kubia-manual-v2 env=debug --overwrite
```

## Selecting pods with label selectors

You can select subsets of pods based on labels and their values.

```
kubectl get pods -l creation_method=manual
```
