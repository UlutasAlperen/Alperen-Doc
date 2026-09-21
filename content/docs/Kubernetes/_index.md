---
title: "Kubernetes"
weight: 3
bookCollapseSection: true
---
[Kubernetes](https://kubernetes.io/) is _the_ container orchestration platform. Nearly every modern DevOps workflow runs on top of it in production. I use Kubernetes to:

- Orchestrate containers across single-node (Minikube) and multi-node aws(WORK IN PROGRESS) clusters
- Declare the desired state with YAML manifests and let controllers reconcile it
- Load balance and expose services through Services and Gateways
- Scale workloads vertically and horizontally (HPA)
- Persist state with volumes, PVCs and persistent storage
- Run stateful workloads like databases with StatefulSets
- Run batch work and scheduled tasks with Jobs and CronJobs
- And much more

## Kubernetes konular (Konuları sırayla takip ederseniz, verdiğim görevlerin birbiriyle bağlantılı olduğunu ve sistemin pratikte nasıl çalıştığını daha net görebilirsiniz.)

1. [kubernetes-minikube](kubernetes-minikube/)
2. [kubernetes-pods-minikube](kubernetes-pods-minikube/)
3. [kubernetes-deployments-minikube](kubernetes-deployments-minikube/)
4. [kubernetes-probes](kubernetes-probes/)
5. [kubernetes-yaml-configurations](kubernetes-yaml-configurations/)
6. [kubernetes-secrets](kubernetes-secrets/)
7. [kubernetes-jobs-cronjobs](kubernetes-jobs-cronjobs/)
8. [kubernetes-services](kubernetes-services/)
9. [kubernetes-gateway-minikube](kubernetes-gateway-minikube/)
10. [kubernetes-namespaces](kubernetes-namespaces/)
11. [kubernetes-scaling-vertical](kubernetes-scaling-vertical/)
12. [kubernetes-scaling-horizontal](kubernetes-scaling-horizontal/)
13. [kubernetes-storage](kubernetes-storage/)
14. [kubernetes-persistent-volumes](kubernetes-persistent-volumes/)
15. [kubernetes-statefulsets](kubernetes-statefulsets/)
16. [kubernetes-nodes-basic](kubernetes-nodes-basic/)

### Temeller

- [kubernetes-minikube](kubernetes-minikube/) = Minikube ile local cluster, `kubectl create deployment`, `port-forward`, Prod vs Minikube farkı
- [kubernetes-pods-minikube](kubernetes-pods-minikube/) = Pod kavramı, ephemeral doğası, `kubectl logs/delete pod`, pod'ların unique IP adresleri
- [kubernetes-deployments-minikube](kubernetes-deployments-minikube/) = Deployment ve ReplicaSet, desired vs current state, `kubectl get/edit deployment`
- [kubernetes-probes](kubernetes-probes/) = Liveness/Readiness/Startup probes, `httpGet`/`tcpSocket`/`exec`, rolling update'lerde probes'un rolü
- [kubernetes-yaml-configurations](kubernetes-yaml-configurations/) = YAML manifest yapısı (`apiVersion`, `kind`, `spec`), ConfigMap, `env`/`envFrom`, Secrets

### Secret'lar ve Batch İşleri

- [kubernetes-secrets](kubernetes-secrets/) = `kubectl create secret`, `secretKeyRef`/`envFrom`/volume mount, base64 ≠ şifreleme, Sealed Secrets/External Secrets
- [kubernetes-jobs-cronjobs](kubernetes-jobs-cronjobs/) = Job (`completions`/`parallelism`/`backoffLimit`), CronJob (`schedule`/`concurrencyPolicy`), db.json yedekleme pipeline'ı

### Ağ (Services & Gateway)

- [kubernetes-services](kubernetes-services/) = `ClusterIP`, `NodePort`, `LoadBalancer`, `ExternalName`, stable endpoint ve load balancing
- [kubernetes-gateway-minikube](kubernetes-gateway-minikube/) = Gateway API (Envoy), `HTTPRoute`, `/etc/hosts` + `minikube tunnel`, annotations
- [kubernetes-namespaces](kubernetes-namespaces/) = `kubectl create ns`, `-n` flag, intra-cluster DNS (`svc.cluster.local`)

### Ölçeklendirme

- [kubernetes-scaling-vertical](kubernetes-scaling-vertical/) = `metrics-server`, `kubectl top`, resource limits (`50m` CPU, `256Mi` RAM), CrashLoopBackOff
- [kubernetes-scaling-horizontal](kubernetes-scaling-horizontal/) = `HorizontalPodAutoscaler` (HPA), `minReplicas`/`maxReplicas`, CPU hedefli otomatik ölçekleme

### Depolama

- [kubernetes-storage](kubernetes-storage/) = Ephemeral filesystem, `emptyDir` volumes, multi-container pod'lar (sidecar), veritabanları
- [kubernetes-persistent-volumes](kubernetes-persistent-volumes/) = `PersistentVolume` (PV), `PersistentVolumeClaim` (PVC), dynamic provisioning, volumeMounts
- [kubernetes-statefulsets](kubernetes-statefulsets/) = Stable identity, `volumeClaimTemplates` ile pod başına PVC, headless services, PostgreSQL örneği

### Prod'e Geçiş

- [kubernetes-nodes-basic](kubernetes-nodes-basic/) = Control plane ve worker node'lar, GKE/EKS/AKS, resource requests vs limits
