---
title: "use-ssm-from-ec2"
weight: 90
---

# Use SSM from EC2

At this point, everything should be in place for our Pinger server to pull its config values from SSM:

- We created the params `/DATABASE_URL` and `/CMO_NAME` in the SSM Parameter Store.
- We added an IAM policy `patientping-ssm-access`, allowing those params to be read.
- That IAM policy is attached to the role `patientping-ec2-readonly-role`.
- Our EC2 instance that runs the Pinger app is authorized to assume that IAM role.

And the Pinger app code is already configured to try to pull config from SSM before falling back to a `.env` file. So once we restart the app, it should "just work."

The key idea here is that we can give one AWS resource (an EC2 instance) access to another resource (SSM params) by allowing it to assume an IAM role that has a relevant policy attached.

It may seem like a lot of ceremony, but this system allows _fine-grained, auditable control_ over who gets access to what in an AWS account. IAM roles and permissions can be updated or revoked at any time, with changes taking effect almost immediately.

## Assignment

**Restart the Pinger server on your EC2 instance, and confirm that the `DATABASE_URL` and `CMO_NAME` values are now coming from SSM.**

If you're having connection issues, double-check that you stored the **full connection string** in SSM (including the `postgresql://` prefix, username, password, and database name) - not just the RDS endpoint hostname. The `DATABASE_URL` value should look like: `postgresql://postgres:PASSWORD@hostname:5432/patientping`

1. [ ] SSH into your `patientping-web-v2` EC2 instance.
2. [ ] Start the app server. If it was already running, stop it and restart it.
    
    ```sh
    cd ~/patientping-web
    uv run patientping.py
    ```
    
3. [ ] On startup, if the SSM params were fetched successfully, the following text should be printed to the console. Otherwise you'll see an error message.
    
    ```text
    Loaded DATABASE_URL and CMO_NAME from SSM (us-east-1)
    ```
    
4. [ ] Open the Pinger web app (`http://EC2.PUBLIC.IP.ADDR:8080`) in a browser. You should also now see the following line in the header:
    
    ```text
    PatientPing Chief Medical Officer: Dr. Strangelove
    ```
    
5. [ ] Back in your SSH session on the EC2 instance, delete the `.env` file from the `patientping-web` directory; it isn't needed anymore.

**Run and submit** the CLI tests.

## Tip

If you want to verify the SSM params and the IAM role and policy via CLI, here are some commands to get you started:

```sh
# From your computer, logged into the AWS CLI as an admin user on your account,
# you can view SSM params directly
aws ssm get-parameter --name /DATABASE_URL
aws ssm get-parameter --name /CMO_NAME

# List inline policies attached to a role, e.g., "patientping-ssm-access"
aws iam list-role-policies --role-name patientping-ec2-readonly-role

# Show the details of an inline policy, given role and policy names
# Effect: "Allow"; Actions: "ssm:GetParameter" and "ssm:GetParameters"
# Resources: /DATABASE_URL and /CMO_NAME
aws iam get-role-policy --role-name patientping-ec2-readonly-role --policy-name patientping-ssm-access

# From an SSH session on your EC2 instance, run this to see its role
# The "Arn" value should have "assumed-role/patientping-ec2-readonly-role"
aws sts get-caller-identity
```