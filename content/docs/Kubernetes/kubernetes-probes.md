---
title: "kubernetes-probes"
weight: 4
---
# Probes

We've created pods, watched them get replaced when we deleted them, and even debugged a `CrashLoopBackOff` or two. But here's the uncomfortable truth: **`Running` does not mean `healthy`**. A pod can be "Running" while the app inside is completely broken - a web server listening on the wrong port, an API that hangs on every request, or a process that deadlocked mid-chat.

[Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) are Kubernetes' way of asking your application: "are you _actually_ okay?" They come in three flavors:

- **Liveness probe**: "Is the container still working?" If it fails, Kubernetes _restarts_ the container.
- **Readiness probe**: "Can this pod take traffic right now?" If it fails, Kubernetes stops routing traffic to the pod (via services), but does _not_ restart it.
- **Startup probe**: "Is the container done booting?" Runs first, and disables the other probes until it succeeds. Great for slow-starting apps.

## Liveness vs Readiness

They sound similar, but they do very different things:

| | Failure action |
|---|---|
| Liveness | Container is **restarted** |
| Readiness | Pod is removed from the service's pool of endpoints |

Think about a heavy app that takes 30 seconds to warm up. If you only had a liveness probe, Kubernetes would keep restarting it before it ever gets ready. If you only had a readiness probe, a truly deadlocked app would _never_ restart - it would just sit there "ready-less" forever.

**Kural basit:**

- Liveness = "ben öldüm, beni yeniden başlat"
- Readiness = "şu an hazır değilim, bana trafik gönderme (ama beni öldürme)"

## Probe Types

Kubernetes has [several ways](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-probes) to check on a container:

- `httpGet`: Make an HTTP request to a path. Any status code in 2xx or 3xx = success
- `tcpSocket`: Try to open a TCP connection to a port
- `exec`: Run a command inside the container. Exit code `0` = success

And each probe has the same tuning knobs:

- `initialDelaySeconds`: Wait N seconds before the first probe
- `periodSeconds`: How often to probe (default: 10)
- `failureThreshold`: How many consecutive failures before it "counts"
- `timeoutSeconds`: How long to wait for a probe to succeed

## Assignment

Remember how the `api` pod returns `404` on `/` but `200` on `/healthz`? That's exactly what probes are for. Open `api-deployment.yaml` and add both probes to the `synergychat-api` container:

```yaml
containers:
  - name: synergychat-api
    image: bootdotdev/synergychat-api:latest
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
```

Apply the changes and watch the rollout:

```bash
kubectl apply -f api-deployment.yaml
kubectl get pods --watch
```

You should see a new pod replace the old one. To double check that the probes are wired up, describe the pod:

```bash
kubectl describe pod <pod-name>
```

In the `Containers` section you should see the `Liveness` and `Readiness` fields showing `http-get http://:8080/healthz`. If you ever need to prove a liveness probe works, temporarily change the path to something that returns `404`, apply, and watch Kubernetes restart the pod over and over. Don't forget to change it back!

## Probes and Rolling Updates

One last thing before we move on: probes are also the gatekeepers of rolling updates. When you `apply` a new version of a deployment, Kubernetes starts the new pod and _waits_ for its readiness probe to pass before terminating the old pod. Without a readiness probe, Kubernetes assumes the new pod is ready instantly - which is how you end up with traffic routed to a pod that's still booting.

for more [yaml-configurations](kubernetes-yaml-configurations/)
