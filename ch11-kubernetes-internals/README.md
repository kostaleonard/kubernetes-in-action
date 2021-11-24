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


