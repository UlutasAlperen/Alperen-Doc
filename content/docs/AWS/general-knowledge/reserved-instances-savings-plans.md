---
title: "reserved-instances-savings-plans"
weight: 10
---

# Reserved Instances and Savings Plans

Companies can spend a _lot_ of money on cloud servers, and part of building on AWS is optimizing your spend and getting the most out of it.

Initially, AWS allowed you to ["reserve" an instance](https://aws.amazon.com/ec2/pricing/reserved-instances/). This meant:

- You commit to paying for at least a certain amount of compute on a specific EC2 instance type, for a period of 1-3 years.
- In return for your commitment, AWS gives you a significant discount on that compute.

It's like buying in bulk, but it comes with constraints. The reservation is tied to a specific instance type and region (sometimes even a specific availability zone, or AZ).

More recently, AWS has introduced [Savings Plans](https://aws.amazon.com/savingsplans/), which offer more flexibility while still giving customers similar discounts. In brief:

- You commit to 1-3 years of usage.
- You can change the AZ of your instances (e.g., from your `public-a` subnet to `public-b`).
- You can change the instance type, within a "family" (e.g., the `t3` in `t3.small` and `t3.medium`).

Savings can range from **26%** to **72%**! In many cases, turning off a server every night after business hours _saves less money_ than just adding a savings plan and running the thing 24/7. If you're sure you're going to run a server for a year, you're wasting money by not having a savings plan.

## Next

If you want to save _even more_, see [spot-instances](spot-instances/) - unused AWS capacity at a ~90% discount, with the caveat that AWS can reclaim it at any time.
