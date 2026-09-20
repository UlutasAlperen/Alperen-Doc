---
title: "kubernetes-secrets"
weight: 6
---
# Secrets

In the [YAML config](kubernetes-yaml-configurations/) chapter we saw that ConfigMaps are a great way to manage _innocent_ environment variables - ports, URLs, feature flags. But we also learned that ConfigMaps are **not** encrypted, and anyone with access to the cluster can read them.

[Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) are the Kubernetes object designed for sensitive data: passwords, tokens, and keys. Functionally, they look a _lot_ like ConfigMaps. The differences are:

- Values are stored base64-encoded in the object
- Access can be restricted with RBAC (who can read which secrets)
- etcd can be configured to encrypt secret data at rest

## Creating a Secret

The imperative way:

```bash
kubectl create secret generic db-auth \
  --from-literal=username=admin \
  --from-literal=password=correct-horse-battery-staple
```

Or the declarative way, in YAML:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-auth
type: Opaque
data:
  password: Y29ycmVjdC1ob3JzZS1iYXR0ZXJ5LXN0YXBsZQ==
```

Notice the `data` values are base64 encoded. You can produce them like this:

```bash
echo -n "correct-horse-battery-staple" | base64
```

There's also a `stringData` field where you can put plaintext values and Kubernetes will encode them for you when the secret is stored:

```yaml
stringData:
  password: correct-horse-battery-staple
```

Let's inspect what we created:

```bash
kubectl get secrets
kubectl describe secret db-auth
```

`describe` won't show you the values, just the keys and their byte sizes. To see the raw (encoded) data:

```bash
kubectl get secret db-auth -o yaml
```

# Using a Secret

Just like ConfigMaps, secrets can be injected into pods in a few ways.

## As Environment Variables

Use `secretKeyRef` instead of `configMapKeyRef`:

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-auth
        key: password
```

Or import _all_ keys with `envFrom`:

```yaml
envFrom:
  - secretRef:
      name: db-auth
```

## As Files

You can also mount a secret as a volume, and each key becomes a file:

```yaml
volumes:
  - name: db-auth-vol
    secret:
      secretName: db-auth
```

# Base64 Şifreleme Değildir!

**Dikkat:** Secret değerlerinin base64 ile kodlanmış olması onları _güvenli_ yapmaz. Base64 sadece bir **encoding** tekniğidir, şifreleme (encryption) değildir. `kubectl get secret db-auth -o yaml` çıktısını görüp `base64 -d` çalıştırabilen herkes değerleri geri okur.

Yani secret'ların "güvenliği" şunlardan gelir:

- **RBAC**: Kimin hangi secret'ı okuyabildiğini kısıtlar
- **etcd encryption at rest**: Disk üzerinde şifreli saklanması
- **Least privilege**: Secret'ları sadece ihtiyacı olan deployment'ların namespace'lerinde tutmak

Kısacası: `kubectl` erişimi olan birinden base64 korumaz, ama yanlışlıkla bir ConfigMap'te `password: super-secret` yazmaktan çok daha iyidir.

# Alternatives

If you need more serious secret management, there are third-party options:

- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets): Encrypt secrets in your git repo, only the cluster can decrypt them
- [External Secrets Operator](https://external-secrets.io/): Sync secrets from external stores like AWS Secrets Manager or HashiCorp [Vault](https://www.vaultproject.io/)

In most production clusters I'd recommend one of these over raw Secrets, especially if secrets live in your git repos.

# Assignment

Let's create a secret for the `synergychat-api`. Create a file called `api-secret.yaml`:

- `apiVersion`: `v1`
- `kind`: `Secret`
- `metadata/name`: `synergychat-api-secret`
- `type`: `Opaque`
- `data/API_KEY`: base64-encoded fake token of your choice (`echo -n "xxx" | base64`)

Apply it:

```bash
kubectl apply -f api-secret.yaml
kubectl get secrets
```

Now connect it to the api deployment. In `api-deployment.yaml`, add to the container's `env` section:

```yaml
- name: API_KEY
  valueFrom:
    secretKeyRef:
      name: synergychat-api-secret
      key: API_KEY
```

Apply, then prove the env var made it into the container:

```bash
kubectl exec <pod-name> -- printenv | grep API_KEY
```

If you see your token, congrats - your deployment is pulling sensitive config from a Secret instead of a plaintext ConfigMap.

for more [jobs-cronjobs](kubernetes-jobs-cronjobs/)
