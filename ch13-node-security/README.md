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

### Preventing processes from writing to the container's filesystem

Preventing write access to a container's filesystem can prevent certain classes of attacks. In the following example, the container's filesystem is marked as read only, but the volume mount is still writable. From `pod-with-readonly-filesystem.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-readonly-filesystem
spec:
  containers:
  - name: main
    image: alpine
    command: ["/bin/sleep", "9999999"]
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: my-volume
      mountPath: /volume
      readOnly: false
  volumes:
  - name: my-volume
    emptyDir:
```

Under this design pattern, you can make the root filesystem read only, but then mount volumes for logs, caches, etc. to which the container should have write access.

**Note: The security context property can be set at the pod or container level (container level overrides pod level).**

### Sharing volumes when containers run as different users

Containers in the same pod can share volumes, but only when the permissions are set correctly. Before, this wasn't an issue because we were running apps as root. If the containers are running as different users, they will need to run as the same group. You can set groups with `fsGroup` and `supplementalGroups`. From `pod-with-shared-volume-fsgroup.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-shared-volume-fsgroup
spec:
  securityContext:
    fsGroup: 555
    supplementalGroups: [666, 777]
  containers:
  - name: first
    image: alpine
    command: ["/bin/sleep", "9999999"]
    securityContext:
      runAsUser: 1111
    volumeMounts:
    - name: shared-volume
      mountPath: /volume
      readOnly: false
  - name: second
    image: alpine
    command: ["/bin/sleep", "9999999"]
    securityContext:
      runAsUser: 2222
    volumeMounts:
    - name: shared-volume
      mountPath: /volume
      readOnly: false
  volumes:
  - name: shared-volume
    emptyDir:
```

Volume mounts will be owned by `fsGroup`. Files that the user creates in this volume will be owned by `fsGroup`; files the user creates in any other location will be owned by the effective group ID (root by default).

## Restricting the use of security-related features in a pod

What is to stop a cluster user from deploying a pod that runs in privileged mode? The cluster administrator can create a PodSecurityPolicy resource to prevent abuse of pod security features.

### Introducing the PodSecurityPolicy resource

PodSecurityPolicy is a cluster-level resource that defines the security-related features users can or can't use in their pods. The following PodSecurityPolicy prevents pods from using the host's IPC, PID, and network namespaces; prevents running privileged containers; and prevents the use of most host ports. From `pod-security-policy.yaml`:

```yaml
apiVersion: extensions/v1beta1
kind: PodSecurityPolicy
metadata:
  name: default
spec:
  hostIPC: false
  hostPIC: false
  hostNetwork: false
  hostPorts:
  - min: 10000
    max: 11000
  - min: 13000
    max: 14000
  privileged: false
  readOnlyRootFilesystem: true
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  seLinux:
    rule: RunAsAny
  volumes:
  - '*'
```

### Understanding runAsUser, fsGroup, and supplementalGroups policies

You can impose limits on the users and groups a pod runs with using the rules in thee `runAsUser`, `fsGroup`, and `supplementalGroups` fields. From `psp-must-run-as.yaml`:

```yaml
runAsUser:
  rule: MustRunAs
  ranges:
  - min: 2
    max: 2
fsGroup:
  rule: MustRunAs
  ranges:
  - min: 2
    max: 10
  - min: 20
    max: 30
supplementalGroups:
  rule: MustRunAs
  ranges:
  - min: 2
    max: 10
  - min: 20
    max: 30
```

**Note: Changing the policy has no effect on existing pods, because PodSecurityPolicies are enforced only when pods are created or updated.**

**Note: The user ID baked into an image can be overridden using MustRunAs.**

### Configuring allowed, default, and disallowed capabilities

You can also define more fine-grained control over the kernel capabilities available to each container. From `psp-capabilities.yaml`:

```yaml
apiVersion: extensions/v1beta1
kind: PodSecurityPolicy
spec:
  allowedCapabilities:
  - SYS_TIME
  defaultAddCapabilities:
  - CHOWN
  requiredDropCapabilities:
  - SYS_ADMIN
  - SYS_MODULE
```

### Constraining the types of volumes pods can use

PodSecurityPolicies can define which volume types users can use in their pods. From `psp-volumes.yaml`:

```yaml
kind: PodSecurityPolicy
spec:
  volumes:
  - emptyDir
  - configMap
  - secret
  - downwardAPI
  - persistentVolumeClaim
```

### Assigning different PodSecurityPolicies to different users and groups

PodSecurityPolicies can be assigned to users and groups using the RBAC method described in the previous chapter. First we will make a PodSecurityPolicy that allows privileged users to create pods with privileged containers. From `psp-privileged.yaml`:

```yaml
apiVersion: extensions/v1beta1
kind: PodSecurityPolicy
metadata:
  name: privileged
spec:
  privileged: true
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  seLinux:
    rule: RunAsAny
  volumes:
  - '*'
```

Now you can create two ClusterRoles for the security policies.

```bash
kubectl create clusterrole psp-default --verb=use --resource=podsecuritypolicies --resource-name=default
kubectl create clusterrole psp-privileged --verb=use --resource=podsecuritypolicies --resource-name=privileged
```

Now create ClusterRoleBindings. Bob is the user we would like to give the `psp-privileged` ClusterRole.

```bash
kubectl create clusterrolebinding psp-all-users --clusterrole=psp-default --group=system:authenticated
kubectl create clusterrolebinding psp-bob --clusterrole=psp-privileged --user=bob
```

## Isolating the pod network

Now we'll explore security configuration for pod networking to limit which pods can talk to which other pods. We'll do this with NetworkPolicy resources.

### Enabling network isolation in a namespace

The following NetworkPolicy prevents all clients from connecting to pods in your namespace. The empty `podSelector` field doesn't match any pods, so no pods will be able to connect. From `network-policy-default-deny.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector:
```

### Allowing only some pods in the namespace to connect to a server pod

Suppose you only want your webserver to be able to connect to your database. With the following NetworkPolicy, only webserver pods will be able to connect to database pods, and only on port 5432. From `network-policy-postgres.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-netpolicy
spec:
  podSelector:
    matchLabels:
      app: database
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: webserver
    ports:
    - port: 5432
```

Client pods connecting through a Service will still be restricted as expected.

### Isolating the network between Kubernetes namespaces

You can also allow/deny access from specific namespaces using namespace selectors. In the following NetworkPolicy, only namespaces with label `tenant=manning` are allowed to access the shopping cart microservice. From `network-policy-cart.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: shoppingcart-netpolicy
spec:
  podSelector:
    matchLabels:
      app: shopping-cart
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tenant: manning
    ports:
    - port: 80
```

### Isolating using CIDR notation

Instead of using label selectors, you can limit access with IP ranges in CIDR notation. From `network-policy-cidr.yaml`:

```yaml
ingress:
- from:
  - ipBlock:
      cidr: 192.168.1.0/24
```

### Limiting the outbound traffic of a set of pods

In the previous examples, we limited traffic using ingress rules. You can also limit traffic using egress rules. In the following NetworkPolicy, the webserver pods are only allowed to connect to the database (and nothing else). From `network-policy-egress.yaml`:

```yaml
spec:
  podSelector:
    matchLabels:
      app: webserver
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
```
