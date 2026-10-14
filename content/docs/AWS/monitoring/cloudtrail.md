---
title: "cloudtrail"
weight: 40
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


![cloudtrail 1](/images/aws/cloudtrail_1.png)

## AWS CloudTrail

CloudTrail tracks user activity and API usage in the AWS Cloud, on premises, and even with other cloud providers. CloudTrail provides a detailed history of API calls, so you can track changes and identify who made them and when. This helps you understand what actions were taken on your AWS resources.

### Benefits: 
CloudTrail provides auditing, security monitoring, and operational troubleshooting. It also helps you prove compliance and improve your security posture.

### Use cases: 
It can be used for compliance and auditing, identifying security incidents, troubleshooting operational issues.

![cloudtrail 2](/images/aws/cloudtrial_2.png)
