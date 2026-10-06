---
title: "cloudtrail"
weight: 370
---

# CloudTrail

[CloudWatch](https://aws.amazon.com/cloudwatch/) is great for monitoring _our_ systems, but how do we get logs for the AWS account itself? Think about this:

- Our CloudWatch agent is happily writing logs to CloudWatch, but what if they're being _rejected by the API_?
- What if the intern we fired last month changed our [VPC](https://aws.amazon.com/vpc/) settings? How would we know what they did?

[CloudTrail](https://aws.amazon.com/cloudtrail/) is the answer. It logs the AWS API calls that we make.

With CloudTrail, you can see what folks have been doing on your AWS account. Each event includes information like:

- Who made the API call (username)
- What API call was made (e.g., `DescribeAlarms`, `CreateInstance`)
- When it occurred (timestamp)
- Whether it was a read or a write operation
- What resources were affected

For example, you might see that user `vvega` ran the API call `DescribeAlarms` at a specific time. This level of detail is critical to troubleshooting things on AWS. It tells you for certain what happened and when, regardless of the health of your monitoring systems.

There's [Organization trails](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/creating-trail-organization.html), which lets us combine multiple AWS accounts into a single trail and see what users are doing across all of them.

**Cost check:** CloudTrail is free for the first copy of management events (create, modify, delete API calls). Data events (S3 object-level activity, Lambda function invocations) cost `$0.10` per 100,000 events.