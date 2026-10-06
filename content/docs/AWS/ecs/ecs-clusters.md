---
title: "ecs-clusters"
weight: 50
---

# ECS Clusters

We've got our image, now it's time to _run_ it.

Enter the [**ECS Cluster**](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/clusters.html). Think of it as a logical grouping of compute resources where containers can run. It doesn't cost anything by itself, but it provides the foundation for running ECS tasks.

A cluster can hold a number of [**capacity providers**](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/create-capacity-provider-console-v2.html), which provide resources to our cluster on an as-needed basis. It's like the ECS version of [**EC2 Auto Scaling Groups**](https://docs.aws.amazon.com/autoscaling/ec2/userguide/auto-scaling-groups.html).

So while we could spin up EC2 instances that are ready to run our container, AWS also provides [**Fargate** and **Fargate Spot**](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-capacity-providers.html) capacity providers. **Fargate** is a "serverless" compute option provided by AWS that manages the underlying infrastructure for you.

Serverless gets a lot of hype and hate. But like many things, there's just a trade-off.

With serverless you typically pay a premium on the compute resources and lose a bit of control in exchange for making the management of those machines simpler. It makes sense for many jobs, but can be a financial catastrophe for others. _YMMV_.

AWS provides the Fargate capacity providers by default. When you configure a cluster, you can set Fargate as the default capacity provider, which means any tasks you create will automatically use Fargate unless you specify otherwise.

## Assignment

**Create an ECS cluster named `patientping-ecs` and configure it to use Fargate.**

**Cost check:** The ECS Cluster and the Fargate capacity provider don't cost anything until we start to use them.

1.  Navigate to "Elastic Container Service" in the AWS Console
2.  Click "Clusters" in the left sidebar, then click "Create Cluster"
3.  **Name**: `patientping-ecs`
4.  Leave all the defaults: we're going to use "Fargate only"
5.  Click "Create" and wait for the cluster to be created
6.  Use this command to double-check that the cluster's "capacity providers" are set to Fargate:

```sh
aws ecs describe-capacity-providers --capacity-providers FARGATE FARGATE_SPOT
```


## Tip

If you prefer to use the CLI:

```sh
aws ecs create-cluster --cluster-name "patientping-ecs"

aws ecs put-cluster-capacity-providers \
     --cluster "patientping-ecs" \
     --capacity-providers FARGATE \
     --default-capacity-provider-strategy "capacityProvider=FARGATE,weight=1"

```