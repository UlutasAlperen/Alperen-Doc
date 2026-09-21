---
title: "kubernetes-deployments-minikube"
weight: 3
---
# Deployments

A _[Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)_ provides declarative updates for Pods and ReplicaSets.

You describe your _desired state_ in a Deployment, and the Deployment Controller's job is to make the _current state_ match the _desired state_. You declare your hopes and dreams, and it's Kubernetes' job to make them come true.

## Why Deleting a Pod Doesn't Feel Like a Deletion

Remember when we had you delete a pod, only to see that a new pod was created in its place? It's kinda like chopping heads off of a hydra.

That's because the _desired state_ described in our Deployment says we want 2 pods running at all times. When we delete one, the Deployment Controller sees that the _current state_ doesn't match the _desired state_, so it creates a new pod to make them match again.


Take a look at the YAML file for your current deployment in the CLI:

```bash
kubectl get deployment synergychat-web -o yaml
```

Edit the deployment and change the number of replicas from 2 to 10:

```bash
kubectl edit deployment synergychat-web
```

Make sure you've got 10 pods running:

```bash
kubectl get pod
```

Keep using `kubectl get pod` to check on your pods until all 10 are in a "ready" state. Once they are, run:

```bash
kubectl proxy
```


# Replica Sets

A [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) maintains a stable set of replica Pods running at any given time. It's the thing that makes sure that the number of Pods you want running is the same as the number of Pods that are actually running.

You might be thinking, "I thought that's what a Deployment does." Well...yes.

A Deployment is a higher-level abstraction that manages the ReplicaSets for you. You can think of a Deployment as a wrapper around a ReplicaSet. Here's the rub:

_You will probably never use ReplicaSets directly._ I just need to mention what they are because you'll hear the term thrown around, and might even see them referenced in logs and such.

## Look at Your Replica Sets

Let's take a look at the ReplicaSets that are running in your cluster:

```bash
kubectl get replicasets
```

Just like with pods, notice that _we never directly created the replica set_. We created a deployment, and the deployment created the replica set.


# YAML Config

Kubernetes resources are primarily configured using YAML files. We've used the `kubectl edit` command to edit resources in the cluster on-demand, but let's inspect our deployment's YAML file a bit more closely.


https://storage.googleapis.com/qvault-webapp-dynamic-assets/lesson_videos/what-is-yaml.mp4


First, download a copy of your deployment's YAML file and save it in your current directory:

```bash
kubectl get deployment synergychat-web -o yaml > web-deployment.yaml
```

Then open it in your text editor. There are 5 top-level fields in the file:

- `apiVersion: apps/v1` - Specifies the version of the Kubernetes API you're using to create the object (e.g., apps/v1 for Deployments).
- `kind: Deployment` - Specifies the type of object you're configuring
- `metadata` - Metadata about the deployment, like when it was created, its name, and its ID
- `spec` - The desired state of the deployment. Most impactful edits, like how many replicas you want, will be made here.
- `status` - The current state of the deployment. You won't edit this directly, it's just for you to see what's going on with your deployment.

Inside your editor, change the number of replicas to 3 and save the file. Notice that you're just editing a file on your machine! It won't yet have any effect on the deployment in your cluster.

To apply the changes, run:

```bash
kubectl apply -f web-deployment.yaml
```

You should get a warning that lets you know that you're missing the `last-applied-configuration` annotation. That's okay! we got that warning because we created this deployment the quick and dirty way, by using `kubectl create deployment` instead of creating a YAML file and using `kubectl apply -f`.

However, because we've now _updated_ it with `kubectl apply`, the annotation is now there, and we won't get the warning again.

Download the YAML file again and take a look at it. You should see the annotation now.

Apply the configuration a second time, you won't get the warning. _Save this YAML file in a git repo for this course! We'll be making more configuration files. Kubernetes is an "infra-as-code" tool, so it's important to keep your configuration files in a git repo._

Finally, start the proxy server:

```bash
kubectl proxy
```


# API Service

We've deployed one service, and we've deployed multiple instances of it. Time to deploy a second service!

This service doesn't serve a webpage! It's a JSON API. It's the backend for our chat application. By deploying the API and configuring the front-end to talk to it, we'll have a functional chat application!

## Create a Deployment Configuration

Let's write a deployment from scratch.

Feel free to reference the [k8s docs here](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#creating-a-deployment) as you go for examples of the proper structure.

1. Create a new file called `api-deployment.yaml`.
2. Add the `apiVersion` and `kind` fields. The `apiVersion` is `apps/v1` and, since this is a deployment, the `kind` is `Deployment`.
3. Add a `metadata/name` field, and let's name our deployment `synergychat-api` for consistency.
4. Add a `metadata/labels/app` field, and also set it to `synergychat-api`. This will be used to select the pods that this deployment manages.
5. Add a `spec/replicas` field and let's set it to `1`. We can always scale up to more pods later.
6. Add a `spec/selector/matchLabels/app` field and set it to `synergychat-api`. This should match the label we set in step 4.
7. Add a `spec/template/metadata/labels/app` field and set it to `synergychat-api`. Again, this should match the label we set in step 4. Labels are important because they're how Kubernetes knows which pods belong to which deployments.
8. Add a `spec/template/spec/containers` field. This actually contains a list of containers that will be deployed:
    1. Note: A hyphen is how you denote a list item in YAML
    2. Set the `name` of the container to `synergychat-api`.
    3. Set the `image` to `bootdotdev/synergychat-api:latest`. This tells k8s where to download the Docker image from.

I don't want to give you the YAML because I want you to type it out, but here's the equivalent JSON:

```json
{
  "apiVersion": "apps/v1",
  "kind": "Deployment",
  "metadata": {
    "name": "synergychat-api",
    "labels": {
      "app": "synergychat-api"
    }
  },
  "spec": {
    "replicas": 1,
    "selector": {
      "matchLabels": {
        "app": "synergychat-api"
      }
    },
    "template": {
      "metadata": {
        "labels": {
          "app": "synergychat-api"
        }
      },
      "spec": {
        "containers": [
          {
            "name": "synergychat-api",
            "image": "bootdotdev/synergychat-api:latest"
          }
        ]
      }
    }
  }
}
```

## Create the Deployment

```bash
kubectl apply -f api-deployment.yaml
```

Next, take a look at all the pods you have running now. You should see pods for the web service and a pod for the api service.

However, you might notice that the api pod isn't in a "ready" state. In fact, it should be stuck in a "CrashLoopBackOff" status. Oh no! We've created a thrashing pod!

Run:

```bash
kubectl proxy
```

for more [probes](../kubernetes-probes/)
