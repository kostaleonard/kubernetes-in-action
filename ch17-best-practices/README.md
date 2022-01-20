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

A pre-stop hook, on the other hand, is executed immediately before a container is terminated. The following is an example of an HTTP GET pre-stop hook. The request is sent to [http://POD_IP:8080/shutdown]. This endpoint is configurable in the `httpGet` fields. Regardless of the exit code from the hook, the pod will be terminated. **You may not even notice if the hook fails, so if it is performing a critical operation, log its output or test it.**

```yaml
    lifecycle:
      preStop:
        httpGet:
          port: 8080
          path: shutdown
```


Lifecycle hooks target containers, not pods, so don't execute entire-pod startup/termination logic.

### Understanding pod shutdown

You can configure the termination grace period, the time the Kubelet allots to your app to execute the pre-stop hook and shutdown on receiving SIGTERM, by setting the `spec.terminationGracePeriodSeconds` field.

Note that there are no guarantees that a pod will be allowed to complete its whole shut-down procedure. Instead, it is better to have a separate pod (perhaps a CronJob) running and checking for orphaned resources (e.g., data in a PersistentVolume) and migrating them to where they need to be.

## Ensuring all client requests are handled properly

Your pods need to follow a few rules to ensure that no client connections are broken.

### Preventing broken client connections when a pod is starting up

All you need to do during start-up is define a readiness probe that returns success only when your app is ready to handle incoming requests. A good first approach is to use an HTTP GET probe to the base URL of the app.

### Preventing broken connections during pod shut-down

When your app receives a SIGTERM to shut down gracefully, you need to be careful how you shut down so as to disrupt user connections as little as possible. The book has an extended explanation, but what your app should do upon receiving SIGTERM is:

1. Wait for a few seconds, then stop accepting new connections. This gives kube-proxies and others watching changes to service Endpoints objects time to remove the pod's IP from `iptables` and similar mechanisms for directing traffic. You can't guarantee that all of these users will update in a timely manner, or that you even know all of the existing users, but giving a few seconds for users to stop sending requests to the pod will significantly improve the user experience.
1. Close all keep-alive connections not in the middle of a request.
1. Wait for all active requests to finish.
1. Shut down completely.

A basic version of this process can be accomplished with a pre-stop hook. In its most simple formulation, that pre-stop hook looks like this:

```yaml
    lifecycle:
      preStop:
        exec:
          command:
          - sh
          - -c
          - "sleep 5"
```

## Making your apps easy to run and manage in Kubernetes

### Making manageable container images

Keep your images small. It's okay to include a few tools (e.g., `ping`, `dig`, `curl`, `vim`) for debugging, but don't package up an entire filesystem.

### Properly tagging your images and using ImagePullPolicy wisely

It is very advisable to use version tags on images instead of the `latest` tag in pods. If you use `latest`, you can (and will) run multiple versions of your app at the same time, unintentionally. Plus, you can't easily rollback the deployment to the previous version. Always use versioned tags on images when you create a pod.

Beware of using the `Always` ImagePullPolicy. It can be costly, and you won't be able to create new pods when the registry cannot be contacted.

## Using multi-dimensional instead of single-dimensional labels
