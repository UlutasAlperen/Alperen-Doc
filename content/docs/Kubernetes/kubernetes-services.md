---
title: "kubernetes-services"
weight: 8
---
# Services

We've spun up pods and connected to them individually, but that's frankly not super useful if we want to distribute real traffic across those pods. That's where services come in.

[Services](https://kubernetes.io/docs/concepts/services-networking/service/) provide a stable endpoint for pods. They are an abstraction used to provide a stable endpoint and load balance traffic across a group of Pods. By "stable endpoint", I just mean that the service will always be available at a given URL, even if the pod is destroyed and recreated.

Let's add a service for our 3 `synergychat-web` pods. If you don't have 3 pods running, edit the deployment to have 3 replicas.

Create a file called `web-service.yaml` and add the following:

- `apiVersion`: `v1`
- `kind`: `Service`
- `metadata/name`: `web-service` (we could call it anything, but this is a fine name)
- `spec/selector/app`: I'm going to let you figure out what should be here. This is how the service knows which pods to route traffic to.
- `spec/ports`: An array of port objects. You need one entry:
    - `protocol`: `TCP` ([TCP will allow us to use HTTP](https://qr.ae/pKCJMf))
    - `port`: `80` (this is the port that the service will listen on)
    - `targetPort`: `8080` (this is the port that the pods are listening on)

This creates a new service called `web-service` with a few properties:

- It listens on port `80` for incoming traffic
- It forwards that traffic to pods that are listening on their port `8080`
- Its controller will continuously scan for pods matching the `app: synergychat-web` label selector and automatically add them to its pool

Create the service:

```bash
kubectl apply -f web-service.yaml
```

Now, let's forward the service's port to our local machine so we can test it out.

```bash
kubectl port-forward service/web-service 8080:80
```

Now, if you hit `http://localhost:8080` in your browser, you should see the web app! It's _better_ this time around because now our requests are being load-balanced across 3 pods.
# Service Types

Take a look at the `yaml` that describes your `web-service`.

```bash
kubectl get svc web-service -o yaml
```

_"svc" is a short-hand alias for "service", either will work in kubectl._

You should see a section that looks like this:

```bash
spec:
  clusterIP: 10.96.213.234
  ...
  type: ClusterIP
```

We didn't specify a service type! Why is this here? Well, it's because `ClusterIP` is the default service type.

The `clusterIP` is the IP address that the service is bound to on the internal Kubernetes network. Remember how we talked about how pods get their own internal, virtual IP address? Well, services can too! However, `type: ClusterIP` is just _one type_ of service! There are [several others](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types), including:

- `NodePort`: Exposes the Service on each Node's IP at a static port.
- `LoadBalancer`: Creates an external load balancer in the current cloud environment (if supported, e.g. AWS, GCP, Azure) and assigns a fixed, external IP to the service.
- `ExternalName`: Maps the Service to the contents of the `externalName` field (for example, to the hostname `api.foo.bar.example`). The mapping configures your cluster's DNS server to return a CNAME record with that external hostname value. No proxying of any kind is set up.

The interesting thing about service types is that they typically build on top of each other. For example, a `NodePort` service is just a `ClusterIP` service with the added functionality of exposing the service on each node's IP at a static port (it still has an internal cluster IP).

A `LoadBalancer` service is just a `NodePort` service with the added functionality of creating an external load balancer in the current cloud environment (it still has an internal cluster IP and node port).

An `ExternalName` service is actually a bit different. All it does is a DNS-level redirect. You can use it to redirect traffic from one service to another.

## Which Type Should I Use?

Well, it depends on a lot of things. If you're working in a microservices environment where many services are only meant to be accessed within the cluster, then `ClusterIP` is going to be your go-to. `NodePort` and `LoadBalancer` are used when you want to expose a service to the outside world. `ExternalName` is primarily for DNS redirects (frankly I've never used it).

We'll talk more about exposing services to the outside world in a later chapter.
# API Service

As you know, the `web` service in the SynergyChat system serves the front-end assets (HTML, CSS, and JavaScript) to the user's browser. The `api` service is responsible for handling requests from the front-end and returning data from the database.

Let's add a "NodePort" type service to expose the `api` service to the outside world.

## Example

Create a copy of your `web-service.yaml` file and name it `api-service.yaml`. Change the following:

1. The name should be `api-service`
2. Make sure it selects pods using the `app: synergychat-api` key
3. Add `type: NodePort` to the `spec` section
4. Add `nodePort: 30080` to the first object in the `ports` list

Here are the docs for the [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) service type in case you get lost.

Once you've made the changes, apply the service. Then check to make sure the service is running:

```bash
kubectl get svc
```
# Change API Service

Remember how I said that `NodePort` and `LoadBalancer` services are used to expose services to the outside world? That's true, but in most cloud-based Kubernetes environments, you'll actually use a Gateway object to expose your services. The Gateway object not only exposes your service to the outside world, but also allows you to do things like:

- Host multiple services on the same IP address
- Host multiple services on the same port (path-based routing)
- Terminate SSL
- Integrate directly with external DNS and load balancers

Because we'll be setting up a Gateway in the next chapter anyway, there's no reason to expose the API service with a `NodePort` service. Let's change it back to a `ClusterIP` service.

Switch the `api-service` back to a `ClusterIP` service.

for more [gateway](kubernetes-gateway-minikube/)
