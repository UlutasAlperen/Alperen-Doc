---
title: "cloudfront-cdn"
weight: 460
---

# CloudFront CDN
How does Netflix serve a 14 GB 4K video file to a user _on demand_? Two options include:

- They can compress the video data as much as possible (_compression_).
- They can chunk the video into smaller pieces and send it piece by piece (_chunking/streaming_).

But neither of those solves the problem of _initial load time_ (how long until the first frame starts after hitting "play"). To get fast initial loads, we also need a quick **[round-trip time](https://www.cloudflare.com/learning/cdn/glossary/round-trip-time-rtt/)** (RTT). If you're a gamer, it's basically the same as server ping: the time it takes for a request to go from your computer to the server and back.

[CloudFront](https://aws.amazon.com/cloudfront/) is AWS's [content delivery network](https://www.cloudflare.com/learning/cdn/what-is-a-cdn/) (CDN). A CDN is a network of servers distributed around the world that cache and serve content in locations that are _physically close_ to users. In simple terms, they cache static content like images, videos, and scripts from an [origin server](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistS3AndCustomOrigins.html) and serve it from the nearest edge location.

![CDN visual](https://storage.googleapis.com/qvault-webapp-dynamic-assets/course_assets/rNV9Nku-1280x720.png)

CDNs solve two main problems:

1. **Latency:** Bringing content physically closer to users reduces the time it takes to load.
2. **Origin Server Load:** Offloading traffic from your main servers reduces bandwidth costs and prevents overload.

## How CloudFront Works

1. **Origin:** Your source content (S3 bucket, EC2 instance, load balancer, etc.)
2. **Distribution:** CloudFront creates a "distribution," i.e., a configuration for how to route requests to your origin.
3. **Edge Locations:** AWS has hundreds of edge locations worldwide.
4. **Caching:** Edge locations cache content based on user requests.
5. **Delivery:** When a user requests content, CloudFront serves it from the nearest edge location.