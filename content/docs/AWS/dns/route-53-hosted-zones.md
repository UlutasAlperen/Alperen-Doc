---
title: "route-53-hosted-zones"
weight: 10
---

# Route 53 Hosted Zones

So far in this course, every time we've needed to communicate with an AWS resource we set up, we've identified it in one of two ways:

- A **raw [IPv4](https://en.wikipedia.org/wiki/IPv4) address**, like we used to load the web app, or to [SSH](https://en.wikipedia.org/wiki/Secure_Shell) into [EC2](https://aws.amazon.com/ec2/) instances.
- An **AWS-provided domain name**, like `patientping-db.random_id.us-east-1.rds.amazonaws.com` for an [RDS](https://aws.amazon.com/rds/) instance.

But usually for a _real_ company, we need to bring _our own domain name_ and point it to AWS resources.

The solution to this problem is [Route 53](https://aws.amazon.com/route53/), AWS' [DNS](https://www.cloudflare.com/learning/dns/what-is-dns/) service. Route 53 translates names like `patientping.io` into IP addresses. It can answer questions such as:

- Where should emails to `cto@patientping.io` be routed?
- Which server runs the website `www.patientping.io`?
- Is this recruiting email really from `jobs@patientping.io`?

In college, my friends used to send each other emails "from" their professor's email address, saying "Please come to my office first thing in the morning..." Which is why domain verification is so important!

To point a domain to your servers, you need somewhere to define and store DNS records. That's what a **hosted zone** is: a "box" for all the DNS records for a domain.

Normally you'd buy a "real" domain from a **registrar** (a company that sells domain names, e.g., Porkbun, Namecheap, and yes, AWS). For this course, we'll use a free `patientping.internal` domain, which works for learning but won't resolve on the public internet.

## Assignment

PatientPing wants internal hostnames like `www.patientping.internal` and `blog.patientping.internal` to resolve inside the VPC (for the app, for docs, for whatever Product thinks of next). A [private hosted zone](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zones-private.html) does exactly that: DNS that only works from your VPC, no need to buy a real domain.

**Create a Route 53 private hosted zone for `patientping.internal` and associate it with your `patientping` VPC.**

**Cost check:** Hosted zones cost $0.50 per zone per month, but the first 25 hosted zones are free for the first 12 months.

1.  Navigate to "Route 53" → "Hosted zones" in the AWS Console.
2.  Click "Create hosted zone."
3.  Configure the zone:
    -  **Domain name:** `patientping.internal`
    -  **Type:** `Private hosted zone`
    -  **Associated VPC:** region `us-east-1`; select `patientping` VPC
4.  Click "Create hosted zone" to confirm.
5.  Take a moment to review the initial records that were added to this hosted zone.

You should see [SOA](https://www.cloudflare.com/learning/dns/dns-records/dns-soa-record/) and [NS](https://www.cloudflare.com/learning/dns/dns-records/dns-ns-record/) records. The NS records specify name servers (e.g. `ns-0.awsdns-00.com`), which will answer DNS queries for `patientping.internal` from within the VPC.

## Tip

To create a private hosted zone via the AWS CLI, you can use the following command structure:

```sh
aws route53 create-hosted-zone \
--name patientping.internal \
--caller-reference ANY-UNIQUE-STRING \
--hosted-zone-config PrivateZone=true \
--vpc VPCRegion=us-east-1,VPCId=YOUR-VPC-ID
```