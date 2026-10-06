---
title: "inline-policies-iam-user"
weight: 30
---

# Inline Policies

[Policies](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html) **allow** or **deny** actions on specific resources. They look something like this:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow", // <--- 1. Are we allowing or denying?
      "Action": [
        "ec2:Describe*" // <--- 2. Which API actions are affected?
      ],
      "Resource": "*" // <--- 3. Which targets are affected?
    }
  ]
}
```

A policy has three main parts:

- **Effect:** `Allow` or `Deny`. Are we permitting or denying these actions?
- **Action:** The action to allow or deny. There are [thousands](https://github.com/awsles/AwsServices/blob/master/AwsServiceActions.txt) of these, like `DescribeInstances`, `CreateBucket`, or `PutObject`.
- **Resource:** Which resources the action targets. For example, "all EC2 servers," or perhaps the [ARN](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference-arns.html) of just a specific one.

## Assignment

Vincent Vega needs to see EC2 instances for a dashboard he's building, but make sure he can't break anything.

**Add an inline read-only policy `patientping-ec2-readonly` (EC2 Describe only) to `patientping-admin-vinny`.**

**Cost check:** Creating IAM policies is free; costs arise from service usage.

_Let's add this policy the wrong way (inline) first, and we'll fix it in the next lesson_.

1.  Navigate to "IAM" → "Users" in the AWS Console.
2.  Select the `patientping-admin-vinny` user.
3.  Go to the "Permissions" tab and click the "Add permissions" dropdown, then "Create inline policy."
4.  In the policy editor, switch to JSON input, _delete all the starter text_, and paste the following:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ec2:Describe*"],
      "Resource": "*"
    }
  ]
}
```

5.  Click the "Next" button at the bottom.
6.  Name the policy `patientping-ec2-readonly`. Then, if everything looks right, click "Create policy" to finalize.
7.  Go back to the `patientping-admin-vinny` user, and verify in the "Permissions" tab that the new inline policy was added.

**Run and submit** the CLI tests.

## Tip

Here's how to create an inline policy with the AWS CLI (assuming you've prepared a JSON file for the policy):

```sh
aws iam put-user-policy --user-name patientping-admin-vinny --policy-name patientping-ec2-readonly --policy-document file://POLICY-FILE.json
```