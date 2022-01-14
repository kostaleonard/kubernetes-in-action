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

Sometimes you want to specify the affinity between pods, rather than between pods and nodes. Suppose, for example, you have frontend and backend pods that you want to keep close together, but don't want to have to specify the exact node or datacenter to which to schedule them.

### Using inter-pod affinity to deploy pods on the same node

First, we'll deploy a backend pod with `kubectl run backend -l app=backend --image busybox -- sleep 9999999`. Now, we'll create the frontend pod with affinity.

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 5
  template:
    ...
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - topologyKey: kubernetes.io/hostname
            labelSelector:
              matchLabels:
                app: backend
      ...
```

This Deployment creates pods that must be scheduled to the same node as the ones backend pods are scheduled to.

If you delete and recreate (or a controller recreates) the backend pod, it will get scheduled back to the node with all the frontend pods. This is because the Scheduler takes other pods' pod affinity rules into account and sees that the backend pod should be with the frontend pods.

### Deploying pods in the same rack, availability zone, or geographic region

Instead of deploying pods with affinity to the same node, you can also use `topologyKey` to deploy to the same general area without forcing pods to be on the same node. For instance, you can set `topologyKey` to `failure-domain.beta.kubernetes.io/zone` to co-locate pods in the same availability zone, or `failure-domain.beta.kubernetes.io/region` for the same region.

You can also use your own `topologyKey` value by adding a label to your nodes (e.g., for the rack that they occupy).

### Expressing pod affinity preferences instead of hard requirements

Both node affinity and pod affinity can be expressed as preferences instead of hard requirements. Simply substitute `requiredDuringSchedulingIgnoredDuringExecution` with `preferredDuringSchedulingIgnoredDuringExecution`, and provide a weight factor.

### Scheduling pods away from each other with pod anti-affinity

To keep pods away from each other, substitude `podAffinity` with `podAntiAffinity`. You might want to do this if two sets of pods would interfere with each other's performance if they run on the same node, or if you want to spread pods across regions for resiliency.
