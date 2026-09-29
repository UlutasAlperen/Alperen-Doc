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

# Checking Permissions

You never have to guess who can do what:

```bash
kubectl auth can-i create deployments --namespace crawler
kubectl auth can-i list secrets --as=system:serviceaccount:crawler:crawler-sa
kubectl auth can-i '*' '*' --as=system:serviceaccount:crawler:crawler-sa
```

You want `no` on that last one. `kubectl auth can-i --list` dumps the full effective rule set for the current user.

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
- `subjects`: the `crawler-sa` ServiceAccount

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

> **Dikkat:** `kubectl auth can-i` ile test ettiğin kimlik, `--as` vermezsen kendi kimliğin oldugundan "ben görebiliyorum ama pod göremiyor" probleminin ana sebebidir  - her zaman pod'un ServiceAccount'ı ile test et.

for more [jobs-cronjobs](../kubernetes-jobs-cronjobs/)
