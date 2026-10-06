---
title: "aws-external-monitoring"
weight: 340
---

# External Monitoring

With an [EC2](https://aws.amazon.com/ec2/) instance running, we get some monitoring data _automatically_ and at no extra charge.

There are certain things that a running instance _must_ communicate to the hardware/system that hosts it. AWS needs to track this information "from the outside" in order to manage its infrastructure. For example:

- How much CPU is the instance using?
- How much network bandwidth is it taking up?
- How much data is it reading from and/or writing to disk?

All these things can be monitored externally by AWS, and they package it into a service called [CloudWatch Metrics](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/working_with_metrics.html). The metrics allow us to answer questions like:

- Is my server _doing anything_? (If not, you'll see idle CPU usage.)
- Is my server _overloaded_? (Watch for sustained high CPU usage.)
- Is my server _consuming too much bandwidth_? (Watch network usage.)

These are a great start, and for some services they're all you need.

## Assignment

**Cost check:** Basic CloudWatch metrics for EC2 (CPU, network, disk) are included at no extra charge. CloudWatch dashboards have a small monthly cost, so delete your test dashboard when you clean up resources.

**Create a dashboard for `patientping-web-v2`, force CPU usage high, verify the spike, then stop the load.**

1.  In the AWS Console, go to "CloudWatch" → "Dashboards" and click "Create dashboard."
2.  Name the dashboard `patientping-dashboard` and click "Create dashboard."
3.  Add a line widget for `patientping-web-v2` CPU:
    1.  Choose "Line" when prompted for widget type.
    2.  In metric namespaces, select "EC2" → "Per-Instance Metrics."
    3.  Search for _your_ instance ID, and select the row where the metric is `CPUUtilization`, then check it.
    4.  Click "Create widget."
    5.  Set the dashboard timeframe to "15 minutes".
    6.  Click "Save dashboard."
4.  SSH into `patientping-web-v2` and start CPU load:
```bash
yes > /dev/null
```
5.  Keep the load running for at least 5 minutes so the dashboard shows a clear CPU spike!
6.  Once you're satisfied that your monitoring is working, stop the load by pressing Ctrl+C in the terminal where you ran the `yes` command.