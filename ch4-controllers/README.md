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
