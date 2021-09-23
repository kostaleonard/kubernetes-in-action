# Chapter 5: Services

## Introducing Services

A Service is a resource you create to make a single, constant point of entry to a group of pods providing the same service. Each Service has an IP address and port that never change while the Service exists. Clients can open connections to that IP and port, and those connections are then routed to one of the pods backing that service. Load balancing occurs automatically.

Label selectors determine which pods belong to a Service.

### Creating services

First, create the pods with the ReplicationController used in chapter 4: `kubectl create -f ../ch4-controllers/kubia-rc.yaml`.

#### Creating a service through `kubectl expose`

In chapter 2, we used `kubectl expose rc kubia --type=LoadBalancer --name kubia-http` to create a Service with the same label selector as the one used in the ReplicationController.

#### Creating a service through a YAML descriptor

From `kubia-svc.yaml`. The service will be available on port 80, and will forward to the pods on port 8080. By default, the service is only exposed internal to the cluster, so you won't be able to `curl` it directly (but you can if you run `minikube ssh` first).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: kubia
```

#### Remotely executing commands in running containers

First list the pods with `kubectl get pods` and choose one from which to execute a command. Next, find the cluster IP address of the service with `kubectl get svc`. Then, run exec with `kubectl exec <pod-name> -- curl -s <svc-ip>`.

**Note: Don't forget the double dash to separate the `kubectl` args from the command args.**

#### Configuring session affinity on the Service

Normally, the Service will redirect traffic to a randomly selected pod. But, you can configure a Service to bind a new client with a pod so that the same client will always connect to the same pod. This is called session affinity. You can configure it with the `sessionAffinity` field in the Service yaml.

#### Exposing multiple ports in the same Service

A Service can forward multiple ports to client pods. From `kubia-svc-multi-port.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia-multi-port
spec:
  ports: 
  - name: http
    port: 80
    targetPort: 8080
  - name: https 
    port: 443
    targetPort: 8443
  selector:
    app: kubia
```

#### Using named ports
