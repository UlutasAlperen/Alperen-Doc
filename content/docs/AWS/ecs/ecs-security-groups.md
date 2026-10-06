---
title: "ecs-security-groups"
weight: 80
---

# ECS Security Groups

Now we need [**Security Groups**](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html) to control who can reach our ECS tasks. We'll place our tasks in public subnets; only the load balancer should be able reach them. Security groups let us define exactly which traffic is allowed. We'll need two of them:

1. One for the **ECS tasks**. This will be an **internal security group** that allows traffic from an [Application Load Balancer (ALB)](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html) to our ECS tasks. A powerful feature of security groups is that they can reference _other_ security groups instead of IP addresses, so we can allow traffic from the load balancer without hardcoding IPs.
2. One for the **Load Balancer**. This will be an **external security group** that allows internet traffic to the load balancer.

Security groups referencing other security groups is one of those AWS features that seems obvious once you see it, but it's easy to miss. It's way better than managing IP addresses manually, especially when things change.

It also communicates the "intent" of the rule which makes it much easier understand as complexity grows.

![sg diagram](https://storage.googleapis.com/qvault-webapp-dynamic-assets/course_assets/uHQsCPY-1110x630.png)

## Example

**Create security groups `patientping-external` and `patientping-internal` in your `patientping` VPC** and configure traffic between them.

**Cost check:** Security groups are free. You only pay for the resources that use them.

1.  In the AWS console navigate to `EC2` → `Security Groups`.
2.  Create a `patientping-external` security group.
    -  Add an inbound rule to allow `HTTP` on port `80` from `0.0.0.0/0`.
    -  Description: "Allow HTTP traffic from the internet to the load balancer"
    -  VPC: `patientping`
3.  Create a `patientping-internal` security group.
    -  VPC: `patientping`
    -  Add an inbound rule:
        -  Type: `All traffic`
        -  Source: Custom. Click on the search bar and select your `patientping-external` security group from the dropdown.
    -  Description: "Allow traffic from the load balancer to ECS tasks"
