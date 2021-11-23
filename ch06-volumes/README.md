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

From `fortune-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune
spec:
  containers:
  - image: kostaleonard/fortune
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts: 
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
```

You can also set the Volume's `medium` property to `Memory` instead of leaving it at the default value, which will use disk storage.

### Using a Git repository as the starting point for a volume

You can also populate a Volume with a Git repository. The Volume directory will be filled before the Pod is started. The Volume will not be kept up to date with the latest commits once the Pods are running, but Pods can be restarted to get the latest changes.

From `gitrepo-volume-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gitrepo-volume-pod
spec:
  containers:
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    gitRepo:
      repository: https://github.com/kostaleonard/kubia-website-example.git
      revision: master
      directory: .
```

**Note: If you want to keep the Volume in sync with the git repository, you can use a sidecar container (a container that augments the operation of the main container of the pod) to get the latest changes regularly. See Chapter 18 for a worked-through example.**

**Note: You can't directly clone a private Git repository with a `gitRepo` Volume, but you can with a sidecar container.**

## Accessing files on the worker node's filesystem

Sometimes a Pod will need to interface directly with the host node's filesystem (think DaemonSets and pods that work with system-level resources like hardware). This is where we want a `hostPath` Volume.

### Introducing the `hostPath` Volume

A `hostPath` Volume points to a specific file or directory on the node's filesystem. `hostPath` Volumes offer persistent storage, but only on the node to which the Pod is scheduled (in general `hostPath` Volumes should not be used to persist data unless it has to do with the node itself, e.g., system-level resources).

### Examining system pods that use `hostPath` Volumes

Try running `kubectl describe pod <pod-name> --namespace kube-system` on some of the Kubernetes system pods. Many of them have `hostPath` Volumes pointing to log directories, certificates, or Kubernetes config files.

## Using persistent storage

Here we will create a pod hosting a MongoDB NoSQL database. Your persistent storage options will vary based on your underlying hardware. For minikube, you can simulate persistent storage with a `hostPath` volume.

### Using a GCE Persistent Disk in a pod volume

From `mongodb-pod-gcepd.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb
spec:
  volumes:
  - name: mongodb-data
    gcePersistentDisk:
      pdName: mongodb
      fsType: ext4
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
```

Create the pod and insert an object into the database.

```bash
kubectl exec -it mongodb mongo
> use mystore
> db.foo.insert({name:'foo'})
```

You can now delete the pod, create a new pod, and verify that the item is present.

```bash
kubectl delete pod mongodb
kubectl create -f mongodb-pod-gcepd.yaml
kubectl exec -it mongodb mongo
> use mystore
> db.foo.find()
```

### Using other types of persistent storage

If your Kubernetes cluster is running in AWS EC2, you can use an `awsElasticBlockStore` volume as persistent storage. If the cluster is running in Microsoft Azure, you can use the `azureFile` or `azureDisk` volumes. If you are running your own set of servers, you can use an `nfs` volume (and many other options).

The problem with the way we've approached these persistent volumes so far is that they are tightly coupled with the underlying infrastructure implementation. What if we want to use a different set of hardware? Our pods won't work correctly.

## Decoupling pods from the underlying storage technology

The goal of Kubernetes is to hide the underlying infrastructure from developers and applications. Developers can request a certain amount of persistent storage from the cluster using PersistentVolumes and PersistentVolumeClaims, and Kubernetes will provision the storage.

### Introducing PersistentVolumes and PersistentVolumeClaims

First, the cluster administrator sets up an underlying storage system (cloud or on-premises) and registers it in Kubernetes by creating a PersistentVolume resource. A cluster user then creates a PersistentVolumeClaim specifying the minimum size and access mode they require, and Kubernetes binds the PersistentVolumeClaim to a PersistentVolume that meets requirements. Pods can then use the PersistentVolumeClaim as a volume.

### Creating a PersistentVolume

We will create a PersistentVolume that uses `hostPath` so that it can be used with minikube. You can also use `gcePersistentDisk`, etc. as the storage mechanism. If the storage mechanism changes at any point, the PersistentVolumes are the only resources that need to change.

From `mongodb-pv-hostpath.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/kubia
    type: DirectoryOrCreate
```

**Note: PersistentVolumes don't belong to any namespace--like nodes, they are a cluster-level resource.**

### Claiming a PersistentVolume by creating a PersistentVolumeClaim

PersistentVolumeClaims allow pods to have persistent data even when nodes fail.

From `mongodb-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim 
metadata:
  name: mongodb-pvc
spec:
  resources:
    requests:
      storage: 1Gi
  accessModes:
  - ReadWriteOnce
  storageClassName: ""
```

Once the PersistentVolumeClaim is created, it is bound to the PersistentVolume.

**Note: The access modes pertain to the number of worker nodes that can use the volume at the same time, not the number of pods!**

### Using a PersistentVolumeClaim in a pod

You can now tie the storage device to a pod by referencing the PersistentVolumeClaim in the pod's manifest.

From `mongodb-pod-pvc.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb
spec:
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
  volumes:
  - name: mongodb-data
    persistentVolumeClaim:
      claimName: mongodb-pvc
```

You can now create the pod and check that the item we created earlier is in the database (if on minikube, create the item, delete the pod, recreate the pod, and check for the item as described above).

**Note: When using a `hostPath` Volume on minikube, the directory is created *in the minikube container*, not on the host OS; to see the contents that are persisted, run `minikube ssh` first.**

### Recycling PersistentVolumes

PersistentVolumes, by default, will not bind to a new PersistentVolumeClaim after the original PersistentVolumeClaim is deleted (although a new pod can use the old PersistentVolumeClaim). This is done to preserve data that the cluster administrator may want to clean up/keep. It is also done for privacy; a pod from a different namespace (likely belonging to a different cluster tenant) could read the data stored in the PersistentVolume if it became available again.

This behavior is specified in the PersistentVolume's `persistentVolumeReclaimPolicy`, which by default is set to `Retain`. To manually recycle the volume, you have to delete and recreate the PersistentVolume resource. While you do this, you can do whatever you need to with the underlying storage (clear the files, etc.).

There are two other reclaim policies.

* `Recycle`: Clears the volume's contents and makes the volume available to be claimed again.
* `Delete`: Deletes the underlying storage.

## Dynamic provisioning of PersistentVolumes

Instead of having a cluster administrator manually provision a PersistentVolume for every storage resource available to the cluster, the admin can deploy a PersistentVolume provisioner and define one or more StorageClass objects to let users choose which kind of PersistentVolume they want.

**Note: StorageClass resources, like PersistentVolumes, are not namespaced.**

StorageClasses allow the system to create a new PersistentVolume every time one is requested through a PersistentVolumeClaim.

### Defining the available storage types through StorageClass resources

From `storageclass-fast-hostpath.yaml`. The provisioner is the Kubernetes plugin that interfaces with the cloud/on-premises infrastructure to create PersistentVolumes.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: k8s.io/minikube-hostpath
parameters:
  type: pd-ssd
```

### Requesting the storage class in a PersistentVolumeClaim

From `mongodb-pvc-dp.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
spec:
  storageClassName: fast
  resources:
    requests:
      storage: 100Mi
  accessModes:
    - ReadWriteOnce
```

### Dynamic provisioning without specifying a storage class

You can also create a PersistentVolumeClaim without specifying the `storageClassName` attribute, in which case the default storage class for the cluster will be used (to find this, run `kubectl get sc`; on minikube, it is called `standard`).

You can also set `storageClassName` to "" if you want the PersistentVolumeClaim to use a pre-provisioned PersistentVolume.

From `mongodb-pvc-dp-nostorageclass.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc2
spec:
  resources:
    requests:
      storage: 100Mi
  accessModes:
    - ReadWriteOnce
```
