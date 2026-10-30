---
title: "AWS SAA-C03"
weight: 7
bookCollapseSection: true
---

# AWS SAA-C03 - Solutions Architect Associate

> Claimer: these notes are built around the official [SAA-C03 exam guide](https://d1.awsstatic.com/training-and-certification/docs-sa-assoc/AWS-Certified-Solutions-Architect-Associate_Exam-Guide.pdf), skillbuilder.aws and exam-focused question banks. The hands-on "how do I click this in the console" parts live in my [AWS](../aws/) notes - this section is the **exam theory**: what to choose, when, and why.

[AWS Certified Solutions Architect - Associate](https://aws.amazon.com/certification/certified-solutions-architect-associate/) is the "bread and butter" AWS certification. It does **not** test how fast you can click through the console - it tests whether you can look at a business requirement and pick the right architecture from a pile of all-valid-but-not-all-correct answers.

## Exam facts

- **SAA-C03** - 65 questions (50 scored, 15 unscored/pilot), **130 minutes**
- Score range **1-1000**, you need **720** to pass
- Question types: multiple choice and multiple response (pick 2-3)
- **No penalty for wrong answers** - never leave a question blank
- Valid for **3 years**
- Domains and weights:

| Domain | Weight |
|---|---|
| 1. Design Secure Architectures | 24% |
| 2. Design Resilient Architectures | 30% |
| 3. Design High-Performing Architectures | 24% |
| 4. Design Cost-Optimized Architectures | 22% |

## How I use these notes

1. Every topic page explains **what the service is, when to pick it, and what the exam loves to trick you with**.
2. Each page ends with an **Exam check** (mini questions with answers) and an **Ozet** (one-line "X = ..." summaries).
3. The [general-knowledge](general-knowledge/) page collects **all** those one-liners in one place - my cheat sheet before the exam.
4. Real labs (VPC (Virtual Private Cloud), EC2 (Elastic Compute Cloud), RDS (Relational Database Service), ECS (Elastic Container Service), EKS (Elastic Kubernetes Service)...) are in the [AWS](../aws/) section - go there when you want to build it, come here when you want to pass it.

## AWS SAA-C03 konular

### [Domain 1 - Design Secure Architectures](secure-architectures/)

1. [iam-policies-roles-cross-account](secure-architectures/iam-policies-roles-cross-account/)
2. [identity-federation-and-cognito](secure-architectures/identity-federation-and-cognito/)
3. [kms-encryption-deep-dive](secure-architectures/kms-encryption-deep-dive/)
4. [secrets-manager-vs-parameter-store](secure-architectures/secrets-manager-vs-parameter-store/)
5. [s3-security-and-access-points](secure-architectures/s3-security-and-access-points/)
6. [network-security-sg-nacl-vpc-endpoints](secure-architectures/network-security-sg-nacl-vpc-endpoints/)
7. [security-services-waf-guardduty-inspector](secure-architectures/security-services-waf-guardduty-inspector/)
8. [governance-monitoring-organizations-config-ssm](secure-architectures/governance-monitoring-organizations-config-ssm/)

### [Domain 2 - Design Resilient Architectures](resilient-architectures/)

1. [ha-vs-fault-tolerance](resilient-architectures/ha-vs-fault-tolerance/)
2. [disaster-recovery-strategies](resilient-architectures/disaster-recovery-strategies/)
3. [elb-and-auto-scaling-resilience](resilient-architectures/elb-and-auto-scaling-resilience/)
4. [decoupling-with-sqs-sns](resilient-architectures/decoupling-with-sqs-sns/)
5. [event-driven-eventbridge-step-functions](resilient-architectures/event-driven-eventbridge-step-functions/)
6. [streaming-kinesis-data-streams-firehose](resilient-architectures/streaming-kinesis-data-streams-firehose/)
7. [rds-and-aurora-failover](resilient-architectures/rds-and-aurora-failover/)
8. [route53-routing-policies-and-failover](resilient-architectures/route53-routing-policies-and-failover/)
9. [vpc-connectivity-peering-tgw-direct-connect](resilient-architectures/vpc-connectivity-peering-tgw-direct-connect/)

### [Domain 3 - Design High-Performing Architectures](high-performing-architectures/)

1. [s3-storage-and-performance](high-performing-architectures/s3-storage-and-performance/)
2. [block-and-file-storage-ebs-efs-fsx](high-performing-architectures/block-and-file-storage-ebs-efs-fsx/)
3. [caching-elasticache-cloudfront](high-performing-architectures/caching-elasticache-cloudfront/)
4. [database-selection-matrix](high-performing-architectures/database-selection-matrix/)
5. [dynamodb-deep-dive](high-performing-architectures/dynamodb-deep-dive/)
6. [compute-performance-ec2-options](high-performing-architectures/compute-performance-ec2-options/)
7. [serverless-and-containers-lambda-ecs-eks](high-performing-architectures/serverless-and-containers-lambda-ecs-eks/)
8. [data-transfer-and-hybrid-services](high-performing-architectures/data-transfer-and-hybrid-services/)

### [Domain 4 - Design Cost-Optimized Architectures](cost-optimized-architectures/)

1. [pricing-models-ri-savings-plans-spot](cost-optimized-architectures/pricing-models-ri-savings-plans-spot/)
2. [s3-cost-and-lifecycle-policies](cost-optimized-architectures/s3-cost-and-lifecycle-policies/)
3. [cost-management-tools](cost-management-tools/)
4. [cost-optimized-architecture-patterns](cost-optimized-architecture-patterns/)

### [Architecture Playbook - 5 Golden Blueprints](architecture-playbook/)

### [General knowledge - cheat sheet](general-knowledge/)

### [Practice Exam - 20 Realistic Scenario Questions](practice-exam/)

> Not: burasi sinav oncesi son tekrar ve deneme bolumu. `architecture-playbook` en cok cikan 5 mimari sablonu, `general-knowledge` her servisi tek satirda ozetleyen cheat sheet'i, `practice-exam` ise gercek sinav zorlugunda 20 senaryo sorusunu icerir.
