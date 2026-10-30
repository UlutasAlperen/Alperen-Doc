---
title: "practice-exam"
weight: 110
---

# AWS SAA-C03 Practice Exam: 20 Realistic Scenario Questions

> **How to use this page:** Every question below is written in the real SAA-C03 style — multiple plausible options where a single requirement keyword (*"least operational overhead"*, *"most cost-effective"*, *"without traversing the public internet"*, *"RPO under 1 second"*) decides the winner. Includes **Multiple Response (Choose TWO)** questions.

---

### Question 1 (Domain 1 - Security & Networking)
A company has established an AWS Direct Connect connection between its on-premises data center and a VPC (Virtual Private Cloud). Applications running on-premises must upload sensitive files to an Amazon S3 (Simple Storage Service) bucket privately without traffic traversing the public internet, and without deploying proxy servers on EC2 (Elastic Compute Cloud). What should the solutions architect configure?

- **A)** Create a VPC Gateway Endpoint for Amazon S3 and update the VPC route table
- **B)** Create an Amazon S3 Interface VPC Endpoint (AWS PrivateLink) in the VPC and route on-premises S3 traffic to it
- **C)** Attach an Internet Gateway to the VPC and use an S3 Bucket Policy with `aws:SourceIp`
- **D)** Configure S3 Transfer Acceleration on the bucket

{{% details "Cevabı göster" %}}
**Correct Answer: B**
- **Why B is correct:** VPC **Interface Endpoints** (powered by PrivateLink) create ENIs with private IPs inside your subnets, which are directly reachable from on-premises networks over Direct Connect or Site-to-Site VPN.
- **Why A is wrong (Classic Exam Trap):** S3 **Gateway Endpoints** only add a prefix-list route for resources *inside* the VPC; they **cannot** be routed to from on-premises VPN, Direct Connect, or peered VPCs.
- **Why C & D are wrong:** Both traverse the public internet.
{{% /details %}}

---

### Question 2 (Domain 2 - Resilience & Serverless)
An e-commerce platform uses AWS Lambda functions inside a VPC to process checkout requests and write transactions to an Amazon RDS (Relational Database Service) for PostgreSQL Multi-AZ DB instance. During promotional events, sudden traffic spikes cause thousands of concurrent Lambda executions, overwhelming the database with new connection requests and causing timeouts. Which solution resolves this with the LEAST code and architectural change?

- **A)** Create three RDS Read Replicas and point the Lambda checkout writes to the replica endpoints
- **B)** Deploy Amazon RDS Proxy in the VPC and update the Lambda function to connect to the proxy endpoint
- **C)** Migrate the PostgreSQL database to Amazon DynamoDB and rewrite all SQL queries
- **D)** Put Amazon ElastiCache for Memcached in front of the RDS database for write operations

{{% details "Cevabı göster" %}}
**Correct Answer: B**
- **Why B is correct:** **RDS Proxy** pools and shares established database connections across thousands of ephemeral Lambda containers, preventing connection exhaustion and cutting failover times by up to 66% with just an endpoint change.
- **Why A is wrong:** Read Replicas only accept *reads*, not checkout *writes*.
- **Why C is wrong:** Huge rewrite effort (violates "LEAST code change").
- **Why D is wrong:** Memcached caches reads, it does not pool relational write connections.
{{% /details %}}

---

### Question 3 (Domain 1 - Storage Governance)
A financial institution stores audit logs in an Amazon S3 bucket. Regulatory compliance requires that once a log file is written, it must not be overwritten or deleted by **any** user — including the AWS account root user — for a period of 7 years. After 7 years, the logs should be transitionable or deletable. Which configuration meets this requirement?

- **A)** Enable S3 Object Lock in **Governance mode** with a retention period of 7 years
- **B)** Enable S3 Object Lock in **Compliance mode** with a retention period of 7 years
- **C)** Attach an S3 Bucket Policy with `Effect: Deny` on `s3:DeleteObject` for all IAM (Identity and Access Management) users
- **D)** Enable S3 Versioning and MFA (Multi-Factor Authentication) Delete on the bucket

{{% details "Cevabı göster" %}}
**Correct Answer: B**
- **Why B is correct:** S3 Object Lock **Compliance mode** guarantees WORM (Write Once, Read Many) protection where **no one, including the AWS account root user**, can delete the object or shorten the retention period until it expires.
- **Why A is wrong:** In *Governance mode*, principals with `s3:BypassGovernanceRetention` (which the root user can grant) can still override retention.
- **Why C & D are wrong:** The root user can always delete a bucket policy or use the root MFA device to disable MFA Delete.
{{% /details %}}

---

### Question 4 (Domain 3 - High-Performing Storage)
A genomics research company stores 200 TB of raw sequencing files in an Amazon S3 bucket. Every Friday, they spin up a cluster of 64 compute-optimized EC2 Linux instances for 4 hours to run parallel analysis over this dataset. The instances need sub-millisecond latency, hundreds of GB/s of aggregate throughput, and POSIX file system access to the S3 objects. If the storage layer fails during the 4-hour window, the job can be re-run. What is the MOST cost-effective solution?

- **A)** Amazon EFS (Elastic File System) Standard with Provisioned Throughput
- **B)** Amazon FSx for Lustre **Scratch** file system linked to the S3 bucket
- **C)** Amazon FSx for Lustre **Persistent** file system linked to the S3 bucket
- **D)** Copy the 200 TB from S3 onto EBS (Elastic Block Store) `io2` Block Express volumes attached to each instance

{{% details "Cevabı göster" %}}
**Correct Answer: B**
- **Why B is correct:** **FSx for Lustre** provides native, high-speed POSIX access backed directly by an S3 data repository (hundreds of GB/s, sub-ms latency). Because the job is temporary (4 hours) and fault-tolerant, a **Scratch** deployment is significantly cheaper than Persistent.
- **Why A is wrong:** EFS does not natively lazy-load objects from S3 and offers lower throughput than Lustre for HPC workloads.
- **Why C is wrong:** Persistent Lustre is more expensive and unnecessary when the job can simply be restarted.
{{% /details %}}

---

### Question 5 (Domain 2 - Event-Driven & Streaming) — *Choose TWO*
A media company ingests real-time clickstream events from its website. The data must be analyzed in real time using SQL window queries to detect trending articles within 10 seconds, AND simultaneously delivered in Apache Parquet format to an Amazon S3 data lake with a 1-minute buffer and minimal operational overhead. Which TWO services should be combined to build this pipeline? **(Choose TWO)**

- **A)** Amazon Kinesis Data Streams + Amazon Managed Service for Apache Flink
- **B)** Amazon Data Firehose with built-in JSON-to-Parquet conversion delivering to S3
- **C)** Amazon SQS (Simple Queue Service) Standard queue + AWS Batch
- **D)** AWS Database Migration Service (DMS)
- **E)** AWS DataSync scheduled every minute

{{% details "Cevabı göster" %}}
**Correct Answers: A and B**
- **Why A & B are correct:** **Kinesis Data Streams + Managed Apache Flink** handles the real-time (10-second window) streaming analytics (A), while **Amazon Data Firehose** reads from the same Kinesis stream, buffers for 60 seconds, converts JSON to Parquet out-of-the-box, and writes directly to S3 (B).
- **Why C, D, E are wrong:** SQS does not support streaming SQL window analytics or multiple independent stream consumers; DMS is for databases; DataSync is for file systems.
{{% /details %}}

---

### Question 6 (Domain 2 - Decoupling & Auto Scaling)
An application processes video transcoding jobs placed in an Amazon SQS Standard queue by a fleet of EC2 instances in an Auto Scaling Group (ASG). Each video takes between 3 and 8 minutes to transcode. Currently, users complain that some videos are transcoded twice, and when a corrupted video file arrives, workers crash repeatedly trying to process it forever. Which TWO actions will fix both issues? **(Choose TWO)**

- **A)** Increase the SQS queue **Visibility Timeout** to at least 10 minutes (or extend it dynamically during processing)
- **B)** Configure a **Dead-Letter Queue (DLQ)** redrive policy on the SQS queue with a `maxReceiveCount` of 3
- **C)** Switch from Long Polling to Short Polling
- **D)** Increase the SQS message retention period to 14 days
- **E)** Put an Application Load Balancer in front of the SQS queue

{{% details "Cevabı göster" %}}
**Correct Answers: A and B**
- **Why A is correct:** The default SQS visibility timeout is 30 seconds. Since jobs take 3–8 minutes, the message becomes visible again before the first worker finishes, causing duplicate processing. Increasing the visibility timeout (A) prevents this.
- **Why B is correct:** A **DLQ** with `maxReceiveCount` isolates poison-pill/corrupted messages after N failed attempts so workers don't loop infinitely.
{{% /details %}}

---

### Question 7 (Domain 4 - Cost-Optimized Networking)
A company runs a data-processing fleet on EC2 instances in private subnets across three Availability Zones. The instances download and upload 80 TB of data per month to Amazon S3 and Amazon DynamoDB in the same region. A monthly billing review shows very high **NAT Gateway data processing charges**. How can the solutions architect eliminate these data processing costs for S3 and DynamoDB traffic?

- **A)** Replace the NAT Gateways with an AWS Transit Gateway
- **B)** Create **VPC Gateway Endpoints** for S3 and DynamoDB and add them to the private subnets' route tables
- **C)** Create VPC Interface Endpoints (PrivateLink) for S3 and DynamoDB
- **D)** Move the EC2 instances into public subnets and use VPN connections

{{% details "Cevabı göster" %}}
**Correct Answer: B**
- **Why B is correct:** **Gateway VPC Endpoints** for S3 and DynamoDB are completely **free** (no hourly fee, no per-GB data processing fee) and route traffic privately within the AWS network via the route table, bypassing the NAT Gateway.
- **Why C is wrong:** Interface Endpoints cost per hour and per GB processed.
{{% /details %}}

---

### Question 8 (Domain 1 - Security & Operations)
A company runs 150 EC2 instances in private subnets across multiple AWS accounts. Security compliance strictly prohibits opening inbound SSH (port 22) or RDP (port 3389) in any Security Group, forbids deploying bastion hosts, and requires every command typed by an administrator during a troubleshooting session to be logged to an encrypted Amazon S3 bucket. What should the architect recommend?

- **A)** Use EC2 Instance Connect over an Internet Gateway
- **B)** Configure **AWS Systems Manager Session Manager** with an IAM instance profile and enable session logging to S3
- **C)** Deploy a Network Load Balancer forwarding port 22 to the private instances
- **D)** Use AWS Config to run shell commands on the instances

{{% details "Cevabı göster" %}}
**Correct Answer: B**
- **Why B is correct:** **SSM Session Manager** uses the outbound HTTPS connection from the SSM Agent (via NAT or SSM VPC Interface Endpoints) — requiring **zero inbound ports, zero SSH keys, and zero bastion hosts** — and natively logs session transcripts to S3 and CloudWatch Logs.
{{% /details %}}

---

### Question 9 (Domain 2 - Disaster Recovery & Databases)
A mission-critical financial application uses Amazon Aurora PostgreSQL in `us-east-1`. The business requires a cross-region disaster recovery solution in `us-west-2` with an **RPO (Recovery Point Objective) of less than 1 second** and an **RTO (Recovery Time Objective) of less than 1 minute**. Which architecture satisfies these requirements?

- **A)** Enable RDS automated backups and use AWS Backup to copy daily snapshots to `us-west-2`
- **B)** Configure **Amazon Aurora Global Database** with a secondary cluster in `us-west-2` and Route 53 failover routing
- **C)** Use Aurora Backtrack with a 24-hour window
- **D)** Create a standard RDS MySQL Multi-AZ deployment in `us-east-1`

{{% details "Cevabı göster" %}}
**Correct Answer: B**
- **Why B is correct:** **Aurora Global Database** uses dedicated storage-layer replication across regions with typical lag **under 1 second (RPO < 1s)** and can promote a secondary region to full read/write in **under 1 minute (RTO < 1m)**.
- **Why A is wrong:** Daily snapshots have an RPO of up to 24 hours and take hours to restore.
- **Why C & D are wrong:** Backtrack and Multi-AZ are single-region features and do not survive a full regional outage.
{{% /details %}}

---

### Question 10 (Domain 3 - Networking & Load Balancing)
A gaming company hosts a multiplayer backend on EC2 instances behind a load balancer. The application uses **UDP and TCP** protocols, requires **ultra-low latency** for millions of packets per second, and partners require **one static IP address per Availability Zone** so they can whitelist the service in their corporate firewalls. Which solution meets these requirements?

- **A)** Application Load Balancer (ALB) with Elastic IPs attached
- **B)** **Network Load Balancer (NLB)** with Elastic IPs assigned per Availability Zone
- **C)** Gateway Load Balancer (GWLB)
- **D)** Amazon CloudFront with an S3 origin

{{% details "Cevabı göster" %}}
**Correct Answer: B**
- **Why B is correct:** **NLB** operates at Layer 4 (TCP/UDP/TLS), handles millions of requests per second with ultra-low latency, and natively supports **static / Elastic IPs per AZ**.
- **Why A is wrong:** ALB only supports Layer 7 (HTTP/HTTPS/gRPC) and cannot have static/Elastic IPs attached directly (without placing an NLB or Global Accelerator in front).
{{% /details %}}

---

### Question 11 (Domain 3 - S3 Replication & Compliance)
A company has an existing S3 bucket containing 50 million objects (40 TB) in `eu-central-1`. Due to a new disaster recovery policy, the architect enables S3 Versioning and configures **S3 Cross-Region Replication (CRR)** to a new bucket in `eu-west-1`. The next day, the team notices that newly uploaded objects appear in `eu-west-1`, but none of the original 50 million objects are in the destination bucket. Why did this happen, and how should it be resolved?

- **A)** CRR only replicates new objects after the rule is created; run an **S3 Batch Replication** job to replicate the existing objects
- **B)** CRR requires S3 Transfer Acceleration to replicate existing objects
- **C)** Versioning must be disabled on the destination bucket for historical objects to copy
- **D)** Existing objects take 30 days to transition via S3 Lifecycle rules before CRR starts

{{% details "Cevabı göster" %}}
**Correct Answer: A**
- **Why A is correct (Classic Exam Trap):** By design, S3 CRR and SRR (Same-Region Replication) **only replicate new or updated objects** written *after* replication is enabled. To replicate objects that already existed before the rule, you must run **S3 Batch Replication**.
{{% /details %}}

---

### Question 12 (Domain 1 - Encryption & KMS Cross-Account)
Account A has an Amazon S3 bucket encrypted with a **Customer Managed Key (CMK)** in AWS KMS (Key Management Service). An IAM role in Account B needs to download and decrypt objects from this bucket. The architect has already updated the S3 Bucket Policy in Account A to allow the IAM role in Account B. What ELSE must be configured for the download to succeed?

- **A)** Nothing else; the S3 Bucket Policy automatically delegates KMS decryption permissions
- **B)** Update the **KMS Key Policy** in Account A to allow Account B, AND ensure the **IAM Role Policy** in Account B allows `kms:Decrypt` on Account A's CMK
- **C)** Switch the bucket encryption to SSE-S3 because SSE-KMS does not support cross-account access
- **D)** Export the KMS key material from Account A and import it into Account B

{{% details "Cevabı göster" %}}
**Correct Answer: B**
- **Why B is correct:** Cross-account access to SSE-KMS objects requires **four** permissions: (1) S3 Bucket Policy in Account A, (2) KMS Key Policy in Account A allowing Account B, and (3 & 4) IAM Role Policy in Account B allowing both `s3:GetObject` AND `kms:Decrypt` on the CMK ARN.
- **Why C is wrong:** Default AWS-managed keys (`aws/s3`) cannot have their key policy edited for cross-account sharing, which is why a **Customer Managed Key (CMK)** is required!
{{% /details %}}

---

### Question 13 (Domain 4 - Compute Pricing & Cost Optimization)
A company runs a steady baseline of containerized web services on **AWS Fargate** in `us-east-1`, several **AWS Lambda** functions, and a fleet of `m6i.large` **EC2** instances that the team plans to migrate to Graviton (`m7g.large`) in `us-west-2` in six months. They want to commit to a 3-year spend to maximize discounts while keeping the flexibility to change instance families, regions, and compute services automatically. Which purchasing option should they choose?

- **A)** EC2 Instance Savings Plans
- **B)** **Compute Savings Plans**
- **C)** Standard Reserved Instances
- **D)** Convertible Reserved Instances

{{% details "Cevabı göster" %}}
**Correct Answer: B**
- **Why B is correct:** **Compute Savings Plans** provide up to 66% discount and automatically apply across **EC2 (any instance family, size, OS, tenancy, or Region), AWS Fargate, and AWS Lambda**.
- **Why A & C & D are wrong:** EC2 Instance Savings Plans and Reserved Instances are locked to a specific Region and do **not** cover AWS Fargate or AWS Lambda usage.
{{% /details %}}

---

### Question 14 (Domain 2 - Hybrid DNS)
A company has connected its on-premises network to an AWS VPC via AWS Direct Connect. They use an Amazon Route 53 Private Hosted Zone (`aws.internal.example.com`) for resources inside the VPC, and an on-premises DNS server for `onprem.example.com`. Servers on-premises need to resolve `aws.internal.example.com`, and EC2 instances in the VPC need to resolve `onprem.example.com`. What should the solutions architect configure?

- **A)** Make the Route 53 Hosted Zone public
- **B)** Create a **Route 53 Resolver Inbound Endpoint** (for on-premises → AWS queries) and a **Route 53 Resolver Outbound Endpoint** with forwarding rules (for AWS → on-premises queries)
- **C)** Enable `enableDnsHostnames` only and create CNAME records in S3
- **D)** Deploy an Application Load Balancer in front of Route 53

{{% details "Cevabı göster" %}}
**Correct Answer: B**
- **Why B is correct:**
  - **Inbound Endpoint**: gives on-premises DNS servers private IP addresses inside the VPC to forward queries *into* Route 53 Resolver.
  - **Outbound Endpoint + Forwarding Rule**: allows Route 53 Resolver inside the VPC to forward specific domain queries (`onprem.example.com`) *out* to on-premises DNS servers.
{{% /details %}}

---

### Question 15 (Domain 3 - Caching & CloudFront Security)
A media company distributes paid training videos stored in an Amazon S3 bucket through Amazon CloudFront. Only paying subscribers who log into the web application should be able to stream a set of 15 video files (HLS segments) for a course during their 2-hour session, without modifying the individual video URLs in the video player playlist. How should this be implemented?

- **A)** Configure **CloudFront Origin Access Control (OAC)** on the S3 bucket and issue **CloudFront Signed Cookies** to authenticated subscribers
- **B)** Issue S3 Pre-Signed URLs for every individual video segment
- **C)** Make the S3 bucket public and restrict access using a NACL (Network Access Control List)
- **D)** Use Cognito User Pools attached directly to the S3 bucket

{{% details "Cevabı göster" %}}
**Correct Answer: A**
- **Why A is correct:**
  - **OAC** keeps the S3 bucket completely private so users must go through CloudFront.
  - **CloudFront Signed Cookies** grant time-limited access to **multiple files** (e.g., all `.ts` HLS video segments under `/courses/101/*`) without changing the URLs inside the playlist! (Use **Signed URLs** for a *single* file; use **Signed Cookies** for *many* files).
{{% /details %}}

---

### Question 16 (Domain 1 - Web Security & DDoS)
A public API hosted behind an Application Load Balancer (ALB) is suffering from credential-stuffing attacks where single IP addresses send 5,000 login requests per minute, as well as occasional SQL injection payloads. The company wants to automatically block any IP address that exceeds 500 requests in a 5-minute window and block SQL injection patterns. Which solution meets these requirements with the LEAST effort?

- **A)** Add deny rules to the subnet Network ACL manually whenever an attack occurs
- **B)** Attach an **AWS WAF (Web Application Firewall)** Web ACL to the ALB with an AWS Managed Rule group for SQL injection and a **rate-based rule** set to 500
- **C)** Enable Amazon Inspector on the ALB
- **D)** Subscribe to AWS Shield Standard

{{% details "Cevabı göster" %}}
**Correct Answer: B**
- **Why B is correct:** **AWS WAF** operates at Layer 7 on the ALB, provides managed rule sets for SQLi/XSS, and supports **rate-based rules** that automatically block offending IPs when they exceed N requests per 5-minute rolling window.
- **Why A is wrong:** Manual NACL updates do not scale, have rule limits (default 20, max 40), and cannot inspect HTTP SQLi payloads.
{{% /details %}}

---

### Question 17 (Domain 2 - Global Traffic & Non-HTTP Failover)
A financial trading application runs on Network Load Balancers (NLBs) in `us-east-1` and `eu-west-1` over custom TCP ports. Clients around the world require **two static Anycast IP addresses** that never change, traffic routed over the **AWS global network backbone** from the nearest edge, and **sub-30-second health-checked failover** between regions without waiting for client DNS cache TTLs to expire. Which service should be placed in front of the NLBs?

- **A)** Amazon CloudFront
- **B)** **AWS Global Accelerator**
- **C)** Amazon Route 53 Geolocation Routing
- **D)** AWS Transit Gateway

{{% details "Cevabı göster" %}}
**Correct Answer: B**
- **Why B is correct:** **AWS Global Accelerator** provides **2 static Anycast IPs**, ingests TCP/UDP traffic at the nearest edge location onto the AWS global backbone, and performs instant health-checked regional failover without relying on client DNS TTL caching.
- **Why A is wrong:** CloudFront caches HTTP/HTTPS content and does not provide static Anycast IPs for arbitrary TCP/UDP protocols.
- **Why C is wrong:** Route 53 failover is subject to client DNS TTL caching delays.
{{% /details %}}

---

### Question 18 (Domain 4 - Observability & Right-Sizing)
An operations team wants to configure a CloudWatch alarm that triggers whenever an EC2 instance's **memory (RAM) utilization** exceeds 85% OR its **root EBS disk space** exceeds 90% full. However, they cannot find these metrics in the Amazon CloudWatch console under the `AWS/EC2` namespace. Why are these metrics missing, and how can they be collected?

- **A)** Memory and disk space are OS-level metrics not visible to the hypervisor; install the **CloudWatch Unified Agent** on the EC2 instance to push them as custom metrics
- **B)** Enable Detailed Monitoring (1-minute intervals) on the EC2 instance
- **C)** Enable VPC Flow Logs on the subnet
- **D)** Upgrade the EBS volume from `gp2` to `gp3`

{{% details "Cevabı göster" %}}
**Correct Answer: A**
- **Why A is correct:** By default, EC2 sends hypervisor-visible metrics (CPU, Network I/O, Status Checks) to CloudWatch, **never OS-level RAM or Disk space utilization**. You must install the **CloudWatch Agent** on the instance (with an IAM role allowing `cloudwatch:PutMetricData`) to collect memory and disk space metrics.
- **Why B is wrong:** Detailed Monitoring only changes the frequency of default metrics from 5 minutes to 1 minute; it does not add memory or disk space metrics.
{{% /details %}}

---

### Question 19 (Domain 3 - Analytics & Data Lake Cost)
A company stores 50 TB of application logs in Amazon S3 as uncompressed JSON files. Analysts run ad-hoc SQL queries using **Amazon Athena** a few times a week, and complain that queries are slow and Athena bills are high (since Athena charges $5 per TB scanned). Which TWO changes will dramatically improve query speed AND reduce Athena costs? **(Choose TWO)**

- **A)** Use AWS Glue or Amazon Data Firehose to convert the JSON files into compressed **Apache Parquet** (columnar format)
- **B)** **Partition** the data in S3 by date (e.g., `s3://bucket/year=2026/month=10/day=30/`) and filter on partition columns in `WHERE` clauses
- **C)** Move the S3 bucket to S3 Glacier Deep Archive
- **D)** Provision an S3 Transfer Acceleration endpoint
- **E)** Increase the number of EC2 instances running Athena

{{% details "Cevabı göster" %}}
**Correct Answers: A and B**
- **Why A & B are correct:** Athena charges by the **amount of data scanned**. Converting row-based JSON to compressed **columnar Parquet/ORC** (A) lets Athena read only the requested columns and shrinks file size by up to 80–90%. **Partitioning** by date (B) lets Athena skip scanning irrelevant folders entirely.
- **Why C is wrong:** Athena cannot directly query objects archived in Glacier Deep Archive without restoring them first.
- **Why E is wrong:** Athena is completely serverless — there are no EC2 instances to manage.
{{% /details %}}

---

### Question 20 (Domain 1 - Multi-Account Security & Guardrails)
An enterprise uses AWS Organizations with 40 member accounts organized into Organizational Units (OUs). A security mandate requires that **no user or role — including the account root user — in any member account** is allowed to launch resources outside `eu-central-1` and `eu-west-1`, or disable AWS CloudTrail. What is the MOST effective way to enforce this centrally?

- **A)** Create an IAM Permission Boundary in each of the 40 accounts
- **B)** Attach a **Service Control Policy (SCP)** to the root or top-level OUs in AWS Organizations that explicitly denies actions in other regions (`aws:RequestedRegion`) and denies `cloudtrail:StopLogging`
- **C)** Configure an AWS Budgets alert in each account
- **D)** Create a Network ACL in each account blocking traffic from other regions

{{% details "Cevabı göster" %}}
**Correct Answer: B**
- **Why B is correct:** **Service Control Policies (SCPs)** in AWS Organizations centrally enforce maximum permission guardrails across all member accounts — **including each member account's root user** — and cannot be overridden by local IAM administrators.
- **Why A is wrong:** IAM Permission Boundaries do not restrict the account root user and would have to be attached to every single IAM entity individually.
{{% /details %}}
