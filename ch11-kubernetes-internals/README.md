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

Add-ons are extra components that are deployed as pods by submitting YAML manifests to the API server. These resources may be managed by Replication Controllers, ReplicaSets, DaemonSets, Deployments, etc. Add-ons include:

* DNS
* The Ingress controller
* The Kubernetes web dashboard

## How controllers cooperate

Here is what happens when a pod is created, in this example through a Deployment resource.

### The chain of events

See the book for an in-depth explanation of the process of creating pods through a Deployment.

### Observing cluster events

Both Control Plane components and Kubelets send events to the API server as they perform actions. These are represented as Event resources, and can be seen with `kubectl get events`. You can also watch events in real time with `kubectl get events --watch`.

## Understanding what a running pod is

In addition to the pod's explicitly defined container(s), one additional container is created on the node as part of the pod: a pod infrastructure container. The purpose of this container is to store namespace information (network interfaces, etc.) for all pods in a container.


## Inter-pod networking

Each pod gets its own unique IP address and can communicate with all other pods through a flat, NAT-less network.

### What the network must be like

All pods must agree on the IP addresses of all pods.

### Diving deeper into how networking works

Pods on the same node communicate with each other via a bridge. Each pod's infrastructure container creates a virtual ethernet interface, and the main container(s) connect to the virtual interface over their eth0 "physical" interface. The connection acts as a pipe to the network bridge, which connects all pods on a node.

Pods on different nodes communicate over a network, which can use overlay or underlay networks or layer 3 routing. Each bridge in the cluster must have a unique IP range. Each bridge is also connected to the network, physically, using the node's eth0 interface. Kubernetes wants to be agnositc to underlying network architecturem, so communication between nodes is handled using Software Defined Networking.

## How services are implemented

### Introducing the kube-proxy

Services are handled by the kube-proxy process running on each node. Remember that services use virtual addresses, IP and port pairs, which are layer 4 constructs (you can't ping a service).

### How kube-proxy uses iptables

When a new service is created, the kube-proxy on every node, which watches among other things new services, creates an iptables rule to forward packets destined for the service virtual address to the physical address of a randomly selected node running the service.

## Running highly available clusters

We usually want Kubernetes to keep apps running at all times even in the case of infrastructure failures. Kubernetes Control Plane components need to be running at all times as well in order to achieve this goal.

### Making your apps highly available

Always manage your app's pods using Deployment resources with more than one replica. Deployments (and their ReplicaSets) will ensure high availability. If your app is not horizontally scalable (only one copy can run at a time), then use a leader election mechanism; only one instance of the app will be performing work (the leader) while all other instances are on standby in case of failure.

**Note: There is no need to implement leader election in the app itself. You can use already-existing sidecar containers that do the hard work.**

### Making Kubernetes Control Plane components highly available

You also need to run multiple copies of the Control Plane components: etcd, the API server, the Controller Manager, and the Scheduler.

etcd is a distributed data store, so all you need to do is run it on multiple nodes and make each instance aware of the other instances in the configuration. Always run 3, 5, or 7 instances (even numbers don't actually help; more than 7 begins to impact performance in most cases).

The API server is also easy; it is stateless except for caching, so just run it on multiple nodes. Usually one API server is collocated with one etcd instance.

The Controller Manager and Scheduler modify the system state, so only one of each can be active at a time. Multiple instances can be run, but only one will be active (the leader). Leader election is the default behavior when running multiple instances of the Controller Manager and Scheduler, so little additional work needs to be done when adding new instances.
