---
title: "postgresql-longhorn-read-replicas"
weight: 12
---
# PostgreSQL Read Replicas with CloudNativePG

The [manual setup](../postgresql-statefulset-replication/) taught us the mechanics: WAL, basebackup, standby. It also showed the catch - failover is a human job, and humans don't get paged at 3am for fun. [CloudNativePG](https://cloudnative-pg.io/) (CNPG) is the operator that automates the whole lifecycle: primary election, failover, replica management, backups - as a declarative `Cluster` CRD. On top of [Longhorn](../longhorn/) it turns our 3-node homelab into a genuinely production-shaped database platform.

## What the Operator Buys You

| | Elle StatefulSet | CloudNativePG |
|---|---|---|
| Replication kurulumu | `pg_basebackup`, `pg_hba`, init script'ler | `instances: 3` |
| Failover | Manuel `pg_promote()` + split-brain riski | Otomatik: primary ölürse en güncel replica promote edilir |
| Primary DNS | Kendi service şemanı yazarsın | `-rw` / `-ro` / `-r` service üçlüsü hazır |
| Pg upgrade/resize | Kendin | Operator rolling update yapar |
| Monitoring | Kendin | Hazır metrics endpoint |

The service trio is the part your apps touch - and it's the read-replica payoff:

- `<cluster>-rw` → always the primary (writes)
- `<cluster>-ro` → only the replicas (reads)
- `<cluster>-r` → any instance (least useful, mostly for tooling)

> One honest warning before we start: Longhorn snapshots are _not_ a PostgreSQL backup strategy. Snapshots capture the filesystem at a moment; a database mid-write can produce a torn state, and there's no point-in-time recovery across WAL. CNPG integrates barman-based backups (object storage) with proper PITR - snapshots are extra safety, never the substitute.

## How to Install CloudNativePG

1. Install the operator with Helm ([helm notes](../helm/)):

```bash
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm repo update
helm install cnpg cnpg/cloudnative-pg --namespace cnpg-system --create-namespace
kubectl -n cnpg-system get pods --watch
```

2. Verify the CRDs and the operator pod:

```bash
kubectl get crd | grep postgresql.cnpg.io
kubectl -n cnpg-system get pods
```

You should see `Cluster`, `Pooler`, `Backup` and friends registered - the operator itself is a single pod, which is refreshingly small.

3. Optional but recommended for CLI workflows: the `kubectl cnpg` plugin:

```bash
curl -sSfL https://github.com/cloudnative-pg/cloudnative-pg/raw/main/hack/install.sh | sh
kubectl cnpg --help
```

## How to Create a PostgreSQL Cluster on Longhorn

1. The `Cluster` CRD - notice there is no StatefulSet, no pg_hba, no basebackup in sight:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: app-db
spec:
  instances: 3
  storage:
    size: 5Gi
    storageClass: longhorn
  bootstrap:
    initdb:
      database: appdb
      owner: app
```

CNPG derives everything: pod identities (`app-db-1/2/3`), the streaming replication user, `pg_hba.conf`, services, and three PVCs on the `longhorn` class - each one a 3-replica Longhorn volume spread across our nodes.

2. Apply and watch the choreography - the operator starts instances _in order_ (`-1` first, as the primary, then joins the others as standbys):

```bash
kubectl apply -f app-db-cluster.yaml
kubectl get pods -o wide --watch
kubectl get pvc
```

3. Inspect the credentials CNPG generated for you - it created a superuser secret automatically:

```bash
kubectl get secret app-db-superuser -o jsonpath='{.data.username}' | base64 -d; echo
kubectl get secret app-db-superuser -o jsonpath='{.data.password}' | base64 -d
```

Same [base64-isn't-encryption](../../kubernetes/kubernetes-secrets/) rules apply as always - the secret's protection is RBAC, not the encoding.

4. Check the topology:

```bash
kubectl cnpg status app-db
```

Primary on one instance, two hot standbys streaming - exactly the manual setup from the previous chapter, minus every manual step.

## How to Use the Read Replicas

1. Look at the three services the operator created:

```bash
kubectl get svc -l cnpg.io/cluster=app-db
```

`app-db-rw` points at the primary, `app-db-ro` only at standbys. That's your application contract: writes to `-rw`, reads to `-ro`.

2. Write through the primary:

```bash
kubectl exec -it app-db-1 -- psql -U app -d appdb \
  -c "CREATE TABLE messages (id serial, body text);"
kubectl exec -it app-db-1 -- psql -U app -d appdb \
  -c "INSERT INTO messages (body) VALUES ('written on primary');"
```

3. Read from a replica - and prove it's read-only:

```bash
kubectl exec -it app-db-2 -- psql -U app -d appdb -c "SELECT count(*) FROM messages;"
kubectl exec -it app-db-2 -- psql -U app -d appdb -c "SELECT pg_is_in_recovery();"
```

`t` - you just read replicated data. And a write against `-ro` fails exactly like the manual standby did: `ERROR: cannot execute ... in a read-only transaction`.

4. From _inside the cluster_, point your app at the service names and you get connection-level read/write splitting for free - a `netshoot` pod test:

```bash
kubectl run nettest --rm -it --image=cilium/netshoot:latest -- \
  pg_isready -h app-db-ro -p 5432
```

## How to Test Failover

1. Find the primary and delete it - the operator's turn to shine:

```bash
kubectl get pod -l cnpg.io/cluster=app-db -L cnpg.io/instanceRole
kubectl delete pod app-db-1
kubectl get pods -o wide --watch
```

Watch the sequence: the most up-to-date standby gets promoted to primary, a replacement instance is _created_ and rejoins as a replica - the roles re-shuffle automatically. Check the labels again; `instanceRole: primary` moved to a different pod.

2. Verify continuity through the `-rw` service - the app-level story:

```bash
kubectl exec -it app-db-2 -- psql -U app -d appdb -c "SELECT count(*) FROM messages;"
```

The `app-db-rw` service followed the new primary; your application never knew.

3. Reconcile with git, if you run [ArgoCD](../gitops-argocd/): the CRD is declarative state, so the operator is self-healing by design - a deleted `Cluster` resource gets recreated, but _deleting the CRD deletes the database_, so treat cluster deletion like `DROP DATABASE`.

> That last line deserves its own emphasis: `kubectl delete cluster app-db` is `DROP DATABASE` for your whole fleet. CNPG uses the `finalizer` mechanism, so deleting the CR tears down the PVCs too. Deleting a pod (as we did above) is safe - it's rescheduled; deleting the `Cluster` is the dangerous one.

## Elle Setup mı Operator mı?

- Öğrenmek için elle StatefulSet kurulumu paha biçilemez - replication'ın ne olduğunu _görüyorsun_
- Ama 3am failover, lag-based failover seçimi, timeline yönetimi, managed upgrade'ler - bunları elle yapmak, operatorün çözdüğü ve **her seferinde** yanlış yapılabilen işler
- Benim kuralım: homelab'da önce elle kurdum (önceki yazı), gerçek kullanım için CNPG. ArgoCD ile birleştirince `Cluster` CRD'si de git'ten yönetilir - GitOps tüm stack'i kapsar

for more, circle back to where it all started: [kubernetes-minikube](../../kubernetes/kubernetes-minikube/)
