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

---

# Additional Database Services


## Amazon DocumentDB

Amazon DocumentDB (with MongoDB compatibility) is fully managed service designed to handle semistructured data, which is information that doesn't conform to rigid relational schemas. Amazon DocumentDB is a MongoDB-compatible database, so it manages JSON-like documents with dynamic schemas.

Amazon DocumentDB is perfect for applications requiring frequent schema changes and document-oriented data. Unlike relational databases or nonrelational databases, you can quickly iterate without relying on predefined schemas. Amazon DocumentDB can store, query, and index JSON data effortlessly, all while benefiting from automatic scaling, continuous backup, and enterprise-grade security features.
Use cases

Some examples of practical use cases for Amazon DocumentDB are content management systems, catalog and inventory management, and user profile and personalization systems.

## Benefits

### MongoDB compatibility
Amazon DocumentDB is fully compatible with MongoDB workloads and supports MongoDB APIs, drivers, and tools. This compatibility means that you can use existing MongoDB code and skills without modification. You can also migrate MongoDB applications to Amazon DocumentDB with minimal changes to their application code.

### Performance and scalability
Amazon DocumentDB automatically scales storage up to 64 TB in 10 GB increments based on your application needs. It can handle millions of requests per second with consistent performance. It also provides the option to scale compute resources up or down as needed.

### Increased read throughput
Amazon DocumentDB improves read throughput for high-volume applications by creating up to 15 replica instances that share underlying storage.



## AWS Backup

AWS Backup streamlines data protection across various AWS resources and on-premises deployments by providing a single dashboard for monitoring and managing backups. It eliminates the complexity of managing multiple backup strategies by supporting multiple storage types, including Amazon Elastic Block Store (Amazon EBS) volumes, Amazon Elastic File System (Amazon EFS) file systems, and various databases.

AWS Backup centralizes and automates data protection processes, improving consistency and reducing administrative overhead. It offers flexible scheduling options, encryption capabilities, and cross-Region backup support for enhanced disaster recovery.

##  Use cases

Some examples of practical use cases for AWS Backup are centralized disaster recovery, consistent backup policies for compliance requirements, and consolidating multiple backup processes through a single interface.

## Benefits

### Centralized backup management
AWS Backup provides a single dashboard to manage backups across multiple AWS services and accounts. You can monitor backup jobs, restore points, and verify compliance status from one central location to reduce operational complexity and potential configuration errors.

You can create automated backup schedules that align with your business requirements and compliance needs. You can set up backup policies that automatically protect new resources as they're created.

### Cross-region backup redundancy
AWS Backup enables automatic replication of backup data across different AWS Regions for disaster recovery purposes. You can quickly restore data from secondary Regions if the primary Region experiences an outage. Cross-Region redundancy helps you meet compliance requirements while guaranteeing data accessibility during Regional failures.


### Streamlined regulatory compliance
AWS Backup maintains detailed audit logs and reports to demonstrate compliance with regulatory requirements. You can use it to enforce backup policies across your organization and track backup activities for security and compliance purposes.

## Amazon Neptune

Neptune is a fully managed, purpose-built graph database service that manages highly connected data sets, like those used in social networking applications. It excels at understanding complex relationships that are difficult to identify in traditional relational databases like user connections, friend networks, and interaction patterns. Neptune can maintain high performance even as data complexity grows, and offers high availability with automatic failover and backups.

## Use cases

Some examples of practical use cases for Amazon Neptune are social network user connection mapping, fraud detection systems, and search and recommendation systems.

## Benefits

### Purpose-built for complex relationships
Neptune excels at storing and querying highly connected data using graph models. It supports both property graph and resource description framework, or RDF, models making it ideal for relationship mapping and pattern matching applications.

### High performance and scalability
Neptune delivers consistent performance at scale, processing billions of relationships in milliseconds. It automatically grows storage up to 64 TB based on your application needs. Its purpose-built engine optimizes graph queries to enable fast traversal of connected data points at scale.
