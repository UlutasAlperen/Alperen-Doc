---
title: "kubernetes-minikube"
weight: 1
---
# Minikube

During this blog, we'll be using [Minikube](https://minikube.sigs.k8s.io/docs/) to practice with Kubernetes. In production, you probably wouldn't use Minikube, you would use a cluster of servers, probably in the cloud. That's expensive! Minikube is a fantastic tool that allows you to run a single-node Kubernetes cluster on your local machine.

## Kubectl

The Kubernetes command-line tool, `kubectl`, allows you to run commands against Kubernetes clusters. It's a client that communicates with a Kubernetes API server.

## Installing `kubectl`

Follow the official [installation instructions for kubectl](https://kubernetes.io/docs/tasks/tools/).

## Verify Installation

Run `kubectl version --client` to verify that kubectl is installed correctly.

## Install Minikube

Follow the official [installation instructions for Minikube](https://minikube.sigs.k8s.io/docs/start/). Select the correct instructions for your system, choose either Linux/WSL or macOS.If you don't have the system requirements, you'll have a hard time getting everything up and running, unfortunately.

## Verify Installation

Run `minikube version` to verify that Minikube is installed correctly.

## Run Minikube

We'll be using Kubernetes with Docker, which is arguably the most common way to use Kubernetes. Make sure your Docker daemon is running before starting Minikube. 

Next, run:

```bash
minikube start 
```

### Previous Minikube Installations

If you've installed minikube in the past, you might have conflicts. If you don't care about your old minikube clusters, you can delete them by running:

```bash
minikube stop
minikube delete
```

Then restart minikube.

        


## Deploying an Image

The `kubectl create deployment` command will create a "deployment" for us. We'll talk more about the nuances of "deployments" later. But to put it simply, we only need to provide two things:

1. The name of the deployment (this can be anything, it's used to identify the deployment)
2. The ID of the Docker image we want to deploy (it would be a full URL if we weren't hosting the image on Docker Hub, which is the default)

```bash
kubectl create deployment synergychat-web --image=docker.io/bootdotdev/synergychat-web:latest
```

This command will deploy a container built from [this Docker image](https://hub.docker.com/r/bootdotdev/synergychat-web) to your local k8s cluster.

## Viewing Deployments

To make sure the deployment was successful, run:

`kubectl get deployments`

## Accessing the Web Page

By default, resources inside of Kubernetes run on a private, isolated network. They're visible to other resources within the cluster, but not to the outside world.

In order to access the application from your local network, you'll need to use `kubectl` to do some port forwarding. First, run:

```bash
kubectl get pods
```

We'll talk more about pods later, but for now, a pod is an abstraction over a container, and remember, a container is just a running instance of an image. To oversimplify, **a pod is a running application**.

You should see something like this:

```bash
NAME                                   READY   STATUS    RESTARTS   AGE
synergychat-web-679cbcc6cd-cq6vx       1/1     Running   0          20m
```

Next, run:

```bash
kubectl port-forward PODNAME 8080:8080
```

Be sure to replace `PODNAME` with _your_ pod's name. In my case, it was `synergychat-web-679cbcc6cd-cq6vx`.


# Minikube vs. Prod

Minikube is a great tool for learning Kubernetes, but it's not a production-scale Kubernetes cluster. The primary difference is that Minikube runs a single-node cluster, whereas production clusters are multi-node distributed systems.

## Distributed Systems Are Complex

Whenever you're dealing with a system that involves multiple machines talking to each other over a network, you're dealing with a distributed system. Distributed systems are inherently complex, and Kubernetes is no exception, but that complexity is generally abstracted away from you as a K8s user. That's what makes Kubernetes so cool! _It does a lot of the hard work for you._

## Resources and Nodes

To zoom way out, Kubernetes' job is to run software applications, and applications require resources. Resources are things like:

- CPU
- Memory
- Disk space

Kubernetes' job is to manage those resources and allocate them to the applications that are running on it. Let's look at an oversimplified example:

### 3 Nodes (Machines)

| Node   | RAM  |
| ------ | ---- |
| Node 1 | 16GB |
| Node 2 | 8GB  |
| Node 3 | 8GB  |

### 5 Pods (Applications)

|App|Required RAM|
|---|---|
|App 1|12GB|
|App 2|2GB|
|App 3|5GB|
|App 4|4GB|
|App 5|4GB|

Kubernetes looks at the resources required by each application and decides which node to run it on. In this case, it might do something like this:

|Node|Apps|RAM Left Over|
|---|---|---|
|Node 1|App 1 (12GB), App 2 (2GB)|2GB|
|Node 2|App 4 (4GB), App 5 (4GB)|0GB|
|Node 3|App 3 (5GB)|3GB|

What happens if we get a new application that requires 10GB of RAM? The cluster doesn't have enough resources to run it! The solution? Easy. Just add another node to the cluster and let Kubernetes figure out where to run it.

## This Won't Work With Minikube

With Minikube, you only get one node! So once your machine runs out of resources, you're out of luck. That's why Minikube is great for learning, but not for production.

Kubernetes clusters are running in production that have _thousands_ of nodes. That's a lot of resources to manage! But that's the beauty of Kubernetes.

_If you're interested, you can find some [case studies here](https://www.cncf.io/case-studies/). I liked [this one](https://www.cncf.io/case-studies/bloomberg/) from Bloomberg that shows they run hundreds of clusters with thousands of nodes each._
for more [pods](../kubernetes-pods-minikube/)
