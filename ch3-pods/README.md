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
