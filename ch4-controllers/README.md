# Chapter 4: Replication and other controllers

## Keeping pods healthy

Kubernetes has several ways of checking pod health. First, if a pod's container crashes, Kubernetes will restart it. But if a container is stuck in an infinite loop, is lagging, or is in deadlock, the container may not crash. Kubernetes also checks if a container is still alive through liveness probes. Liveness probes can either be HTTP GET probes, TCP Socket probes, or Exec probes based on how they check if a container is alive.

Both of the above operations (restarting a container when it crashes and performing liveness checks) are responsibilities of the Kubelet running on an individual node; the Kubernetes Control Plane is not involved. So, if an entire node goes down, these techniques won't keep apps alive.

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

Create the pod with `kubectl create -f kubia-liveness-probe.yaml`, and then watch its restart count using `kubectl get pod kubia-liveness`. To see why the container had to be restarted, use `kubectl describe pod kubia-liveness`.

To view the logs of a crashed container (not the current one), use `kubectl logs mypod --previous`.

## Configuring additional porperties of the liveness probe

Additional properties can be configured in the yaml. **Always remember to set an initial delay to liveness probes to allow your apps to start up.**

From `kubia-liveness-probe-initial-delay.yaml`:

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
      initialDelaySeconds: 15
```
