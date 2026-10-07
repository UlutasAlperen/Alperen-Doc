---
title: "eks-deploy-app"
weight: 30
---

# Deploy App to EKS

The cluster is up; now we run the PatientPing app on it. Nothing about this part is AWS-specific — it's plain Kubernetes — but we'll use AWS to expose the app to the internet at the end.

We need two objects:

- A **Deployment**: tells Kubernetes "keep 2 replicas of the `patientping-web` container running".
- A **Service**: gives those pods a stable address. With `type: LoadBalancer` on EKS, Kubernetes asks AWS to create an [Application Load Balancer](../../ecs/application-load-balancer/) for us — the same kind we built by hand in the ECS section.

## Assignment

**Deploy `patientping-web` to the `patientping-eks` cluster and reach it through a load balancer.**

1.  Create a file called `patientping-web.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: patientping-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: patientping-web
  template:
    metadata:
      labels:
        app: patientping-web
    spec:
      containers:
        - name: web
          image: YOUR_ECR_IMAGE_URI
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: patientping-web
spec:
  type: LoadBalancer
  selector:
    app: patientping-web
  ports:
    - port: 80
      targetPort: 8080
```

> If you don't have a pushed image yet, build and push `patientping-web` to ECR first (same steps as the [ECR lesson](../../ecs/ecr-repo/)) — EKS pulls images the same way ECS does.

2.  Apply it:

```bash
kubectl apply -f patientping-web.yaml
```

3.  Watch the rollout:

```bash
kubectl get pods -w
```

Wait until both pods are `Running` and `Ready` (it may pull the image for a minute or two).

4.  Get the load balancer's address:

```bash
kubectl get svc patientping-web
```

The `EXTERNAL-IP` column shows the ALB's DNS name once AWS has provisioned it (~2-3 minutes). Then:

```bash
curl http://EXTERNAL-IP
```

You should get the PatientPing app's response.

**Cost check:** The load balancer that Kubernetes created for you costs roughly **$0.025 per hour** plus data transfer, same as the ECS ALB. Two `t3.small` pods don't cost anything extra beyond the worker nodes you're already paying for.

> Not: `kubectl get svc`'deki EXTERNAL-IP bazen `<pending>` görünür — AWS ALB'yi arka planda oluşturuyordur, 2-3 dakika verin.

## Cleanup later

When you're done with this section, `kubectl delete -f patientping-web.yaml` removes the deployment _and_ the load balancer. Leaving an orphaned ALB around is a classic way to wake up to a surprise bill.
