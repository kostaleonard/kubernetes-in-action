# Chapter 4: Replication and other controllers

## Keeping pods healthy

Kubernetes checks if a container is still alive through liveness probes. Liveness probes can either be HTTP GET probes, TCP Socket probes, or Exec probes based on how they check if a container is alive.

## Creating an HTTP-based liveness probe

We'll use a modified version of our good Node.js app that returns a 500 error code on every request after the 5th. This image is available at `luksa/kubia-unhealthy`. We'll check its health using `kubia-liveness-probe.yaml`, reproduced below.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-liveness
spec:
  containers:
  - image: luksa/kubia-unhealthy
    name: kubia
    livenessProbe:
      httpGet:
        path: /
        port: 8080
```
