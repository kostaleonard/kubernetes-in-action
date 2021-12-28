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


