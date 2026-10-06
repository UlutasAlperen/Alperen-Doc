---
title: "relational-database-service"
weight: 180
---

# RDS: Relational Database Service

[Amazon RDS](https://aws.amazon.com/rds/) (Relational Database Service) is AWS' _managed database service_. If you don't want to manage/host a database server yourself, you can let AWS handle provisioning, backups, patching, and monitoring. You get a database you can connect to with (nearly) zero operational overhead.

That said, RDS can get **very** expensive, especially if you have lots of data or need to operate in multiple AZs. But for many applications, the cost of a managed service is reasonable, and far outweighed by its benefits.

Boot.dev's site is powered by a managed database service! It used to be on AWS RDS, and now its on Google Cloud SQL, but they're basically the same thing.

## Prerequisites

Before we can create a database, we need to set up the networking infrastructure.

1.  Stand up a **VPC** named `patientping`, or use your existing one from the EC2 module, if you still have it.
2.  Stand up **subnets** within the VPC:
    1.  A **public subnet** named `patientping-public-a` for your application servers
    2.  A **private subnet** named `patientping-private-a` for your database
3.  Add an **internet gateway (IGW)** to the VPC, and make sure the public subnet's route table points to it with a `0.0.0.0/0` default route.
4.  Make sure you have a **key pair** that you can use to SSH into EC2 instances.
5.  Stand up an **Amazon Linux AMI** server on a **t3.micro** instance, named `patientping-web-v2`, in your public subnet.
6.  Add a **security group** to the instance that allows you to SSH into it (named `patientping-public` or similar).
7.  Make sure your instance has a **public IP address**, preferably by attaching an Elastic IP to it.

This EC2 instance will serve as an application server that we'll use to test connections to the database.

Whew! That's a lot of setup. If you _just_ completed the EC2 chapter, you might already have most of this in place. If so, feel free to reuse it.

**Cost check:** A `t3.micro` instance costs about `$0.0104` per hour (~$7.60 per month). The public IP is free when attached to a running instance.

I recommend checking your work in the AWS CLI with these commands:

```sh
# Check your VPC
aws ec2 describe-vpcs --filters Name=tag:Name,Values=patientping

# Check your subnets
aws ec2 describe-subnets --filters Name=tag:Name,Values=patientping-public-a
aws ec2 describe-subnets --filters Name=tag:Name,Values=patientping-private-a

# Check your instance
aws ec2 describe-instances --filters Name=tag:Name,Values=patientping-web-v2

# Make sure your instance has a public IP
aws ec2 describe-instances --filters Name=tag:Name,Values=patientping-web-v2 --query 'Reservations[*].Instances[*].PublicIpAddress'
```

Once you have your VPC, subnets, IGW, and test instance ready, **answer the multiple choice question**. We'll be creating an RDS database shortly.