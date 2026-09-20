---
title: "kubernetes-jobs-cronjobs"
weight: 7
---
# Jobs

Everything we've deployed so far has been a long-running service. A [deployment](kubernetes-deployments-minikube/) promises "keep N replicas running at all times". But a lot of real work is the opposite: run once, finish, exit. Think database backups, schema migrations, or batch processing.

A [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/) creates one or more pods and keeps retrying them until the specified number of pods _successfully terminate_.

| | Deployment | Job |
|---|---|---|
| Amaç | Servis sürekli çalışsın | İş tamamlanana kadar çalışsın |
| Pod durumu | `Running` (olması beklenen) | `Completed` (başarı sinyali) |
| Çalışma biçimi | Sonsuz döngü | Bitince exit code ile biter |
| Örnek | web, api, crawler | migration, backup, batch |

A minimal job looks like this:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: example-job
spec:
  template:
    spec:
      containers:
        - name: example
          image: busybox
          command: ["echo", "hello from the job"]
      restartPolicy: Never
```

A few important spec fields:

- `completions`: How many pods need to _complete successfully_ (default `1`)
- `parallelism`: How many pods may run _concurrently_ (default `1`)
- `backoffLimit`: How many failed attempts before the Job is marked `Failed` (default `6`)
- `restartPolicy`: Must be `Never` or `OnFailure`. `Always` is _not_ allowed for Jobs - a Job pod that would never exit would defeat the point.

Apply one and watch its lifecycle:

```bash
kubectl apply -f example-job.yaml
kubectl get jobs --watch
kubectl get pods
```

Notice that the pod will show `STATUS: Completed`, not `Running`. That's the whole point!

To see the output:

```bash
kubectl logs job/example-job
```

# CronJobs

A [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) creates Jobs on a repeating schedule. If you've ever written a crontab, this will feel like home:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: example-cronjob
spec:
  schedule: "*/5 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: example
              image: busybox
              command: ["echo", "tick"]
          restartPolicy: OnFailure
```

## Cron Formatı Hatırlatması

```
* * * * *
│ │ │ │ │
│ │ │ │ └── haftanın günü (0-6, 0=Pazar)
│ │ │ └──── ay (1-12)
│ │ └────── ayın günü (1-31)
│ └──────── saat (0-23)
└────────── dakika (0-59)
```

- `*/5 * * * *` = her 5 dakikada bir
- `0 */6 * * *` = her 6 saatte bir (0. dakikada)
- `0 3 * * *` = her gün saat 03:00'te

Two more useful spec fields:

- `concurrencyPolicy`: `Allow` (default), `Forbid` (skip if the previous job is still running) or `Replace` (kill the old one)
- `startingDeadlineSeconds`: Max allowed lateness, otherwise the run is skipped

# Assignment

Remember the `db.json` file that our api writes to its persistent volume (see [storage](kubernetes-storage/) and [persistent volumes](kubernetes-persistent-volumes/))? Let's build a backup CronJob for it. Create `api-backup-cronjob.yaml`:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: api-db-backup
spec:
  schedule: "0 */6 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: busybox:1.36
              command:
                - sh
                - -c
                - cp /persist/db.json /backup/db-$(date +%Y%m%d-%H%M%S).json && ls -la /backup && cat /persist/db.json
              volumeMounts:
                - name: api-data
                  mountPath: /persist
                - name: backup
                  mountPath: /backup
          volumes:
            - name: api-data
              persistentVolumeClaim:
                claimName: synergychat-api-pvc
            - name: backup
              emptyDir: {}
```

A few things to notice:

- We mount the _same_ PVC the api pod uses. It's `ReadWriteOnce`, and since Minikube is a single-node cluster, multiple pods on that node can mount it.
- The backup target is an `emptyDir` volume, which dies with the pod. For this exercise we only verify the backup via logs; in real life you'd push the file to S3/object storage or a dedicated backup PVC.
- `restartPolicy: OnFailure` means a failed backup attempt will retry.

Apply it:

```bash
kubectl apply -f api-backup-cronjob.yaml
kubectl get cronjobs
```

We don't want to wait up to 6 hours to see if it works. Manually trigger one job right now:

```bash
kubectl create job --from=cronjob/api-db-backup manual-backup
```

Then check the results:

```bash
kubectl get jobs
kubectl logs job/manual-backup
```

You should see the timestamped `db-YYYYMMDD-HHMMSS.json` file in the listing, and the contents of the messages database in the logs. Congratulations - you now have a scheduleable, retryable backup pipeline in a single YAML file.

for more [services](kubernetes-services/)
