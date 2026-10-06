---
title: "other-lambda-use-cases"
weight: 690
---

# Other Lambda Use Cases

We built a simple HTTP API with Lambda, but that's just scratching the surface. Lambda's real power comes from its ability to automatically trigger handlers based on events from across AWS and beyond.

## S3 Triggers: Automatic Image Processing

One popular Lambda use case is processing files when they're uploaded to [Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html). For example:

1. A user uploads an image to S3 bucket `user-uploads`
2. S3 sends an event to Lambda: "A new file was uploaded"
3. A Lambda function runs that:
    - Downloads the image from S3
    - Resizes it to thumbnail size (using an image processing library)
    - Uploads the thumbnail to another S3 bucket `user-thumbnails`
4. Done!

No servers to maintain, and you only pay when images are actually uploaded. If you get 1,000 uploads one day and 10 the next, Lambda automatically scales up and down.

## Scheduled Tasks: Cron-Based Automation

Lambda can run on a schedule using [**Amazon EventBridge**](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html). Think of it like a [cron job](https://en.wikipedia.org/wiki/Cron), but serverless. For example:

1. EventBridge triggers Lambda every night at 2 AM
2. A Lambda function runs that:
    - Connects to your RDS database
    - Creates a snapshot
    - Uploads it to S3
    - Sends a notification email if it fails

EventBridge uses cron syntax to define schedules, so you can run functions every hour, daily, weekly, or on complex schedules like "first Monday of every month."

## Webhooks: Discord/Slack Bots

Say you want a [Discord](https://docs.discord.com/developers/intro) bot to post a message when your production deployment completes. For example:

1. Your CI/CD pipeline (GitHub Actions, GitLab CI, etc.) finishes deploying
2. The pipeline sends a webhook to [API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html)
3. API Gateway triggers a Lambda handler
4. The Lambda function runs and:
    - Parses the deployment info
    - Formats a nice Discord message
    - Sends it to your Discord channel via Discord's API

The Lambda function only runs when deployments happen. No bot server needs to be running continuously. You can use the same pattern for [Slack](https://docs.slack.dev/messaging/sending-messages-using-incoming-webhooks/) notifications, Teams messages, or any service that accepts webhooks.

## When Lambda Makes Sense

- **The workload is event-driven:** Responding to file uploads, webhooks, messages
- **Usage is unpredictable:** Traffic varies widely (spiky or infrequent)
- **Execution is short:** Tasks complete in under 15 minutes
- **Stateless:** Each invocation is independent
- **Cost-sensitive:** Don't want to pay for idle servers

## When Lambda Doesn't Make Sense

- **Workload is constant:** Server running 24/7 might be cheaper
- **Execution is long:** Need to run for hours
- **Needs state:** Require persistent connections or in-memory caching
- **Cold start time matters:** Can't tolerate 200-500 ms initialization delay
- **Complex dependencies:** Need large libraries or specific OS customization