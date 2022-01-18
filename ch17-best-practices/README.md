# Chapter 17: Best practices for developing apps

Now we know how to run apps in Kubernetes, but we want to make sure they run smoothly, which is what we cover in this chapter.

## Bringing everything together

A typical application contains one or more Deployment and/or StatefulSet objects. Pods that provide services are exposed through Services objects. When they need to be reachable outside the cluster, they are configured to be `LoadBalancer` or `NodePort` Services, or are exposed through an Ingress.

Secrets are configured by cluster administrators and used by pods via ServiceAccounts for pulling images and configuring credentials.

ConfigMaps are used to initialize environment variables or create configuration files with volumes. Persistent storage is handled through PersistentVolumeClaims and PersistentVolumes, the latter usually created at runtime from StorageClass objects defined by cluster administrators.

Some applications need to use Jobs or CronJobs for one-off tasks. DaemonSets are typically not part of applications, but are often created by cluster administrators to run system services. HorizontalPodAutoscalers manage the Deployment scaling. Cluster administrators also create LimitRange and ResourceQuota objects to control cluster resource consumption.

Most resources are labeled and annotated to improve organization.

## Understanding the pod's lifecycle

Pods are like VMs dedicated to running only a single application, except that pods can be killed at any time.

### Applications must expect to be killed and relocated

Application developers need to make sure their apps allow being moved (i.e., killed and started elsewhere) relatively often. Apps need to be able to handle changes in IP and hostname (although stateful apps can be assigned the same hostname). Never base decisions on an app's IP, and if basing on the hostname, the app should be backed by a StatefulSet.

App data written to disk may not be available on restart, unless you mount persistent storage. App data written to disk may also be deleted or overwritten even when a pod is not evicted; a pod may be restarted at any time (e.g., if it fails a liveness probe or if there is an OOM error on the node), and the new container(s) will not have the same filesystem (although they will retain the data in Volumes--if the pod is not moved--and PersistentVolumes).

### Rescheduling of dead or partially dead pods

If a pod's container keeps crashing, the time between restarts will continue to increase exponentially until it reaches 5 minutes. At this point, the pod is essentially dead (or partially dead if some of its containers are working fine), but Kubernetes won't reschedule it automatically--Kubernetes sees the pod as running fine (regardless of the container crashing), so the replica count won't be adjusted or any action taken.

### Starting pods in a specific order

Kubernetes systems are usually defined in a single YAML or JSON file. There is no guarantee (especially if any pods get restarted) that any one pod will start before any other, so temporal dependencies cannot be captured in such a manner. However, you can include in a pod an extra "init" container. Init containers run before the main container(s) of a pod, and only allow the main container(s) to start once some precondition is met (e.g., successful connection to a service). Init containers are defined in a pod's `spec.initContainers` field.

With that said, it is even better practice to make sure apps work (or fail gracefully) when dependencies are absent or not ready. You never know when the service will go down.

### Adding lifecycle hooks

You can define two kinds of lifecycle hooks that execute commands or requests after a pod has started (post-start hooks) and just before it is stopped (pre-stop hooks). The options are similar to readiness probes:

* Execute a command.
* Perform an HTTP GET request.

A pod with a post-start hook will remain in the `Waiting` state until the hook has finished, and if it exits with an error, the pod will be killed. From `post-start-hook.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-poststart-hook
spec:
  containers:
  - image: luksa/kubia
    name: kubia
    lifecycle:
      postStart:
        exec:
          command:
          - sh
          - -c
          - "echo 'hook will fail with exit code 15'; sleep 5; exit 15"
```
