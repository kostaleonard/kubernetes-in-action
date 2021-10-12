# Chapter 7: ConfigMaps and Secrets

There are 3 main ways to pass configuration to an app in Kubernetes.

1. Passing command-line arguments to containers
1. Setting custom environment variables in each container
1. Mounting configuration files into containers through a special kind of volume

## Passing command-line arguments to containers

In a Dockerfile, there are 2 related keywords for specifying the container executable: `ENTRYPOINT` and `CMD`. `ENTRYPOINT` is meant to specify the program to be executed; `CMD` specifies the default arguments. But, because it's possible to specify the entire command in both `ENTRYPOINT` and `CMD`, you will see both.
