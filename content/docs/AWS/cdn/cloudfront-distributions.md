---
title: "cloudfront-distributions"
weight: 20
---

# CloudFront Distributions

A [**CloudFront distribution**](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/distribution-working-with.html) is the configuration that tells CloudFront:

- Where your content is (the origin, like your S3 bucket)
- How to deliver it (caching behavior, compression, etc.)
- Who can access it (security settings)

When you create a distribution, CloudFront assigns it a domain name, like `d1234567890.cloudfront.net`, which you can use to access your content through the CDN.

## Key Configuration Options

- **Origin:** The source of your content (S3 bucket, EC2 instance, load balancer, etc.)
- **Default root object:** What file to serve for the root URL (e.g. `index.html`)
- **Distribution settings:** Caching behavior, compression, HTTPS, etc.
- **Price class:** Which edge locations to use (affects cost and coverage)

CDNs are typically used to serve static content, like images, videos, and documents. You may have heard of [Lambda@Edge](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-at-the-edge.html) for actual computational request/response logic close to users... more on that later.

## Why We Need a Distribution

Right now, your favicon is sitting in an S3 bucket. Users can access it, but they're hitting S3 _directly_, which means:

- Slower load times for users far from your bucket's region
- All traffic goes through S3 (more bandwidth costs)
- No caching benefits

A CloudFront distribution solves this by creating a network that caches your content at edge locations worldwide. Users get faster access, and you reduce load on S3.

![visualization of CloudFront distribution](https://storage.googleapis.com/qvault-webapp-dynamic-assets/course_assets/Cf6hhGs-961x806.png)

## example

Create a CloudFront distribution that points to your S3 bucket. This will make the favicon accessible through CloudFront's global edge network.

**Cost check:** CloudFront data transfer costs vary by region and volume. The first 1 TB per month is free, then it's roughly $0.085 per GB.

Some new AWS accounts must be verified before they can create CloudFront resources. If AWS says your account must be verified before adding CloudFront resources, contact AWS Support and include that error message.

1.  Navigate to `CloudFront` using the AWS search bar.
2.  Click `Create distribution`.
    1.  Distribution name: `patientping-favicon-distro`
    2.  Distribution type: "Single website or app"
    3.  Click `Next`
3.  Specify origin.
    1.  Origin type: `Amazon S3`
    2.  **Origin:** Click "Browse S3" and select your S3 bucket (e.g. `patientping-favicon-bucket-YOUR_SUFFIX.s3.amazonaws.com`).
    3.  Origin settings → "Use recommended origin settings"
    4.  Cache settings → "Use recommended cache settings tailored to serving S3 content"
    5.  Click `Next`
4.  Enable security.
    1.  WAF: leave default
    2.  Click `Next` → review everything looks correct and click `Create distribution`
5.  Wait for the distribution to deploy (it can take 5-15 minutes).
    1.  The status will show `In Progress` then change to `Enabled`.
    2.  Note the **Distribution domain name** (e.g. `YOUR-ID.cloudfront.net`).
6.  Once deployed, test the distribution:
    1.  Copy the distribution domain name.
    2.  Append `/favicon.ico` to it (e.g. `https://YOUR-ID.cloudfront.net/favicon.ico`).
    3.  Open the URL in a browser to verify that the favicon loads.
    4.  Open your browser's [developer tools](https://developer.chrome.com/docs/devtools/) → Network tab → Reload the page. Take a look at the load time – mine was 20ms, which is super fast! I'm probably fairly close to an edge location.

![devtools](https://storage.googleapis.com/qvault-webapp-dynamic-assets/course_assets/fjNhjq8-1280x189.png)

## Tip

If you prefer to use the CLI, here's a command structure to get started with CloudFront distributions:

```sh
aws cloudfront create-distribution --distribution-config '{"CallerReference":"<unique-string>","Origins":{"Quantity":1,"Items":[{"Id":"<origin-id>","DomainName":"<bucket-domain>","S3OriginConfig":{"OriginAccessIdentity":""}}]},"DefaultCacheBehavior":{"TargetOriginId":"<origin-id>","ViewerProtocolPolicy":"redirect-to-https","AllowedMethods":{"Quantity":2,"Items":["GET","HEAD"]},"CachePolicyId":"<cache-policy-id>"},"Enabled":true}'
```