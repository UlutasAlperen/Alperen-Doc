---
title: "route-tables"
weight: 60
---

# Route Tables

We've attached an internet gateway to our VPC, but nothing tells the subnets it exists or when to use it.

To do that we need a **route**, and routes live in the creatively named **route table**. It is literally just a list of routes.

We can add a special kind of route, a "default route," to our table. It is a catch-all rule that says:

> "If you don't have a specific address for something, look here."

This lets us create rules for things we care about (like local traffic). If we _don't_ have a specific rule for something, we send it to the internet. When we're done with this lesson, our VPC will look like this:

![](https://storage.googleapis.com/qvault-webapp-dynamic-assets/course_assets/pYyDT6S-899x632.png)

## how to create to gateway for internet access

**Create a route table that sends public subnet traffic to the internet gateway.**

1.  Navigate to "VPC" using the search bar in the AWS Console.
2.  Click on "Route tables" in the left-hand menu.
3.  Click the orange "Create route table" button at the top right.
4.  Name the new route table `patientping-public-rt`, select our `patientping` VPC, and click "Create route table."
5.  Click the "Actions" dropdown button at the top right, then click "Edit subnet associations."
6.  Select both `patientping-public-a` and `patientping-public-b` (and only those!), then click "Save associations."
7.  Back in the list of route tables, select `patientping-public-rt` again, and click "Actions" → "Edit routes."
8.  Click "Add route."
9.  Set the **destination** to `0.0.0.0/0`. **This "everything CIDR" is what makes it the default route.**
10. For the **target**, select "Internet Gateway." It should open a new input field, where you can select the gateway you just created (`patientping-igw`).
11. Click "Save changes."

Whew! Finally, we have "public" subnets that can connect to the internet.

Take a moment to review your "Internet gateways" and "Route tables" in the VPC dashboard.

## Tip

If you want to use the CLI instead, here's the command structure:

```sh
# Create a route table
aws ec2 create-route-table --vpc-id <vpc-id> --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=<name>}]'
# Associate the route table with a subnet (run once per public subnet)
aws ec2 associate-route-table --route-table-id <rt-id> --subnet-id <subnet-id>
# Add a route to the route table
aws ec2 create-route --route-table-id <rt-id> --destination-cidr-block 0.0.0.0/0 --gateway-id <igw-id>
```