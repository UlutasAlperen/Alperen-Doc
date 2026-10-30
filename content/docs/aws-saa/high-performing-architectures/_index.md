---
title: "high-performing-architectures"
weight: 30
bookCollapseSection: true
---

# Domain 3 - Design High-Performing Architectures (24%)

- 001 - [s3-storage-and-performance](s3-storage-and-performance/) = storage class'lar, multipart, Transfer Acceleration, CRR/SRR kurallari
- 002 - [block-and-file-storage-ebs-efs-fsx](block-and-file-storage-ebs-efs-fsx/) = EBS (`gp3`, `io2`, Multi-Attach), EFS, FSx (Windows, Lustre, ONTAP, OpenZFS)
- 003 - [caching-elasticache-cloudfront](caching-elasticache-cloudfront/) = Redis vs Memcached, DAX (DynamoDB Accelerator), cache stratejileri
- 004 - [database-selection-matrix](database-selection-matrix/) = is yukunden veritabani secimi - RDS (Relational Database Service) mi DynamoDB mi Redshift mi
- 005 - [dynamodb-deep-dive](dynamodb-deep-dive/) = partition, WCU (write capacity unit)/RCU (read capacity unit), GSI (global secondary index)/LSI (local secondary index), DAX, Global Tables
- 006 - [compute-performance-ec2-options](compute-performance-ec2-options/) = instance aileleri, placement groups, enhanced networking
- 007 - [serverless-and-containers-lambda-ecs-eks](serverless-and-containers-lambda-ecs-eks/) = Lambda limitleri, RDS Proxy, API Gateway, ECS/EKS ve Fargate
- 008 - [data-transfer-and-hybrid-services](data-transfer-and-hybrid-services/) = DataSync, Transfer Family, Storage Gateway, Snow, DMS (Database Migration Service)

Bu domain "performans" sorulari: **en hizli/en olceklenebilir secenek hangisi**. Yaygin tuzak: dogru calisan ama performansi olcilmemis secenekler. Ipucu: sinav **yonetilmis (managed) servisleri** ve **dogru veritabani tipini** sever.

> Not: S3 (Simple Storage Service) ve CloudFront'un pratik tarafi [AWS - S3](../../aws/s3/) ve [AWS - CDN](../../aws/cdn/) notlarinda.
