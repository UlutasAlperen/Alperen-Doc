---
title: "accessing-ssm-parameters-from-ec2"
weight: 80
---

# Accessing SSM Parameters from EC2

Storing parameters in the SSM Parameter Store is only half the equation. Your EC2 instances still need **permission** to retrieve them, which they can get through IAM. To allow an instance to access SSM parameters, you need to:

1. Create or attach an IAM policy that grants `ssm:GetParameter` permissions.
2. Attach that policy to the instance's IAM role.

However, doing this for each individual parameter would get tedious, real quick. Instead, we can scope access with parameter names or path namespaces using a policy like this:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowReadSSMParameters",
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

This gives us a pretty flexible system. For example, you could organize parameters using namespaces like:

- A `/global/` namespace for all servers in the environment
- A `/cloudsy/` namespace for all servers that are part of a `cloudsy` application
- An `/env/debug/` namespace for all servers in the `debug` environment

**Cost check:** IAM roles and policies are free. Granting EC2 access to SSM parameters doesn't add cost; you only pay for the EC2 instances (and SSM "advanced parameters," if any).

## Assignment

**Add an inline policy `patientping-ssm-access` to the role `patientping-ec2-readonly-role`. Allow `ssm:GetParameter` and `ssm:GetParameters` for `/DATABASE_URL` and `/CMO_NAME`.**

1.  Navigate to "IAM" → "Roles" in the AWS Console.
2.  Select `patientping-ec2-readonly-role` and go to its "Permissions" tab.
3.  Remove the `patientping-deny-all` policy that was attached to it.
4.  Click "Add permissions" → "Create inline policy."
5.  In the policy editor, switch to JSON input, _delete all the starter text_, and paste the following:

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
6.  Click "Next."
7.  Name the policy `patientping-ssm-access`, then click "Create policy."
8.  Verify that `patientping-ec2-readonly-role` has two policies attached, `patientping-ec2-readonly` (managed) and `patientping-ssm-access` (inline).

## Tip

For CLI users, here's how to create and attach an inline policy:

```sh
aws iam put-role-policy --role-name ROLE-NAME --policy-name POLICY-NAME --policy-document file://POLICY-FILE.json
```