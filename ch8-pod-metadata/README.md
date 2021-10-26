# Chapter 8: Accessing pod metadata and other resources from applications

Apps can, if necessary, talk directly to the Kubernetes API server to get information on the resources deployed in the cluster. In general, apps should be Kubernetes-agnositc, but this information is available if necessary.

## Passing metadata through the Downward API

The Kubernetes Downward API allows you to expose metadata (e.g., pod name, node hostname, pod IP) to pods as environment variables or files (in a `downwardAPI` volume).

### Exposing metadata through environment variables

From `downward-api-env.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward
spec:
  containers:
  - name: main
    image: busybox
    command: ["sleep", "9999999"]
    resources:
      requests:
        cpu: 15m
        memory: 100Ki
      limits:
        cpu: 100m
        memory: 4Mi
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    - name: SERVICE_ACCOUNT
      valueFrom:
        fieldRef:
          fieldPath: spec.serviceAccountName
    - name: CONTAINER_CPU_REQUEST_MILLICORES
      valueFrom:
        resourceFieldRef:
          resource: requests.cpu
          divisor: 1m
    - name: CONTAINER_MEMORY_LIMIT_KIBIBYTES
      valueFrom:
        resourceFieldRef:
          resource: limits.memory
          divisor: 1Ki
```

### Passing metadata through files in a `downwardAPI` volume

You can also expose these metadata through files in a `downwardAPI` volume. If you want to expose a pod's labels or annotations, you have to use a `downwardAPI` volume (this is partly because labels and annotations can have `=` characters as well as newlines and other stop characters, and partly because Kubernetes keeps the `downwardAPI` label/annotation files up to date if those parameters change at any point).

From `downward-api-volume.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward
  labels:
    foo: bar
  annotations:
    key1: value1
    key2: |
      multi
      line
      value
spec:
  containers:
  - name: main
    image: busybox
    command: ["sleep", "9999999"]
    resources:
      requests:
        cpu: 15m
        memory: 100Ki
      limits:
        cpu: 100m
        memory: 4Mi
    volumeMounts:
    - name: downward
      mountPath: /etc/downward
  volumes:
  - name: downward
    downwardAPI:
      items:
      - path: "podName"
        fieldRef:
          fieldPath: metadata.name
      - path: "podNamespace"
        fieldRef:
          fieldPath: metadata.namespace
      - path: "labels"
        fieldRef:
          fieldPath: metadata.labels
      - path: "annotations"
        fieldRef:
          fieldPath: metadata.annotations
      - path: "containerCpuRequestMillicores"
        resourceFieldRef:
          containerName: main
          resource: requests.cpu
          divisor: 1m
      - path: "containerMemoryLimitBytes"
        resourceFieldRef:
          containerName: main
          resource: limits.memory
          divisor: 1
```

## Talking to the Kubernetes API server

The Downward API can only pass to pods their own metadata. Sometimes you need to access the metadata of other pods or resources, and for that you can talk to the Kubernetes API server.

### Exploring the Kubernetes REST API

First, get the Kubernetes API server IP address with `kubectl cluster-info`. You can see the IP address and port (likely 8443), but you won't be able to connect with `curl`, even if you use `-k` for insecure mode.

Instead, you can run `kubectl proxy` to start a proxy connection to the API server that accepts connections on localhost, vanilla HTTP. The proxy will handle all authentication with the API server.

With the proxy running, sending requests to the proxy will return Kubernetes metadata. For example, `curl localhost:8001/apis/batch` returns a description of the `batch` API group.

### Talking to the API server from within a pod

Pods won't usually have access to `kubectl proxy`, so if we want to talk to the API server from within a pod, we have to do it differently.

For this example, we will use a container that does nothing (`sleep`), but that has the `curl` command. From `curl.yaml`:

```yaml
apiVersion: v1 
kind: Pod
metadata:
  name: curl
spec:
  containers:
  - name: main
    image: tutum/curl
    command: ["sleep", "9999999"]
```

Create the pod and enter into a shell with `kubectl exec -it curl -- bash`. We are trying to interface with the Kubernetes API server, which is exposed at the `kubernetes` service. The IP/port of this service are populated in environment variables in all pods, but we can also just resolve the hostname with DNS by running `curl https://kubernetes`.