---
title: "aws-internal-monitoring"
weight: 20
---

# Internal Monitoring

Beyond what is easily observable from the outside, there are useful things to monitor that live _inside_ the domain of the server's operating system. Things like:

- Disk usage (i.e., how much data is on disk and how much space is free)
- Memory used vs. available memory
- Application logs

AWS doesn't provide these metrics out of the box. To solve this, there are hundreds of options: [Grafana](https://grafana.com/), [Prometheus](https://prometheus.io/), [Datadog](https://www.datadoghq.com/), [Splunk](https://www.splunk.com/), etc. AWS also provides a simple option with the [CloudWatch Agent](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html).

CloudWatch Agent is a small piece of software that takes a single-file configuration and sends information from our server's operating system to AWS. The data can be basic metrics like CPU, disk, and memory usage, or the agent can be configured to upload specific OS or application logs.

Setting up a monitoring agent on an [EC2](https://aws.amazon.com/ec2/) instance is straightforward:

1. Grant it the necessary permissions to send data to CloudWatch (via an IAM role and policy).
2. Install the CloudWatch Agent software on the server.
3. Configure the agent with a config file that specifies what metrics and logs to collect.
4. Start the agent, and it will begin sending data to CloudWatch.

## Assignment

The PatientPing ops team wants more detailed info on how app servers are behaving. **Set up CloudWatch Agent on our EC2 instance.**

**Cost check:** CloudWatch Logs charges $0.50 per GB ingested and $0.03 per GB stored per month. CloudWatch Metrics are free for the first 10 custom metrics.

1.  Create an IAM role with CloudWatch permissions:
    1.  Navigate to "IAM" → "Roles" in the AWS Console.
    2.  Click "Create role."
        -  **Trusted entity type:** `AWS service`
        -  **Use case:** `EC2`
        -  Click "Next."
        -  Don't attach any policies yet; click "Next" again.
        -  Name the role `patientping-monitoring-role`.
        -  Without changing anything else, click "Create role" at the bottom.
2.  Create and attach CloudWatch policy:
    1.  Navigate to "IAM" → "Roles" in the AWS Console.
    2.  Click the `patientping-monitoring-role` role name (not its checkbox), then go to the "Permissions" tab.
    3.  Under "Permissions policies," click "Add permissions" → "Create inline policy."
    4.  In the policy editor, switch to JSON input, _delete all the starter text_, and paste the following:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogStreams"
      ],
      "Resource": ["arn:aws:logs:*:*:*"]
    }
  ]
}
```
 5.  Click "Next."
 6.  Name the policy `patientping-cloudwatch-logs-access`, then click "Create policy."
 7.  Add one more inline policy on `patientping-monitoring-role` so the instance can still read `/DATABASE_URL` and `/CMO_NAME` from SSM Parameter Store:
 ```json
 {
   "Version": "2012-10-17",
   "Statement": [
     {
       "Effect": "Allow",
       "Action": ["ssm:GetParameter", "ssm:GetParameters"],
       "Resource": [
         "arn:aws:ssm:*:*:parameter/DATABASE_URL",
         "arn:aws:ssm:*:*:parameter/CMO_NAME"
       ]
     }
   ]
 }
 ```
        Name this policy `patientping-ssm-access`.
3.  Switch your instance to use the new IAM role:
    1.  Navigate to "EC2" → "Instances" in the AWS Console.
    2.  Select `patientping-web-v2`.
    3.  Click "Actions" → "Security" → "Modify IAM role."
    4.  Select `patientping-monitoring-role` and click "Update IAM role."
4.  Set up CloudWatch Agent:
    1.  SSH into your instance and install CloudWatch Agent:
```bash
ssh patientping
sudo dnf upgrade
sudo dnf install amazon-cloudwatch-agent
```
    2.  Create/open a new CloudWatch Agent config file in the terminal text editor of your choice (e.g. `nano` or `vim`):
	
```bash
sudo nano /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.d/patientping-monitoring.json
```
    2.  Paste the following content into the config file. This will track a log file (`patientping.log`), as well as collecting metrics on memory and swap usage, with an interval of 60 seconds.

```json
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "root"
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/patientping.log",
            "log_group_name": "patientping-monitoring",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  },
  "metrics": {
    "append_dimensions": {
      "AutoScalingGroupName": "${aws:AutoScalingGroupName}",
      "ImageId": "${aws:ImageId}",
      "InstanceId": "${aws:InstanceId}",
      "InstanceType": "${aws:InstanceType}"
    },
    "metrics_collected": {
      "mem": { "measurement": ["mem_used_percent"] },
      "swap": { "measurement": ["swap_used_percent"] }
    }
  }
}
```
    3.  Save and exit the file (Ctrl+X, Y, Enter in `nano`; [good luck](https://stackoverflow.com/questions/11828270/how-do-i-exit-vim) in `vim`).
    4.  Start and enable the CloudWatch Agent service:
	
  ```bash
  sudo systemctl start amazon-cloudwatch-agent
  sudo systemctl enable amazon-cloudwatch-agent
  ```
  
    3.  Re-run the web server and redirect the logs to the file:
        
```bash
cd ~/patientping-web
PYTHONUNBUFFERED=1 uv run patientping.py 2>&1 | sudo tee -a /var/log/patientping.log
```

    4.  Navigate to the CloudWatch Console in AWS → "Logs" → "Log Management."
    5.  Click on your `patientping-monitoring` log group, then click on the log stream with your instance ID. **You should see your server's "PatientPing listening on..." log messages streaming in!**

## Tip

If you'd rather deal with the IAM role and policy via the CLI, here's some command structure to get you started:

```sh
# Create IAM role with trust policy
aws iam create-role --role-name ROLE-NAME --assume-role-policy-document file://TRUST-POLICY.json

# Create and attach inline permissions policy
aws iam put-role-policy --role-name ROLE-NAME --policy-name POLICY-NAME --policy-document file://POLICY.json

# Create IAM instance profile and attach role
aws iam create-instance-profile --instance-profile-name PROFILE-NAME
aws iam add-role-to-instance-profile --instance-profile-name PROFILE-NAME --role-name ROLE-NAME

# Associate instance profile with EC2 instance
aws ec2 associate-iam-instance-profile --instance-id EC2-INSTANCE-ID --iam-instance-profile Name=PROFILE-NAME
```
