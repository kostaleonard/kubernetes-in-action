# Chapter 12: Securing the Kubernetes API server

This chapter is about ServiceAccounts and configuring cluster permissions.

## Understanding authentication

When the API server receives a request, it goes through a list of authentication plugins to determine the identity of the requestor.

### Users and groups

An authentication plugin returns the username and group(s) of the authenticated user.

There are 2 types of clients: (human) users, and pods. Users should authenticate into the cluster using an external framework such as a Single Sign On (SSO), but pods authenticate using ServiceAccounts, which are Kubernetes cluster resources.

Groups are used to grant permissions to several users at once. Groups are represented by arbitrary strings.

### Introducing ServiceAccounts

ServiceAccounts are how pods authenticate with the API server. When a pod makes a request at the API server, it includes its ServiceAccount's token. ServiceAccounts are namespaced resources; every namespace includes a "default" ServiceAccount, which is what all pods so far have used. Pods can be assigned a specific ServiceAccount in the manifest.

### Creating ServiceAccounts

You can create ServiceAccounts using `kubectl create serviceaccount foo`; you can also create a ServiceAccount yaml.

### Assigning a ServiceAccount to a pod

Now that you have a ServiceAccount, you can assign it to a pod in the pod's `spec.serviceAccountName` field. This field can't be changed after pod creation. From `curl-custom-sa.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: curl-custom-sa
spec:
  serviceAccountName: foo
  containers:
  - name: main
    image: curlimages/curl
    command: ["sleep", "9999999"]
  - name: ambassador
    image: luksa/kubectl-proxy:1.6.2
```

You can now try to talk to the API server with `kubectl exec -it curl-custom-sa -c main -- curl localhost:8001/api/v1/pods`; you may get a 403 response depending on your account's permissions.

## Securing the cluster with role-based access control

Role-based access control (RBAC) is a plugin enabled by default on modern Kubernetes. It prevents unauthorized users from viewing or modifying the cluster state.

### Introducing the RBAC authorization plugin

The RBAC plugin determines whether a client connecting to the API server is allowed to perform the requested HTTP verb on the requested resource (e.g., GET and Pods, POST and Service, PUT and Secrets). RBAC can also apply specific permissions to specific resources.

### Introducing RBAC resources

There are 4 RBAC resources, which can be split into 2 groups:

1. Roles and ClusterRoles, which specify which verbs can be performed on which resources.
1. RoleBindings and ClusterRoleBindings, which bind the above roles to specific users, groups, and ServiceAccounts.

Roles and RoleBindings are the same thing as ClusterRoles and ClusterRoleBindings, except that the former are namespaced while the latter are not.

### Using Roles and RoleBindings

The following Role allows users to GET and LIST services in the foo namespace. From `service-reader.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: foo
  name: service-reader
rules:
- apiGroups: [""]
  verbs: ["get", "list"]
  resources: ["services"]
```

We'll also create a similar role in the bar namespace using `kubectl create role service-reader --verb=get --verb=list --resource=services -n bar`.

Now we need to create RoleBindings to bind the Roles to ServiceAccounts. We can do this with `kubectl create rolebinding test --role=service-reader --serviceaccount=foo:default -n foo`. The test pod (and any other pods with the default service account in namespace foo) can now get and list the services in namespace foo. You could also add the default service account in namespace bar to the rolebinding so that pods in namespace bar could get and list services in namespace foo.

### Using ClusterRoles and ClusterRoleBindings

A Role only allows access to resources in the same namespace as the Role. If you want to allow someone access to resources across different namespaces, you need to make a new Role in every namespace to access the resources in that namespace. Also, some resources aren't namespaced, and Roles can't grant access to those resources. ClusterRoles allow access to non-namespaced resources and can be used as a common role to be bound inside individual namespaces.

You can create a ClusterRole to read PersistentVolumes with `kubectl create clusterrole pv-reader --verb=get,list --resource=persistentvolumes`.

ClusterRoles can be bound to subjects with regular RoleBindings: `kubectl create rolebinding pv-test --clusterrole=pv-reader --serviceaccount=foo:default -n foo`.

**Note: A RoleBinding referencing a ClusterRole will only work for namespaced resources; to access non-namespaced resources, you need to use a ClusterRoleBinding.**

Now create a ClusterRoleBinding: `kubectl create clusterrolebinding pv-test --clusterrole=pv-reader --serviceaccount=foo:default`.

The automatically-created `system:discovery` ClusterRole and identically named ClusterRoleBinding allow access to non-resource URLs. If you inspect the ClusterRoleBinding (`kubectl get clusterrolebinding system:discovery -o yaml`), you'll see that it is bound to all authenticated users, meaning that authenticated users can GET non-resource URLs.

You can also create a ClusterRole to read/write namespaced resources. When bound with a ClusterRoleBinding, the user can access namespaced resources in all namespaces; when bound with a RoleBinding, the user can only access namespaced resources in the namespace of the RoleBinding.

Another automatically-created ClusterRole is `view`, which allows read access to many resources. Create a ClusterRoleBinding with `kubectl create clusterrolebinding view-test --clusterrole=view --serviceaccount=foo:default`. You can also create a RoleBinding with `kubectl create rolebinding view-test --clusterrole=view --serviceaccount=foo:default -n foo`.
