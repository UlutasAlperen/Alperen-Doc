---
title: "vpc-and-networking-setup-for-ecs"
weight: 520
---

# VPC and Networking Setup

As we dive into [ECS](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html) (Elastic Container Service), I want to re-emphasize that having your networking set up correctly is critical. Luckily, we've already done it! Just know that it will be important that it's all still set up.

For ECS to work properly, we need:

1. A **[VPC](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html)** with CIDR `10.0.0.0/22` (our `patientping` VPC)
2. **Two public [subnets](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html)** (`patientping-public-a` and `patientping-public-b`)
3. An [**Internet Gateway**](https://docs.aws.amazon.com/vpc/latest/userguide/working-with-igw.html#create-igw) attached to the VPC (`patientping-igw`)
4. A [**route table**](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html) that sends `0.0.0.0/0` to the Internet Gateway, associated with both public subnets (`patientping-public-rt`)

We'll run our Fargate tasks and the load balancer in these public subnets. Tasks get outbound internet access (for example, to pull images from ECR) through the Internet Gateway, so we don't need a [NAT Gateway](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html) for this course. (In production, you'd often use private subnets and a NAT Gateway or [VPC endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/concepts.html) for stronger isolation.)