---
title: "ecs-services"
weight: 630
---

# ECS Services

Finally! Time to build the [**ECS Service**](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs_services.html) that ties everything together, and answers these questions:

- Where do we place our tasks?
- How do we scale them?
- How do we connect to the load balancer?

Once we create an ECS service with load balancer settings, ECS automatically registers tasks with the [target group](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html) as they start.

## Assignment

**Create an ECS service `patientping-svc`** that runs your task definition and connects it to the load balancer (cluster `patientping-ecs`, ALB `patientping-alb`, target group `patientping-tg`).

**Cost check:** An ECS service creates the serverless containers (Fargate tasks). It costs about `$0.012/hr` for the task. The load balancer we created earlier costs about `$0.0225/hr` (~`$16/mo`). Combined, you're looking at roughly `$0.0345/hr` or about $25/mo total! Don't leave these up if you're going to step away from the lessons.

1.  Create the ECS service from the console:
    1.  In the AWS Console, navigate to `Elastic Container Service`, open `Clusters`, then click on `patientping-ecs`.
    2.  On the cluster page, go to the **Services** tab and click **Create**.
    3.  Under **Service details**, set:
        - **Family** (task definition) to `patientping-ecs`.
        - **Service name** to `patientping-svc`.
    4.  Under **Environment**, use **Capacity provider strategy** with the `FARGATE` capacity provider.
    5.  Under **Networking**:
        -  Choose the `patientping` VPC
        -  Select both public subnets (`patientping-public-a` and `patientping-public-b`)
        -  Select the `patientping-internal` security group, and set **Public IP** to **Turned on**.
            
            Now you may wonder why we're using a public subnet. Well, when a new Fargate container starts, it needs to be able to interact with the container repository (ECR) and pull in a fresh container. Putting this in a public subnet means it can just leverage our internet gateway, and save us from setting up a specific endpoint to fetch them privately.
            
    6.  Under **Load balancing**, enable **Application Load Balancer**, "use existing":
        -  Container: `patientping-ecs`
        -  Use existing load balancer: `patientping-alb`
        -  Use existing listener: `HTTP:80` (the one that routes traffic to the target group)
        -  Use existing target group: `patientping-tg`
    7.  Create the service.
        
        It may take a minute or two for the service to start tasks and register them with the load balancer. Once they're registered, your containers will be ready to receive traffic through the load balancer.
        
2.  Once its online, verify the service by testing the endpoint:
    1.  In the **Services** tab of the `patientping-ecs` cluster, confirm that `patientping-svc` shows `1/1` running tasks.
    2.  In the AWS Console, navigate to `EC2` → `Load Balancers`, select `patientping-alb`, and copy the **DNS name**.
    3.  Visit `http://<DNS_NAME>` in your browser. (If it hasn't come online yet, you should get a `503` from the ALB).
    4.  Once online, confirm that the service serves up a `Hello from the container! From Dr. Strangelove.` response.
        
        Remember, we set this service up with permissions to read the `/CMO_NAME` from SSM parameters!
        
