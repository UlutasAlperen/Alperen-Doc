---
title: "cloudwatch-log-groups"
weight: 110
---

# CloudWatch Log Groups

Now, how will we know if something goes wrong with our containers at runtime? We need to set up [**CloudWatch Log Groups**](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html) so we can see what's happening inside our containers (`stdout`, `stderr`, application logs, etc.).

Think of [CloudWatch log groups and log streams](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/Working-with-log-groups-and-streams.html) as folders for your logs. Each group can have multiple streams (one per container instance), so you can see all logs for a service in one place.

If you don't create the log group beforehand, ECS will try to create it automatically when the first task starts. However, your task execution role needs the `logs:CreateLogGroup` permission for this to work. Creating it explicitly gives you more control and avoids potential permission issues. The task execution role we created earlier already has permissions to write logs (`logs:CreateLogStream` and `logs:PutLogEvents`), so once the log group exists, tasks can start logging immediately.

## Assignment

**Create a CloudWatch log group `/ecs/patientping-ecs`** and set a short retention (e.g. 7 days) so we don't hoard logs forever.

**Cost check:** CloudWatch Logs charges `$0.50` per GB ingested and $0.03 per GB stored per month. For small applications, this is fairly minimal, but setting a retention policy automatically deletes logs after a certain period, which helps keep costs down while still giving you recent logs for debugging.

1.  In the AWS Console, navigate to `CloudWatch`, then open `Log Management` under `Logs`.
2.  Click **Create log group**.
3.  Set **Log group name** to `/ecs/patientping-ecs`.
4.  For **Retention setting**, choose **7 days** so old logs are cleaned up automatically.
5.  Click **Create**.

**Run and submit** the tests to verify your log group is configured correctly.