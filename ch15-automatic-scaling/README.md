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


