---
title: "kubernetes-troubleshooting"
weight: 5
---
# Troubleshooting

Everything that follows builds on one rule: **status codes first, logs second, `describe` always**. After writing dozens of deployments in the [v1 notes](../../kubernetes/), here's the operator-grade field guide.

## Pod Status Quick Reference

| Status | Anlamı | İlk bakılacak yer |
|---|---|---|
| `Pending` | Node'a schedule edilemedi | `describe` → Events (resources? taints? affinity? quota?) |
| `ImagePullBackOff` | Image çekilemiyor (tag/repo/auth) | `describe` → Events + image adı |
| `CrashLoopBackOff` | Başlıyor, çöküyor, üstel beklemeyle tekrar | `logs --previous` + exit code |
| `ErrImagePull` | `ImagePullBackOff`'un ilk hali | Aynı |
| `Running` ama broken | Process yaşıyor, app ölü | `logs` + `port-forward` ile elle dene |
| `OOMKilled` | Memory limiti aşıldı | `describe` → `Last State` |

## Exit Code Cheat Sheet

The exit code in `Last State: Terminated` usually tells the whole story:

| Code | Kim gönderdi | Ne anlama gelir |
|---|---|---|
| `0` | Uygulama | Temiz çıkış - Job'larda başarı, servislerde şüpheli |
| `1` | Uygulama | Genel hata - stack trace'e bak |
| `126` | Runtime | Komut çalıştırılamadı (izin/execute bit) |
| `127` | Runtime | Komut yok (`command:` yazım hatasının klasik belirtisi) |
| `137` | Kernel (SIGKILL) | `128+9` → **OOMKilled** ya da elle `kill -9` |
| `139` | Kernel (SIGSEGV) | Segmentation fault |
| `143` | Kernel (SIGTERM) | `128+15` → k8s'in normal kapatması (`graceful shutdown`) |

> `137` and `143` are just `128 + signal number`. Seeing `137` on a memory-limited pod? That's the kernel enforcing your limit - go re-read the [scaling notes](../../kubernetes/kubernetes-scaling-vertical/) on limits vs requests.

## The Golden Path

```bash
# 1. Hangi pod, hangi durumda?
kubectl get pods

# 2. Detay + Events - okunacak en zengin kaynak
kubectl describe pod <pod-name>

# 3. Loglar; çöken pod'un önceki attempt'ı için --previous
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
kubectl logs <pod-name> -c <container-name>

# 4. İçeri girip elle dene
kubectl exec -it <pod-name> -- sh

# 5. Cluster geneli olay akışı (neden Pending olduğunu burada da görürsün)
kubectl get events --sort-by='.lastTimestamp'
kubectl get events --field-selector reason=FailedScheduling
```

Useful structured queries once `describe` walls of text get tiring:

```bash
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}{" exit="}{.status.containerStatuses[0].lastState.terminated.exitCode}{"\n"}'
kubectl get pod <pod> -o jsonpath='{.spec.containers[*].image}{"\n"}'
kubectl get events --field-selector involvedObject.name=<pod>
```

And when you're unsure what a field even _is_:

```bash
kubectl explain pod.spec.securityContext --recursive | less
```

> Don't forget `get endpoints <service>` for "service returns 404/connection refused" mysteries: an empty endpoints list means the _service's selector_ matches no pods - the app may be perfectly healthy. Half of "broken ingress" bugs are selector mismatches, not app bugs.

# kubectl debug: Debugging Without a Shell

`exec` requires a shell in the image - distroless images have none, and adding tools to prod images is bad practice anyway. [Ephemeral containers](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/#ephemeral-debugging-container) solve this:

```bash
kubectl debug -it <pod-name> --image=cilium/netshoot:latest --target=app
```

What happens: a temporary container joins the pod's _process namespace_, so you can inspect the target container's processes (`ps aux`) - no shell required in the target image. `--target` even shares the target's process namespace specifically.

For node-level problems (disk, kubelet, network interfaces):

```bash
kubectl debug node/<node-name> -it --image=ubuntu:22.04
```

You get a container on that node with the host's filesystem mounted at `/host` - the node's logs live at `/host/var/log`, kubelet's journal via `journalctl` (run `chroot /host` first for full tooling).

Türkçe homelab checklist (kafam karıştığında sırayla uyguladığım liste):

1. Pod durumu ne? (`get pods`)
2. Events ne diyor? (`describe` - Events ve Last State bölümleri)
3. Log'da stack trace var mı? (`logs --previous`)
4. Exit code ne? (yukarıdaki tablo - `137` = OOM, `127` = command yazım hatası)
5. Env var'lar configmap/secret'tan geliyorsa, o key gerçekten var mı?
6. Selector'lar eşleşiyor mu? (deployment selector ↔ pod labels; service selector ↔ endpoints)
7. PVC bağlıysa mount path ve izinler doğru mu? (`fsGroup` hatırlatması: [security-context](security-context/))
8. İzinler/root sorunları distroless image'larda mı çıktı? ([distroless notları](../../docker/docker-distroless-container-images/))

## How to Diagnose a CrashLoopBackOff

1. Cause one: point the api deployment at a ConfigMap key that doesn't exist (or delete a key it needs), then watch:

```bash
kubectl get pods --watch
```

2. Read the state machine:

```bash
kubectl get pod <api-pod> -o jsonpath='{.status.containerStatuses[0].state}{"\n"}'
```

`waiting.reason: CrashLoopBackOff` - and `restartCount` climbing.

3. Get the _last_ attempt's story:

```bash
kubectl logs <api-pod> --previous | tail -n 20
```

4. Cross-check the exit code in `describe`:

```bash
kubectl describe pod <api-pod> | grep -A5 "Last State"
```

> The distinction between `logs` and `describe` is where beginners lose time: `logs` shows what the app printed before dying; `describe`'s Events and `Last State` show what k8s _observed_. A container that dies before printing anything has an empty `logs` and a telling `describe` - missing env, missing file, bad command (exit `127`).

5. Fix the root cause (restore the ConfigMap key), and watch `restartCount` reset with the new pod. Note: k8s does _not_ reset the backoff timer for the same pod; after repeated fixes, `delete pod` is faster than waiting out a long backoff.

## How to Debug a Shell-less Container

1. Distroless-style: attach a toolbox container to a running pod and share process visibility:

```bash
kubectl debug -it <api-pod> --image=cilium/netshoot:latest --target=synergychat-api
```

2. Inside the debug container, inspect the target's processes:

```bash
ps aux
```

You'll see the target container's process tree (thanks to `--target` + process namespace sharing).

3. Test network reachability from inside the pod's network namespace - the debug container shares the pod's IP, so this tests exactly what the app pod sees:

```bash
curl --max-time 3 http://api-service:8080/healthz
```

4. Cleanup happens automatically when you exit (ephemeral containers don't survive pod restarts) - verify with `kubectl get pod <pod> -o jsonpath='{.spec.ephemeralContainers}'`.

## How to Trace Scheduling Failures

1. Reproduce a `Pending` from the v1 notes - set an absurd resource request on `testram`:

```bash
kubectl set resources deployment/synergychat-testram --limits=memory=40000Mi --requests=memory=40000Mi
kubectl get pods --watch
```

2. Filter the noise out of global events:

```bash
kubectl get events --field-selector reason=FailedScheduling
```

You'll see exactly why: `0/1 nodes are available: 1 Insufficient memory`.

3. Confirm the decision logic in `describe` (the Events section repeats the same verdict with more context), then reason about the fix path: lower the request, add a node (`minikube node add`), or remove a [taint](taints-affinity-quotas/) that blocks the only candidate node.

4. Clean up and verify recovery:

```bash
kubectl set resources deployment/synergychat-testram --limits=memory=256Mi --requests=memory=128Mi
kubectl get pods
```

for more [gitops-argocd](gitops-argocd/)
