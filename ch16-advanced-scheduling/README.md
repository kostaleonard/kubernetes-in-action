# Chapter 16: Advanced scheduling

Aside from labeling nodes, there are additional strategies for determining where your pods are scheduled.

## Using taints and tolerations to repel pods from certain nodes

A taint is a marking applied to a node that prevents any pod from being scheduled to it unless the pod tolerates that taint. You can display a node's taints with `kubectl describe node <node>`.

### Introducing taints and tolerations

Taints have a key, value, and effect, presented as `key=value:effect`. Value may be null. Three possible effects exist:

1. NoSchedule: Pods won't be scheduled to the node if they don't tolerate the taint.
2. PreferNoSchedule: The scheduler will try to avoid scheduling pods to the node if they don't tolerate the taint, but will schedule it to the node if they can't schedule it elsewhere.
3. NoExecute: In addition to preventing scheduling, this effect evicts any pods that don't tolerate the taint and are already running.

### Adding custom taints to a node

Add a taint with `kubectl taint`, e.g., `kubectl taint node <node> node-type=production:NoSchedule`. Now, no one can deploy non-production pods to production nodes.

### Adding tolerations to pods

In the pod spec, add a `tolerations` field that indicates which taints are tolerated. You can use comparison operators such as `Equal` and `Exists` for the key/value.

### Understanding what taints and tolerations can be used for

Taints and tolerations are used to partition the cluster according to some schema, often production vs. development. It can also be used when some nodes provide special hardware that only some pods need to use (you would want to use NoSchedule or PreferNoSchedule to repel pods that don't need the special hardware resources). You can also configure taint-based eviction of pods when a node fails.

## Using node affinity to attract pods to certain nodes

Node affinity tells Kubernetes to schedule pods only to specific subsets of nodes. Node selectors are a less powerful version of this feature, and will be deprecated, so node affinity is the way to go moving forward. Node affinity selects nodes based on labels (as with node selectors).

### Specifying hard node affinity rules

In chapter 3, we created a pod that used a node selector to only be deployed on GPU nodes. Here is the same pod using node affinity. From `kubia-gpu-nodeaffinity.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-gpu
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: gpu
            operator: In
            values:
            - "true"
```

### Prioritizing nodes when scheduling a pod

One feature node affinity gives us that node selectors don't on their own is the ability to specify which nodes the scheduler should prefer when scheduling a pod. From `preferred-deployment.yaml`:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: pref
spec:
  template:
    ...
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80
            preference:
              matchExpressions:
              - key: availability-zone
                operator: In
                values:
                - zone1
          - weight: 20
            preference:
              matchExpressions:
              - key: share-type
                operator: In
                values:
                - dedicated
```

Here, availability zone will be prioritized over share type for scheduling decisions, although the scheduler will try to honor both. Note that the scheduler has other prioritization rules for selecting the node for a given pod; one of these promotes spreading pods across multiple nodes to maintain function if one of the nodes goes down. So, even if you prefer only one node, pods may be spread across several.

## Co-locating pods with pod affinity and anti-affinity
