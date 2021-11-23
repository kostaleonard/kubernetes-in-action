# Chapter 2: First steps with Docker and Kubernetes

## The App

### Write a simple Node.js app

This app handles HTTP requests on port 8080 and responds with 200 OK and the hostname. When run from a Docker container, the hostname will be the container's ID.

```js
const http = require('http');
const os = require('os');

console.log("Kubia server starting...");

var handler = function(request, response) {
  console.log("Received request from " + request.connection.remoteAddress);
  response.writeHead(200);
  response.end("You've hit " + os.hostname() + "\n");
};

var www = http.createServer(handler);
www.listen(8080);
```

## Docker image and container

### Writing a Dockerfile to define the image

The Dockerfile tells Docker how to build the image for the app.

```Dockerfile
FROM node:7
ADD app.js /app.js
ENTRYPOINT ["node", "app.js"]
```

### Build a Docker image for the app

This command will build the image and name it `kubia` (from Kubernetes in Action).

```bash
docker build -t kubia .
```

### Run the container

This command runs a new container called `kubia-container` from the `kubia` image. It also forwards port 8080 on the host machine to 8080 on the container. `-d` runs the container in detached mode (in the background).

```bash
docker run --name kubia-container -p 8080:8080 -d kubia
```

### Explore the container

This command allows you to enter a shell in the container (`-it` flags are both necessary for an interactive terminal).

```bash
docker exec -it kubia-container bash
```

### Stop the container

This command stops the container, but does not remove it.

```bash
docker stop kubia-container
```

### Remove the container

This command removes a container.

```bash
docker rm kubia-container
```

## Pushing the image to an image registry

There are several image registries; in this example, we will use Docker Hub.

### Add additional tag to image

"Docker Hub will allow you to push an image if the image's repository name starts with your Docker Hub ID." (Luksa, 35)

My Docker Hub ID is `kostaleonard`. Note that this does not rename the image; it only adds an additional tag pointing to the same image ID. After running this, both `kubia` and `kostaleonard/kubia` point to the same image ID.

```bash
docker tag kubia kostaleonard/kubia
```

### Push to Docker Hub

You may need to log in to Docker Hub first.

```bash
docker login
```

Now you can push your image.

```bash
docker push kostaleonard/kubia
```

### Run the image on a different machine

If you don't have a different machine handy, you can also demo this by running `docker rmi <image-id>` so that you no longer have a local copy.

```bash
docker run --name kubia-container -p 8080:8080 -d kostaleonard/kubia
```

## Setting up a Kubernetes cluster

We will use Minikube (available [here](https://minikube.sigs.k8s.io/docs/start/)) for single-node development. `kubectl` is also needed and may or may not be installed when you install `minikube`.

### Starting a Kubernetes cluster with Minikube

This command will start a Kubernetes cluster.

```bash
minikube start
```

### Using a hosted Kubernetes cluster with Google Kubernetes Engine

Visit [the GKE quickstart guide](https://cloud.google.com/kubernetes-engine/docs/quickstart) for installation information.

## Running the app on Kubernetes

### Creating a pod

Usually we will use a JSON or YAML manifest to deploy an app on Kubernetes, but if our app is simple enough we can just specify command line arguments.

Arguments:

* `NAME`: Tells Kubernetes what to name the pod containing the app.
* `--image`: The container image that you want to run.
* `--port`: The port that the container exposes.

```bash
kubectl run kubia --image=kostaleonard/kubia --port=8080
```

We have now created a pod called `kubia`.

### Pod scheduling

**Pod scheduling** is the process of assigning a pod to a worker node for execution. The pod is run immediately (not at some time in the future, as the term might lead you to believe).

### Accessing the app

Each pod gets an IP address, but this address is only accessible from inside the cluster. You can verify this by running the following.

```bash
# Get the pod's internal IP for the curl command.
# Mine is 172.17.0.3.
kubectl describe pod kubia | grep IP
minikube ssh
curl 172.17.0.3:8080
```

To make the pod accessible to the outside world, you can expose it through a Service object.

```bash
kubectl expose pod kubia --type=LoadBalancer --name=kubia-http
```

This creates a service to expose the pod. We can list services with `kubectl get services`. We need to wait for the `LoadBalancer` to give the `kubia-http` service an external IP address. If you are using `minikube` and not GKE or other cloud resources, your service will never obtain an external IP address. Instead, you can create a tunnel from your host to the service using the following command.

```bash
minikube service --url=true kubia-http
```

The `--url=true` flag tells `minikube` to print the URL of the service to stdout rather than automatically launching a browser and navigating to the service.

### Kubernetes dashboard

Another way to inspect Kubernetes objects is through the Kubernetes web dashboard.

#### GKE/Cloud

With cloud deployments, you can get the URL of the dashboard with the following.

```bash
kubectl cluster-info | grep dashboard
```

To get the username and password to access this web page, run the following.

```bash
gcloud container clusters describe kubia | grep -E "(username|password):"
```

#### Minikube

Launch the dashboard with the following command.

```bash
minikube dashboard
```

## Selected Kubernetes commands

These are some of the commands that you can use to interact with Kubernetes.

* `kubectl cluster-info`: Show Kubernetes cluster info; verify cluster health.
* `minikube ssh`: Log into the Minikube VM.
* `kubectl get pods`: Show running pods.
