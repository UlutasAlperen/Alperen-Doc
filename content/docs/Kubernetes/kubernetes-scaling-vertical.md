---
title: "kubernetes-scaling-vertical"
weight: 11
---
# Top

We've already learned how to look at logs for k8s pods, but sometimes that's not enough when it comes to debugging. Sometimes we want to know about the _resources_ that a pod is using.

To get metrics working, we need to enable the `metrics-server` addon. Run:

```bash
minikube addons enable metrics-server
```

Take a look inside the `kube-system` namespace:

```bash
kubectl -n kube-system get pod
```

You should see a new "metrics-server" pod. _It might take a couple of minutes to get started_, but once that pod is ready, you should be able to run:

```bash
kubectl top pod
```

You should see something like this:

```
NAME                               CPU(cores)   MEMORY(bytes)
synergychat-api-76b796b58d-x5wpk   1m           14Mi
synergychat-web-846d86c444-d9c8q   1m           15Mi
synergychat-web-846d86c444-sk6n4   1m           15Mi
synergychat-web-846d86c444-w2pqg   1m           15Mi
```

The `kubectl top` command (just like the [unix top command](https://en.wikipedia.org/wiki/Top_\(software\))) will show you the resources that each pod is using. In the example above, each pod is using about 1 milliCPU and 15 megabytes of memory.

# Vertical and Horizontal Scaling

Generally speaking, there are two ways to scale an application: vertically and horizontally. When I say "Scaling", I'm talking about increasing the capacity of an application. For example, maybe we have a web server, and to handle roughly 1000 requests per second, it uses about:

- 1/2 of a CPU core
- 1 GB of RAM

If we want to "scale up" to handle 2000 requests per second, we could double the CPU and RAM:

- 1 CPU core
- 2 GB of RAM

This is called "vertical scaling" because we're increasing the capacity of the application by increasing the resources available to it. We're scaling up. Scaling up works until it doesn't. You can only scale up as much as your hardware will allow (the maximum number of CPUs and amount of RAM your node has).

The other way to scale is horizontally. Instead of increasing the resources available to the application, we increase the number of _instances_ of the application (pods). Pods can be distributed _across_ nodes, so we can scale horizontally until we run out of nodes. When working in a system like Kubernetes, it's generally better to scale horizontally than vertically.

# Resource Limits

None of our current deployments have any resource limits set. We have very little traffic, so it's not currently an issue, but in a production environment, we would want to set resource limits to ensure that our pods don't consume too many resources.

We wouldn't want a pod to hog all the CPU and RAM on its node, suffocating all of the other pods on the node.

## Setting Limits

We can set [resource limits](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) in our deployment files. Here's an example:

```yaml
spec:
  containers:
    - name: <container-name>
      image: <image-name>
      resources:
        limits:
          memory: <max-memory>
          cpu: <max-cpu>
```

Memory is measured in bytes, so we can use the suffixes `Ki`, `Mi`, and `Gi` to specify [kibibytes, mebibytes, and gibibytes](https://en.wikipedia.org/wiki/Byte#Multiple-byte_units), respectively. For example, `512Mi` is 512 mebibytes.

CPU is [measured in cores](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#meaning-of-cpu), so we can use the suffix `m` to specify milli-cores. For example, `500m` is 500 milli-cores, or 0.5 cores.

## Assignment

It would be really hard to test resource limits with our SynergyChat web application because we have no production traffic. Instead, I've created a couple of custom applications we can use to test and debug resource limits.

The `bootdotdev/synergychat-testcpu:latest` image on Docker Hub is an application that simply consumes as much CPU power as it can.

Create a new file called `testcpu-deployment.yaml` with the following:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: synergychat-testcpu
  name: synergychat-testcpu
spec:
  replicas: 1
  selector:
    matchLabels:
      app: synergychat-testcpu
  template:
    metadata:
      labels:
        app: synergychat-testcpu
    spec:
      containers:
        - image: bootdotdev/synergychat-testcpu:latest
          name: synergychat-testcpu
		  resources:
            limits:
              cpu: 50m
```

Add a CPU limit of `50m` to the deployment.

Apply the deployment, then make sure the pod is running:

```bash
kubectl get pod
```

It might take a minute or so, but soon you should be able to see its metrics with `top`:

```bash
kubectl top pod
```

Assuming everything is working properly, you should see that the pod is using about 50 milli-cores of CPU. That's because k8s is throttling the pod to ensure that it doesn't use more than 50 milli-cores.

# Limits - RAM

We've successfully throttled the CPU usage of our `testcpu` pod, but what about RAM?

Create a new file called `testram-deployment.yaml` with the following:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: synergychat-testram
  name: synergychat-testram
spec:
  replicas: 1
  selector:
    matchLabels:
      app: synergychat-testram
  template:
    metadata:
      labels:
        app: synergychat-testram
    spec:
      containers:
        - image: bootdotdev/synergychat-testram:latest
          name: synergychat-testram
```

Add a memory limit of `256Mi` (256 Megabytes) to the deployment. Remember, this is the [syntax](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) to do so:

```yaml
spec:
  containers:
    - name: <container-name>
      image: <image-name>
      resources:
        limits:
          memory: <max-memory>
          cpu: <max-cpu>
```

Additionally, create a ConfigMap called `testram-configmap.yaml` with the name "synergychat-testram-configmap" for this deployment that specifies a single environment variable:

- `MEGABYTES`: `"200"`

That will tell the application to allocate 200 megabytes of memory. Update the deployment to use the config map, then apply both.

Make sure the pod is healthy:

```bash
kubectl get pods
```

After a minute or so, you should be able to see the memory usage of the pod:

```bash
kubectl top pods
```

The memory usage should be a little over 200 megabytes, but not more than 256 megabytes.

# Breaking the Limits

You may have noticed that with the `testcpu` application, we never "told" the application how much CPU to use. That's because generally speaking, applications don't know how much CPU they should use. They just go as "fast" as they can when they're doing computations.

Memory is different, applications allocate memory based on a variety of factors, and while an application can have its CPU throttled and just "go slower", if an application runs out of available memory, it will crash.

Let's test that!

Update the `MEGABYTES` environment variable for the `testram` application to `500` and apply the change.

Delete the `testram` pod so that the new environment variable takes effect. Assuming you did everything correctly, the pod should crash. You'll be able to check with:

```bash
kubectl get pods
```

Then describe the individual pod:

```bash
kubectl describe pod <pod-name>
```

Look for a section in the output that looks like this:

```
Containers:
  synergychat-testram:
    Container ID:   docker://453facc1515e05ec553ad755c6a0edffd3e67b62c14b4ddb328cc0f8d5c67250
    Image:          bootdotdev/synergychat-testram:latest
    Image ID:       docker-pullable://bootdotdev/synergychat-testram@sha256:a127779899f29d7b2e1fc80ed75e001eaed8e7cec0985707a802319fcdd9bec1
    Port:           <none>
    Host Port:      <none>
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       XXX
```




for more [scaling-horizontal](kubernetes-scaling-horizontal/)
