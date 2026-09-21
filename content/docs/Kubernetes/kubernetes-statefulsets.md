---
title: "kubernetes-statefulsets"
weight: 15
---
# StatefulSets

In [storage](../kubernetes-storage/) we learned that the filesystem inside a pod is ephemeral, and in [persistent volumes](../kubernetes-persistent-volumes/) we attached a PVC to keep data alive across pod restarts. But there's one piece of the puzzle missing.

Deployments treat all their pods as **identical, interchangeable clones**. `synergychat-web-679cbcc6cd-cq6vx` could die at any moment and a fresh `synergychat-web-679cbcc6cd-x9k2f` takes its place - nobody cares which is which. That's perfect for web servers and APIs.

Databases are different. Each instance needs:

- **Stable identity**: "I am node 0 of 3", not a random hash that changes on every reschedule
- **Stable storage**: _this_ pod's data must survive pod deletion and follow it when it's rescheduled
- **Ordered operations**: primaries should start before replicas; scale-down should retire the highest index first

A [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) is the controller designed for exactly this.

| | Deployment | StatefulSet |
|---|---|---|
| Pod adı | Rastgele (`api-646c6fd585-dk5db`) | Sabit ve sıralı (`db-0`, `db-1`) |
| Storage | Ortak PVC (yoksa volume yok) | Her pod için ayrı PVC (`volumeClaimTemplates`) |
| Ölçekleme sırası | Önemsiz | Sıralı: `db-1` olmadan `db-2` başlamaz |
| DNS | Sadece service üzerinden | Her pod'un kendi stabil DNS'i var |
| Uygun workload | Stateless servisler | Veritabanları, mesaj kuyrukları, etc |

## Headless Services

StatefulSets are almost always paired with a [headless service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) - a service with `clusterIP: None`.

Remember how a normal service gives you one stable virtual IP and load balances across pods? A headless service does the opposite: DNS queries against it return the _pod IPs directly_, one per pod:

```
nslookup postgres-headless
→ db-0.postgres-headless.default.svc.cluster.local 10.244.0.5
→ db-1.postgres-headless.default.svc.cluster.local 10.244.0.6
```

That's what makes peer-to-peer protocols (database replication, cluster gossip) possible - each pod is addressable individually, by name, forever.

# Assignment

Let's deploy a single-replica PostgreSQL with a StatefulSet. First, we need a password. Instead of a plaintext ConfigMap, we'll use the [Secret](../kubernetes-secrets/) approach from the last chapter:

```bash
kubectl create secret generic postgres-secret \
  --from-literal=POSTGRES_PASSWORD=correct-horse-battery-staple
```

Now create a headless service, `postgres-headless.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
```

And the StatefulSet, `postgres-statefulset.yaml`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_PASSWORD
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

The key part is `volumeClaimTemplates`. Instead of us hand-creating a PVC, the StatefulSet creates **one PVC per pod, automatically**, named `<template-name>-<statefulset-name>-<ordinal>`. Apply everything:

```bash
kubectl apply -f postgres-headless.yaml
kubectl apply -f postgres-statefulset.yaml
kubectl get pods --watch
kubectl get pvc
```

Notice the pod is named `postgres-0` and the PVC is `postgres-data-postgres-0`. Now for the test:

1. `kubectl exec -it postgres-0 -- psql -U postgres -c "CREATE TABLE test (id int); INSERT INTO test VALUES (1);"`
2. `kubectl delete pod postgres-0`
3. Wait for the replacement pod to become `Running`
4. `kubectl exec -it postgres-0 -- psql -U postgres -c "SELECT * FROM test;"`

The table survived! A brand new pod mounted the _same_ PVC, picked up the old data, and your table is still there. That's the StatefulSet contract: **pod may be ephemeral, its identity and its storage are not**.

## Not: Deployment vs StatefulSet Kararı

Her workload StatefulSet gerektirmez. Karar kuralım şu:

- Uygulamanın pods arasında _farkı yoksa_ ve veri paylaşılmıyorsa → **Deployment** + tek PVC yeterli (SynergyChat'in `db.json`'u böyle)
- Her instance'ın kendi verisi, kendi kimliği veya sıralı kurulum gereksinimi varsa → **StatefulSet**

Ne zaman bir veritabanını Kubernetes'te çalıştırmak istesem de, [Databases](../kubernetes-storage/#databases) bölümünde bahsettiğim gibi operasyonel yükünü hesaba katmak gerekir - ama bunu yapacaksam, doğru araç StatefulSet'tir.

for more, see the follow-up chapters in v2: [streaming replication on Longhorn](../../kubernetes_v2/postgresql-statefulset-replication/)
