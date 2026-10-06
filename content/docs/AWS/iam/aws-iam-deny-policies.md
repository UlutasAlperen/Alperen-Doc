---
title: "aws-iam-deny-policies"
weight: 60
---

# Deny Policies

An explicit `Deny` policy always wins; even if other policies `Allow` the same action. This is your shield when you need to set absolute boundaries.

Follow the [principle of least privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege): grant people and resources only the permissions they actually need, nothing more. So don't "allow all actions by default" and then add `Deny` rules for dangerous actions. Prefer to add permissions as they're needed.

That said, there are situations where `Deny` policies are helpful:

> Something is misbehaving on server 3; can you quickly disable its AWS access?

A deny-all policy is like putting someone in time-out. They can't do anything, no matter what other permissions they might have. Useful for quarantine, break-glass prevention, or stopping a rogue service from causing more damage.

Hackers _love_ getting access to random EC2 instances and **using the permissions to the AWS account they're in**. Sometimes for data exfiltration, sometimes to mine crypto, and sometimes just to cause chaos for your team.

## Assignment

Security drill: assume the EC2 instance is compromised. Lock it down immediately.

**Create a deny-all policy `patientping-deny-all` and attach it to the role `patientping-ec2-readonly-role`.**

**Cost check:** Creating IAM policies is free; costs arise from service usage.

1.  SSH into your `patientping-web-v2` EC2 instance: `ssh patientping`
2.  Run an `aws` command to see if you have access to _read_ information about EC2 instances. **This should work**:

```bash
aws ec2 describe-instances --no-cli-pager
```

3.  Create the deny-all policy:
    1.  Navigate to "IAM" → "Policies" in the AWS Console.
    2.  Click "Create policy."
    3.  In the policy editor, switch to JSON input, _delete all the starter text_, and paste the following:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyEverything",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*"
    }
  ]
}
```
	4.  Click "Next."
	5.  Name the new policy `patientping-deny-all`. Leave everything else as-is, and click "Create policy" at the bottom.
1.  Attach the policy to the role:
    1.  Navigate to "IAM" → "Roles" in the AWS Console.
    2.  Select the `patientping-ec2-readonly-role` that you created earlier.
    3.  Go to the "Permissions" tab and click "Add permissions" → "Attach policies."
    4.  Use the search field to find the new `patientping-deny-all` policy, select it, and click "Add permissions."
2.  Back on the EC2 instance, run the same `aws` command again:
```bash
aws ec2 describe-instances --no-cli-pager
```

This time, you should get an error message indicating that access is denied!

At this point, the role has no effective AWS access (`Deny` **always** wins)!

## Tip

You can accomplish the same thing via CLI. Here's the command structure:

```sh
# Create the deny-all policy
aws iam create-policy --policy-name POLICY-NAME --policy-document file://DENY-POLICY.json

# Attach the policy to a role
aws iam attach-role-policy --role-name ROLE-NAME --policy-arn POLICY-ARN
```