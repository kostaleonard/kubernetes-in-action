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

Configuration is meant to be divorced from source code because it changes frequently and between environments. Similarly, we want to keep configuration out of pod definitions, which are basically extensions of source code for apps and are not meant to change between environments.

### Introducing ConfigMaps

Kubernetes includes the ConfigMap resource that allows users to define configuration in key/value pairs. ConfigMap entries can then be passed to containers as command-line arguments or through `configMap` volumes.

Pods are now decoupled from configuration and the same pod definition can be used in development, production, and other namespaces; each namespace will have a different ConfigMap to match the environment.

### Creating a ConfigMap

First, we are going to use `kubectl create configmap` to create a ConfigMap.

```bash
kubectl create configmap fortune-config --from-literal=sleep-interval=25
```

You can inspect the yaml with `kubectl get configmap fortune-config -o yaml`; we could also have written this ConfigMap in yaml form. Entire configuration files can also be added as values within a ConfigMap using `--from-file`.

### Passing a ConfigMap entry to a container as an environment variable

To get a value from a ConfigMap, your pod can use an environment variable defined in the yaml. From `fortune-pod-env-configmap.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-env-from-configmap
spec:
  containers:
  - image: luksa/fortune:env
    env:
    - name: INTERVAL
      valueFrom:
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval
```

### Passing all entries of a ConfigMap as environment variables at once

You can expose all of the keys in a ConfigMap as environment variables at once by setting `envFrom` in the yaml definition.

**Note: Only valid environment variable names will be gathered. Names containing invalid characters, e.g. "-", will be skipped.**

### Passing a ConfigMap entry as a command-line argument

If you want to pass a configuration parameter as a command-line argument to a container's main process, then you can set an environment variable and reference it in `args` (you can't reference the ConfigMap key directly in `args`). From `fortune-pod-args-configmap.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata: 
  name: fortune-args-from-configmap
spec:
  containers:
  - image: luksa/fortune:args
    env:
    - name: INTERVAL
      valueFrom:
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval
    args: ["$(INTERVAL)"]
```

### Using a configMap volume to expose ConfigMap entries as files

When a ConfigMap contains entire files for app configuration, it can be more convenient to expose those to containers as actual files. A configMap volume allows you to expose each entry of a ConfigMap as a file.

From `my-nginx-config.conf`:

```conf
server {
  listen        80;
  server_name   www.kubia.example.com;

  gzip on;
  gzip_types text/plain application/xml;

  location / {
    root        /usr/share/nginx/html
    index       index.html index.htm;
  }
}
```

We place this file and the `sleep-interval` file in the `configmap-files` directory. Then, we will create the ConfigMap directly with `kubectl create configmap fortune-config --from-file=configmap-files`.

Now we can create a pod with a configMap Volume that references the ConfigMap we have created. From `fortune-pod-configmap-volume.yaml`:

TODO working offline, took best guess at the config without debugging. See luksa repo for working yaml.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-configmap-volume
spec:
  containers:
  - image: kostaleonard/fortune:args
    env:
    - name: INTERVAL
      valueFrom:
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval
    args: ["$(INTERVAL)"]
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
    - name: config
      mountPath: /etc/nginx/conf.d
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir:
      medium: Memory
  - name: config
    configMap:
      name: fortune-config
```

If you use this yaml, the `web-server` container will have both `my-nginx-config.conf` and `sleep-interval` in the `/etc/nginx/conf.d` directory. Ideally, `sleep-interval` would be omitted from this directory. To only include certain items from a ConfigMap in the configMap volume, you can specify the `items` property as follows:

```yaml
...
  volumes:
  - name: html
    emptyDir:
      medium: Memory
  - name: config
    configMap:
      name: fortune-config
      items:
      - key: my-nginx-config.conf
        path: gzip.conf
```

**Note: By mounting the volume at `/etc/nginx/conf.d` (a directory), you shadowed all of the files that might have been in that directory. In this case, there were none, but if there were, the consequences could have been disastrous. To avoid this, use the `subPath` property in `volumeMounts`.**

You can mount a specific ConfigMap entry into a specific file with `subPath` as follows. Here, `/etc/someconfig.conf` is a file, not a directory, and `myconfig.conf` is an entry in the ConfigMap.

```yaml
spec:
  containers:
  - image: some/image
  volumeMounts:
  - name: myvolume
    mountPath: /etc/someconfig.conf
    subPath: myconfig.conf
```

