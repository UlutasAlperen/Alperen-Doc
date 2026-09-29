---
title: "kubernetes-ingress"
weight: 11
---
# Ingress

In the [services](../kubernetes-services/) chapter we promised to talk more about exposing things to the outside world. Time to pay that off too.

A `LoadBalancer` service gives every app its own cloud load balancer and its own public IP. That's fine when you have one app. With three apps, three IPs, three DNS records, three TLS certificates, three bills. What we actually want is one entry point that routes `chat.example.com` to the web app and `api.example.com` to the API - and gives us HTTPS while it's at it.

[Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) is that entry point. It's a set of _rules_ for routing HTTP traffic by hostname and path to services inside the cluster. It's the front door with a very organized mailroom.

**Özetlersek Service**= uygulamanın dengelenmiş/atanmis(rastgele veya degil) adresi; Ingress = "hangi URL, hangi servise gider". Ingress kendisi trafiğini taşımaz, sadece talimat yazar.

# Ingress Controller

Here's the rub: an Ingress object is just YAML. Somebody has to _read_ those rules and actually proxy the packets. That somebody is an [Ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) - nginx, traefik, Envoy, HAProxy, and friends. Install one and it watches for Ingress objects; don't install one and your Ingress sits there doing literally nothing.

Minikube ships with the nginx controller as an addon:

```bash
minikube addons enable ingress
kubectl get pods -n ingress-nginx
```

Give the controller a minute to come up. On cloud clusters the usual choice is [ingress-nginx](https://kubernetes.github.io/ingress-nginx/deploy/) installed with Helm - same idea, more knobs.

> Which controller should you pick? For the exam and for most clusters, ingress-nginx. It's the most widely documented one and the rules syntax people mean when they say "Ingress".

# Ingress Resources

Now the fun part. A [route](https://kubernetes.io/docs/concepts/services-networking/ingress/#the-ingress-resource) is a list of host/path rules, each ending at a service:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: synergychat-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: synchat.local
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

A few fields worth pinning down:

- `ingressClassName`: which controller owns this object. On clusters with two controllers this is the difference between "works" and "nothing happens"
- `host`: the HTTP `Host` header to match. Omit it and the rule catches everything
- `path` + `pathType`: `Prefix` matches path segments (`/api` matches `/api/v1` but not `/apiary`); `Exact` demands a full match; `ImplementationSpecific` is "whatever the controller feels like"
- `backend.service`: where the traffic lands - always a Service, never a pod

Order matters when paths overlap: controllers match the most specific path first, which is why `/api` can sit above `/` without eating the website.

**TLS** is one more block:

```yaml
spec:
  tls:
    - hosts:
        - synchat.local
      secretName: synchat-tls
```

The `secretName` points at a Secret holding `tls.crt` and `tls.key` - exactly the format we saw in [secrets](../kubernetes-secrets/). Real clusters usually get this issued automatically by cert-manager instead of hand-rolling it.

# Ingress vs LoadBalancer

| | LoadBalancer Service | Ingress |
|---|---|---|
| Amaç | Tek uygulama, TCP | Birden çok HTTP uygulaması |
| Routing | Yok | host + path kuralları |
| TLS | Yok (eklemen gerekir) | `tls` bloğu + Secret |
| Ne kadar dış IP | Uygulama başına 1 | Genelde 1 tane |

# Assignment

Let's put SynergyChat behind one front door.

1. Make sure the controller is running:

```bash
minikube addons enable ingress
kubectl get pods -n ingress-nginx --watch
```

2. Create a file called `synchat-ingress.yaml`:

- `apiVersion`: `networking.k8s.io/v1`
- `kind`: `Ingress`
- `metadata/name`: `synchat-ingress`
- `spec/ingressClassName`: `nginx`
- one rule with `host: synchat.local`
- path `/api` (Prefix) → `api-service` port `80`
- path `/` (Prefix) → `web-service` port `80`

Remember the [gateway](../kubernetes-gateway-minikube/) chapter used `/etc/hosts` to fake DNS locally. Same trick here - add the line so the hostname resolves:

```bash
echo "$(minikube ip) synchat.local" | sudo tee -a /etc/hosts
```

3. Apply it and see what the controller says:

```bash
kubectl apply -f synchat-ingress.yaml
kubectl get ingress
kubectl describe ingress synchat-ingress
```

`ADDRESS` fills in once the controller claims the object. The `describe` events are the first place to look when it doesn't.

4. Test both routes:

```bash
curl http://synchat.local/
curl http://synchat.local/api/
```

You want the web app on `/` and the API on `/api/`, from a single hostname. If `curl` hangs rather than fails, check `/etc/hosts` first - that's almost always the culprit in Minikube.

5. Break it on purpose: change the backend service name to `api-services` (with an `s`) and re-apply. `curl` now returns the web app's `404` for `/api` instead of routing - and `kubectl describe ingress` shows the backend doesn't exist. Revert it and watch it heal.

for more [gateway](../kubernetes-gateway-minikube/)
