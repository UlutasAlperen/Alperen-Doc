---
title: "aws-lambda"
weight: 640
---

# AWS Lambda

So you've got a simple application you want to run in the cloud. Maybe it's an API endpoint, a scheduled task, or a function that processes files uploaded to [Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html). You could spin up an [EC2 instance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html), set up an [ECS cluster](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/clusters.html), or run code directly on a server.

[AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html) is a serverless compute service. You write your code and upload it, and AWS handles the rest: provisioning servers, scaling, patching, and monitoring. You only pay for the compute time you actually use.

## The Serverless Pitch

Here's the thing: with EC2 or ECS, you're paying for servers whether they're busy or idle. If your application only handles 100 requests a day, you're still paying for a server to sit there 24/7 waiting for those requests.

Lambda flips this model. Your code only runs when it's triggered, and you only pay for those milliseconds of execution time. No requests? No charges (beyond a tiny storage fee for the code itself).

## When Lambda Makes Sense

- **Event-driven workloads:** Process S3 uploads, respond to [SNS](https://docs.aws.amazon.com/sns/latest/dg/welcome.html) messages, react to [DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html) changes
- **APIs with variable traffic:** Handle thousands of requests per second or just a handful per day
- **Scheduled tasks:** Run a job every hour without maintaining a server
- **Background processing:** Resize images, generate PDFs, send emails
- **Prototypes and [MVPs](https://en.wikipedia.org/wiki/Minimum_viable_product):** Get something running fast without infrastructure setup

## When Lambda Does Not Make Sense

- **Long-running processes:** Lambda functions time out after 15 minutes max
- **Stateful applications:** Each invocation is isolated; you can't keep state in memory between requests
- **Applications needing large dependencies:** 250 MB unzipped deployment package limit
- **Consistent high traffic:** If your function runs constantly, EC2 or ECS might be cheaper
- **Applications requiring specific OS customization:** You get a standard runtime environment

## How Lambda Works

When a Lambda function is invoked:

1. **[Cold start](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html#cold-start-latency)** (first time or after idle): AWS provisions a container, loads your code, and starts the runtime. This adds latency (often a few hundred milliseconds for [Python](https://www.python.org/), longer for Java or .NET).
2. **Warm execution** (subsequent requests): If another request comes in quickly, AWS reuses the existing container. This is fast (single-digit milliseconds).
3. **Scaling:** If requests come in faster than the available containers can handle, AWS automatically creates more containers. Lambda scales up to thousands of concurrent executions.