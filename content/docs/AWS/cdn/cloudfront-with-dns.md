---
title: "cloudfront-with-dns"
weight: 60
---

# CloudFront With DNS

Your CloudFront distribution has a domain name like `d1234567890abcdef.cloudfront.net`, but that's a long, hard-to-remember address.

To make it easier for users to find your content, you can create _your own_ DNS record that points a friendly domain name to your CloudFront distribution. Instead of typing that long CloudFront URL, users can type something like `cdn.yourcoolsite.com`.

If you're using [Route 53](https://docs.aws.amazon.com/route53/), you can use an [ALIAS (A) record](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-to-cloudfront-distribution.html) instead of a [CNAME](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/CNAMEs.html). ALIAS records work better at the zone apex and are free. They also are a bit more plug-and-play within AWS if you're using CloudFront, but CNAME also works just fine.

However, ALIASing a CloudFront distribution requires using a Public hosted zone, so because ours is private, we'll use a CNAME record.

**Cost check:** CNAME records are free, but if you're using Route 53, you'll pay for hosted zones (`$0.50`/mo) and DNS queries (`$0.40` per million queries).

## Assignment

In production, custom CloudFront domains also need to be configured as "Alternate domain names (CNAMEs)", usually with a matching certificate. For now, just create the Route 53 CNAME below.

1.  Navigate to Route 53 in the AWS console.
2.  Select your `patientping.internal` hosted zone.
3.  Create a new `CNAME` record:
    -  **Name:** `cdn` (or whatever subdomain you want)
    -  **Value:** Your CloudFront distribution domain (e.g. `d1234567890abcdef.cloudfront.net`)
4.  Save the record.
