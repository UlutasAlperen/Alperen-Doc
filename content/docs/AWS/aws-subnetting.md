---
title: "aws-subnetting"
weight: 40
---

# Subnetting

A VPC's address space is split into smaller networks called **subnets**. Each subnet lives in exactly one Availability Zone and gets its own CIDR block inside the VPC CIDR.

- **Public subnets** have a route to an [Internet Gateway](internet-gateways-igw/) — resources here can be reached from (and reach out to) the internet.
- **Private subnets** have no direct route to the internet — good for databases and internal services.

![VPC subnet layout with private and public subnets](/images/aws/vpc-subnetting-overview.png)

![Subnetting diagram](/images/aws/subnetting-diagram.png)

> **TODO:** bu sayfanın içeriği tamamlanacak (CIDR bölme, subnet maskeleri, AZ yerleşimi ve pratik örnek).
