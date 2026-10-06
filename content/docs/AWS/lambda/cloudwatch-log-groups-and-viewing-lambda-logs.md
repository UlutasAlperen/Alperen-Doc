---
title: "cloudwatch-log-groups-and-viewing-lambda-logs"
weight: 50
---

# CloudWatch Log Groups and Viewing Lambda Logs

When something goes wrong with your Lambda, how do you see what's happening inside the function? Debugging Lambda is trickier than a local app: you can't just attach a debugger to a process. There are [ways to run Lambda locally](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-using-debugging.html) (e.g., SAM CLI) for breakpoints, or [AWS Toolkit for VSCode](https://docs.aws.amazon.com/toolkit-for-vscode/latest/userguide/lambda-remote-debug.html) for IDE integration. For day-to-day debugging, though, you'll rely on **logs**.

Lambda automatically sends everything your function writes to [`stdout`](https://en.wikipedia.org/wiki/Standard_streams) and [`stderr`](https://en.wikipedia.org/wiki/Standard_streams) (including `print()` in Python) to **Amazon CloudWatch Logs**. As long as your Lambda has the basic execution role (which we attached in the deploy lesson), it can write logs. AWS creates a **log group** for your function and streams each run's output there.

## What Lambda Logs Automatically

For every invocation, Lambda adds a few lines before and after your code's output:

- **START:** Request ID and version
- Your **print()** (or other stdout/stderr) output
- **END:** Request ID
- **REPORT:** Duration, billed duration, memory used, and so on

That REPORT line is handy for spotting slow runs or functions that are close to the memory limit.

## Assignment

**View your Lambda function's logs in CloudWatch using the console.** They are created automatically when your function is invoked.

**Cost check:** A small Lambda like ours produces only a few KB per invocation. You'll stay well within the 5 GB free tier for this lesson, so expect $0.

1.  Go to [**CloudWatch** → **Log management**](https://console.aws.amazon.com/cloudwatch/home#logsV2:log-groups).
2.  Open the log group `/aws/lambda/patientping-ip-function`
3.  Open the most recent **log stream** and scroll through events. You should see `START`, `END`, and `REPORT` lines, as well as the "Received IP: ..." from the `print()` statement in your handler