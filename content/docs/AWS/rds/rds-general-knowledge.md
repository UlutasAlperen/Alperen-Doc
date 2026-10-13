---
title: "rds-general-knowledge"
weight: 70
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


# Relational databases

Relational databases store data in a way that relates it to other pieces of data, and they use structured query language, or SQL, to manage and query data. This approach stores data in an easily understandable, consistent, and scalable way that works great for applications requiring structured data management.

AWS offers fully managed relational database solutions that remove the burden of database administration while maintaining high availability and security. AWS relational databases support popular database engines like MySQL, PostgreSQL, and Oracle, making it easier to migrate existing databases to AWS.

An example of a relational database would be an inventory management system for a restaurant. Each record in the database includes data for a single item, such as product name, size, price, and so on. The following table shows how this type of data is configured in a relational database.

---

![database 1](/images/aws/database_1.png)


## Amazon Relational Database Service (Amazon RDS)

Amazon RDS is a managed relational database service that handles routine database tasks such as backups, patching, and hardware provisioning. Amazon RDS supports multiple database instance class types that optimize for memory, performance, or input/output (I/O).

To improve data resilience, Amazon RDS offers Multi-AZ deployment and automated backups, but you can also manually create backups using DB snapshots. These are full backups of your entire database instance, which can be useful for specific point-in-time recovery or long-term data archiving purposes. Amazon RDS offers security features including network isolation, encryption in transit, and encryption at rest. You can readily scale database resources vertically or horizontally as needed.

### Supported database engines

Amazon RDS supports different database engines, including Amazon Aurora, MySQL, PostgreSQL, Microsoft SQL Server, MariaDB, and Oracle Database.

### Use cases

Some examples of practical use cases for Amazon RDS are web applications, enterprise workloads, and product inventories for e-commerce platforms.

![database 2](/images/aws/database_2.png)
![database 3](/images/aws/database_3.png)

---

![database 4](/images/aws/database_4.png)

## Amazon Aurora

Aurora is a managed relational database designed to help reduce unnecessary I/O operations. It's compatible with MySQL and PostgreSQL, provides high performance and availability, and automatically scales alongside your workloads. Aurora replicates data across multiple Availability Zones for enhanced durability and fault tolerance, and features automated backups, encryption at rest, and continuous monitoring.

### Use cases

Some examples of practical use cases for Aurora are gaming applications, media and content management, and real-time analytics.
Benefits


![database 5](/images/aws/database_5.png)
