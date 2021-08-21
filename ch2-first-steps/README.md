# Chapter 2: First steps with Docker and Kubernetes

## The App

### Write a simple Node.js app

This app handles HTTP requests on port 8080 and responds with 200 OK and the hostname. When run from a Docker container, the hostname will be the container's ID.

```
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

```
FROM node:7
ADD app.js /app.js
ENTRYPOINT ["node", "app.js"]
```

### Build a Docker image for the app

This command will build the image and name it `kubia` (from Kubernetes in Action).

```
docker build -t kubia .
```

### Run the container

This command runs a new container called `kubia-container` from the `kubia` image. It also forwards port 8080 on the host machine to 8080 on the container. `-d` runs the container in detached mode (in the background).

```
docker run --name kubia-container -p 8080:8080 -d kubia
```

### Explore the container

This command allows you to enter a shell in the container (`-it` flags are both necessary for an interactive terminal).

```
docker exec -it kubia-container bash
```

### Stop the container

This command stops the container, but does not remove it.

```
docker stop kubia-container
```

### Remove the container

This command removes a container.

```
docker rm kubia-container
```

## Pushing the image to an image registry

There are several image registries; in this example, we will use Docker Hub.

### Add additional tag to image

"Docker Hub will allow you to push an image if the image's repository name starts with your Docker Hub ID." (Luksa, 35)

My Docker Hub ID is `kostaleonard`. Note that this does not rename the image; it only adds an additional tag pointing to the same image ID. After running this, both `kubia` and `kostaleonard/kubia` point to the same image ID.

```
docker tag kubia kostaleonard/kubia
```

### Push to Docker Hub

You may need to log in to Docker Hub first.

```
docker login
```

Now you can push your image.

```
docker push kostaleonard/kubia
```

### Run the image on a different machine

If you don't have a different machine handy, you can also demo this by running `docker rmi <image-id>` so that you no longer have a local copy.

```
docker run --name kubia-container -p 8080:8080 -d kostaleonard/kubia
```

## Setting up a Kubernetes cluster

We will use Minikube (available [here](https://minikube.sigs.k8s.io/docs/start/)) for single-node development. `kubectl` is also needed and may or may not be installed when you install `minikube`.

### Starting a Kubernetes cluster with Minikube

This command will start a Kubernetes cluster.

```
minikube start
```

### Selected Kubernetes commands

These are some of the commands that you can use to interact with Kubernetes.

* `kubectl cluster-info`: show Kubernetes cluster info; verify cluster health.
* `minikube ssh`: log into the Minikube VM.
