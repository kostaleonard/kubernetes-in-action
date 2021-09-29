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

You can also name a pod's ports and refer to them by those names in the Service spec. This can clarify the purpose of the service when using non-standard ports, and also be helpful when you decide to change the port number in the pod spec without changing the service spec. The service will correctly send traffic to the old port on the old pods, and the new port on the new pods. Here is an example pod definition:

```yaml
kind: Pod
spec:
  containers:
  - name: kubia
    ports:
    - name: http
      containerPort: 8080
    - name: https
      containerPort: 8443
...

```

The corresponding Service definition:

```yaml
apiVersion: v1
kind: Service
spec:
  ports:
  - name: http
    port: 80
    targetPort: http
  - name: https
    port: 443
    targetPort: https
...

```

### Discovering services

Services have static IP addresses with which clients can interact. But how do pods learn about the IP addresses of services?

#### Discovering services through environment variables

When a pod is created, Kubernetes adds environment variables for all existing services. If the service was created after the pods, then the pods will have to be destroyed and recreated. To see the environment variables, run `kubectl exec <pod-name> -- env`. You will see environment variables like `KUBIA_SERVICE_HOST` that indicate the location of the service.

#### Discovering services through DNS

Pods can also use DNS (available from the DNS pod in the `kube-system` namespace) to discover services. Pods can open a connection using the FQDN, which has the form `<svc-name>.<namespace>.<cluster-domain-suffix>`. In our example, the FQDN for the `kubia` service running in the `default` namespace is `kubia.default.svc.cluster.local`. But, you can omit the cluster suffix and, if running in the same namespace, the namespace. So, the service can be reached at the FQDN `kubia`.

We can demonstrate this by running `kubectl exec <pod-name> -- curl -s kubia`.

#### Running a shell in a pod's container

Instead of running `kubectl exec` every time you want to run something in a pod's container, you can also enter a shell directly with `kubectl exec -it <pod-name> -- bash`.

From inside the container, you can see that `curl kubia.default.svc.cluster.local`, `curl kubia.default`, and `curl kubia` all hit the service.

**Note: The cluster IP of a service is a virtual IP, which only has meaning when combined with a port. This means that layer 3 protocols (e.g., ICMP) won't work on the virtual IP.**

## Connecting to services living outside the cluster

Sometimes you want to use a service to redirect connections to external IPs instead of pods in the cluster.

### Introducing service endpoints

Endpoints is a Kubernetes resource that sits between Services and pods. A Service stores in the Endpoints resource a list of IP/port pairs that correspond with the pods the service exposes. See more about Endpoints with `kubectl get endpoints`.

### Manually configuring service endpoints

A Service will automatically configure Endpoints based on its label selector, but if the label selector is left unspecified, you can configure Endpoints manually.

First create a Service without a selector. From `external-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  ports:
  - port: 80
```

Next create an Endpoints resource for the Service. The name of the Endpoints object must match the name of the service. From `external-service-endpoints.yaml`:

```yaml
apiVersion: v1 
kind: Endpoints
metadata:
  name: external-service
subsets:
  - addresses:
    - ip: 11.11.11.11 
    - ip: 22.22.22.22
    ports:
    - port: 80
```

Now pods within the cluster can connect to the service and the connections will be load balanced between the external IPs.

### Creating an alias for an external service

You can also expose an external Service by FQDN by setting the `externalName` field in the Service spec. Notice that the Service type is now `ExternalName`, not `ClusterIP`. From `external-service-externalname.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  type: ExternalName
  externalName: someapi.somecompany.com
  ports:
  - port: 80
```

Pods will now be able to connect to `someapi.somecompany.com:80` via `external-service.default.svc.cluster.local` or just `external-service`. `ExternalName` services are implemented at the DNS level, and don't get a cluster IP address.

## Exposing services to external clients

We want to be able to expose some services (like web servers) to external clients. We have 3 options to accomplish this:

1. Setting the service type to NodePort.
1. Setting the service type to LoadBalancer, an extension of the NodePort type.
1. Creating an Ingress resource.

### Using a NodePort service

Creating a NodePort service causes Kubernetes to reserve a port on every node in the cluster (the same on all of them) and forward incoming connections to the service.

#### Creating a NodePort service

From `kubia-svc-nodeport.yaml`. `port` refers to the port of the service's internal cluster IP; `targetPort` is the port to which the service forwards traffic; `nodePort` is the port that makes the service accessible from all nodes. Specifying the `nodePort` is not mandatory, and Kubernetes will choose a port if omitted.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia-nodeport
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30123
  selector:
    app: kubia
```

Once the NodePort service is created, you can access your service through any one of the nodes' external IP addresses on port 30123 (you may have to change some firewall rules).

**Note: When using minikube, you can access your NodePort services using `minikube service <service-name>`. This will establish a tunnel into the cluster.**

Now your service is accessible externally. But how can we load balance between the nodes and ensure that clients can always connect to the service even if one of the nodes goes down? For that, we will use a LoadBalancer service.

### Exposing a service through an external load balancer

The load balancer service will have its own unique, publicly accessible IP address and will redirect all connections to your service. If your Kubernetes cluster does not support LoadBalancer services (e.g., minikube), the load balancer will not be provisioned, but the service will still function as a NodePort service since LoadBalancer is an extension of NodePort.

Your pods must still be externally routable for the load balancer to work properly. The load balancer exists outside of the cluster.

#### Creating a LoadBalancer service

From `kubia-svc-loadbalancer.yaml`. We leave the node port unspecified, letting Kubernetes choose.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia-loadbalancer
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: kubia
```

You can see the load balancer redirect you to different pods with `curl <svc-external-ip>` or `kubectl exec <pod-name> -- curl -s <svc-cluster-ip>` (the latter will always work and is useful if you can't obtain an external IP, as with minikube).

### Understanding the peculiarities of external connections

You can configure a service to redirect external traffic only to pods running on the node that received the connection by specifying the `externalTrafficPolicy: Local` field in spec. This will prevent unneccessary network hops, but if the node that received the traffic has no pods belonging to the service, the connection will hang. To get around this, you need to make sure the load balancer forwards connections only to nodes that have at least one such pod.

There is also no guarantee that the client IP is preserved from an external connection.

## Exposing services externally through an Ingress resource

Whereas each LoadBalancer service requires its own load balancer with its own public IP address, an Ingress requires only 1 IP address and can provide access to many services. Ingresses operate at the application layer and can provide features like cookie-based session affinity unavailable when using services.

**Note: To use Ingress, you may need to run minikube using the hyperkit driver (`minikube start --driver=hyperkit`) and then enable the ingress addon (`minikube addons enable ingress`).**

### Creating an Ingress resource

**Note: To create an Ingress, you must have an Inrgess controller running in your cluster. You can confirm this with `kubectl get pods --all-namespaces`.**

From `kubia-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kubia
spec:
  rules:
  - host: kubia.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kubia-nodeport
            port:
              number: 80
```

Your service is now accessible through [http://kubia.example.com](http://kubia.example.com) as long as the domain name resolves to the IP of the Ingress controller.

### Accessing the service through the Ingress

First, get the Ingress controller's IP address with `kubectl get ingresses`.

To make it so that your host resolves the domain `kubia.example.com` to the Ingress controller's IP, you can either configure your DNS server to resolve the domain or you can add the following line to `/etc/hosts`: `<server-ip> kubia.example.com`. Then, you should be able to `curl http://kubia.example.com` and hit one of the pods.

### Exposing multiple services through the same Ingress

Ingresses can contain paths to multiple hosts (domains) and paths to mutliple services on each host.

### Configuring Ingress to handle TLS traffic

You can configure the Ingress to handle TLS traffic using a Secret resource. The connection will be passed to the pod without TLS, so the pod can just handle HTTP traffic.

**Question: Does this violate principles of zero-trust architecture? If one of the nodes was compromised, wouldn't it be able to snoop on unencrypted traffic?**

To create a Secret, you need to generate a private key and certificate.

```bash
openssl genrsa -out tls.key 2048
openssl req -new -x509 -key tls.key -out tls.cert -days 360 -subj /CN=kubia.example.com
kubectl create secret tls tls-secret --cert=tls.cert --key=tls.key
```

Now we can update the Ingress manifest to accept TLS connections. From `kubia-ingress-tls.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kubia
spec:
  tls:
  - hosts:
    - kubia.example.com
    secretName: tls-secret
  rules:
  - host: kubia.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kubia-nodeport
            port:
              number: 80

```

You can then send requests with `curl -k -v https://kubia.example.com`. You will have to use `-k` (insecure) for self-signed certificates. `curl http://kubia.example.com` will get a `308 Permanent Redirect` response from the Ingress (this is intended behavior); you can follow redirects with `-L`, so you can reach the service with `curl -L -k http://kubia.example.com`.

## Signaling when a pod is ready to accept connections

Pods usually need some time to start up before they are ready to accept connections.

### Introducing readiness probes

Readiness probes indicate when a pod is ready to accept requests from a service. When a probe returns success, the pod is considered ready. Readiness probes come in the same 3 types as liveness probes (exec, HTTP GET, TCP Socket). If a pod reports that it is not ready, it is removed from any services' endpoints; when it becomes ready again, it is added back. Failing a readiness probe does not cause a pod to restart.

From `kubia-rc-readinessprobe.yaml`. This probe succeeds if `/var/ready` exists in the container. Real-world probes should check that the app is ready to receive requests.

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    app: kubia
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: luksa/kubia
        readinessProbe:
          exec:
            command:
            - ls
            - /var/ready
        ports:
        - containerPort: 8080
```


