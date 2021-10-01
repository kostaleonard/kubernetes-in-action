# Chapter 6: Volumes

Containers in pods share some resources (e.g., CPU, RAM, network interfaces), but not filesystems; that is a container-level resource. Also, Kubernetes can restart unhealthy containers in a pod, and files do not persist between restarts.

## Introducing Volumes

Volumes are a Kubernetes resource that belong to pods. A volume is available to all containers in a pod, but must be mounted in each container that needs access to it. Volumes will persist between container restarts, and can be configured to persist between pod restarts.

Volumes come in various types, including, but not limited to:

* `emptyDir`
* `hostPath`
* `gitRepo`
* `nfs`
* `gcePersistentDisk`, `awsElasticBlockStore`, `azureDisk`
* `configMap`, `secret`, `downwardAPI`
* `persistentVolumeClaim`

## Using volumes to share data between containers


