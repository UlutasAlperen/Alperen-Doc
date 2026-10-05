---
title: "Kubernetes v2"
weight: 4
bookCollapseSection: true
---
These are the follow-up notes to my [Kubernetes](../kubernetes/) section - same Minikube cluster, next level: packaging, hardening, operations and multi-node. I use this section to:

- Package and distribute applications with Helm charts
- Manage my own manifests with Kustomize overlays
- Lock down pod-to-pod traffic with NetworkPolicies and secure containers with Security Contexts
- Understand the extension interfaces (CNI/CSI/CRI) and what CRDs and operators really are
- Debug broken workloads - and broken control planes - like an operator, not a tourist
- Move CD from push (GitHub Actions + rsync) to pull (ArgoCD)
- Complete the autoscaling trio (HPA → VPA → Cluster Autoscaler)
- Schedule pods with taints, tolerations, spread constraints and priorities on multi-node clusters
- Upgrade clusters, restore etcd from snapshots and run a highly-available control plane
- Operate the control plane like a platform team: identity, RBAC tiers, break-glass, Helm add-ons, GitOps self-service
- Automate TLS with cert-manager and run a real monitoring stack (Prometheus or VictoriaMetrics)
- Run distributed block storage with Longhorn and build real PostgreSQL HA with read replicas
- And much more

## Kubernetes v2 konular

1. [helm](helm/)
2. [kustomize](kustomize/)
3. [cluster-extensions](cluster-extensions/)
4. [network-policy](network-policy/)
5. [security-context](security-context/)
6. [kubernetes-troubleshooting](kubernetes-troubleshooting/)
7. [control-plane-troubleshooting](control-plane-troubleshooting/)
8. [gitops-argocd](gitops-argocd/)
9. [vpa-cluster-autoscaler](vpa-cluster-autoscaler/)
10. [taints-affinity-quotas](taints-affinity-quotas/)
11. [pod-priority-disruption](pod-priority-disruption/)
12. [multi-node-kubeadm](multi-node-kubeadm/)
13. [kubeadm-upgrade-etcd](kubeadm-upgrade-etcd/)
14. [ha-control-plane](ha-control-plane/)
15. [devops-control-plane](devops-control-plane/)
16. [cert-manager](cert-manager/)
17. [cert-manager-production](cert-manager-production/)
18. [prometheus-stack](prometheus-stack/)
19. [victoriametrics](victoriametrics/)
20. [longhorn](longhorn/)
21. [postgresql-statefulset-replication](postgresql-statefulset-replication/)
22. [postgresql-longhorn-read-replicas](postgresql-longhorn-read-replicas/)

### Araçlar ve Genişletmeler

- [helm](helm/) = Chart anatomy (`crds/`, subcharts), values precedence, hooks & weights, release storage; How-to: chart inceleme, Prometheus kurulumu, rollback
- [kustomize](kustomize/) = `base`/`overlays`, strategic merge vs JSON6902, `configMapGenerator` hash trick; How-to: overlay yapısı, dev→prod promotion, config change auto-roll
- [cluster-extensions](cluster-extensions/) = CNI/CSI/CRI arayüzleri, `crictl`, CRD ve Custom Resource'lar, operator reconcile döngüsü; How-to: CRD inceleme, Helm ile operator kurulumu

### Güvenlik ve Ağ

- [network-policy](network-policy/) = additive/union semantiği, AND vs OR selectors, `ipBlock`, CNI gerçekliği; How-to: default-deny zone, web→api kuralı, netshoot doğrulama
- [security-context](security-context/) = `runAsNonRoot`/`fsGroup`/`seccompProfile`, readOnlyRootFilesystem, Pod Security Admission; How-to: non-root API, restricted profile geçişi

### Operasyon

- [kubernetes-troubleshooting](kubernetes-troubleshooting/) = pod durumları + exit code cheat-sheet, `kubectl debug` (ephemeral containers, node debug), jsonpath; How-to: CrashLoopBackOff teşhisi, shell-less debug, scheduling trace
- [control-plane-troubleshooting](control-plane-troubleshooting/) = static pod'lar, kubelet/`NotReady` karar ağacı, `etcdctl endpoint health/defrag`, kube-proxy, scheduler/CM leader election; How-to: dead apiserver, NotReady node, Türkçe checklist
- [gitops-argocd](gitops-argocd/) = GitOps prensipleri, sync waves/hooks, app-of-apps/ApplicationSet; How-to: ArgoCD kurulum + CLI login, auto-sync onboarding, drift testi

### Ölçeklendirme ve Scheduling

- [vpa-cluster-autoscaler](vpa-cluster-autoscaler/) = HPA→VPA→CA üçlüsü, VPA bileşen mimarisi (recommender/webhook/updater), PDB ilişkisi; How-to: öneri okuma, controlled mode uygulama
- [taints-affinity-quotas](taints-affinity-quotas/) = Taints/Tolerations, affinity, topologySpreadConstraints, ResourceQuota/LimitRange; How-to: node rezerve etme, replica dağıtma, namespace cap
- [pod-priority-disruption](pod-priority-disruption/) = DaemonSet, PriorityClass/preemption, PodDisruptionBudget (`minAvailable`) ve drain etkileşimi; How-to: ajan-per-node, drain sırasında servis koruma

### Multi-Node (Homelab)

- [multi-node-kubeadm](multi-node-kubeadm/) = kubeadm ile 4 node'lu k8s (1 CP + 3 worker; GPG'li apt, containerd, Cilium + kube-proxy replacement), etcd backup; How-to: bootstrap, worker join, node remove/re-add
- [kubeadm-upgrade-etcd](kubeadm-upgrade-etcd/) = `kubeadm upgrade plan/apply`, drain sırası, `etcdctl snapshot save/restore`, `kubeadm certs check-expiration/renew`; How-to: CP upgrade, worker upgrade, etcd restore, cert renew
- [ha-control-plane](ha-control-plane/) = stacked vs external etcd, quorum, `controlPlaneEndpoint` + API LB, `kubeadm join --control-plane`; How-to: 2. CP node, external etcd, failover testi

### Platform Mühendisliği

- [devops-control-plane](devops-control-plane/) = kubeconfig/identity (cert, SA token, OIDC), platform vs app RBAC katmanları, break-glass + audit, Helm ile add-on yönetimi, GitOps self-service, managed control plane karşılaştırması, Kyverno/OPA admission; How-to: scoped kubeconfig, add-on kurulumu, break-glass, namespace-as-a-service

### TLS ve Gözlemlenebilirlik

- [cert-manager](cert-manager/) = `Issuer`/`ClusterIssuer`/`Certificate`, SelfSigned vs `isCA`+`CA` zinciri, Gateway `certificateRefs` + `cert-manager.io/cluster-issuer`, Ingress vs Gateway TLS; How-to: kurulum, elle sertifika, Gateway'den üretme, CA zinciri, stuck certificate teşhisi
- [cert-manager-production](cert-manager-production/) = ACME (staging/prod, rate limitler), HTTP-01 vs DNS-01, wildcard, solver selector, renewal/`duration`/`renewBefore`/ARI, trust-manager `Bundle` + rotasyon tuzağı, Prometheus alert, hardening/HA; How-to: issuer çifti, wildcard, alert, token-silme testi
- [prometheus-stack](prometheus-stack/) = kube-prometheus-stack bileşenleri (operator, prometheus, alertmanager, grafana, node-exporter, kube-state-metrics), `ServiceMonitor`/`PrometheusRule` CRD'ları, `release:` label kuralı, CRD upgrade kuralı; How-to: helm kurulumu, kendi app'ini scrape etme, alert yazma, e2e kırma testi
- [victoriametrics](victoriametrics/) = VMSingle/VMCluster + VMAgent/VMAlert mimarisi, VM CRD'ları (`VMServiceScrape`/`VMRule`), Prometheus vs VM karşılaştırması, MetricsQL; How-to: helm kurulumu, Prometheus objelerini çevirme, single vs cluster seçimi, VMUI doğrulama

### Depolama ve HA

- [longhorn](longhorn/) = dağıtık block storage CSI, engine/replica mimarisi, snapshot vs backup; How-to: node hazırlığı (iscsid), helm kurulumu, default SC, node failover testi
- [postgresql-statefulset-replication](postgresql-statefulset-replication/) = WAL streaming replication elle: replication user, `pg_basebackup -R`, `pg_stat_replication`; manuel `pg_promote` ve split-brain uyarısı
- [postgresql-longhorn-read-replicas](postgresql-longhorn-read-replicas/) = CloudNativePG `Cluster` CRD, `-rw`/`-ro` service üçlüsü, otomatik failover; snapshot ≠ PG backup uyarısı
