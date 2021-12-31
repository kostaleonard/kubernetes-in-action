# Chapter 14: Managing pods' computational resources

Setting how much a pod is expected and allowed to consume is a vital part of any pod definition. It affects resource allocation (obviously) and scheduling.

## Requesting resources for a pod's containers

* Request: The amount of CPU/memory a container needs.
* Limits: An upper bound on the CPU/memory a container can consume.

### Creating pods with resource requests

From `requests-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: requests-pod
spec:
  containers:
  - image: busybox
    command: ["dd", "if=/dev/zero", "of=/dev/null"]
    name: main
    resources:
      requests:
        cpu: 200m
        memory: 10Mi
```

You are requesting 200 millicores (one-fifth of a CPU core's time; 5 such pods can run sufficiently fast on a single CPU core) and 10 mebibytes of memory.

### Understanding how resource requests affect scheduling

Requests determine which nodes are available for scheduling; if a node cannot meet the resource requests for a pod, then the Scheduler will not assign the pod to that node. The Scheduler doesn't look at the amount of resources being consumed on a node when making this decision. It looks at the sum of the resources requested by the existing pods deployed on the node.

To view the resources available to a node, run `kubectl describe nodes`. This will tell you, among other things, how many CPU cores and how much memory your node has available. Note that system pods (in the `kube-system` namespace) may consume some node resources.

### Understanding how CPU requests affect CPU time sharing

CPU requests affect both scheduling and how CPU time is distributed between containers. If one container requests some amount of CPU, and another container on the same node requests 5 times as much CPU, the CPU's time will be split between the containers in a 1 to 5 ratio (unless one or both of the two containers is idle).

### Defining and requesting custom resources

You can add custom resources to a node and request them in the pod's resource requests. Add the custom resource to the Node object's `capacity` field. You will have to provide a name and an integer quantity. The Scheduler will make sure pods are not scheduled to a node unless there are enough instances of the custom resource for new pods.

## Limiting resources available to a container

Resource requests ensure that containers get the minimum resources they require. Limits set the maximum a container can consume.

### Setting a hard limit for the amount of resources a container can use

CPU is a compressible resource, meaning it can be throttled without affecting the process running in the container adversely. Memory is not compressible, so it is important to limit the amount of memory a container can use.

**Note: Limits do not affect scheduling, so if your program consumes more memory than it requests, it could make the node unusable for other pods (even though there may appear to be enough memory to fill the requests of new pods).**

Limits are specified in almost the same way as requests. From `limited-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: limited-pod
spec:
  containers:
  - image: busybox
    command: ["dd", "if=/dev/zero", "of=/dev/null"]
    name: main
    resources:
      limits:
        cpu: 1
        memory: 40Mi
```

**Note: If your pod fails to run (error `FailedCreatePodSandBox` or similar), you may need to increase the memory limit. I doubled the memory limit given in the book and the pod ran successfully.**

**Note: Resource requests default to the limit values provided.**

Resources can be overcommitted. In other words, it is possible to have less than 100% of the resources on a node requested, but have more than 100% in use because the limits exceed the requests. This may cause Kubernetes to kill containers.

### Exceeding the limits

When a CPU limit is set for a container, the process isn't given more CPU time than the configured limit. But with memory, the container will be killed if it tries to allocate more memory than its limit provides.

### Understanding how apps in containers see limits

Containers see the node's memory and CPUs, not the container's. The container will not be aware of limits you set for it. You can, however, pass limits to apps using the Downward API. This can help alleviate the problem.

## Understanding pod QoS classes

When a node can't provide enough resources to all its pods, which pod(s) should be killed? This depends on each pod's Quality of Service class:

* `BestEffort` (lowest priority)
* `Burstable`
* `Guaranteed` (highest priority)

### Defining the QoS class for a pod

QoS class is assigned to pods not through a field in the manifest, but from a combination of resource requests and limits.

* `BestEffort`: No resource/limit requests.
* `Burstable`: Does not meet qualifications of `BestEffort` or `Guaranteed`.
* `Guaranteed`: Requests equal limits for all resources, and requests include both CPU and memory for each container.

A pod's QoS class is shown when running `kubectl describe pod` and in the pod's manifest in the `status.qosClass` field.

### Understanding which process gets killed when memory is low

First to get killed are `BestEffort` pods, then `Burstable` pods, then `Guaranteed` pods. Within the same QoS class, the system kills the pod with the highest OutOfMemory (OOM) score. OOM scores are based on the percentage of requested memory in use; a pod using 90% of its requested memory will be killed before a pod using 70%, assuming both are in the same QoS class.

## Setting default requests and limits for pods per namespace

If you don't set requests or limits for some pods, they could experience starvation. It is a good idea to set requests and limits on every container.

### Introducing the LimitRange resource

Instead of declaring requests and limits for every container, you can do it by creating a LimitRange resource. It allows you to set the minimum and maximum resource limits a container can request, and the default requests and limits for containers that don't specify them explicitly.

Another good use case for LimitRanges is to prevent users from creating pods that are bigger than any node in the cluster.

### Creating a LimitRange object

From `limits.yaml`:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: example
spec:
  limits:
  - type: Pod
    min:
      cpu: 50m
      memory: 5Mi
    max:
      cpu: 1
      memory: 1Gi
  - type: Container
    defaultRequest:
      cpu: 100m
      memory: 10Mi
    default:
      cpu: 200m
      memory: 100Mi
    min:
      cpu: 50m
      memory: 5Mi
    max:
      cpu: 1
      memory: 1Gi
    maxLimitRequestRatio:
      cpu: 4
      memory: 10
  - type: PersistentVolumeClaim
    min:
      storage: 1Gi
    max:
      storage: 100Gi
```

Here, limits for several types of objects are compiled into one LimitRange, but you can just as easily make separate LimitRange objects for each resource if you prefer.

### Enforcing the limits

The API server will reject resource creation requests that violate any of your established limits.

### Applying default resource requests and limits

Deploy `kubia-manual` from chapter 3 again: `kubectl apply -f ../ch03-pods/kubia-manual.yaml`. You will see that it uses your default requests and limits.

## Limiting the total resources available in a namespace

LimitRanges only apply to individual pods, but cluster admins also need a way to limit the total amount of resources available in a namespace (say, so that one team doesn't eat up all the resources in the cluster).

### Introducing the ResourceQuota object

A ResourceQuota limits the amount of resources pods, PersistentVolumeClaims, and the like consume; it can also limit the number of pods, claims, public IPs, and other objects.

With the following ResourceQuota in place, the current namespace (ResourceQuotas are namespaced) will be limited to 400m in total CPU requests and 600m in total CPU limits. From `quota-cpu-memory.yaml`:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: cpu-and-mem
spec:
  hard:
    requests.cpu: 400m
    requests.memory: 200Mi
    limits.cpu: 600m
    limits.memory: 500Mi
```

If you create a ResourceQuota object, the API server will reject any manifest that doesn't have requests and limits defined for the enumerated resources, so you will likely also want to have a LimitRange object alongside the ResourceQuota to define defaults.

### Specifying a quota for persistent storage

The following ResourceQuota limits total requested storage to 500Gi, and sets specific limits for each storage class. From `quota-storage.yaml`:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage
spec:
  hard:
    requests.storage: 500Gi
    ssd.storageclass.storage.k8s.io/requests.storage: 300Gi
    standard.storageclass.k8s.io/requests.storage: 1Ti
```

**Question: Is the 1Ti request a typo? Does it make sense for a single storage class to have a request limit greater than the overall storage request limit for the namespace?**

### Limiting the number of objects that can be created

From `quota-object-count.yaml`:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: objects
spec:
  hard:
    pods: 10
    replicationcontrollers: 5
    secrets: 10
    configmaps: 10
    persistentvolumeclaims: 4
    services: 5
    services.loadbalancers: 1
    services.nodeports: 2
    ssd.storageclass.storage.k8s.io/persistentvolumeclaims: 2
```

### Specifying quotas for specific pod states and/or QoS classes

Quotas can be limited by scope. Four scopes exist (at the time of publication of the book): `BestEffort`, `NotBestEffort`, `Terminating`, `NotTerminating`. ResourceQuotas can be assigned scopes, and resources must match all specified scopes for the quota to apply.

From `quota-scoped.yaml`:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: besteffort-notterminating-pods
spec:
  scopes:
  - BestEffort
  - NotTerminating
  hard:
    pods: 4
```

## Monitoring pod resource usage


