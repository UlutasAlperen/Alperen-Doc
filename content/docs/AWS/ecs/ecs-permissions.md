---
title: "ecs-permissions"
weight: 60
---

# ECS Permissions

We've dealt with [**IAM roles**](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html) with EC2, but they also apply to ECS. We have two problems to solve:

1. We need to give ECS access to specific containers to pull them in and start them up.
2. After the container boots up, it needs to have its own set of permissions. In our case, we'll give it permissions to read SSM parameters.

Two different permission sets, and AWS handles them with:

1. A **Task Execution Role:** Essentially, what permissions does ECS need to start up the task definition?
2. A [**Task Role**](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html): After the task starts up, what permissions does my application running _inside that container_ require?

I've met a lot of engineers confused about this concept, and it's partly because AWS tries to be helpful and "create" roles automatically... _sometimes_.

So an engineer might have tasks running happily for quite a while before they need to think about what roles/permissions they need to give to their tasks.

AWS will happily suggest creating the roles for you in the console, but we'll do it by hand so you can see exactly what's going on.

Remember, every **IAM Role** has a set of **Permissions** (what can be done) and a **Trust Policy** (who can use it).

## Example

**Create the IAM roles and policies needed for ECS tasks.**

**Cost check:** IAM roles and policies are free. You only pay for the resources that use these roles.

1.  Navigate to the "IAM" service in the AWS console.
2.  Select "Policies" from the left-hand menu, and then click "Create policy".
3.  Using the JSON editor, paste in this file that allows for pulling images from ECR and writing logs to CloudWatch:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["logs:CreateLogStream", "logs:PutLogEvents"],
      "Resource": "arn:aws:logs:*:*:log-group:/ecs/patientping-ecs:*"
    }
  ]
}
```

4.  Name the policy `patientping-ecs-execution-policy` and create it.
5.  Create another following the same steps called `patientping-ecs-task-policy` with this JSON so your container can read the SSM parameters you created in `5.7` (`/DATABASE_URL` and `/CMO_NAME`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadSsmParameters",
      "Effect": "Allow",
      "Action": ["ssm:GetParameter", "ssm:GetParameters"],
      "Resource": ["arn:aws:ssm:us-east-1:*:parameter/CMO_NAME"]
    }
  ]
}
```

6.  Create the execution role in IAM:
    -  In `IAM` → `Roles`, click `Create role`.
    -  Trusted entity type: `AWS service`.
    -  Use case: `Elastic Container Service` → `Elastic Container Service Task` → "Next."
    -  Permissions: attach `patientping-ecs-execution-policy`.
    -  Role name: `patientping-ecs-execution-role`.
    -  Click `Create role`.
7.  Create the task role in IAM:
    -  Again, click `Create role`.
    -  Trusted entity type: `AWS service`.
    -  Use case: `Elastic Container Service` → `Elastic Container Service Task`.
    -  Permissions: attach `patientping-ecs-task-policy`.
    -  Role name: `patientping-ecs-task-role`.
    -  Click `Create role`.
8.  Open both roles and verify their trust relationship allows `ecs-tasks.amazonaws.com` to assume the role