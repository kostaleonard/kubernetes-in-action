# Chapter 9: Deployments: updating applications declaratively

How can we update Kubernetes apps with zero downtime? To handle that requirement, Kubernetes provides the Deployment resource.

## Updating applications running in pods

If you want to manually roll out an update to your application image, you have to replace the old pods with new ones. You have two ways to approach this:

1. Delete old pods and replace them with new ones. This approach will cause a short period of downtime.
1. Create a second set of pods, then delete the old ones. This approach will potentially cause issues because you will have 2 different versions of your app running at the same time (different data schemas, interfaces, etc.).

### Deleting old pods and replacing them with new ones

This approach simply involves updating the ReplicationController/ReplicaSet pod template to reference the new version of the image, then delete the pods. There will be a short period of downtime as the new pods are created.

### Spinning up new pods and then deleting the old ones

Alternatively, you could create a new ReplicationController/ReplicaSet with the new image, then switch the Service's label selector to point to the new pods. You can only do this if your application supports the multiple versions running simultaneously for a period of time. If you switch the Service's label selector to point to the new pods all at once, then you have performed a **blue-green deployment**. If your label selector includes both sets of pods, and you scale up the number of new pods as you scale down the number of old pods, then you have performed a **rolling update (or deployment)**.

## Performing an automatic rolling update with a ReplicationController

First we'll look at how to automate rolling deployments using `kubectl`. This is now an outdated method, but it is good to understand for context.

### Running the initial version of the app

The app is the same as the NodeJS app we used in the initial chapters to show the basics of Kubernetes, except this time the app will also print out its version in the HTTP response.

We will put the ReplicationController and Service definitions in the same YAML file. A YAML manifest can contain multiple objects delimited with a line containing 3 dashes. From `kubia-rc-and-service-v1.yaml`:

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: kubia-v1
spec:
  replicas: 3
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
      - image: luksa/kubia:v1
        name: nodejs
---
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  type: LoadBalancer
  selector:
    app: kubia
  ports:
  - port: 80
    targetPort: 8080
```

To perform a rolling update to the v2 container, run the following. This replaces the `kubia-v1` ReplicationController with one running v2 of your app.

```bash
kubectl rolling-update kubia-v1 kubia-v2 --image=luksa/kubia:v2
```

**Note: `kubectl rolling-update` has been deprecated, so this will not work on newer versions of Kubernetes. Instead, use Deployments.**

## Using Deployments for updating apps declaratively

When updating apps backed by ReplicationControllers or ReplicaSets, you need to introduce a second controller to scale up the number of copies of your new version while simultaneously scaling down the number of copies of your old version. To help coordinate this action, as well as enable rollbacks and other handy features, Kubernetes introduced the Deployment resource.

### Creating a Deployment

Deployments look very similar to ReplicationControllers in terms of the YAML. A Deployment will create a ReplicaSet, which will create the pods. From `kubia-deployment-v1.yaml`:

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
```

In addition to `kubectl get deployment` and `kubectl describe deployment`, you can also check the status of a Deployment with `kubectl rollout status deployment <name>`.

### Updating a Deployment

Updating the image used by a Deployment automatically triggers an update. The default update strategy is `RollingUpdate`, but there is also `Recreate` which will delete all of the current pods and start new ones.

You can update the image with:

```bash
kubectl set image deployment kubia nodejs=luksa/kubia:v2
```

**Note: You can also update the Deployment (and other resources) using a new YAML file with `kubectl apply -f <file>` or `kubectl replace -f <file>`. This could be a better way to ensure all your changes end up in version control.**

### Rolling back a deployment
