# Chapter 7: ConfigMaps and Secrets

There are 3 main ways to pass configuration to an app in Kubernetes.

1. Passing command-line arguments to containers
1. Setting custom environment variables in each container
1. Mounting configuration files into containers through a special kind of volume

## Passing command-line arguments to containers

In a Dockerfile, there are 2 related keywords for specifying the container executable: `ENTRYPOINT` and `CMD`. `ENTRYPOINT` is meant to specify the program to be executed; `CMD` specifies the default arguments. But, because it's possible to specify the entire command in both `ENTRYPOINT` and `CMD`, you will see both. Containers can be passed arguments as part of `docker run`.

### Overriding the command and arguments in Kubernetes

In the pod yaml, you can override `ENTRYPOINT` and `CMD` with `command` and `args`, respectively. Only rarely will you change the executable, but you may need to change the arguments.

From `fortune-pod-args.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune2s
spec:
  containers:
  - image: kostaleonard/fortune:args
    args: ["2"]
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
    emptyDir:
      medium: Memory
```

## Setting environment variables for a container

Kubernetes allows you to define environment variables for each container in a pod.

From `fortune-pod-env.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune2s
spec:
  containers:
  - image: luksa/fortune:env
    env:
    - name: INTERVAL
      value: "30"
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
    emptyDir:
      medium: Memory
```

You can refer in the yaml to any environment variables you've defined with `$(VAR)` syntax. You can even reference the environment variables in the `command` and `args` attributes.

The drawback of using environment variables to store pod configuration is that you may need separate environment variables, and therefore pod definitions, for development and production. To decouple the configuration from the pod, we will use a ConfigMap resource.

## Decoupling configuration with a ConfigMap


