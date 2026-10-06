---
title: "ecs-target-groups"
weight: 100
---

# Target Groups

Now it's time to configure the load balancer to actually route traffic to our ECS tasks (instead of just returning its own fixed response). This is where [**Target Groups**](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html) come in.

A **target group** tells the load balancer where to send traffic. It defines:

- **Protocol and port:** What protocol (HTTP/HTTPS) and which port your tasks are listening on
- **Health checks:** How the load balancer determines if a target is healthy
- **Target type:** Whether targets are IP addresses (for Fargate) or instance IDs (for EC2)

For ECS with Fargate, we'll use the `ip` target type because each task gets its own IP address. The load balancer will route traffic directly to the task's IP address.

## Health Checks

Target groups perform health checks to determine if targets are healthy and ready to receive traffic. The health check configuration includes:

- **Health check path:** The endpoint to check (e.g., `/` or `/health`)
- **Health check interval:** How often to check (e.g., every 30 seconds)
- **Health check timeout:** How long to wait for a response (e.g., 5 seconds)
- **Healthy threshold:** How many consecutive successful checks before marking as healthy (e.g., 2)
- **Unhealthy threshold:** How many consecutive failed checks before marking as unhealthy (e.g., 2)

If a target fails its health checks, the load balancer stops sending traffic to it. Once it passes health checks again, traffic resumes automatically.

Health checks are crucial for reliability. If a container crashes or becomes unresponsive, the load balancer will detect it and stop routing traffic to it. This prevents users from hitting broken containers. Remember, ECS can run _many_ backing containers!

## Assignment

**Create a target group named `patientping-tg`** and update the load balancer listener to forward traffic to it instead of the static response.

1.  Create the target group from the console:
    1.  In the AWS Console, navigate to `EC2`, then open `Target Groups` under `Load Balancing`.
    2.  Click **Create target group**.
    3.  For **Target type**, select **IP addresses**.
    4.  Set **Target group name** to `patientping-tg`.
    5.  For **Protocol** and **Port**, use **HTTP** and `8000` (the port your container listens on).
    6.  Select your `patientping` VPC.
    7.  Under **Health checks**, keep the **Path** as `/` and leave the other defaults (interval, timeout, thresholds) as-is, or mirror the values shown in the lesson if you want to practice tuning them.
    8.  Click **Next**, then **Create target group**. _We won't register targets yet_.
2.  Update the existing listener on `patientping-alb` to forward to this target group:
    1.  In the AWS Console, navigate to `EC2` → `Load Balancers`, and select `patientping-alb`.
    2.  Go to the **Listeners** tab and click on the HTTP (port 80) listener.
    3.  Click **Edit** (or **View/edit rules**) and change the **Default action** from **Return fixed response** to **Forward to target groups**.
    4.  Choose the `patientping-tg` target group you just created.
    5.  Save the changes.

The target group is now configured, but we don't have its _targets_ configured yet. Don't worry, that's coming soon.