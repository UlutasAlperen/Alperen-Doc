---
title: "ecs"
weight: 90
bookCollapseSection: true
---

# AWS - ECS - Elastic Container Service

- 001 - [vpc-and-networking-setup-for-ecs](vpc-and-networking-setup-for-ecs/)
- 002 - [why-ecs](why-ecs/)
- 003 - [elastic-container-registry](elastic-container-registry/) = Before we can deploy any containers, we need somewhere to _store their images_. Those images need to be available all the time because containers are regularly replaced and may need to pull a fresh copy of the software.
- 004 - [ecr-repo](ecr-repo/)
- 005 - [ecs-clusters](ecs-clusters/)
- 006 - [ecs-permissions](ecs-permissions/)
- 007 - [ecs-task-definitions](ecs-task-definitions/)
- 008 - [ecs-security-groups](ecs-security-groups/)

![ECS security groups diagram](/images/aws/ecs-security-groups-diagram.png)

- 009 - [application-load-balancer](application-load-balancer/) = A [load balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/what-is-load-balancing.html) sits in front of our service and distributes incoming traffic across your task(s). It handles things like health checks, SSL termination, and routing traffic to healthy containers. We'll place the load balancer in public subnets so it can receive traffic from the internet.

![Application load balancer diagram](/images/aws/application-load-balancer-diagram.png)

- 010 - [ecs-target-groups](ecs-target-groups/) = A **target group** tells the load balancer where to send traffic. It defines:
    - **Protocol and port:** What protocol (HTTP/HTTPS) and which port your tasks are listening on
    - **Health checks:** How the load balancer determines if a target is healthy
    - **Target type:** Whether targets are IP addresses (for Fargate) or instance IDs (for EC2)
- 011 - [cloudwatch-log-groups](cloudwatch-log-groups/)
- 012 - [ecs-services](ecs-services/)
