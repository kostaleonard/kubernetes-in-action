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

You can add better support for your custom objects by implementing a custom API server. The main API server will forward requests on those objects to your custom API server. This differentiation would give you greater control over custom resource validation, and you wouldn't even have to implement CustomResourceDefinitions; custom objects would be implemented directly in the custom API server.

A custom API server can be deployed as a pod (backed by a Deployment) and exposed through a Service. Then, create an APIService resource similar to the following:

```yaml
apiVersion: apiregistration.k8s.io/v1beta1
kind: APIService
metadata:
  name: v1alpha1.extensions.example.com
spec:
  group: extensions.example.com
  version: v1alpha1
  priority: 150
  service:
    name: website-api
    namespace: default
```

## Extending Kubernetes with the Kubernetes Service Catalog

The Kubernetes Service Catalog is an extension that is designed to make using services (in a general sense, not specifically the Kubernetes Service resource; e.g., a SQL database for your app) very easy.

### Introducing the Service Catalog

The Service Catalog allows users to browse and provision instances of the services listed in the catalog without having to deal with "low-level" Kubernetes resources (e.g., Pods, Services, ConfigMaps). The Service Catalog introduces 4 generic API resources:

* ClusterServiceBroker: Describes a system (possibly external) that can provision services.
* ClusterServiceClass: Describes a type of service that can be provisioned.
* ServiceInstance: One instance of a service that has been provisioned.
* ServiceBinding: Represents a binding between a ser of clients (pods) and a ServiceInstance.

### Introducing the Service Catalog API server and Controller Manager

The Service Catalog is a distributed system composed of 3 components:

* Service Catalog API server, which handles the 4 aforementioned resources
* etcd as the storage
* Controller Manager, where all the controllers run

### Introducing Service Brokers and the OpenServiceBroker API

A cluster administrator can register ServiceBrokers in the Service Catalog. Every Broker must implement the OpenServiceBroker API, which is a REST API with operations to provision and manage service instances. The cluster administrator registers ServiceBrokers by posting a YAML manifest as shown below to the Service Catalog API server.

```yaml
apiVersion: servicecatalog.k8s.io/v1alpha1
kind: ClusterServiceBroker
metadata:
  name: database-broker
spec:
  url: http://database-osbapi.myorganization.org
```

### Provisioning and using a service

Once the ServiceBroker is registered, you can have the service provisioned for you by creating a ServiceInstance resource as shown below.

```yaml
apiVersion: servicecatalog.k8s.io/v1alpha1
kind: ServiceInstance
metadata:
  name: my-postgres-db
spec:
  clusterServiceClassName: postgres-database
  clusterServicePlanName: free
  parameters:
    init-db-args: --data-checksums
```

To use a provisioned ServiceInstance in your pods, create a ServiceBinding resource as shown below.

```yaml
apiVersion: servicecatalog.k8s.io/v1alpha1
kind: ServiceBinding
metadata:
  name: my-postgres-db-binding
spec:
  instanceRef:
    name: my-postgres-db
  secretName: postgres-secret
```

Right now, the way you bind to pods is to mount the Secret in them.

### Unbinding and deprovisioning

You can delete ServiceBindings and ServiceInstances in the same way you delete other Kubernetes resources, but the Broker will decide exactly how to handle its own unbinding/deprovisioning operation for each.

## Platforms built on top of Kubernetes

Many companies are now building PaaS systems on top of Kubernetes.

### Red Hat OpenShift Container Platform

OpenShift is a PaaS for rapid development, deployment, scaling, and maintenance of applications. OpenShift defines several additional Kubernetes resources.

### Deis Workflow and Helm

Deis provides PaaS systems called Workflow and Helm, also built on Kubernetes. Like OpenShift, Workflow provides a source to image mechanism straight from your git repository, but Workflow runs on an already-existing Kubernetes cluster rather than standalone. Workflow also provides log aggregation, metrics, and alerting.

Helm is a package manager for Kubernetes. It runs as a pod with a command line tool, `helm`.
