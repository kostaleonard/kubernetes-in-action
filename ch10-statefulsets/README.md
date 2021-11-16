# Chapter 10: StatefulSets: deploying replicated stateful applications

## Replicating stateful pods

Suppose each pod keeps its own state in your application (i.e., not a RESTful application, but a stateful application). You will likely need to allow each pod to keep its own volume. But, if the pod template uses a PersistentVolumeClaim and pods are managed by a ReplicationController or ReplicaSet, all of them will share the same claim and, therefore, volume.

### Running multiple replicas with separate storage for each

There are ways to create separate storage for replicated pods, but they are cumbersome and error-prone.

### Providing a stable identity for each pod

Pods are ephemeral and can be moved at will. Sometimes you need a stable identity (e.g., network address) for each pod. This is common in distributed stateful applications. An ugly solution is to have a service for every single pod. This will work because service IPs are stable, but pods won't be able to know through which service they are exposed.

## Understanding StatefulSets

StatefulSets were designed to solve the above problems. They allow the user to treat instances of the app as non-fungible entities, each having a stable name and state.

### Comparing StatefulSets with ReplicaSets

Stateful apps are like pets and stateless apps are like cattle (fungibility). StatefulSets manage stateful apps, while ReplicaSets and ReplicationControllers manage stateless apps.

**Specifically, pods created by StatefulSets each have their own set of volumes (i.e., persistent state) that differentiates them from their peers, and a predictable, stable network identity in the form of pod name and hostname.**

### Providing a stable network identity

Kubernetes gives a stable network identity to each of the pods by making you create a headless "governing" service. Through the headless service, each pod will have its own DNS entry. When a pod backed by a StatefulSet is rescheduled, it acquires the same name and hostname as the pod that disappeared.

### Providing stable dedicated storage to each stateful instance

As discussed before, each pod in a StatefulSet needs its own PersistentVolumeClaim (in a ReplicaSet, all pods share a single PersistentVolumeClaim and PersistentVolume). StatefulSets therefore have, in addition to a pod template, one or more volume claim templates to create new PersistentVolumeClaims along with each pod. When a StatefulSet is scaled down, only the pod is deleted, leaving any claim(s) to be reattached to a new pod if the StatefulSet is scaled up. The PersistentVolumeClaim(s) must be deleted manually, since that may not be what is intended.

### Understanding StatefulSet guarantees

Kubernetes guarantees not only stable, persistent storage and network identity for each pod, but also that **at most one** instance of each pod is running. This is to prevent collisions in network identity and storage.

## Using a StatefulSet