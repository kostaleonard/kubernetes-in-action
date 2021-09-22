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

## Introducing ReplicationControllers

**Note: ReplicationControllers are deprecated and have been completely replaced with ReplicaSets.**

A ReplicationController is a Kubernetes resource that ensures its pods are always kept running, even if the nodes to which they are scheduled fail. A ReplicationController makes sure that an exact number of pods always matches its label selector.

### Creating a ReplicationController

From `kubia-rc.yaml`:

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    app: kubia
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: luksa/kubia
        ports:
        - containerPort: 8080
```

**Note: if using ReplicationControllers, don't specify the label selector; it will automatically be taken from the template if left unspecified.**

Create the controller with `kubectl create -f kubia-rc.yaml`. You can verify that the ReplicationController does its job by running `kubectl get pods -l app=kubia`, then deleting a pod with `kubectl delete pod <name>`.

### Getting information about a ReplicationController

View ReplicationController info with `kubectl get replicationcontroller` or `kubectl describe replicationcontroller kubia`. You can substitute `rc` for `replicationcontroller`.

### Moving pods in and out of the scope of a ReplicationController

Since ReplicationControllers count the number of pods in their scope based on labels, changing pod labels will add pods to or remove pods from the ReplicationController's scope.

Overwrite one of the pod's `app` label to take it out of the scope of the ReplicationController.

```bash
kubectl label pod <pod-name> app=foo --overwrite
```

List pods with `kubectl get pods -L app` and verify that there are now 4 pods, 3 of which have the label `app=kubia` and 1 of which has the label `app=foo`.

You might want to take a pod out of a ReplicationController's scope to debug that specific pod. When you have finished debugging, delete the pod.

```bash
kubectl delete pod -l app=foo
```

### Changing the pod template

You can edit the pod template for an existing ReplicationController by running `kubectl edit rc kubia` and editing the `template` section. New pods will be created with the updated template; old pods will not be affected. Editing the running ReplicationController's configuration won't update any yaml file used to create it, so this method won't pass changes into any version control system you may be using.

### Horizontally scaling pods

Scaling pods is as easy as changing the number of replicas required in the ReplicationController's configuration.

One way, using `kubectl scale`:

```bash
kubectl scale rc kubia --replicas=10
```

Another way, by editing the ReplicationController definition. Go to the `spec` section and set `replicas` to the desired number.

```bash
kubectl edit rc kubia
```

You can scale down in the same manner.

### Deleting a ReplicationController

You can delete a ReplicationController with `kubectl delete rc kubia`. By default, this will also delete that ReplicationController's pods. To keep the pods, run with the `--cascade=false` flag.

## Using ReplicaSets instead of ReplicationControllers

ReplicaSets are the replacement for the now-deprecated ReplicationController. Here we will make one manually, but they are usually created automatically when you make a Deployment resource. ReplicaSets are like ReplicationControllers, but you can use the full range of label selection patterns on them (instead of just `key=value`, you also have `key!=value`, boolean operators, selecting based on the presence/absence of a key, etc.).

### Defining a ReplicaSet

From `kubia-replicaset.yaml`. Note the change in API version.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kubia
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: luksa/kubia
```

Then run `kubectl create -f kubia-replicaset.yaml`.

### Using the ReplicaSet's more expressive label selectors

The `spec.selector.matchLabels` selector is the less expressive way to select by labels; you can also use `spec.selector.matchExpressions`. Each expression must contain a `key`, an `operator`, and possibly (depending on the operator) a list of `values`. Valid operators include `In`, `NotIn`, `Exists`, and `DoesNotExist`. You can use multiple expressions (they are AND-ed).

From `kubia-replicaset-matchexpressions.yaml`:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    matchExpressions:
      - key: app
        operator: In
        values:
          - kubia
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: luksa/kubia
```

## Running exactly one pod on each node with DaemonSets

Sometimes you want to run a pod exactly once on each node in the cluster. An example of this use case might be infrastructure-related pods that perform system-level operations like log collection or resource monitoring. DaemonSets fill this use case. They are like ReplicaSets, but they automatically create one replica of the pod on each node in the cluster. You can also deploy pods to only those nodes that match certain labels (usually hardware-oriented, like `disk=ssd`).

With DaemonSets, if a node goes down, a replacement pod is not created. But if a new node is added to the cluster, the DaemonSet will deploy a pod to it.

**Note: Even if a pod is made "unschedulable" to prevent pods from being deployed to it, a DaemonSet will still deploy its pod to those nodes because the DaemonSet bypasses the Kubernetes Scheduler entirely. This is usually desirable, since DaemonSets are meant to run system services that usually need to run even on unschedulable nodes.**

### Creating a DaemonSet YAML definition

From `ssd-monitor-daemonset.yaml`:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ssd-monitor
spec:
  selector:
    matchLabels:
      app: ssd-monitor
  template:
    metadata:
      labels:
        app: ssd-monitor
    spec:
      nodeSelector:
        disk: ssd
      containers:
      - name: main
        image: luksa/ssd-monitor
```

Label a node with `disk=ssd` and then use `kubectl create`.

## Running pods that perform a single completable task

The Job resource runs a pod whose container is not restarted when the process inside finishes successfully. If the node on which the pod is running fails before the completion of the process, the pod is rescheduled to a new node.

From `exporter.yaml`. Note that the `spec.template.spec.restartPolicy`, which defaults to `Always` for pods, has been set to `OnFailure`. The Job will be invalid if it uses the `Always` policy. Additionally, the label selector has been omitted and will be filled in manually based on the template's labels.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  template:
    metadata:
      labels:
        app: batch-job
    spec:
      restartPolicy: OnFailure
      containers:
      - name: main
        image: luksa/batch-job
```

Completed pods won't be deleted. This allows you to see the logs with `kubectl logs <pod-name>`.

### Running multiple pod instances in a Job

Jobs may be configured to create more than one pod instance and run them in parallel or sequentially. Use `spec.completions` and `spec.parallelism` for these.

#### Running Job pods sequentially

From `multi-completion-batch-job.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: multi-completion-batch-job
spec:
  completions: 5
  template:
    metadata:
      labels:
        app: batch-job
    spec:
      restartPolicy: OnFailure
      containers:
      - name: main
        image: luksa/batch-job
```

#### Running Job pods in parallel

From `multi-completion-parallel-batch-job.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: multi-completion-batch-job
spec:
  completions: 5
  parallelism: 2
  template:
    metadata:
      labels:
        app: batch-job
    spec:
      restartPolicy: OnFailure
      containers:
      - name: main
        image: luksa/batch-job
```
