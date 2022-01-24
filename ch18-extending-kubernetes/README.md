# Chapter 18: Extending Kubernetes

This chapter looks at how to define custom API objects and their controllers, as well as building PaaS solutions on top of Kubernetes.

## Defining custom API objects

As Kubernetes evolves, people add more and more custom objects to increase the level of abstraction. Some custom resources represent entire apps.

### Introducing CustomResourceDefinitions

To define a new resource type, post a CustomResourceDefinition object to the API server. Users will then be able to create instances of your custom resource. Each custom resource usually has its own custom controller.

Imagine we want to create a Website resource with a yaml in the following structure. From `imaginary-kubia-website.yaml`:

```yaml
kind: Website
metadata:
  name: kubia
spec:
  gitRepo: https://github.com/luksa/kubia-website-example.git
```

To make Kubernetes accept the custom resource, you need to post a CustomResourceDefinition. From `website-crd.yaml`:

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: websites.extensions.example.com
spec:
  scope: Namespaced
  group: extensions.example.com
  version: v1
  names:
    kind: Website
    singular: website
    plural: websites
```

Now we modify the Website resource definition to include the API version we specified in the CRD. From `kubia-website.yaml`:

```yaml
apiVersion: extensions.example.com/v1
kind: Website
metadata:
  name: kubia
spec:
  gitRepo: https://github.com/luksa/kubia-website-example.git
```

Now you can see your website with `kubectl get websites`.

The Website resource you created doesn't do anything yet; we need to add a controller. Not all custom resources need to do anything. Some are just meant to store data in a more concrete format than a ConfigMap.

### Automating custom resources with custom controllers

The Website controller (which runs as a pod managed by a Deployment) will watch for Website objects and create a Deployment and a Service for each. The source code for the controller is available [here](https://github.com/luksa/k8s-website-controller).

The Deployment backing the controller is as follows. From `website-controller.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: website-controller
spec:
  selector:
    matchLabels:
      app: website-controller
  replicas: 1
  template:
    metadata:
      name: website-controller
      labels:
        app: website-controller
    spec:
      serviceAccountName: website-controller
      containers:
      - name: main
        image: luksa/website-controller
      - name: proxy
        image: luksa/kubectl-proxy:1.6.2
```

Create the ServiceAccount with:

```bash
kubectl create serviceaccount website-controller
kubectl create clusterrolebinding website-controller --clusterrole=cluster-admin --serviceaccount=default:website-controller
```

Now you can create the controller and website objects.

### Validating custom objects

Kubernetes does not  (cannot) validate custom object YAML files posted to the API server without a schema indicating which fields can and cannot be specified. At the time of publication, this feature was in alpha.

### Providing a custom API server for your custom objects
