---
title: "postgresql-statefulset-replication"
weight: 11
---
# PostgreSQL Streaming Replication (The Manual Way)

In the [StatefulSet notes](../../kubernetes/kubernetes-statefulsets/) we ran a single-replica PostgreSQL: pod dies, new pod mounts the same PVC, data survives. But there's still exactly _one_ copy of the data, and if the primary goes down, your database is down until it comes back. On a 3-node cluster with [Longhorn](longhorn/) underneath, we can finally do the real thing: a primary plus streaming replicas.

## Two Different Layers of Protection

This is the concept to get straight before touching YAML, because people conflate them constantly:

| Katman | Ne korur | Ne yapmaz |
|---|---|---|
| **Longhorn** (block replicas) | Disk/node kaybında verinin _kendisi_ ölmesin | Uygulama seviyesinde kullanılabilirlik sağlamaz - aynı primary hâlâ tek nokta |
| **PostgreSQL streaming replication** | Primary düşerse hot standby hazır dursun + okuma yükünü dağıt | Disk corruption / node kaybını kendisi çözmez |

**Kemer + askı örneği:** Longhorn, `kworker1` yanınca postgres volume'ünü başka node'a taşıyabilir (block layer). Streaming replication ise bir primary'ye bir şey olduğu an devralabilecek _ikinci bir postgres process'i_ tutar. Gerçek HA için ikisine de ihtiyacın var; birini diğerinin alternatifi sanmak, en pahalı veri kaybı hatalarından biridir.

## How Streaming Replication Works

- The primary writes every change to **WAL** (write-ahead log) before applying it
- A **standby** streams those WAL records in real time and replays them - it's always one (or few) steps behind
- The standby runs in recovery mode (`hot_standby = on`): it serves _reads_ while replaying, never writes
- Replication here is **asynchronous** by default: a commit ACK doesn't wait for the standby. That means a tiny window of data loss is possible on failover - the price of write latency. (Synchronous mode exists: `synchronous_standby_names` - you pay latency for zero loss.)

The plan: one StatefulSet for the primary, one for the 2 replicas, all volumes on the `longhorn` StorageClass so each postgres pod carries its own replicated volume.

## How to Deploy a Replication-Ready Primary

1. Secrets first - the app password and, crucially, a dedicated replication user password ([secrets notes](../../kubernetes/kubernetes-secrets/)):

```bash
kubectl create secret generic postgres-auth \
  --from-literal=POSTGRES_PASSWORD=app-pass \
  --from-literal=REPLICATION_PASSWORD=repl-pass
```

2. A ConfigMap that runs once at initdb: create the replication role and allow it through `pg_hba.conf`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-primary-init
data:
  init-replication.sh: |
    #!/bin/bash
    set -e
    psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname postgres <<-EOSQL
      CREATE ROLE repl_user WITH REPLICATION LOGIN PASSWORD '$REPLICATION_PASSWORD';
    EOSQL
    echo "host replication repl_user 0.0.0.0/0 scram-sha-256" >> "$PGDATA/pg_hba.conf"
```

3. The primary StatefulSet. The official `postgres` image runs anything in `/docker-entrypoint-initdb.d` after initdb - that's our hook:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-primary
spec:
  serviceName: postgres-primary
  replicas: 1
  selector:
    matchLabels:
      app: postgres-primary
  template:
    metadata:
      labels:
        app: postgres-primary
    spec:
      containers:
        - name: postgres
          image: postgres:16
          env:
            - name: POSTGRES_USER
              value: app
            - name: POSTGRES_DB
              value: appdb
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-auth
                  key: POSTGRES_PASSWORD
            - name: REPLICATION_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-auth
                  key: REPLICATION_PASSWORD
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
            - name: init
              mountPath: /docker-entrypoint-initdb.d
      volumes:
        - name: init
          configMap:
            name: postgres-primary-init
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: longhorn
        resources:
          requests:
            storage: 5Gi
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-primary
spec:
  selector:
    app: postgres-primary
  ports:
    - port: 5432
```

4. Apply and wait for `postgres-primary-0`:

```bash
kubectl apply -f postgres-primary.yaml
kubectl get pods -o wide
kubectl get pvc
```

Three PVCs will eventually exist on Longhorn - one per pod, each a 3-replica Longhorn volume underneath.

## How to Join Streaming Replicas with pg_basebackup

A replica starts life as a _copy_ of the primary's data directory plus a "follow this WAL stream" instruction. [pg_basebackup](https://www.postgresql.org/docs/current/app-pgbasebackup.html) does both at once, and with `-R` it even writes the connection config into the data directory so the replica knows where to stream from.

1. Replica StatefulSet - an initContainer builds the data dir, the main container then starts as a standby:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-replica
spec:
  serviceName: postgres-replica
  replicas: 2
  selector:
    matchLabels:
      app: postgres-replica
  template:
    metadata:
      labels:
        app: postgres-replica
    spec:
      initContainers:
        - name: basebackup
          image: postgres:16
          env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-auth
                  key: REPLICATION_PASSWORD
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          command:
            - sh
            - -c
            - |
              rm -rf /var/lib/postgresql/data/pgdata/*
              pg_basebackup -h postgres-primary -U repl_user \
                -D /var/lib/postgresql/data/pgdata -Fp -Xs -P -R
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
      containers:
        - name: postgres
          image: postgres:16
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
      terminationGracePeriodSeconds: 30
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: longhorn
        resources:
          requests:
            storage: 5Gi
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-replica
spec:
  selector:
    app: postgres-replica
  ports:
    - port: 5432
```

The `-R` flag writes `standby.signal` plus a `primary_conninfo` into the data directory - that's the "stream from postgres-primary forever" instruction. The official image's entrypoint finds a non-empty PGDATA, skips `initdb`, sees `standby.signal` and starts in recovery mode. No custom images needed.

> Watch out: `pg_basebackup -R` embeds the replication password inside `primary_conninfo` in the data directory. It works, but be aware your DB password now lives in a file inside the PVC too - the same class of trade-off we discussed in the [secrets notes](../../kubernetes/kubernetes-secrets/). Production setups use `.pgpass` files or certificates instead.

2. Also create a headless service (no `clusterIP`) if you later want each replica addressable by DNS name - for now the ordinary service above load-balances reads for you.

## How to Verify Replication and Test Reads

1. On the primary - who is streaming from me?

```bash
kubectl exec -it postgres-primary-0 -- psql -U app -c \
  "SELECT client_addr, state, sync_state, write_lag, flush_lag FROM pg_stat_replication;"
```

You want two rows in `streaming` state (the two replicas).

2. On a replica - am I in recovery, and is the data there?

```bash
kubectl exec -it postgres-replica-0 -- psql -U app -d appdb -c "SELECT pg_is_in_recovery();"
```

`t` = this instance is a standby. Create a table on the primary, then read it through the _replica service_:

```bash
kubectl exec -it postgres-primary-0 -- psql -U app -d appdb -c "CREATE TABLE t (id int);"
kubectl exec -it postgres-replica-0 -- psql -U app -d appdb -c "SELECT count(*) FROM t;"
```

The table exists on the standby within milliseconds - that's WAL streaming doing its job.

3. Prove the standby is read-only:

```bash
kubectl exec -it postgres-replica-0 -- psql -U app -d appdb -c "CREATE TABLE nope (id int);"
```

You get `ERROR: cannot execute CREATE TABLE in a read-only transaction`. This is exactly the boundary your application will feel when connecting to the replica service.

## The Catch: Failover Is Manual Here

Kill the primary and the replicas keep serving reads - but the cluster is _write-disabled_ until a human promotes one:

```bash
kubectl exec -it postgres-replica-0 -- psql -c "SELECT pg_is_in_recovery();"   # t
kubectl exec -it postgres-replica-0 -- psql -U app -d appdb -c "SELECT pg_promote();"
```

> Promoting a standby is a one-way door: it detaches from the primary permanently. If the old primary comes back, it's still convinced it's the primary - that's the classic **split-brain**, two postgreSQL instances both accepting writes. Rejoining the old primary means wiping its data and re-cloning. This is the real reason bare-managed replication is a learning exercise: automating this dance safely (elections, fencing, timeline management) is what operators like CloudNativePG do - that's the next chapter.

for more [postgresql-longhorn-read-replicas](postgresql-longhorn-read-replicas/)
