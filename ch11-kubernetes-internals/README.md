# Chapter 11: Understanding Kubernetes internals

## Understanding the architecture

A Kubernetes cluster consists of 2 main components: the Kubernetes Control Plane and the nodes. The Control Plane directs the cluster, and runs from the master node. The worker nodes run containers.

### The distributed nature of Kubernetes components

Kubernetes system components communicate only with the API server.

The components of the Control Plane may be distributed across many nodes and with several instances running in parallel to increase availability. These components can either be deployed on systems directly, or run as pods. The Kubelet, which runs on all nodes (including master) is the only component that always runs as a regular system component rather than a pod.

List all of the pods in the `kube-system` namespace (with custom columns):

```bash
kubectl get pods -o custom-columns=POD:metadata.name,NODE:spec.nodeName --sort-by spec.nodeName -n kube-system
```

### How Kubernetes uses etcd

etcd is the (only) place where Kubernetes stores cluster state and metadata, including object manifests. Only the API server talks to etcd, which is a fast, distributed, consistent key-value store. etcd v2 is a hierarchical store (like a file system), but etcd v3 is flat (keys can have slashes in the name, so you can still think of the store as hierarchical). etcd should always be deployed in an odd number of instances, because an even number will increase the probability of cluster failure (due to the consensus algorithm).

You can interact with etcd using `etcdctl`.

### What the API server does

The Kubernetes API server is the central component for querying and modifying the cluster state over a RESTful API. It stores that state in etcd. Clients like `kubectl` interface with the API server.

### Understanding how the API server notifies clients of resource changes

Clients can request to watch a resource for updates. The API server will then notify clients if any changes are made to the resource. For example, the Controller Manager watches pods to make sure the correct number of replicas exist at all times.

### Understanding the Scheduler

The Scheduler determines the node on which a pod should run. The Scheduler doesn't actually assign a node to create a pod; it updates the pod definition to indicate the node on which the pod should run. The API server sends an update to all watchers (the cluster nodes), and when the Kubelet on the target node sees that the pod has been scheduled to its node, it creates the pod.

You can define custom scheduling rules to select the best node for a given pod, a process which can be complex and application-dependent. You can also create multiple Schedulers for the cluster, and specify which Scheduler a pod should use.

### Introducing the controllers running in the Controller Manager

The Controller Manager is the component that makes sure the actual state of the system converges toward the desired end state. Controllers include:

* Replication Manager (for ReplicationControllers)
* ReplicaSet, DaemonSet, and Job controllers
* Deployment controller
* StatefulSet controller
* Node controller
* Service controller
* Namespace controller
* PersistentVolume controller
* Others

In general, controllers run a reconciliation loop that takes the actual state (specified in the resource's `status`) and brings it toward the desired state (specified in the resource's `spec`). Controllers don't create and run resources themselves; they only post manifests to the API server, which uses the Scheduler and the Kubelet to schedule and run the pod, respectively.

### What the Kubelet does

In contrast with all the controllers, which are part of the Kubernetes Control Plane and run on the master node(s), the Kubelet and Service Proxy both run on the worker nodes. The Kubelet is the component responsible for everything running on a worker node. It registers the node on which it is running by creating a Node resource in the API server; then it monitors the API server for pods that are scheduled to its node, and starts those pods' containers. The Kubelet also reports resource status to the API server.

### The role of the Kubernetes Service Proxy

Every worker node also runs the Kubernetes Service Proxy (`kube-proxy`), whose purpose is to make sure clients can connect to services. The proxy makes sure connections to the service IP and port end up at one of the pods backing the service.

### Introducing Kubernetes add-ons
