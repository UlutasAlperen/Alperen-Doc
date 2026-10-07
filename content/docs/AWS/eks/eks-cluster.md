---
title: "eks-cluster"
weight: 10
---

# EKS Cluster

An EKS cluster has two halves:

- The **control plane**: the Kubernetes API server, etcd, scheduler, and controllers. AWS runs these for you, patches them, and keeps them highly available. You never SSH into it; you just get an API endpoint and a `kubeconfig`.
- The **data plane**: the worker nodes (EC2 instances) or Fargate tasks that actually run your pods. These live in _your_ VPC, in the subnets you choose.

For the worker nodes we'll use a **managed node group**: AWS creates the EC2 instances from a launch template, joins them to the cluster, and can roll updates for us. ([Fargate](https://docs.aws.amazon.com/eks/latest/userguide/fargate.html) is the serverless alternative - no nodes to think about, but less control and a different cost model.)

We'll build the cluster with [`eksctl`](https://eksctl.io/), the official CLI from the EKS team. Doing the same thing by clicking through the console takes about forty steps; `eksctl` takes one command.

## Assignment

**Create an EKS cluster named `patientping-eks` with a small managed node group in the `patientping` VPC's private subnets.**

**Cost check:** The EKS **control plane** costs about **$0.10 per hour** (~$73/month) from the moment the cluster is `Active`. The `t3.small` worker node adds a few dollars per month on top. This is the most expensive lesson in the whole document - **delete the cluster when you're done.**

1.  Install `eksctl` if you don't have it (see the [official instructions](https://eksctl.io/installation/)), and make sure your AWS CLI credentials are the `alperen-admin` user, not the root user.

2.  Write the cluster config to a file called `patientping-eks.yaml`:

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: patientping-eks
  region: us-east-1

vpc:
  id: "vpc-REPLACE_WITH_YOUR_VPC_ID"
  clusterEndpoints:
    publicAccess: true
    privateAccess: true
  subnets:
    private:
      us-east-1a: { id: "subnet-PRIVATE_A_ID" }
      us-east-1b: { id: "subnet-PRIVATE_B_ID" }

managedNodeGroups:
  - name: patientping-workers
    instanceType: t3.small
    desiredCapacity: 2
    privateNetworking: true
```

> The VPC and subnet IDs are the ones from the Networking section - the same `patientping` VPC where `patientping-db` (RDS) lives. Keeping the nodes in **private** subnets means they don't get public IPs; they reach the internet through the NAT gateway.

3.  Create the cluster:

```bash
eksctl create cluster -f patientping-eks.yaml
```

This takes roughly **10-15 minutes**. `eksctl` provisions the control plane, creates a security group, writes the `kubeconfig` entry for you, and joins the two worker nodes.

4.  Verify the cluster:

```bash
kubectl get nodes
```

You should see two nodes in `Ready` state (it can take a minute after the command finishes for them to join).

```bash
kubectl get svc
```

The `kubernetes` service in `default` should point at your new API endpoint.

> Trouble? `eksctl utils write-kubeconfig --cluster patientping-eks` refreshes your `kubeconfig`, and `aws eks update-kubeconfig --name patientping-eks` does the same thing with the AWS CLI.

**That's it - you now have a real Kubernetes cluster.** The rest of this section is just Kubernetes: manifests, secrets, and wiring up AWS services to it.
