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

---

# NoSQL databases

NoSQL databases are sometimes referred to as non-relational databases because their structures are different than relational databases like Amazon RDS. Instead of row and column relationships, NoSQL databases build a structure for the data that they contain using key-value pairs instead. With key-value pairs, data is organized into items identified by unique keys.

Each key has one or more associated attributes, or values, that represent various characteristics of the data. You can think of a key as a word entry in a dictionary, and the value as its associated definition. Not every item in the table has to have the same attributes, and you can add or remove attributes at any time.

The following table shows an example of how key-value paired data is stored in NoSQL databases.

![database 6](/images/aws/database_6.png)

![database 7](/images/aws/database_7.png)

## Amazon DynamoDB

DynamoDB is a fully managed NoSQL database service that provides fast and predictable performance for both document and key-value data structures. It's a powerful and incredibly fast database option for use cases that require a flexible schema, and is ideal for applications that require high performance and seamless scaling.

DynamoDB seamlessly scales alongside your data without impacting performance, which means that you only pay for the resources that you use. It also includes built-in security features for enhanced protection, and automatically spreads your data across multiple servers to handle your workload.
Use cases

Some examples of practical use cases for DynamoDB are gaming platforms, financial service applications, and mobile applications with global user bases.


![database 8](/images/aws/database_8.png)
![database 9](/images/aws/database_9.png)

---

![database 10](/images/aws/database_10.png)

# Amazon ElastiCache

ElastiCache is a fully managed in-memory caching service that was built to help reduce the complexity of administering in-memory caching systems. This means that you can continue to use the same Redis, Valkey, or Memcached tools and configurations to scale your workloads. It automatically detects and replaces failed nodes, which makes it ideal for applications that need consistent high performance.
Use cases

Some examples of practical use cases for ElastiCache are session data management, database query enhancement, and gaming leaderboards.


### Benefits


![database 11](/images/aws/database_11.png)
