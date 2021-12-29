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


