---
title: "cdn"
weight: 80
bookCollapseSection: true
---

# AWS - CDN - CloudFront

- 001 - [cloudfront-cdn](cloudfront-cdn/) = [CloudFront](https://aws.amazon.com/cloudfront/) is AWS's [content delivery network](https://www.cloudflare.com/learning/cdn/what-is-a-cdn/) (CDN). A CDN is a network of servers distributed around the world that cache and serve content in locations that are _physically close_ to users. In simple terms, they cache static content like images, videos, and scripts from an [origin server](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistS3AndCustomOrigins.html) and serve it from the nearest edge location.

![CloudFront CDN diagram](/images/aws/cloudfront-cdn-diagram.png)

- 002 - [cloudfront-distributions](cloudfront-distributions/)

![CloudFront distributions diagram](/images/aws/cloudfront-distributions-diagram.png)

- 003 - [cloudfront-invalidation](cloudfront-invalidation/)
- 004 - [presigned-urls](presigned-urls/)
- 005 - [why-use-presigned-urls](why-use-presigned-urls/) = http web sunucu servisi gibi file serve icin kullaniliabilir
- 006 - [cloudfront-with-dns](cloudfront-with-dns/)
