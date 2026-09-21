---
title: "kubernetes-yaml-configurations"
weight: 5
---
# YAML Config

Kubernetes resources are primarily configured using YAML files. We've used the `kubectl edit` command to edit resources in the cluster on-demand, but let's inspect our deployment's YAML file a bit more closely.
## Assignment

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

## Create a Deployment Configuration

Let's write a deployment from scratch.

Feel free to reference the [k8s docs here](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#creating-a-deployment) as you go for examples of the proper structure.

1.  Create a new file called `api-deployment.yaml`.
2.  Add the `apiVersion` and `kind` fields. The `apiVersion` is `apps/v1` and, since this is a deployment, the `kind` is `Deployment`.
3.  Add a `metadata/name` field, and let's name our deployment `synergychat-api` for consistency.
4.  Add a `metadata/labels/app` field, and also set it to `synergychat-api`. This will be used to select the pods that this deployment manages.
5.  Add a `spec/replicas` field and let's set it to `1`. We can always scale up to more pods later.
6.  Add a `spec/selector/matchLabels/app` field and set it to `synergychat-api`. This should match the label we set in step 4.
7.  Add a `spec/template/metadata/labels/app` field and set it to `synergychat-api`. Again, this should match the label we set in step 4. Labels are important because they're how Kubernetes knows which pods belong to which deployments.
8.  Add a `spec/template/spec/containers` field. This actually contains a list of containers that will be deployed:
    1. Note: A hyphen is how you denote a list item in YAML
    2. Set the `name` of the container to `synergychat-api`.
    3. Set the `image` to `bootdotdev/synergychat-api:latest`. This tells k8s where to download the Docker image from.

here's the equivalent JSON:

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

# Config Maps

There are several ways to manage environment variables in Kubernetes. One of the most common ways is to use [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/). ConfigMaps allow us to decouple our configurations from our container images, which is important because we don't want to have to rebuild our images every time we want to change a configuration value.

In a Dockerfile we can set environment variables like this:

```dockerfile
ENV PORT=3000
```

The trouble is, that means that everyone using that image will have to use port 3000. It also means that if we want to change the port, we have to rebuild the image.

## Example

First, let's take a closer look at our crashing pod and try to figure out why it's crashing.

```bash
kubectl get pods
```

Copy the pod name and then get the logs:

```bash
kubectl logs <pod-name>
```

You should see that a specific environment variable is missing! Let's fix that.

Create a new file. Let's call it `api-configmap.yaml`. Add the following YAML to it:

- `apiVersion`: `v1`
- `kind`: `ConfigMap`
- `metadata/name`: `synergychat-api-configmap`
- `data/API_PORT`: `"8080"`

Next, apply the config map:

```bash
kubectl apply -f api-configmap.yaml
```

Now, we haven't yet connected the config map to our pod, so it should still be crashing. However, for now, let's just validate that the config map was created successfully:

```bash
kubectl get configmaps
```
# Applying the Config Map

Now that we have a config map, we need to connect it to our deployment.

Open up your `api-deployment.yaml` file. We're going to add a few things to it. Under the `containers` section, add the following to the first (and only) entry:

```yaml
env:
  - name: API_PORT
    valueFrom:
      configMapKeyRef:
        name: synergychat-api-configmap
        key: API_PORT
```

This tells Kubernetes to set the `API_PORT` environment variable to the value of the `API_PORT` key in the `synergychat-api-configmap` config map. Reference the [official docs](https://kubernetes.io/docs/concepts/configuration/configmap/) if you're confused about the structure of the yaml.

Next, apply the deployment. Hopefully, you remember the command for this by now.

Once it's applied, you should be able to take a look at the pods and see that a new API pod has been deployed and isn't crashing!

Let's forward the API pod's `8080` port to our local machine so we can test it out.

```bash
kubectl port-forward <pod-name> 8080:8080
```

Make sure it returns a `404` response when you hit the root:

```bash
curl http://localhost:8080
```

# Config Maps Are Insecure

ConfigMaps are a great way to manage innocent environment variables in Kubernetes. Things like:

- Ports
- URLs of other services
- Feature flags
- Settings that change between environments, like `DEBUG` mode

However, they are _not_ cryptographically secure. ConfigMaps aren't encrypted, and they can be accessed by anyone with access to the cluster.

If you need to store sensitive information, you should use [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) or a third-party solution. See the [secrets](../kubernetes-secrets/) chapter for a full walkthrough.

# Crawler

We've got one last application to deploy to our cluster: the crawler. This is an application that continuously crawls [Project Gutenberg](https://www.gutenberg.org/) and exposes the juicy data that it finds via a JSON API.

## Assignment

### Add a New Config Map

Create a copy of your `api-configmap.yaml` file and call it `crawler-configmap.yaml`. We're going to make a few changes to it.

1. Name it `synergychat-crawler-configmap` instead of `synergychat-api-configmap`.
2. Remove the `API_PORT` environment variable.
3. Add some new environment variables:
    - `CRAWLER_PORT`: `"8080"`
    - `CRAWLER_KEYWORDS`: `love,hate,joy,sadness,anger,disgust,fear,surprise`

It's okay that the `CRAWLER_PORT` in the `crawler` deployment is the same as the `API_PORT` in the `api` deployment. They're in different pods, and these are pod-internal ports.

Here's a [reference](https://github.com/bootdotdev/synergychat/tree/main#crawler-service) to the docs for the SynergyChat microservices on GitHub in case you want additional info about the crawler.

Deploy the config map:

```bash
kubectl apply -f crawler-configmap.yaml
```

### Add a New Deployment

Create a copy of your `api-deployment.yaml` file and call it `crawler-deployment.yaml`. We're going to make a few changes to it.

1. Update all `synergychat-api` references to `synergychat-crawler`.
2. Update the image URL to `bootdotdev/synergychat-crawler:latest`.
3. Update the environment variable references to match the new config map. (See below)

For the `api` service, we used this syntax to connect the config map to the deployment:

```yaml
spec:
  containers:
    - image: bootdotdev/synergychat-api:latest
      name: synergychat-api
      env:
        - name: API_PORT
          valueFrom:
            configMapKeyRef:
              name: synergychat-api-configmap
              key: API_PORT
```

If we use this same format, it gets kinda verbose and repetitive to list out each environment variable:

```yaml
spec:
  containers:
    - image: bootdotdev/synergychat-api:latest
      name: synergychat-api
      env:
        - name: THING_ONE
          valueFrom:
            configMapKeyRef:
              name: synergychat-api-configmap
              key: THING_ONE
        - name: THING_TWO
          valueFrom:
            configMapKeyRef:
              name: synergychat-api-configmap
              key: THING_TWO
        - name: THING_THREE
          ...
```

We can use the `envFrom` key instead of the `env` key to reference the _entire_ config map and make it available to the pods in the deployment:

```yaml
envFrom:
  - configMapRef:
      name: synergychat-crawler-configmap
```

Once you've updated the deployment, apply it:

```bash
kubectl apply -f crawler-deployment.yaml
```

If the pod isn't "ready", check the logs to see if there's an error. If the error is related to environment variables, debug your config map and deployment files and reapply them.

Once it's ready, forward the pod's `8080` port to your local machine:

```bash
kubectl port-forward <pod-name> 8080:8080
```


for more [secrets](../kubernetes-secrets/)
