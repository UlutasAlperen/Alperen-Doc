---
title: "iam-groups"
weight: 40
---

# IAM Groups

While inline policies are attached directly to a single user, the _preferred_ way to manage permissions in AWS is to create **groups** and give _them_ the policies (sometimes called [customer-managed policies](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_managed-vs-inline.html#customer-managed-policies)).

This [DRYs](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) up your permissions management and makes it much easier to ensure that everyone has only the access levels they need.

Let's convert our inline policy into a customer-managed policy, then attach it to a group.

## Assignment

PatientPing's CISO has "thoughts" about our inline policies. Per last week's security review: _no more inline policies on users_.

**Create a customer-managed policy `patientping-ec2-readonly` and a group `patientping-ec2-readers`. Attach the policy to the group, add `patientping-admin-vinny` to the group, and remove the inline policy from the user.**

**Cost check:** Creating IAM groups and policies is free; costs arise from service usage.

1.  Create a customer-managed policy `patientping-ec2-readonly`:
    1.  Navigate to "IAM" → "Policies" in the AWS Console.
    2.  Click "Create policy."
    3.  In the policy editor, switch to JSON input, _delete all the starter text_, and paste the following:
	
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

2.  Click "Next" at the bottom.
3.  Name the policy `patientping-ec2-readonly`, then finalize by clicking "Create policy."

4.  Create a user group and attach the policy:
    1.  Navigate to "IAM" → "User groups" in the AWS Console.
    2.  Click the "Create group" button at the top right.
    3.  Name the new group `patientping-ec2-readers`.
    4.  Attach the `patientping-ec2-readonly` policy (use the search field to find it).
    5.  Click "Create group."
5.  Add `patientping-admin-vinny` to the group:
    1.  Navigate to "IAM" → "User groups" in the AWS Console.
    2.  Select the `patientping-ec2-readers` group and go to its "Users" tab.
    3.  Click "Add users," select `patientping-admin-vinny`, and click the "Add users" button.
6.  Remove the inline policy from `patientping-admin-vinny`:
    1.  Navigate to "IAM" → "Users" in the AWS Console.
    2.  Select `patientping-admin-vinny` and go to the "Permissions" tab.
    3.  Under "Permissions policies," select the **inline policy** (it should be clear which one that is).
    4.  Click "Remove" and confirm.

## Tip

For CLI users, here's how to create IAM groups, policies, and manage group membership:

```sh
# Create a customer-managed policy
aws iam create-policy --policy-name POLICY-NAME --policy-document file://POLICY-FILE.json

# Create a group
aws iam create-group --group-name GROUP-NAME

# Attach policy to group
aws iam attach-group-policy --group-name GROUP-NAME --policy-arn POLICY-ARN

# Add user to group
aws iam add-user-to-group --group-name GROUP-NAME --user-name USER-NAME

# Remove the inline policy from the user
aws iam delete-user-policy --user-name USER-NAME --policy-name POLICY-NAME
```