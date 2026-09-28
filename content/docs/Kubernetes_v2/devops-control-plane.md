---
title: "devops-control-plane"
weight: 15
---
# Who Owns the Control Plane

Everything until now assumed _we_ own the cluster: we built it on [multi-node VMs](../multi-node-kubeadm/), we [SSH'd into it](../control-plane-troubleshooting/) when it broke, we [joined control-plane nodes](../ha-control-plane/) by hand. That's the homelab model. The moment a second team shows up, the question changes from "how do I run this" to **"who is allowed to do what"** - and that's a platform engineering question, not a kubectl one.

The split that every real org lands on eventually:

- **Platform / DevOps team**: owns the control plane, the nodes, the cluster add-ons (CNI, CSI, ingress, observability), upgrades, and the guardrails. Small team, high blast radius
- **Application teams**: own workloads inside their namespaces. Deploy, scale, debug - but not the cluster

The control plane stops being "a server I log into" and becomes **an API with policies around it**. Nobody SSH's to `kmaster` to deploy a feature. The platform team's job is to make sure nobody needs to.

**Küçük model:** cluster bir bina; platform ekibi bina sahibi (asansör, elektrik, yangın merdiveni), uygulama ekipleri kiracılar (kendi dairelerinde serbest, ama duvarı yıkmak yok). İyi bir platform, kiracıya "istek at" der - "ticket aç" değil.

# Kubeconfig and Identity

Access starts with identity. On our kubeadm cluster the first thing `kubeadm init` hands you is `admin.conf` - a kubeconfig with cluster-admin baked in. That file is the cluster's master key. Handing it to everyone "just for now" is how clusters end up with no audit trail and no boundaries.

A user's access is a kubeconfig of their own:

```yaml
apiVersion: v1
kind: Config
clusters:
  - name: homelab
    cluster:
      server: https://k8s-api.local:6443
      certificate-authority-data: <base64-ca>
users:
  - name: alice
    user:
      client-certificate-data: <base64-cert>
      client-key-data: <base64-key>
contexts:
  - name: alice@homelab
    context:
      cluster: homelab
      user: alice
current-context: alice@homelab
```

Three identity models, from most DIY to most real-world:

| Model | Nasıl çalışır | Kim kullanır |
|---|---|---|
| Client certs | CA ile imzalanmış kullanıcı sertifikası (`kubeadm certs`) | Homelab, küçük ekipler |
| ServiceAccount token | Pod'lara ve CI'ya verilen token | Otomasyon, pipeline'lar |
| OIDC / SSO | Google/Azure/Dex/Keycloak ile login | Gerçek şirketlerin standardı |

The kubeconfig mechanics are worth being fast at:

```bash
kubectl config get-contexts
kubectl config use-context alice@homelab
kubectl config set-context --current --namespace=backend
KUBECONFIG=~/.kube/homelab:~/.kube/prod kubectl config get-contexts
```

Merged `KUBECONFIG` (colon-separated) is how operators juggle clusters without overwriting each other's credentials. And when you're helping someone debug _their_ problem, you don't need their kubeconfig - you impersonate:

```bash
kubectl get pods -n backend --as=alice
kubectl auth can-i list secrets -n backend --as=alice
```

Impersonation is the support tool of choice: "let me see exactly what you see" without sharing credentials. The [rbac](../../kubernetes/kubernetes-rbac/) notes' `--as` flag was practice for this.

# Platform RBAC vs App RBAC

The [rbac](../../kubernetes/kubernetes-rbac/) notes showed _how_ to write a Role. Platform engineering decides _who gets which kind_ - and the rule of thumb is embarrassingly simple:

- **Platform team**: cluster-scoped permissions - nodes, namespaces, CRDs, storage classes, cluster add-ons. Their tooling needs `ClusterRole`s
- **App teams**: namespaced permissions inside their own namespace - deployments, services, configmaps, secrets, pods/log. A `Role` + `RoleBinding`, nothing wider
- **CI pipelines**: a `ServiceAccount` with exactly the verbs the pipeline needs (`create`, `update` deployments; not `delete` namespaces)

**Türkçe özet - yetki katmanları:**
- **Cluster-admin** = sadece platform, sadece break-glass (aşağıda)
- **Namespace admin** = takım liderleri, kendi namespace'inde
- **Namespace edit** = geliştiriciler, günlük işler
- **View / pod exec** = destek ve gözlemci roller

Namespace-as-a-service is this pattern automated: a team asks for a namespace, and the platform hands over a namespace that already contains the `RoleBinding`s, a `ResourceQuota`, a `LimitRange` and (from the [taints and quotas](../taints-affinity-quotas/) notes) sensible defaults. Nobody files a ticket for "give me access to my own stuff".

> The sharp edge of cluster-scoped roles: `ClusterRoleBinding` multiplies blast radius by every namespace. Bind a `ClusterRole` with a namespaced `RoleBinding` instead - same role, one namespace. That single substitution is the most common RBAC fix I make.

# Break-Glass and Audit

Policies will, at some point, lock out the wrong person - sometimes the only person who can fix the cluster. Every platform setup needs a **break-glass** path: a documented, audited, last-resort way in when RBAC says no and the API is otherwise healthy.

What that looks like on our kubeadm cluster:

1. `admin.conf` is copied to the control-plane node's `/root/.kube/config` and nowhere else - not in git, not in Slack, not on laptops
2. To use it, you SSH to the control-plane node ([control-plane troubleshooting](../control-plane-troubleshooting/) style) and work from there
3. The SSH session and the subsequent `kubectl` calls both land in the audit log - which you keep

Audit logging is the difference between "we think someone deleted the namespace" and a timeline:

```yaml
# /etc/kubernetes/audit-policy.yaml (hooked into the apiserver manifest)
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["secrets", "namespaces"]
```

Add `--audit-policy-file` and `--audit-log-path` to the static apiserver manifest and restart it - the [control-plane notes](../control-plane-troubleshooting/) cover the mechanics of editing and restarting static pods.

> **Dikkat:** break-glass bir _prosedürdür_, bir dosya değildir. Mühürlü bir kasa, yazılı bir sıra (kim, ne zaman, neden), ve sonrasında otomatik bir "break-glass kullanıldı" alarmı yoksa; o dosya 3 ay sonra herkesin laptop'unda duruyordur. Yılda en az bir kez gerçekten kullanarak test et.

Two other pieces of the "who did what" toolkit:

```bash
kubectl auth whoami
kubectl get events -A --sort-by=.lastTimestamp | tail -20
```

`kubectl auth whoami` (with the `SelfSubjectReview` API) tells you which identity your kubeconfig actually maps to - the first command to run when "why am I forbidden?" shows up.

# Helm as the Platform's Delivery Tool

The [helm](../helm/) notes were written from a developer's seat. From the platform seat, Helm is how **cluster add-ons** are installed and upgraded - the CNI from the [multi-node](../multi-node-kubeadm/) notes, the ingress controller from [ingress](../../kubernetes/kubernetes-ingress/), Longhorn, cert-manager, the observability stack:

```bash
helm list -A
helm -n ingress-nginx history ingress-nginx
helm -n ingress-nginx upgrade ingress-nginx ingress-nginx/ingress-nginx -f values-prod.yaml
```

The division of labor that keeps this sane:

- **Platform team** runs Helm for anything cluster-scoped: CNI, CSI, ingress, cert-manager, monitoring. Values live in a git repo, one file per environment
- **App teams** either get a curated internal chart repo (with guardrails baked in) or deploy raw manifests through GitOps - but they don't `helm install` the ingress controller

`helm list -A` is the platform team's inventory of what it has actually installed. If something is running and it's not in that list (and not in GitOps), it's shadow infrastructure - someone installed it by hand on a Friday.

> Upgrading a cluster add-on is a change to _every_ team's traffic. The platform's Helm releases deserve the same review, staging and rollback discipline as a production deployment - `helm rollback` from the helm notes is your friend, but only if you remember `helm history` first.

# GitOps and Self-Service

The endgame of platform engineering is self-service: app teams ship without the platform team being a bottleneck. The [gitops-argocd](../gitops-argocd/) notes showed the mechanism - Git is the desired state, ArgoCD reconciles it. Combined with namespace-as-a-service, the flow becomes:

1. A team gets a namespace (with RBAC, quota, limits pre-installed)
2. Their manifests live in their own folder/repo
3. A PR merges → ArgoCD syncs → the platform team is not involved
4. The platform team's own add-ons go through the same door, just in a different folder

The thing that makes this work is what it _removes_: no shared kubeconfigs, no `kubectl apply` from laptops to prod, no "who has the current version of that YAML". The [drift test](../gitops-argocd/) from those notes is how you prove the model holds.

> A common failure: ArgoCD installed with cluster-admin "to make it work". ArgoCD only needs permissions on what it manages. Give it a `ClusterRole` scoped to managed namespaces and the day it's compromised, it can't delete `kube-system`.

# Managed Control Planes

On EKS, GKE or AKS the platform team's job changes shape, because someone else runs the control plane:

| | Kubeadm (self-hosted) | Managed (EKS/GKE/AKS) |
|---|---|---|
| etcd erişimi | Tam - backup/restore bizde | Yok - sağlayıcıda |
| Sertifika yönetimi | `kubeadm certs` bizde | Sağlayıcı yönetir |
| Control-plane upgrade'i | `kubeadm upgrade apply` | API çağrısı / console |
| SSH ile node erişimi | Var | Genelde yok (veya kısıtlı) |
| Fiyat | Sadece VM | Cluster ücreti + iş yükü |
| Sorumluluk | Her şey bizde | "Shared responsibility" |

What you lose is exactly what the [kubeadm-upgrade-etcd](../kubeadm-upgrade-etcd/) and [control-plane troubleshooting](../control-plane-troubleshooting/) notes teach: there is no `/etc/kubernetes/manifests` to edit, no `etcdctl snapshot restore`, no static pods. When the API is down, it's the provider's page to answer - and your job becomes the things you still own: node pools, add-ons, RBAC, workloads.

What you gain is real: no control-plane upgrades at 2am, no cert renewal calendar, no etcd quorum math ([HA notes](../ha-control-plane/) become theory you should still know for the interview). Most companies run managed control planes and self-managed everything above them - which is precisely the platform team's domain.

> Learn the self-hosted path anyway. Managed clusters hide the machinery, and the machinery is what breaks at 3am on the one cluster that isn't managed.

# Policy and Admission

The last platform lever: instead of telling teams "don't do that", make "that" impossible. Admission controllers validate (and mutate) every object before it's stored - the [Pod Security Admission](../security-context/) notes used the built-in version. The policy engines extend it to anything:

```yaml
# Kyverno-style: require an owner label on every Deployment
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-owner-label
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-owner
      match:
        any:
          - resources:
              kinds: ["Deployment"]
      validate:
        message: "every deployment needs an owner label"
        pattern:
          metadata:
            labels:
              owner: "?*"
```

Two flavors, with different reputations:

- **Kyverno**: policies are YAML, very "k8s native", easy to read. `Enforce` blocks, `Audit` only reports
- **OPA Gatekeeper**: policies in Rego (a real language), more expressive, steeper learning curve

Both turn platform standards into code that runs on every PR - naming conventions, image registries, required probes, forbidden `:latest` tags. Combined with [ResourceQuota and LimitRange](../taints-affinity-quotas/), it's the difference between a platform and a wiki page.

**Küçük model:** RBAC "kim ne yapabilir", admission "ne tür nesneler var olabilir", quota "ne kadar kaynak harcanabilir". Platform üçünü birden kullanır; sadece biri yetmez.

# How to Issue a Scoped kubeconfig for an App Team

1. Create the identity - a ServiceAccount (for CI) or a user cert (for humans):

```bash
kubectl create ns backend
kubectl -n backend create serviceaccount backend-ci
```

2. Grant exactly what they need - a `Role`, not a `ClusterRole`:

```bash
kubectl -n backend apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: backend-edit
  namespace: backend
rules:
  - apiGroups: ["apps", ""]
    resources: ["deployments", "services", "configmaps", "pods", "pods/log"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
EOF
kubectl -n backend create rolebinding backend-edit --role=backend-edit --serviceaccount=backend:backend-ci
```

3. Mint a token and build a kubeconfig around it (or wire the OIDC claim):

```bash
kubectl -n backend create token backend-ci --duration=8760h
```

4. Verify the boundary from both sides - this is the whole point:

```bash
kubectl auth can-i create deployments -n backend --as=system:serviceaccount:backend:backend-ci
kubectl auth can-i create deployments -n kube-system --as=system:serviceaccount:backend:backend-ci
```

`yes` then `no`. Ship that kubeconfig (or the token) to the team and the contract is complete.

# How to Install a Cluster Add-On as the Platform Team

1. Pick the add-on and inspect the chart before touching the cluster ([helm](../helm/) notes: look before you leap):

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm show values ingress-nginx/ingress-nginx > values-ingress.yaml
```

2. Install it into a dedicated namespace with an explicit values file - one file per environment, committed to git:

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress-nginx --create-namespace \
  -f values-ingress.yaml
```

3. Verify it's healthy and record it in the inventory:

```bash
helm list -A
kubectl get pods -n ingress-nginx
```

4. Practice the rollback on purpose: push a bad value (`replicaCount: 0`), `helm upgrade`, then `helm rollback ingress-nginx 1 -n ingress-nginx` and confirm. Do this before you need it.

# How to Set Up Break-Glass Access

1. Keep `admin.conf` only on the control-plane node, and make the SSH path the documented one:

```bash
scp /etc/kubernetes/admin.conf root@kmaster:/root/.kube/config
```

2. Turn on audit logging (policy file + apiserver flags), per the break-glass section above, and ship the log somewhere it survives the node dying.

3. Write the procedure down - who may use it, for what, and what happens after (incident review, credential rotation). Store it where the on-call can reach it when the wiki is down.

4. Test it quarterly: have someone use the break-glass path to fix a staged problem, and verify the audit trail recorded it. Then rotate whatever credentials were touched.

# How to Provision Namespace-as-a-Service

1. Template the whole package - namespace, `RoleBinding`, `ResourceQuota`, `LimitRange` ([taints and quotas](../taints-affinity-quotas/) for the field reference):

```bash
kustomize build platform/namespaces/backend | kubectl apply -f -
```

2. Wire the app team's repo into [ArgoCD](../gitops-argocd/) pointing at their folder, with their ServiceAccount as the sync identity.

3. Let them verify self-service works end to end - deploy, break something, roll back, read logs - without asking you for a single `kubectl` command.

4. Audit monthly: `kubectl get rolebindings -A`, `helm list -A`, `argocd app list` - three commands that answer "who can do what, what did we install, what is deployed".

Türkçe homelab→platform geçiş checklist:

1. `admin.conf` sadece control-plane node'da mı? Laptop'ta değil
2. Her ekibin kendi `Role` + `RoleBinding`'i var mı (ClusterRoleBinding yok mu)
3. CI'lar ServiceAccount token ile mi çalışıyor (insan kubeconfig'si değil)
4. `kubectl auth whoami` ile kimlikler doğrulanabiliyor mu
5. Break-glass prosedürü yazılı, mühürlü ve test edilmiş mi
6. Cluster add-on'ları Helm listesinde + git'te mi
7. Uygulama değişiklikleri PR + ArgoCD ile mi (laptop'tan apply yok)
8. Quota/LimitRange her namespace'de, admission policy repo'da mı

for more [longhorn](../longhorn/)
