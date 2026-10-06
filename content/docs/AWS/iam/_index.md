---
title: "iam"
weight: 50
bookCollapseSection: true
---

# Identity and Access Management (IAM)

- 001 - [identity-and-access-management](identity-and-access-management/)

At some stage in building a project on AWS, you'll need to give other people (or perhaps automated systems) access to your cloud resources as well. [IAM](https://aws.amazon.com/iam/) is how AWS controls access to resources: who can log in, and what they can do once they're in.

![IAM diagram](/images/aws/iam-diagram.png)

- 002 - [create-iam-user](create-iam-user/) = Each user gets a **username**/**password** or **API keys** to access AWS. Users are **long-lived identities**: they stick around until you delete them. Once you create a user and give them credentials, those credentials will work until they expire or you revoke them.
- 003 - [inline-policies-iam-user](inline-policies-iam-user/) = [Policies](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html) **allow** or **deny** actions on specific resources.
- 004 - [iam-groups](iam-groups/) = While inline policies are attached directly to a single user, the _preferred_ way to manage permissions in AWS is to create **groups** and give _them_ the policies (sometimes called [customer-managed policies](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_managed-vs-inline.html#customer-managed-policies)).
- 005 - [iam-roles](iam-roles/) = IAM has [roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html). A role is a **temporary identity** that can be used or **assumed** by trusted parties.

![IAM roles diagram](/images/aws/iam-roles-diagram.png)

- 006 - [aws-iam-deny-policies](aws-iam-deny-policies/) = explicit Deny her zaman kazanır
- 007 - [aws-ssm-parameter-store](aws-ssm-parameter-store/) = centralized safe where you can keep configuration data, secrets, and parameters that your applications need
- 008 - [accessing-ssm-parameters-from-ec2](accessing-ssm-parameters-from-ec2/) = uzak sunucumun ssm parametlerine iznini vermek
- 009 - [use-ssm-from-ec2](use-ssm-from-ec2/)
