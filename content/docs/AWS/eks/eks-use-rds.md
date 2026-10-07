---
title: "eks-use-rds"
weight: 50
---

# Use RDS from EKS

Our `patientping-db` Postgres instance is sitting in a **private subnet**, exactly where it belongs. Now the pods need to reach it. Two things have to be right:

1. **Network path** — the RDS security group must allow traffic from the worker nodes on port `5432`.
2. **Credentials** — the app needs `DATABASE_URL`, and that password shouldn't live in the `Deployment` YAML (anything in a manifest ends up in git and in `kubectl get deploy -o yaml`).

Kubernetes [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) are the second half of the answer, and security groups are the first.

## Assignment

**Connect the PatientPing app on EKS to the RDS database and verify the connection.**

**Cost check:** The RDS instance is already running and billing — this lesson adds **no new AWS charges**. (If you deleted it after the RDS section, [re-create it](../../rds/create-a-postgresql-database-in-aws/) first.)

### 1. Open the network path

The worker nodes got a security group when `eksctl` created the cluster. We add _that_ group as an inbound source on the RDS security group — not the pods' IPs (which churn), and not `0.0.0.0/0` (which is how databases end up in breach reports).

1.  Find the node security group:

```bash
aws eks describe-cluster --name patientping-eks \
  --query "cluster.resourcesVpcConfig.clusterSecurityGroupId" --output text
```

2.  In the RDS console, open `patientping-db` → **Connectivity & security** → the VPC security group → **Edit inbound rules** → add:

    - **Type:** PostgreSQL (`5432`)
    - **Source:** the security group ID from the previous step

### 2. Put the connection string in a Secret

1.  Create the secret from your local shell (not from a manifest in git!):

```bash
kubectl create secret generic patientping-db-secret \
  --from-literal=DATABASE_URL='postgresql://postgres:PASSWORD@patientping-db.RANDOM-ID.us-east-1.rds.amazonaws.com:5432/patientping'
```

2.  In `patientping-web.yaml`, add the env var to the container spec:

```yaml
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: patientping-db-secret
                  key: DATABASE_URL
```

3.  Roll it out:

```bash
kubectl apply -f patientping-web.yaml
kubectl rollout restart deployment patientping-web
```

### 3. Verify

1.  Check the app's logs for a successful DB connection:

```bash
kubectl logs deploy/patientping-web --tail=50
```

2.  Or run `psql` from inside a pod, the same way we did from the EC2 instance:

```bash
kubectl exec -it deploy/patientping-web -- \
  psql "$DATABASE_URL" -c "SELECT now();"
```

If you get a timestamp back, the pod is talking to RDS. That's the whole pipeline running on EKS: pods on private subnets, scoped S3 access via IRSA, and the database wired up through a Secret.

> Not: Secret'lar base64 encode edilir ama şifrelenmez. Prod ortamlarda [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) veya [External Secrets Operator](https://external-secrets.io/) kullanın — secret'ları AWS Secrets Manager'dan çeker.

> Classic mistake: `DATABASE_URL`'i `Deployment`'a düz `value:` olarak yazmak. Manifest'i commit eden herkes artık production veritabanının şifresine sahip. Secret + `secretKeyRef` bunun için var.
