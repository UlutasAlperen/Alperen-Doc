---
title: "security-context"
weight: 4
---
# Security Context

In [manage-docker-as-non-root-user](../../docker/manage-docker-as-non-root-user/) we made the case for not running containers as root on Docker. Kubernetes has the same problem: most container images still run their process as `uid 0`. A [Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) sets the security options for a pod or container - and there's a whole family of them, not just "the user".

## The Field Set

**Container level** (per container in `spec.containers[].securityContext`):

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault
```

- `runAsNonRoot: true` - kubelet _refuses to start_ the container if the image's effective user resolves to 0. The most valuable single line in this chapter
- `runAsUser`/`runAsGroup` - override the image's user with numeric UID/GID (names don't work; `id -u` of the image user is what you need)
- `allowPrivilegeEscalation: false` - blocks `setuid` binaries and capability-gain inside the container
- `readOnlyRootFilesystem: true` - container filesystem becomes immutable; writes must target mounted volumes
- `capabilities.drop: ["ALL"]` - drops Linux capabilities; add back only what the app genuinely needs (`add: ["NET_BIND_SERVICE"]` for binding ports < 1024)
- `seccompProfile: RuntimeDefault` - applies the container runtime's default syscall filter. In restricted PSA this is required, and most runtimes' default is a reasonable baseline

**Pod level** (`spec.securityContext`) - applies to every container and to volumes:

```yaml
spec:
  securityContext:
    fsGroup: 2000
    fsGroupChangePolicy: OnRootMismatch
```

`fsGroup` sets group ownership of mounted volumes so a uid-1000 process can write to a PVC. `fsGroupChangePolicy: OnRootMismatch` skips recursive chown when the volume root already has the right group - a meaningful startup-time saver for big volumes.

## Who Is the Image's User, Anyway?

`runAsUser` is only sane if it matches what the image expects. Two common situations:

- Image declares `USER 1000` (distroless runtimes, many official images) → just `runAsNonRoot: true` suffices, no UID override needed
- Image declares nothing → defaults to root → you must supply `runAsUser` pointing at a UID that owns nothing the app needs to write at build time, plus a writable volume or `fsGroup` for its data

[distroless](../../docker/docker-distroless-container-images/) images ship dedicated `:nonroot` tags (uid 65532) exactly for this. If the image hardcodes user 0, `runAsNonRoot: true` simply refuses to start it - that refusal is the feature working.

# Pod Security Admission

Hand-writing every field doesn't scale. [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/) lets a namespace _enforce_ a profile:

- `privileged`: no restrictions (today's implicit default)
- `baseline`: blocks the obviously dangerous (hostNetwork, hostPath, privileged containers, most hostPaths)
- `restricted`: heavily hardened - non-root, no escalation, seccomp required, dropped capabilities, readonly root filesystem _not_ required but capabilities must be dropped

```yaml
metadata:
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.31
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

Türkçe not: üç etiket üç farklı davranışı kontrol eder - `enforce` geçersiz pod'u _reddeder_, `warn` kabul eder ama API yanıtına uyarı ekler, `audit` sadece audit log'a yazar. Geçiş stratejim: önce `warn` + `audit` koy, bir hafta logları izle, hangi deployment'ların ihlal ettiğini gör, düzelt, en son `enforce`'a geç. Version label'ı (`enforce-version`) pinlemek de önemli - aksi halde cluster'ı yükseltince yeni API'ler ihlal sayılabilir ve pod'lar aniden reddedilmeye başlar.

## Not: Rootless Container ≠ Rootless Node

İki farklı root konusu var, karıştırmayalım:

- **Container içindeki root**: bu yazının konusu - `runAsNonRoot` ile process root olmuyor
- **Node üzerindeki container runtime'ı**: Docker yazısındaki rootless daemon ve `userns-remap` konusu - k8s tarafında node hardening'i ayrı bir başlık; burada sadece pod'ların process izolasyonunu konuşuyoruz

## How to Run the API as Non-Root

1. Find out what the image does by default (before guessing a UID):

```bash
docker inspect bootdotdev/synergychat-api:latest --format '{{.Config.User}}'
```

Empty output means "runs as root by default" - we must override.

2. Add to the container spec in `api-deployment.yaml`:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
```

3. And at pod level, since the api writes `db.json` to its PVC:

```yaml
securityContext:
  fsGroup: 2000
  fsGroupChangePolicy: OnRootMismatch
```

4. Apply and verify inside the container:

```bash
kubectl apply -f api-deployment.yaml
kubectl exec <api-pod> -- id
```

You want `uid=1000`, `gid=2000` (the fsGroup supplement) - not `uid=0`.

> If the pod crashes with `Permission denied` writing to `/persist`, that's not a bug - it's your config being honest. Inspect the mount: `kubectl exec <pod> -- ls -la /persist`. The directory should be group-owned by `2000`. If it isn't, either the volume was created before `fsGroup` existed (recreate the PVC) or you picked the wrong GID.

5. Tighten further: add `readOnlyRootFilesystem: true` and check the app still works - any temp-file needs become an `emptyDir` mount at `/tmp`:

```yaml
volumeMounts:
  - name: tmp
    mountPath: /tmp
volumes:
  - name: tmp
    emptyDir: {}
```

## How to Pass the restricted PSA Profile

1. Warn-and-audit first, so you can _see_ violations without breaking anything:

```bash
kubectl label ns default pod-security.kubernetes.io/warn=restricted pod-security.kubernetes.io/audit=restricted
```

2. Re-apply the api deployment. The API response should now be clean (no warnings) if you did the non-root work above. If it still warns, the messages name each violated field - fix them one by one (`runAsNonRoot` set? seccomp set? capabilities dropped? escalation off?).

3. Enforce, pinned to your cluster's version:

```bash
kubectl label ns default pod-security.kubernetes.io/enforce=restricted pod-security.kubernetes.io/enforce-version=v1.31
```

4. Prove enforcement with an obviously-bad pod:

```bash
kubectl run bad --image=busybox:1.36
```

It must be rejected with a `Forbidden` message listing the violated rules. Then run the same image with a proper `securityContext` and watch it start - that contrast is your regression test.

> PSA is namespace-scoped and _not_ inherited. Label every namespace you care about, and prefer pinning `enforce-version` so a cluster upgrade doesn't silently change your security posture.

for more [kubernetes-troubleshooting](kubernetes-troubleshooting/)
