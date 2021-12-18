# Chapter 13: Securing cluster nodes and the network

In this chapter, we'll focus on configuring pods and their namespaces to improve cluster security.

## Using the host node's namespaces in a pod

Each pod's containers use Linux namespaces to provide isolation.

### Using the node's network namespace in a pod

Some pods (usually system pods) need to operate in the host's default namespaces to see and manipulate node-level resources. One example is a pod that uses the host's network interfaces instead of its own, as in `pod-with-host-network.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-host-network
spec:
  hostNetwork: true
  containers:
  - name: main
    image: alpine
    command: ["/bin/sleep", "9999999"]
```

### Binding to a host port without using the host's network namespace

You can use the `hostPort` property (not to be confused with a NodePort service) to forward connections from a port on the host's network interface to a port on the container's network interface.

**Note: If a pod is running with a host port, there can only be one copy of that pod per node.**

From `kubia-hostport.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-hostport
spec:
  containers:
  - image: luksa/kubia
    name: kubia
    ports:
    - containerPort: 8080
      hostPort: 9000
      protocol: TCP
```

### Using the node's PID and IPC namespaces

You can set the `hostPID` and `hostIPC` properties to true to have containers use the node's PID and IPC namespaces, respectively. From `pod-with-host-pid-and-ipc.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-host-pid-and-ipc
spec:
  hostPID: true
  hostIPC: true
  containers:
  - name: main
    image: alpine
    command: ["/bin/sleep", "9999999"]
```

## Configuring the container's security context

Besides nanespaces, other security-related features can be configured on the pod and its containers with `securityContext` properties.

### Running a container as a specific user
