---
title: "regions-and-availability-zones"
weight: 10
---

# Regions and Availability Zones

AWS has many regions to choose from. Each region is where your services are **geographically located**.

- An outage in one region will (_usually_) not affect another region.
- The farther a server is from the user, the higher the latency. Choose a region close to the folks using your service.
- Some services are available only in certain places, or are significantly more expensive in some regions. So we need to make sure a region meets our needs.
- As a service grows, we may want to host in more than one region. We also pay a toll for transferring data between regions.

![](https://storage.googleapis.com/qvault-webapp-dynamic-assets/course_assets/U9stnq2-1224x606.png)

## Availability Zones (AZs)

Each region is split into "zones," which means separate infrastructure within the same region.

- An outage in one zone will (_usually_) not affect servers hosted in another zone.
- We pay a toll for transferring data between zones.

## Assignment

**Set your default AWS region to `us-east-1`.**

**Cost check:** Listing regions and availability zones is free, and changing your default region is also free.

1. List all available AWS regions using the AWS CLI:

```bash
aws ec2 describe-regions --output table
```
> Press "q" to quit when you're done looking at the output.

2.  List availability zones for your current region:
```bash
aws ec2 describe-availability-zones --output table
```

3. Switch your default region to `us-east-1` (if it isn't already):

 ```bash
 aws configure set region us-east-1
 ```
 
4.  In the AWS web console, open the region dropdown and select "N. Virginia" (`us-east-1`) to match your CLI region.

5.  Verify your region is set correctly:

```bash
aws configure get region
```

>You should now see `us-east-1` as the output.