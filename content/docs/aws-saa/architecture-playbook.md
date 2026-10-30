---
title: "architecture-playbook"
weight: 90
---

# AWS SAA-C03 Architecture Playbook: 5 Golden Reference Blueprints

Almost every scenario question on the SAA-C03 exam is a variation of **five reference architectures**. When you read a long 5-sentence question, map it to one of these blueprints first, then find the missing or broken link.

---

## 1. The Multi-AZ 3-Tier Web Architecture (HA + Scalable + Secure)

Used when: *"A company runs a traditional web application on EC2 (Elastic Compute Cloud) and RDS (Relational Database Service) that must survive an AZ (Availability Zone) outage, scale with traffic, block SQL injection, and offload static assets."*

```mermaid
flowchart TB
    user[Global Users] --> r53[Route 53 + ACM TLS]
    r53 --> waf[AWS WAF + Shield]
    waf --> cf[CloudFront CDN]
    cf -->|static assets + OAC| s3[(S3 Private Bucket)]
    cf -->|dynamic /api/*| alb[ALB in Public Subnets]
    alb --> asg[EC2 ASG in Private Subnets across 2+ AZs]
    asg -->|session state & query cache| redis[ElastiCache Redis Multi-AZ]
    asg -->|writes| rds[(RDS / Aurora Primary)]
    rds -.sync replication.-> stby[(Multi-AZ Standby)]
    rds -.async replication.-> rr[(Read Replicas)]
    asg -->|reads| rr
```

### How exam questions mutate this blueprint:
- **"Users lose their shopping cart when an EC2 instance terminates"** → Move session state out of EC2 memory into **ElastiCache for Redis** or **DynamoDB** (avoid sticky sessions).
- **"Database CPU hits 100% due to identical product catalog queries"** → Add **ElastiCache** (or **Read Replicas** if queries are dynamic reporting).
- **"Prevent users from bypassing CloudFront to hit the ALB directly"** → Have CloudFront add a custom secret HTTP header (stored in Secrets Manager) and configure an **ALB listener rule** to drop requests missing that header (or use the AWS-managed CloudFront prefix list in the ALB Security Group).
- **"Security Group Chain"**: `ALB SG` allows `80/443` from `0.0.0.0/0` → `App EC2 SG` allows `8080` **only from `ALB SG`** → `RDS SG` allows `3306/5432` **only from `App EC2 SG`**.

---

## 2. The Serverless Microservices Blueprint (Zero Server Management)

Used when: *"Spiky/unpredictable traffic, minimal operational overhead, no idle compute cost, mobile/web authentication."*

```mermaid
flowchart LR
    client[Web / Mobile App] -->|1. Sign in| up[Cognito User Pool]
    client -->|2. Static UI| cf[CloudFront + S3 OAC]
    client -->|3. API + JWT| apigw[API Gateway]
    apigw --> lambda[AWS Lambda]
    lambda -->|NoSQL single-digit ms| ddb[(DynamoDB + DAX)]
    lambda -->|Relational SQL| proxy[RDS Proxy] --> aurora[(Aurora Serverless v2)]
    lambda -->|Secrets| sm[Secrets Manager]
```

### How exam questions mutate this blueprint:
- **"Lambda exhausts RDS connections during traffic spikes"** → Insert **Amazon RDS Proxy**.
- **"Mobile users must upload 2 GB videos directly without hitting API Gateway's 10 MB payload limit"** → Lambda generates an **S3 Pre-Signed URL**; the client uploads **directly to S3**!
- **"Need global active-active with sub-second cross-region replication"** → **Route 53 Latency Routing + API Gateway + Lambda + DynamoDB Global Tables** (or **Aurora Global Database**).

---

## 3. The Event-Driven Decoupled Worker Blueprint (Fault Tolerance & Spikes)

Used when: *"Orders or media uploads must never be lost if worker nodes crash or traffic spikes 10x for an hour."*

```mermaid
flowchart LR
    prod[Web Tier / S3 Upload] --> sns[SNS Topic: Fan-Out]
    sns --> sqs1[SQS Queue: Image Processing]
    sns --> sqs2[SQS Queue: Billing]
    sqs1 -->|failed 3x| dlq[(SQS DLQ)]
    sqs1 -->|polled by| asg[EC2 Spot/On-Demand ASG]
    cw[CloudWatch: Backlog Per Instance] -.scales.-> asg
```

### How exam questions mutate this blueprint:
- **"How should the Auto Scaling Group scale properly?"** → Never scale queue workers on CPU alone! Calculate a **Backlog Per Instance** metric (`ApproximateNumberOfMessagesVisible / RunningInstanceCount`) and use a **Target Tracking** policy.
- **"A poison-pill message causes workers to crash repeatedly in a loop"** → Configure a **Dead-Letter Queue (DLQ)** with `maxReceiveCount` (e.g., 3–5) and alarm on DLQ depth.
- **"Workers process the same message twice because the job takes 8 minutes"** → Increase the queue's **Visibility Timeout** (default is 30 seconds) or call `ChangeMessageVisibility` dynamically.

---

## 4. The Modern Data Lake & Real-Time Analytics Blueprint

Used when: *"Ingest streaming clickstream/IoT data, store cheaply, run ad-hoc SQL queries, and visualize in BI dashboards without managing servers."*

```mermaid
flowchart LR
    iot[Streaming Producers] --> kds[Kinesis Data Streams]
    kds -->|real-time windowing| flink[Managed Apache Flink]
    kds --> fh[Amazon Data Firehose]
    fh -->|convert JSON to Parquet| s3[(S3 Data Lake)]
    s3 --> glue[AWS Glue Crawler & Catalog]
    glue --> athena[Amazon Athena: Serverless SQL]
    athena --> qs[Amazon QuickSight: BI Dashboard]
    s3 -->|petabyte OLAP joins| rs[(Amazon Redshift / Spectrum)]
```

### How exam questions mutate this blueprint:
- **"Athena queries are slow and expensive"** → Convert raw CSV/JSON into **columnar Apache Parquet or ORC** (via Data Firehose or AWS Glue) and **partition S3 prefixes by date** (`s3://bucket/year=2026/month=10/`).
- **"Discover PII (credit cards, SSNs) in the S3 Data Lake automatically"** → **Amazon Macie**.
- **"Centralize fine-grained column/row-level permissions across the S3 Data Lake"** → **AWS Lake Formation**.

---

## 5. The Enterprise Hybrid & Multi-VPC Network Blueprint

Used when: *"Connect 20 VPCs (Virtual Private Cloud) and an on-premises datacenter with private connectivity, resilient failover, and private access to AWS services."*

```mermaid
flowchart TB
    dc[On-Premises Datacenter]
    dc ===|Primary: private fiber| dx[AWS Direct Connect]
    dc -.Backup: IPsec over internet.-> vpn[Site-to-Site VPN]
    dx --> dxgw[Direct Connect Gateway]
    dxgw & vpn --> tgw[AWS Transit Gateway]
    tgw --- vpc_a[VPC A: Prod]
    tgw --- vpc_b[VPC B: Shared Services]
    tgw --- vpc_c[VPC C: Egress Inspection + Network Firewall]
    vpc_a -->|Gateway Endpoint: free| s3[(Amazon S3)]
    dc -->|PrivateLink Interface Endpoint| vpce[S3 Interface Endpoint in VPC B] --> s3
```

### How exam questions mutate this blueprint:
- **"On-premises servers need to access S3 privately over Direct Connect / VPN"** → **S3 Gateway Endpoints do NOT work from on-premises!** You must create an **S3 Interface Endpoint (PrivateLink)** inside the VPC.
- **"Resolve AWS Private Hosted Zone DNS names from on-premises (and vice versa)"** → Deploy **Route 53 Resolver Inbound Endpoints** (on-prem → AWS) and **Outbound Endpoints + Forwarding Rules** (AWS → on-prem).
- **"Private EC2 instances in 3 AZs download 100 TB/month from S3 and NAT Gateway costs are huge"** → Add a free **S3 Gateway VPC Endpoint** to the private route tables.
