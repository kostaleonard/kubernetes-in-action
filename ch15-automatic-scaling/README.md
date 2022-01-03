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
```
