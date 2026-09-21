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

## Kubectl

The Kubernetes command-line tool, `kubectl`, allows you to run commands against Kubernetes clusters. It's a client that communicates with a Kubernetes API server.

## Install

Follow the official [installation instructions for kubectl](https://kubernetes.io/docs/tasks/tools/).

## Verify Installation

Run `kubectl version --client` to verify that kubectl is installed correctly.

## Minikube

During this blog, we'll be using [Minikube](https://minikube.sigs.k8s.io/docs/) to practice with Kubernetes. In production, you probably wouldn't use Minikube, you would use a cluster of servers, probably in the cloud. That's expensive! Minikube is a fantastic tool that allows you to run a single-node Kubernetes cluster on your local machine.

### Run Minikube

We'll be using Kubernetes with Docker, which is arguably the most common way to use Kubernetes. Make sure your Docker daemon is running before starting Minikube. 

Next, run:

```bash
minikube start 
```

### Previous Minikube Installations

If you've installed minikube in the past, you might have conflicts. If you don't care about your old minikube clusters, you can delete them by running:

```bash
minikube stop
minikube delete
```

Then restart minikube.

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
