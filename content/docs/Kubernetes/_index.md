---
title: "Kubernetes"
weight: 3
bookCollapseSection: true
---
# What Is Kubernetes?

> Kubernetes, also known as K8s, is an open-source system for automating deployment, scaling, and management of containerized applications.
> 
> -- The [Kubernetes](https://kubernetes.io/) team

Kubernetes orchestrates and manages collections of containers (often using container runtimes like containerd). It takes care of scaling, distribution, and connectivity among these containers. Think of it as a _system to manage many containers and the infrastructure they run on_.

For example, you _could_ install Docker on a single server, and route traffic directly to it. That's fairly simple to set up, but what if you want 10 instances of that server? What about 1000 instances? What if you want to deploy many different services, each scaling up with more instances depending on load? Those are the problems that Kubernetes solves.

[Kubernetes](https://kubernetes.io/) is _the_ container orchestration platform. Nearly every modern DevOps workflow runs on top of it in production. I use Kubernetes to:

- Orchestrate containers across single-node (Minikube) and multi-node aws(WORK IN PROGRESS) clusters
- Declare the desired state with YAML manifests and let controllers reconcile it
- Load balance and expose services through Services, Ingress and Gateways
- Control who can do what with RBAC
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
5. [kubernetes-deployment-rollouts](kubernetes-deployment-rollouts/)
6. [kubernetes-yaml-configurations](kubernetes-yaml-configurations/)
7. [kubernetes-secrets](kubernetes-secrets/)
8. [kubernetes-rbac](kubernetes-rbac/)
9. [kubernetes-jobs-cronjobs](kubernetes-jobs-cronjobs/)
10. [kubernetes-services](kubernetes-services/)
11. [kubernetes-ingress](kubernetes-ingress/)
12. [kubernetes-gateway-minikube](kubernetes-gateway-minikube/)
13. [kubernetes-namespaces](kubernetes-namespaces/)
14. [kubernetes-dns-coredns](kubernetes-dns-coredns/)
15. [kubernetes-scaling-vertical](kubernetes-scaling-vertical/)
16. [kubernetes-scaling-horizontal](kubernetes-scaling-horizontal/)
17. [kubernetes-storage](kubernetes-storage/)
18. [kubernetes-persistent-volumes](kubernetes-persistent-volumes/)
19. [kubernetes-storage-classes](kubernetes-storage-classes/)
20. [kubernetes-statefulsets](kubernetes-statefulsets/)
21. [kubernetes-nodes-basic](kubernetes-nodes-basic/)

### Temeller

- [kubernetes-minikube](kubernetes-minikube/) = Minikube ile local cluster, `kubectl create deployment`, `port-forward`, Prod vs Minikube farkı
- [kubernetes-pods-minikube](kubernetes-pods-minikube/) = Pod kavramı, ephemeral doğası, `kubectl logs/delete pod`, pod'ların unique IP adresleri
- [kubernetes-deployments-minikube](kubernetes-deployments-minikube/) = Deployment ve ReplicaSet, desired vs current state, `kubectl get/edit deployment`
- [kubernetes-probes](kubernetes-probes/) = Liveness/Readiness/Startup probes, `httpGet`/`tcpSocket`/`exec`, rolling update'lerde probes'un rolü
- [kubernetes-deployment-rollouts](kubernetes-deployment-rollouts/) = Rolling update (`maxSurge`/`maxUnavailable`), `kubectl set image`, `rollout status/history/undo`, bozuk imajdan kurtarma
- [kubernetes-yaml-configurations](kubernetes-yaml-configurations/) = YAML manifest yapısı (`apiVersion`, `kind`, `spec`), ConfigMap, `env`/`envFrom`, Secrets

### Secret'lar, RBAC ve Batch İşleri

- [kubernetes-secrets](kubernetes-secrets/) = `kubectl create secret`, `secretKeyRef`/`envFrom`/volume mount, base64 ≠ şifreleme, Sealed Secrets/External Secrets
- [kubernetes-rbac](kubernetes-rbac/) = ServiceAccount (token mount), Role/ClusterRole, RoleBinding/ClusterRoleBinding, `kubectl auth can-i --as`
- [kubernetes-jobs-cronjobs](kubernetes-jobs-cronjobs/) = Job (`completions`/`parallelism`/`backoffLimit`), CronJob (`schedule`/`concurrencyPolicy`), db.json yedekleme pipeline'ı

### Ağ (Services, Ingress & Gateway)

- [kubernetes-services](kubernetes-services/) = `ClusterIP`, `NodePort`, `LoadBalancer`, `ExternalName`, stable endpoint ve load balancing
- [kubernetes-ingress](kubernetes-ingress/) = Ingress controller (nginx) vs Ingress resource, host/path kuralları, `ingressClassName`, TLS + Secret
- [kubernetes-gateway-minikube](kubernetes-gateway-minikube/) = Gateway API (Envoy), `HTTPRoute`, `/etc/hosts` + `minikube tunnel`, annotations
- [kubernetes-namespaces](kubernetes-namespaces/) = `kubectl create ns`, `-n` flag, intra-cluster DNS (`svc.cluster.local`)
- [kubernetes-dns-coredns](kubernetes-dns-coredns/) = CoreDNS + Corefile ConfigMap, `dnsPolicy`/`ndots`, `nslookup` ile DNS troubleshooting, kube-dns servisi

### Ölçeklendirme

- [kubernetes-scaling-vertical](kubernetes-scaling-vertical/) = `metrics-server`, `kubectl top` (pod + node), resource limits (`50m` CPU, `256Mi` RAM), CrashLoopBackOff
- [kubernetes-scaling-horizontal](kubernetes-scaling-horizontal/) = `HorizontalPodAutoscaler` (HPA), `minReplicas`/`maxReplicas`, CPU hedefli otomatik ölçekleme

### Depolama

- [kubernetes-storage](kubernetes-storage/) = Ephemeral filesystem, `emptyDir` volumes, multi-container pod'lar (sidecar), veritabanları
- [kubernetes-persistent-volumes](kubernetes-persistent-volumes/) = `PersistentVolume` (PV), `PersistentVolumeClaim` (PVC), dynamic provisioning, volumeMounts
- [kubernetes-storage-classes](kubernetes-storage-classes/) = StorageClass (`provisioner`/`volumeBindingMode`/`allowVolumeExpansion`), access modes (RWO/ROX/RWX), `reclaimPolicy` Retain/Delete, static PV
- [kubernetes-statefulsets](kubernetes-statefulsets/) = Stable identity, `volumeClaimTemplates` ile pod başına PVC, headless services, PostgreSQL örneği

### Prod'e Geçiş

- [kubernetes-nodes-basic](kubernetes-nodes-basic/) = Control plane ve worker node'lar, GKE/EKS/AKS, resource requests vs limits
