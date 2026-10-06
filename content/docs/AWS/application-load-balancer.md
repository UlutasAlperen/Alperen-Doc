---
title: "application-load-balancer"
weight: 600
---

# Application Load Balancer

I know that it looks like we're working on anything _but_ `ECS` and `containers` at the moment, but I promise we're getting closer. We just have to [shave a few more yaks](https://en.wiktionary.org/wiki/yak_shaving).

Now it's time to start listening for traffic. To do that, we'll set up a **Load Balancer** and a [**Listener**](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html).

![listener diagram](https://storage.googleapis.com/qvault-webapp-dynamic-assets/course_assets/GQ1RvYY-638x721.png)

A [load balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/what-is-load-balancing.html) sits in front of our service and distributes incoming traffic across your task(s). It handles things like health checks, SSL termination, and routing traffic to healthy containers. We'll place the load balancer in public subnets so it can receive traffic from the internet.

To make this work, we need several pieces:

1. **Load Balancer:** The actual [ALB](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html) that receives traffic
2. **Listener:** [Rules](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html) that tell the load balancer what to do with incoming traffic
3. **Target Group:** These tell the Listener where to send that traffic (more on this next lesson).

## Assignment

**Create an Application Load Balancer named `patientping-alb`** with a listener that returns a static response.

**Cost check:** This is the part that will cost money. An ALB costs about **`$0.0225` per hour (~`$16`/mo)** plus data transfer costs. This is an important part to tear down when we tell you to.

1. Create the ALB from the console:
    
    1.  In the AWS Console, navigate to `EC2`, then open `Load Balancers` under `Load Balancing`.
    2.  Click **Create load balancer** and choose **Application Load Balancer**.
    3.  Name it `patientping-alb`.
    4.  Set the **Scheme** to **Internet-facing** and **IP address type** to **IPv4**.
    5.  Under **Network mapping**, select your `patientping` VPC and choose both public subnets (`patientping-public-a` and `patientping-public-b`) in the appropriate availability zones.
    6.  Under **Security groups**, switch to the `patientping-external` security group.
    7.  Under **Listeners and routing**, keep a single HTTP listener on port `80`. For now, set the **Default action** to **Return fixed response**, with:
        - **Status code**: `200`
        - **Content type**: `text/plain`
        - **Response body**: `Load balancer is working!`
    8.  All other defaults are fine, scroll to the bottom and click **Create load balancer** and wait for its state to become **active**.
2. Grab the "DNS name" from the load balancer's page. Once the ALB is active (no longer "provisioning"), visit `http://<DNS name>` in your browser.
    
    We've only configured an HTTP listener, so include the `http://` prefix. You should see the message "Load balancer is working!"