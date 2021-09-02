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

* Resource contains (or does not contain) a label with a certain key: `kubectl get pods -l env`, `kubectl get pods -l '!env'`
* Resource contains a label with a certain key and value: `kubectl get pods -l creation_method=manual`, `kubectl get pods -l 'env in (prod,devel)'`, `kubectl get pods -l 'env notin (prod,devel)'`
* Resource contains a label with a  certain key, but with a value not equal to the one you specify: `kubectl get pods -l creation_method!=manual`
* Multiple comma-separated criteria: `kubectl get pods -l env,creation_method=manual`

## Using labels to constrain pod scheduling

Usually, we don't care how Kubernetes decides to schedule pods to worker nodes; this is how Kubernetes is designed--all worker nodes are exposed to the developer as a single, large deployment platform. However, sometimes we want to be able to constrain pod scheduling. For instance, if the worker nodes do not have homogeneous hardware, we may want to schedule certain pods only on worker nodes that have GPU acceleration.

Don't specify the exact node you want something deployed on, though. Just specify the requirements of the node. This prevents tight coupling of the app with infrastructure.

From `kubia-gpu.yaml`. Notice the GPU node selector that forces the app to be deployed on a GPU node.

```
apiVersion: v1
kind: Pod
metadata:
  name: kubia-gpu
spec:
  nodeSelector:
    gpu: "true"
  containers:
  - image: kostaleonard/kubia
    name: kubia
```

Note that if no node exists that matches the selection specifications, the pod will fail to deploy (status will be stuck at "Pending").

## Annotations

Annotations are key-value pairs similar to labels, but they can't be used for matching. They are used to store resource information that could be useful (e.g., a short description of the resource, the author of the resource).

### Looking up an object's annotations

You can look up an object's annotations either by requesting the full YAML with `kubectl get pod kubia -o yaml` or by running `kubectl describe pod kubia`.

### Modifying an object's annotations

Annotations can be added in the YAML or modified later with `kubectl annotate pod kubia-manual mycompany.com/someannotation="foo bar"`. Since various tools and libraries will annotate your objects, a good way to prevent collisions is to format annotation keys with prefixes as shown above, e.g., `mycompany.com/xyz`.

## Namespaces

What if you want to split a large, complex system with many components into smaller distinct groups so that they can be more easily managed with Kubernetes? Or what if you are operating in a multi-tenant environment and need to split resources based on that? Namespaces split Kubernetes resources into separate, non-overlapping groups. Resource names need only be unique within a namespace. Kubernetes namespaces are not the same as Linux namespaces, which Docker uses to provide process isolation and Kubernetes uses to provide pod isolation.

Some resources are not namespaced (i.e., they are cluster-level resources). One of these is the Node resource.

Namespaces also allow Kubernetes to split off system resources so the default namespace isn't polluted and you don't inadvertantly delete them.

### Listing namespaces and their pods

List all namespaces in the cluster with `kubectl get ns`. List all pods in a specific namespace with `kubectl get pods --namespace default`. You can also try other namespaces like `kube-system`.

### Creating a namespace from a YAML

From `custom-namespace.yaml`:

```
apiVersion: v1
kind: Namespace
metadata:
  name: custom-namespace
```

### Creating a namespace with `kubectl create namespace`

You can also create a namespace directly with `kubectl create namespace custom-namespace`.

### Managing objects in other namespaces

To create resources in a specific (non-default) namespace, either add a `namespace: custom-namespace` entry to the resource's YAML, or specify the namespace at creation time with `kubectl create -f kubia-manual.yaml -n custom-namespace`.

`-n` is usually short for `--namespace`.

The default namespace can be changed with `kubectl config`. Try `alias kcd='kubectl config set-context $(kubectl config current-context) --namespace'` so you can switch namespace contexts with `kcd some-namespace`.

## Stopping and removing pods

### Deleting a pod by name

Deleting a pod causes Kubernetes to send a `SIGTERM` to all processes in the pod's container(s). After some time (default 30 seconds), Kubernetes will send a `SIGKILL` if the process has not terminated. So, to shut down gracefully, your processes need to be able to handle a `SIGTERM`.

To delete a pod, run `kubectl delete pod kubia-gpu`.

### Deleting pods using label selectors

To delete pods by label, use `kubectl delete pods -l creation_method=manual`

### Deleting pods by deleting the whole namespace

You can delete a namespace and all pods within it using `kubectl delete ns custom-namespace`.

### Deleting all pods in a namespace, while keeping the namespace

To delete all pods in a namespace, but preserve the namespace itself, run `kubectl delete pods --all`.

### Deleting (almost) all resources in a namespace

To delete all resources in a namespace, run `kubectl delete all --all`. The first `all` specifies that you want to delete resources of all types, and the second `all` specifies that you want to delete all instances of those resource types.

Using `kubectl delete all` won't delete some resources, e.g., Secrets.

This command will also delete the `kubernetes` service, but that will be recreated automatically after a few moments.
