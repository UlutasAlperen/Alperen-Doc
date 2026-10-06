---
title: "ecs-task-definitions"
weight: 580
---

# ECS Task Definitions

With our **container image** in **ECR** and our **Fargate**-ready **ECS Cluster** set up, we need to tell ECS exactly how to run our container. What ports should it expose? How much memory does it need? What permissions should it have?

We answer all these questions with a [**Task Definition**](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definitions.html). It's a JSON file that describes everything ECS needs to know about running our container, for example:

```json
{
  "family": "patientping-ecs",
  "cpu": "256",
  "memory": "512",
  "networkMode": "awsvpc",
  "executionRoleArn": "arn:aws:iam::<MY_ACCOUNT_ID>:role/patientping-ecs-execution-role",
  "taskRoleArn": "arn:aws:iam::<MY_ACCOUNT_ID>:role/patientping-ecs-task-role",
  "runtimePlatform": { "operatingSystemFamily": "LINUX" },
  "requiresCompatibilities": ["FARGATE"],
  "volumes": [],
  "containerDefinitions": [
    {
      "name": "patientping-ecs",
      "image": "<MY_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/patientping-ecs",
      "portMappings": [
        {
          "containerPort": 8000,
          "hostPort": 8000
        }
      ],
      "essential": true,
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/patientping-ecs",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

| Property                                | Description                                                                                   |
| --------------------------------------- | --------------------------------------------------------------------------------------------- |
| `.family`                               | The name of the task definition                                                               |
| `.cpu`                                  | The amount of CPU to allocate to the task, in units of roughly 1/1024th of a core             |
| `.memory`                               | The amount of memory to allocate to the task, in MB                                           |
| `.networkMode`                          | The network mode to use for the task; `awsvpc` assigns an IP address directly within the VPC. |
| `.executionRoleArn`                     | This is an IAM role that's used **while the task is booting up**.                             |
| `.runtimePlatform`                      | The operating system family to use for the task                                               |
| `.requiresCompatibilities`              | The compute resources that the task can use                                                   |
| `.volumes`                              | Optional volume configuration for the task.                                                   |
| `containerDefinitions.name`             | The name of the container                                                                     |
| `containerDefinitions.image`            | The container image to run                                                                    |
| `containerDefinitions.portMappings`     | The ports to expose from the container                                                        |
| `containerDefinitions.essential`        | If `true`, the task will try to recover if this container exits                               |
| `containerDefinitions.logConfiguration` | The logging configuration for the container (more details on this later)                      |
|                                         |                                                                                               |

## Example

The cluster is ready, but ECS doesn't know what to run or how. The task definition is the recipe: which image, how much CPU and memory, which IAM roles, where to send logs. Our app reads `/CMO_NAME` from SSM at runtime and returns it in the response when available.

**Create a task definition for your ECS container** with family and container name `patientping-ecs` to match the cluster and image from the previous lessons.

**Cost check:** ECS task definitions don't cost anything. You only pay when tasks are actually running.

1. [ ] Open the AWS Console, navigate to `Elastic Container Service`, then open `Task definitions`.
2. [ ] Click **Create new task definition**, then choose **Create new task definition with JSON** and paste in this JSON:
    
    ```json
    {
      "family": "patientping-ecs",
      "cpu": "256",
      "memory": "512",
      "networkMode": "awsvpc",
      "executionRoleArn": "<EXECUTION_ROLE_ARN>",
      "taskRoleArn": "<TASK_ROLE_ARN>",
      "runtimePlatform": { "operatingSystemFamily": "LINUX" },
      "requiresCompatibilities": ["FARGATE"],
      "volumes": [],
      "containerDefinitions": [
        {
          "name": "patientping-ecs",
          "image": "<IMAGE_URI>",
          "portMappings": [
            {
              "containerPort": 8000,
              "hostPort": 8000
            }
          ],
          "essential": true,
          "logConfiguration": {
            "logDriver": "awslogs",
            "options": {
              "awslogs-group": "/ecs/patientping-ecs",
              "awslogs-region": "us-east-1",
              "awslogs-stream-prefix": "ecs"
            }
          }
        }
      ]
    }
    ```
3.  Update the 3 account-specific values before creating:
    -  Set `image` to your ECR image URI for `patientping-ecs`.
    -  Set `executionRoleArn` to your `patientping-ecs-execution-role` ARN.
    -  Set `taskRoleArn` to your `patientping-ecs-task-role` ARN.
4.  Click **Create** to register the task definition.
    
    If you typo something, you can delete the task definition by "deregistering" all individual versions for that specific task definition. Then after a minute or two AWS will clean up Task definitions with no available versions registered.
5.  Verify `patientping-ecs` appears in the task definition list with a new revision.