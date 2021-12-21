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

To run a pod under a different user ID than the one in the container image, set the `securityContext.runAsUser` property. From `pod-as-user-guest.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-as-user-guest
spec:
  containers:
  - name: main
    image: alpine
    command: ["/bin/sleep", "9999999"]
    securityContext:
      runAsUser: 405
```

### Preventing a container from running as root

You can prevent a pod from running as root with `securityContext.runAsNonRoot`. From `pod-run-as-non-root.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-run-as-non-root
spec:
  containers:
  - name: main
    image: alpine
    command: ["/bin/sleep", "9999999"]
    securityContext:
      runAsNonRoot: true
```

If you deploy this pod, it will get scheduled, but will not run.

### Running pods in privileged mode

Some pods (e.g., kube-proxy) need to use protected system devices or other kernel features. From `pod-privileged.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-privileged
spec:
  containers:
  - name: main
    image: alpine
    command: ["/bin/sleep", "9999999"]
    securityContext:
      privileged: true
```

### Adding individual kernel capabilities to a container

To have more fine-grained control over privileged functions, you can grant a pod specific kernel capabilities. For example, suppose you want a container to be able to change the system time. Add `CAP_SYS_TIME` to the container's capabilities list. From `pod-add-settime-capability.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-add-settime-capability
spec:
  containers:
  - name: main
    image: alpine
    command: ["/bin/sleep", "999999"]
    securityContext:
      capabilities:
        add:
        - SYS_TIME
```

### Dropping capabilities from a container

You can also drop capabilities that may be available to the container. From `pod-drop-chown-capability.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-drop-chown-capability
spec:
  containers:
  - name: main
    image: alpine
    command: ["/bin/sleep", "9999999"]
    securityContext:
      capabilities:
        drop:
        - CHOWN
```
