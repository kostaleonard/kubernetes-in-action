# Chapter 5: Services

## Introducing Services

A Service is a resource you create to make a single, constant point of entry to a group of pods providing the same service. Each Service has an IP address and port that never change while the Service exists. Clients can open connections to that IP and port, and those connections are then routed to one of the pods backing that service. Load balancing occurs automatically.

Label selectors determine which pods belong to a Service.

### Creating services

First, create the pods with the ReplicationController used in chapter 4: `kubectl create -f ../ch4-controllers/kubia-rc.yaml`.

#### Creating a service through `kubectl expose`

In chapter 2, we used `kubectl expose rc kubia --type=LoadBalancer --name kubia-http` to create a Service with the same label selector as the one used in the ReplicationController.

#### Creating a service through a YAML descriptor
