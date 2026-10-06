---
title: "aws-s3"
weight: 10
---

# AWS S3

[Amazon S3](https://aws.amazon.com/s3/) (**Simple Storage Service**) is AWS' [object storage](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html) service. Think of it as slower, but _very_ scalable storage for files (_objects_) in containers (_buckets_). It's relatively cheap and very reliable at huge scales.

By "slow," I mean it's not designed for low-latency access like a database, and it's not designed to be easily searchable. It's designed for storing and retrieving files by their key (filename).

![S3 object storage overview](/images/aws/s3-overview.png)

You probably store user records (first name, last name, email, password) in a traditional [relational database](https://en.wikipedia.org/wiki/Relational_database), like [Amazon RDS](https://aws.amazon.com/rds/). However, you might store PDFs, images, and videos in S3.

A key feature of S3 is that it's "serverless." It's not serverless **compute**, but rather serverless **storage**. You don't have to manage, scale, or secure the individual servers that store your files. AWS does all that for you.

Instead of interacting with a file system, your server makes network requests to the S3 API to read and write files.

Bucket names must be **globally unique** across all AWS accounts (they appear in URLs). So, choose a random codename for your bucket and append it to the end. In this course we use the pattern `patientping-favicon-bucket-${codename}` (e.g. `a1b2c3d4`).

**Cost check:** S3 storage costs about $0.023 per GB per month. You pay for storage and requests; for the small buckets and objects used in this course, the cost is minimal.