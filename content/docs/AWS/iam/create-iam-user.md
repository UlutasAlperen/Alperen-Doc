---
title: "create-iam-user"
weight: 30
---

# IAM Users

Let's make a new **user**! Each user gets a **username**/**password** or **API keys** to access AWS.

Users are **long-lived identities**: they stick around until you delete them. Once you create a user and give them credentials, those credentials will work until they expire or you revoke them.

Managing more than a handful of users manually is painful. If your company already uses something like Azure's [Active Directory](https://www.microsoft.com/en-us/security/business/identity-access/microsoft-entra-id) or [Google Workspace](https://workspace.google.com/), you can use [AWS Identity Center](https://aws.amazon.com/iam/identity-center/) to integrate with those services and manage users more easily.

## Assignment

HR just onboarded Vincent Vega to the PatientPing dev team! Definitely **do not** give him root access to the AWS account. Instead, start by giving him his own IAM user, with no console login or permissions yet. Standard corporate hygiene.

**Create an IAM user named `patientping-admin-vinny` (no console access, no policies yet).**

**Cost check:** Creating IAM users is free; costs arise from service usage (EC2, S3 etc.), not from IAM.

1.  Navigate to "IAM" using the search bar at the top of the AWS Console.
2.  Click "Users" in the left menu, then "Create user."
3.  Set the user name to `patientping-admin-vinny`. Do _not_ provide console access. Click "Next."
4.  On the "Set permissions" step, don't do anything; just click "Next" again.
5.  Review the new user details, then click "Create user."
6.  Back in the list of users, confirm that `patientping-admin-vinny` is now present.


## Tip

Want to use the CLI instead? Here's the command structure:

```sh
aws iam create-user --user-name patientping-admin-vinny
```