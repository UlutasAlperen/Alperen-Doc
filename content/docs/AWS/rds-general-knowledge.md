---
title: "rds-general-knowledge"
weight: 240
---

# RDS Storage and IOPS

When you create an RDS instance, you choose a type and amount of storage. AWS offers different storage types optimized for different workloads. Let's go over them.

## General Purpose SSD (`gp3`)

This is the **recommended default** for most production workloads. It's a good balance of performance and cost:

- **Baseline performance:** 3,000 IOPS (Input/Output Operations Per Second)
- **Scalable:** Can provision up to 16,000 IOPS if needed
- **Cost-effective:** Lower cost than provisioned IOPS
- **Use case:** Most applications, up to moderate I/O requirements

## Provisioned IOPS SSD (`io1`/`io2`)

This is for **demanding workloads** that need guaranteed performance:

- **Predictable performance:** Guaranteed IOPS (1,000 to 256,000)
- **Low latency:** Consistent performance
- **Higher cost:** More expensive than general purpose storage
- **Use case:** Databases with high transaction rates, large databases, I/O-intensive workloads

## What Are IOPS?

[**IOPS** (Input/Output Operations Per Second)](https://en.wikipedia.org/wiki/IOPS) measures _approximately_ how many read/write operations your database can perform per second. Think of it like the speed limit of your database storage.

- **Higher IOPS** = faster database operations
- **Lower IOPS** = slower database operations

For most applications, the default 3,000 IOPS from general purpose storage is plenty. You only need provisioned IOPS if you have a specific performance requirement that general purpose can't meet.
