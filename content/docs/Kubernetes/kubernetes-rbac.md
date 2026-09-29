---
title: "kubernetes-rbac"
weight: 8
---
# RBAC

In the [secrets](../kubernetes-secrets/) chapter we said that secret data "can be restricted with RBAC". That was a promise we haven't paid off yet. Let's pay it off.

[RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) - Role Based Access Control - answers one question: _who_ is allowed to do _what_ to _which_ objects. It's the bouncer at the club door. Everyone with a `kubeconfig` on your laptop is a cluster-admin, which is why it never came up until now. But the moment you hand a namespace to a team, or give an app a token, the bouncer starts working.

Two halves, and they're separate on purpose:

- **Role**: a list of permissions ("can `get` and `list` pods")
- **RoleBinding**: attaches that role to a user, group, or ServiceAccount ("and _this_ app gets those permissions")

*özetlersek* Role = yetki listesi, RoleBinding = "kim" ile "yetkisi nedir" desek daha dogru olur. *Rol* tek başına kimseye yetki vermez, RoleBinding'i olmayan bir Role'de bir şey yaptıramazsin.

> RBAC is **additive and default-deny**. There are no `deny` rules at all - if no rule allows an action, it's denied. And when several rules apply, the answers are unioned: one "yes" anywhere wins. That's why giving someone `cluster-admin` "just to debug" is so hard to walk back - you can't subtract, you can only edit the rules.

# ServiceAccounts

Every request to the API server runs as _some_ identity. Humans use certificates; pods use [ServiceAccounts](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/). If you don't name one, the pod gets `default` in its namespace - which is one more reason not to leave the `default` namespace full of half-finished experiments.

Create one the imperative way:

```bash
kubectl create serviceaccount crawler-sa
```

Or in YAML:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: crawler-sa
```

A pod claims it like this:

```yaml
spec:
  serviceAccountName: crawler-sa
```

> Modern Kubernetes mounts the token as a _projected_ volume at `/var/run/secrets/kubernetes.io/serviceaccount/` - `token`, `ca.crt`, and `namespace` files. The `automountServiceAccountToken: false` option on a pod spec turns that off entirely, which is the right move for pods that never talk to the API.

Need to authenticate as that ServiceAccount outside the pod - to test, or to hand a CI job a credential? Mint a short-lived token:

```bash
kubectl create token crawler-sa -n crawler
```

> Older docs will tell you to read the token out of a `Secret` of type `kubernetes.io/serviceaccount-token`. That still works, but the projected volume plus `kubectl create token` is the modern path - the token rotates, and you never have to copy a long-lived secret around.

**Özetlersek:** ServiceAccount = pod'un kimliği. İnsalar sertifika ile, podlar SA token ile konuşur. Her namespace'te bir `default` SA var ama ona yetki yüklemek, o namespace'teki *her* pod'un aynı yetkiye sahip olması demek - her uygulamaya kendi SA'sini ver.

# Roles and ClusterRoles

A [Role](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole) is namespaced - it grants permissions inside one namespace. A `ClusterRole` grants cluster-wide, and is also what you need for non-namespaced things (nodes, persistent volumes).

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: crawler
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

The three fields that matter:

- `apiGroups`: `""` means the core group (pods, services, secrets...); `"apps"` covers deployments and statefulsets
- `resources`: what objects (`"pods"`, `"deployments"`, `"secrets"`, or `"pods/log"` for subresources)
- `verbs`: what actions - `get`, `list`, `watch`, `create`, `update`, `patch`, `delete`, and the wildcard `*`

> Verbs are exactly what they say on the tin: `list` without `get` means "you may see the names, but not the contents". `watch` is what `kubectl get -w` needs. `create` without `delete` is a fun one to hand out.

## apiGroups and subresources

The two you'll type most are `""` (core) and `apps`. But the exam likes to trip people on the rest:

| `apiGroups` | `resources` |
|---|---|
| `""` | `pods`, `services`, `configmaps`, `secrets`, `events`, `namespaces` |
| `apps` | `deployments`, `statefulsets`, `daemonsets`, `replicasets` |
| `batch` | `jobs`, `cronjobs` |
| `networking.k8s.io` | `ingresses`, `networkpolicies` |
| `rbac.authorization.k8s.io` | `roles`, `rolebindings`, `clusterroles` |

Subresources are written as `parent/child` and need their own entry in `resources`:

```yaml
resources: ["pods", "pods/log", "pods/exec"]
```

`pods/log` is the "read the output" permission. `pods/exec` is `kubectl exec` - that one is a shell inside the container, so hand it out like you'd hand out a root password.

## resourceNames

Rules can be narrowed to specific objects by name:

```yaml
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["app-config"]
    verbs: ["get"]
```

> `resourceNames` only restricts requests that name a single object: `get`, `update`, `patch`, `delete`. `list` and `watch` don't take a name, and `create` doesn't have one yet - so a rule with `resourceNames` and `verbs: ["list"]` or `["create"]` silently matches nothing. If you need "read these three configmaps", use `resourceNames` with `get`, not with `list`.

## ClusterRole

A `ClusterRole` looks identical to a `Role`, minus `metadata.namespace` - and it's the _only_ way to grant permissions on cluster-scoped resources: `nodes`, `persistentvolumes`, `namespaces`, `storageclasses`, `certificatesigningrequests`.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list"]
```

> **Dikkat:** "Role ile node yetkisi ver" bir sınav klasiğidir ve doğru cevabı yoktur. Node, PV, Namespace gibi cluster-scoped nesneler hiçbir namespace'e ait değildir; `Role` yalnızca tek namespace içinde çalışır. Bu yüzden bu tür bir görevde `ClusterRole` yazman gerekir - Role yazarsan görevi kaybedersin.

A `ClusterRole` is also handy as a _reusable template_ even for namespaced resources - more on that in the combinations table below.

## nonResourceURLs

RBAC also covers things that aren't Kubernetes objects at all - HTTP paths on the API server like `/healthz`, `/metrics`, `/version`:

```yaml
rules:
  - nonResourceURLs: ["/healthz", "/metrics"]
    verbs: ["get"]
```

Note there is no `apiGroups` and no `resources` here - `nonResourceURLs` is a different rule shape.

> **Dikkat:** nonResourceURLs kuralının çalışması için `ClusterRole` + `ClusterRoleBinding` gerekir. `Role` içine yazarsan obje belki `apply` olur ama hiçbir zaman eşleşmez - bu yollar namespace'e ait değildir. Sınavda "monitoring'e `/metrics` okuma yetkisi ver" tarzı bir görevde `RoleBinding` yazmak klasik kaybedilen puandır.

Where do these rules live in a real cluster? Kubernetes ships a few of them already: `system:public-info-viewer` grants `get` on `/healthz`, `/livez`, `/readyz`, `/version` to **everyone including anonymous**, `system:discovery` grants the API discovery paths to authenticated users, and `system:monitoring` grants `/metrics` plus health endpoints to the `system:monitoring` group. Knowing that `system:public-info-viewer` exists explains why `curl /healthz` works before you've configured any RBAC at all.

# Bindings

A [RoleBinding](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#rolebinding-and-clusterrolebinding) sticks a role to a subject:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: crawler-pod-reader
  namespace: crawler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
subjects:
  - kind: ServiceAccount
    name: crawler-sa
    namespace: crawler
```

Note the `roleRef` is not a reference you can edit in place - changing which role a binding points at means deleting and recreating the binding. That's a Kubernetes quirk, not a bug in your YAML.

A `ClusterRoleBinding` does the same with cluster-wide reach. Watch out: a `ClusterRoleBinding` to `cluster-admin` is how people accidentally give every pod in the cluster full control. Bind narrowly.

## Subjects: User, Group, ServiceAccount

`subjects` is a list, and it has three kinds. This is where most RBAC YAML goes wrong:

```yaml
subjects:
  - kind: ServiceAccount
    name: crawler-sa
    namespace: crawler # REQUIRED - the SA lives somewhere specific
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io # REQUIRED for User
  - kind: Group
    name: dev-team
    apiGroup: rbac.authorization.k8s.io # REQUIRED for Group
```

The rules:

- **ServiceAccount** — the identity of a pod. `namespace` is **not** taken from the binding, you must write it out. A binding in `crawler` pointing at `crawler-sa` without a `namespace` field is a broken binding.
- **User** — a human. Kubernetes has no `User` object; the name is whatever the authentication layer said it was. With client certs it's the certificate's `CN` field.
- **Group** — a set of users, coming from a certificate's `O` field or a token claim. Some are built in, like `system:masters` and `system:serviceaccounts:<ns>`.

And the `apiGroup` swap that trips everyone: ServiceAccount takes `apiGroup: ""` (or nothing at all), while `User` and `Group` take `rbac.authorization.k8s.io`. Get this wrong and the binding is rejected at apply time - which is at least loud about it.

> The string `system:serviceaccount:<namespace>:<name>` is the ServiceAccount's internal username. That's why `kubectl auth can-i --as=system:serviceaccount:crawler:crawler-sa` works - you're impersonating it by its full internal name.

## ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: alice-node-reader
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: node-reader
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
```

Cluster-wide reach, no `metadata.namespace` - a `ClusterRoleBinding` is not in any namespace.

## The Four Combinations

This table is the single highest-value thing to memorize. The permission _bundle_ (Role / ClusterRole) and the _binding_ (RoleBinding / ClusterRoleBinding) can be mixed - and the mix decides how far the permissions reach:

| Bundle | Binding | What the subject can do |
|---|---|---|
| `Role` | `RoleBinding` | the Role's rules, inside the binding's namespace only |
| `ClusterRole` | `RoleBinding` | the ClusterRole's rules, but **only inside the binding's namespace** |
| `ClusterRole` | `ClusterRoleBinding` | the ClusterRole's rules, everywhere (all namespaces + cluster-scoped) |
| `Role` | `ClusterRoleBinding` | **not possible** - a Role is namespace-bound and can't be granted cluster-wide |

Row two is the one that pays for itself. Write the ClusterRole once ("can read deployments"), then give every team a `RoleBinding` to it in their own namespace. One permission definition, N teams, zero overlap. That's namespace isolation without copy-pasting roles.

Row four is not a mistake you can make quietly - the API server rejects the object.

**Özetlersek:** Rol + RoleBinding = "tek namespace'te şu yetkiler". ClusterRole + RoleBinding = "aynı yetkiler ama sadece bu namespace'te" (yeniden kullanım, sınırlı erişim). ClusterRole + ClusterRoleBinding = "tüm cluster'da". Role + ClusterRoleBinding = imkansız.

# Imperative Generators

The exam is timed. YAML is clearer, but the generators get the skeleton down in seconds - and every one of them takes `--dry-run=client -o yaml`, so you can generate and then edit:

```bash
# Roles
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n crawler
kubectl create role config-reader --verb=get --resource=configmaps --resource-name=app-config -n crawler
kubectl create clusterrole node-reader --verb=get,list --resource=nodes

# Bindings
kubectl create rolebinding crawler-pod-reader \
  --role=pod-reader \
  --serviceaccount=crawler:crawler-sa -n crawler

kubectl create rolebinding alice-can-view \
  --clusterrole=view --user=alice -n crawler

kubectl create clusterrolebinding alice-node-reader \
  --clusterrole=node-reader --user=alice
```

Note the two subject spellings - `--serviceaccount=namespace:name` for generators, versus `--as=system:serviceaccount:namespace:name` for impersonation. Same identity, different syntax. Mixing them up is the fastest way to waste five minutes.

# Checking Permissions

You never have to guess who can do what:

```bash
kubectl auth can-i create deployments --namespace crawler
kubectl auth can-i list secrets --namespace crawler --as=system:serviceaccount:crawler:crawler-sa
kubectl auth can-i '*' '*' --as=system:serviceaccount:crawler:crawler-sa
```

You want `no` on that last one. `kubectl auth can-i --list` dumps the full effective rule set for the current user; add `--as=...` to see the whole rulebook for someone else, and `--namespace` to scope it:

```bash
kubectl auth can-i --list --as=system:serviceaccount:crawler:crawler-sa -n crawler
```

> Always pass `--namespace` explicitly. Without it, `can-i` checks against whatever namespace your kubeconfig context happens to be pointing at - and you'll "prove" the wrong thing.

There is also `--as-group` when you want to impersonate a group rather than a user:

```bash
kubectl auth can-i list pods -n crawler --as=alice --as-group=dev-team
```

**Özetlersek:** RBAC'i test etmek tahmin işi değildir. `kubectl auth can-i --as=<kimlik> --namespace=<ns>` her zaman yanındadır - ve `--list` ile o kimliğin tüm yetki defterini tek seferde görebilirsin.

# Troubleshooting RBAC

RBAC shows up as a symptom, rarely as an error message that says "your YAML is wrong". The usual shapes:

**Pod logunda `forbidden: User "system:serviceaccount:x:y" cannot get resource "secrets"`** — the app is missing a permission. Don't edit the app; find the gap:

```bash
kubectl auth can-i --list --as=system:serviceaccount:x:y -n x
```

**Yazdın, `apply` oldu ama hâlâ izin yok** — in order of how often it bites:

1. You created the `Role` but never the `RoleBinding`. A Role alone grants nothing.
2. `subjects[].namespace` is missing or wrong for a ServiceAccount. The binding's namespace does **not** substitute for it.
3. You bound the right role to the wrong subject - the pod is running as `default`, not the SA you wired up. Check `kubectl get pod <name> -o jsonpath='{.spec.serviceAccountName}'`.
4. You're missing `watch`. A controller that lists-and-watches works for `list` and then hangs forever on `watch`.
5. You tested it with `kubectl auth can-i` without `--as`, so you tested your own admin identity instead.

**Bir anda herkes her şeyi yapabiliyor** — look for a `ClusterRoleBinding` to `cluster-admin` or `edit`. `kubectl get clusterrolebindings` and grep for it. This is why `ClusterRole + RoleBinding` is the safer default: same role, blast radius of one namespace.

**Bir şeyi değiştirmeye çalışıyorsun ama olmadı** — `roleRef` is immutable. Delete and recreate the binding.

# Assignment

Our crawler team at SynergyChat should be able to watch their own pods and read their logs - and absolutely nothing else.

1. Create a file called `crawler-role.yaml` with a `Role` in the `crawler` namespace:

- `metadata/name`: `crawler-pod-reader`
- `metadata/namespace`: `crawler`
- `rules/apiGroups`: `[""]`
- `rules/resources`: `["pods", "pods/log"]`
- `rules/verbs`: `["get", "list", "watch"]`

2. Create a file called `crawler-rolebinding.yaml`:

- `kind`: `RoleBinding`
- `metadata/namespace`: `crawler`
- `roleRef`: `Role` named `crawler-pod-reader`
- `subjects`: the `crawler-sa` ServiceAccount (remember the `namespace` field)

3. Create the ServiceAccount if you haven't already, and wire the crawler deployment to it:

```bash
kubectl create serviceaccount crawler-sa -n crawler
```

```yaml
spec:
  serviceAccountName: crawler-sa
```

4. Apply everything and test the boundaries:

```bash
kubectl apply -f crawler-role.yaml -f crawler-rolebinding.yaml
kubectl auth can-i list pods -n crawler --as=system:serviceaccount:crawler:crawler-sa
kubectl auth can-i delete pods -n crawler --as=system:serviceaccount:crawler:crawler-sa
kubectl auth can-i get secrets -n crawler --as=system:serviceaccount:crawler:crawler-sa
```

You want `yes`, `no`, `no`. That asymmetry is the entire point of RBAC - the crawler can see itself, but can't touch its neighbors or read the secrets it doesn't need.

5. Now for the part exams actually ask. The ops team (user `alice`) needs to `get` and `list` nodes - cluster-wide, because nodes aren't in a namespace. Write `node-reader` as a `ClusterRole`, bind it to `alice` with a `ClusterRoleBinding`, and prove it:

```bash
kubectl auth can-i list nodes --as=alice           # yes
kubectl auth can-i delete nodes --as=alice          # no
```

6. Bonus, and the pattern worth keeping: the crawler team wants to read `configmaps` in their own namespace using a `ClusterRole` as a template. Create a `ClusterRole` named `cm-reader` that allows `get`/`list` on `configmaps`, then bind it to `crawler-sa` with a **`RoleBinding`** (not a `ClusterRoleBinding`) in `crawler`. Prove that the same SA still cannot read configmaps in `kube-system`. That gap is exactly what makes row two of the combinations table worth knowing.

> **Dikkat:** `kubectl auth can-i` ile test ettiğin kimlik, `--as` vermezsen kendi kimliğin oldugundan "ben görebiliyorum ama pod göremiyor" probleminin ana sebebidir - her zaman pod'un ServiceAccount'ı ile test et.

for more [jobs-cronjobs](../kubernetes-jobs-cronjobs/)
