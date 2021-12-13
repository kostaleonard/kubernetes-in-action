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
