# Chapter 15: Automatic scaling of pods and cluster nodes

Pods can be scaled in two ways.

1. Horizontally: Increasing the number of application pods running, e.g., by increasing the replica count in the controller.
2. Vertically: Increasing the resources available to the pod, e.g., by increasing the requests and limits at pod creation.

We want to be able to scale automatically, rather than manually. This chapter covers pod and node autoscaling.

## Horizontal pod autoscaling

Horizontal pod autoscaling is the automatic scaling of the number of pod replicas managed by a controller. This process is enabled by the HorizontalPodAutoscaler (HPA) resource.

### Understanding the autoscaling process

There are 3 steps to autoscaling.

1. Obtain metrics of all managed pods.
2. Calculate the number of pods required to bring the metrics close to the target value.
3. Update the replica count.

HPAs get metrics from Heapster, so Heapster must be running in the cluster. Any resource that has a Scale sub-resource can be horizontally scaled automatically. These are (at the time of textbook publication) Deployments, ReplicaSets, ReplicationControllers, and StatefulSets.

### Scaling based on CPU utilization

CPU usage is perhaps the most important metric on which to scale pods. HPAs calculate CPU utilization percentage based on **actual** CPU usage vs. **requested** CPU. For this calculation to work, pods must have a CPU request specified. Note that the target CPU usage should never be set to 100% (80% is a good target) because the node may not be able to handle load spikes.

First we'll define the Deployment to scale. From `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubia
spec:
  replicas: 3
  selector:  
    matchLabels:
      app: kubia
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
      - image: luksa/kubia:v1
        name: nodejs
        resources:
          requests:
            cpu: 100m
```

With the Deployment created, you can also create an HPA. You can write a yaml, or you can make an HPA with `kubectl autoscale deployment kubia --cpu-percent=30 --min=1 --max=5`.

**Note: Always autoscale Deployments instead of underlying ReplicaSets so that the autoscaled replica count is preserved across application updates.**

Since the pods are receiving no requests, their CPU utilization should be close to 0%. The autoscaler will scale the number of pod replicas down to 1 (this could take a few minutes for metrics data to aggregate and for the autoscaler to be sure it should decrease the replica count).

Now we will expose our pods to a service and force an autoscale up. Start the service with `kubectl expose deployment kubia --port=80 --target-port=8080`. You can then watch the HPA (on MacOS) with `kubectl get hpa --watch`. Run a load generating pod with `kubectl run -it --rm --restart=Never loadgenerator --image=busybox -- sh -c "while true; do wget -O - -q http://kubia.default; done"`.

HPAs have a maximum upscaling rate of twice the number of pods. Additionally, scale-up operations occur only every 3 minutes, and scale-down operations every 5 minutes.

### Scaling based on memory consumption

Memory-based autoscaling doesn't work exactly like CPU-based autoscaling, since creating more replicas of a pod doesn't have any effect on the memory consumption of an app. Discussion of memory-based autoscaling is beyond the scope of this book.

### Scaling based on other and custom metrics

The metrics used for autoscaling can be any of the following types, including combinations.

* Resource: The resource metrics like those defined in a container's resource requests.
* Pods: Any metric related to the pod directly, e.g., queries per second (QPS) or message queue length.
* Object: Any metric not related to pods directly, e.g., query latency (an Ingress metric).

### Determining which metrics are appropriate for autoscaling

Not all metrics are good bases for autoscaling. If increasing the replica count won't result in a relatively linear decrease in the metric (as with memory consumption), then the autoscaler won't function properly.

### Scaling down to zero replicas

HPAs don't allow you to scale down to 0 replicas, but in future (this may already be implemented since the book is a few years old), Kubernetes will provide a feature that allows you to scale down to 0 replicas; when a new request comes in and there are no pods to back the service, the request is blocked while the pod is brought online, and the request is handled.

## Vertical pod autoscaling

Some applications can't be scaled horizontally, and instead need to be scaled vertically by adding CPU and/or memory. This feature may or may not be fully implemented in the current Kubernetes version (it was a WIP at the time of publication).

### Automatically configuring resource requests

There are experimental features (again, at publication time), e.g. the InitialResources Kubernetes resource, that provide automatic configuration of resource requests and limits. In the case of InitialResources, new pods are automatically assigned resource requests and limits based on the historical resource usage of previous pods.

### Modifying resource requests while a pod is running

At the time of publication, the finalized proposal for this feature was just published. It may be available in the current version of Kubernetes.

## Horizontal scaling of cluster nodes


