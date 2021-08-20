# Chapter 2: First steps with Docker and Kubernetes

### Write a simple Node.js app

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

### Build a Docker image for the app

```
docker build -t kubia .
```

### Run the container image

```
docker run --name kubia-container -p 8080:8080 -d kubia
```

