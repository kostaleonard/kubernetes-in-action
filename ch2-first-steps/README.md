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

My Docker Hub ID is `kostaleonard`.

```
docker tag kubia kostaleonard/kubia
```
