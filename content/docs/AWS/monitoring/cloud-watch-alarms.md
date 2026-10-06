---
title: "cloud-watch-alarms"
weight: 30
---

# CloudWatch Alarms

Our metrics and logs are being sent to AWS, but we have to check them manually to find out if something is wrong. Not ideal...

[CloudWatch Alarms](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Alarms.html) lets us set up automatic "actions" to be taken when certain criteria are met. For example:

- Our service is in an [Auto Scaling Group](https://docs.aws.amazon.com/autoscaling/ec2/userguide/auto-scaling-groups.html) of 1-10 servers based on CPU usage, and we just hit 10!
- We have a service emitting a high number of `5XX` errors!
- One of our servers is at 95% disk usage!

Let's create a simple alarm that triggers when our server goes above **20% CPU usage**. When triggered, it will send a notification via SNS ([Simple Notification Service](https://aws.amazon.com/sns/)) to a given email address.

## Assignment

**Create a CloudWatch alarm that trigger an email when CPU usage exceeds 20%.**

**Cost check:** CloudWatch Alarms are free for the first 10 alarms. SNS notifications cost `$0.50` per 100,000 requests.

1.  Create the alarm:
    1.  Navigate to "CloudWatch" → "Alarms" in the AWS Console.
    2.  Click "Create alarm."
    3.  Select the metric:
        - Click "Select metric."
        -  Under "EC2," click "Per-Instance Metrics."
        -  Search for your instance ID and select the row where "Metric name" is `CPUUtilization`.
        -  Click "Select metric."
    4.  Configure the alarm conditions:
        -  **Statistic:** `Average`
        -  **Period:** `1 minute`
        -  **Threshold type:** `Static`
        -  **Whenever `CPUUtilization` is:** `Greater` than threshold
        -  **than:** `20`
        -  Click "Next."
    5.  Configure the notification:
        -  **Alarm state trigger:** `In alarm`
        -  **SNS topic:** `Create new topic`
        -  **Topic name:** `patientping-alerts`
        -  **Email endpoints:** enter your own email address
        -  Click "Create topic."
        -  Scroll to the bottom of the page and click "Next."
    6.  Name and describe the alarm:
        -  **Alarm name:** `patientping-cpu-alarm`
        -  **Alarm description:** `Alert when CPU exceeds 20%`
        -  Click "Next."
    7.  If everything looks right, click "Create alarm."
        
        When you first create an alarm, it may show `Insufficient data`; this just means it hasn't gathered enough data points yet to make a decision. This will resolve itself after a few periods.
        
2.  Test the alarm:
    1.  check your email for a confirmation from AWS that you were subscribed to an SNS topic, and click the link to confirm the subscription.
    2.  SSH into your EC2 instance and start cranking the CPU:
	
```bash
ssh patientping
yes > /dev/null
```

    3.  Leave `yes` running for a few minutes, and check your email for an alarm notification from AWS.
    4.  After you get the notification, stop the `yes` command by pressing Ctrl+C in your terminal.

## Tip

Want to use the AWS CLI instead? Here's the command structure:

```sh
# Create SNS topic
aws sns create-topic --name TOPIC-NAME

# Subscribe email address to topic
aws sns subscribe --topic-arn TOPIC-ARN --protocol email --notification-endpoint EMAIL-ADDRESS

# Create alarm
aws cloudwatch put-metric-alarm --alarm-name ALARM-NAME --metric-name METRIC-NAME --namespace NAMESPACE --statistic Average --period PERIOD-SECONDS --threshold THRESHOLD --comparison-operator GreaterThanThreshold --evaluation-periods 1 --alarm-actions TOPIC-ARN
```