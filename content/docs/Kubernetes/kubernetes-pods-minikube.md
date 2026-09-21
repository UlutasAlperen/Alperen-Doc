---
title: "kubernetes-pods-minikube"
weight: 2
---
# Pods

> "Pods are the smallest deployable units of computing that you can create and manage in Kubernetes."
> 
> -- The [Kubernetes team](https://kubernetes.io/docs/concepts/workloads/pods/)

A Pod is the smallest and simplest unit in the Kubernetes object model that you create or deploy. It represents one (or sometimes more) running container(s) in a cluster. In a simple web application, you might have one single pod: the web server. As traffic grows, you might deploy that same code to multiple pods to handle the increased load. Several pods, one codebase. In a more complex backend system, you might have several pods for the web server and several pods that handle video processing. Multiple pods, multiple codebases.

![](https://storage.googleapis.com/qvault-webapp-dynamic-assets/course_assets/dK2fL05-1280x655.png)

Pods are just wrappers around containers. You can think of it as a Docker container with a little extra Kubernetes magic. The container is the actual application, and the Pod is the Kubernetes abstraction that manages the container and the resources it needs to run.

## Assignment

Let's deploy a second pod!

Use the `kubectl get pods` again to see a list of all your running pods. You should still only see the one `synergychat-web` pod. Let's add a second instance!

Run `kubectl edit deployment synergychat-web` to edit the deployment. This will open the deployment in your default text editor. You should see a big 'ol [yaml](https://yaml.org/) file. This is the configuration of your deployment. Under the "spec" section, you should see the `replicas` field set to `1`. Change it to `2`, save the file, and close the editor.

```yaml
spec:
  ...
  replicas: 2
  ...
```

Run `kubectl get pods` again. You should see two pods now!


# Ephemeral

Pods die, they die often, and sometimes without warning.

The ephemeral (fancy word for "temporary") nature of Pods is one of the defining features of Kubernetes. Unlike traditional virtual machines (VMs) or physical servers that might run indefinitely (or until hardware failure), Pods are designed to be spun up, torn down, and restarted at a moment's notice.

- **Why are they temporary?** Flexibility and resilience. If a Pod encounters a problem, it can be easily terminated and replaced with a new, healthy instance. This model not only allows for high availability but also promotes immutability. Instead of manually patching or updating _existing_ environments, you spin up new versions of the entire environment.
- **How does it affect me?** As a developer, it's crucial to understand that it's rarely a good idea to store persistent data on a Pod. They can be terminated and replaced, and any locally saved data will be lost. Plan on your image restarting from scratch often!

Get a list of your running pods:

```bash
kubectl get pods
```

Print the logs (what the container is printing to stdout) of your _older_ pod:

```bash
kubectl logs PODNAME
```

Kill that _older_ pod (this might take several seconds to complete):

```bash
kubectl delete pod PODNAME
```

# Unique IP Addresses

Every Pod in a Kubernetes cluster has a unique internal-to-k8s IP address. By giving each Pod a unique IP, Kubernetes simplifies communication and service discovery within the cluster. Pods within the same Node or across different Nodes can easily communicate.

All the resources inside a k8s cluster are virtualized. So, the IP address of a Pod is not the same as the IP address of the Node it's running on. It's a virtual IP address that is only accessible from within the cluster.

## Example

Run this command to get a "wide" output of your pods:

```bash
kubectl get pods -o wide
```

It gives a few more columns of information, including the IP address of each Pod. Notice that each Pod has a unique IP address!

Next, run:

```bash
kubectl proxy
```

This will start a proxy server on your local machine, probably on `127.0.0.1:8001`. Assuming that's the host, navigate to `http://127.0.0.1:8001/api/v1/namespaces/default/pods` in your browser. You should see a big nasty JSON blob that describes the pods that you have running.


for more [deployments](../kubernetes-deployments-minikube/)
