---
title: "virtual-private-cloud-vpc"
weight: 10
---

# Virtual Private Cloud (VPC)

we need a network "box" to put cloud resources into. AWS calls this box a **VPC**, or [Virtual Private Cloud](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html).

- We choose what _region_ this virtual network will be in.
- We choose _how many things_ we plan to put into it, i.e., the size of the address space.

This box (the VPC) is really just a group of IP addresses within a region we choose.

Sometimes we want multiple VPCs, for example to separate production resources from development servers. Or we might want some servers in a primary location and others in a backup region.

![Searching for the VPC service in the AWS console](/images/aws/vpc-console-search.png)

![](https://storage.googleapis.com/qvault-webapp-dynamic-assets/course_assets/Gswk81B-970x760.png)

Running out of addresses is not fun. The only thing worse is [bike shedding](https://en.wikipedia.org/wiki/Law_of_triviality) with a fellow engineer about what "the perfect amount" is. It's best to aim for what you need _plus a bit of a buffer_. Adding new addresses later isn't as annoying as it used to be.

## PatientPing

During this documents, the infrastructure we create will be for a fictional company you've recently joined called **PatientPing**. It's a patient management application for local doctors and dentists. It allows them to keep track of appointments, schedule follow-ups, send automated reminders, etc.

## Assignment

**Cost check:** There are no costs associated with empty VPCs.

**Create a VPC named `patientping` with CIDR `10.0.0.0/22`.**

1. In the AWS Console, ensure your region is set to "N. Virginia (us-east-1)" using the dropdown in the top-right corner.

2. Navigate to the "VPC" section using the search bar at the top.

![](https://storage.googleapis.com/qvault-webapp-dynamic-assets/course_assets/YYMbsFv-1280x416.png)

2. Select "Your VPCs" in the left-hand menu. You'll notice that you already have a "default" VPC, but we'll be making a dedicated one for this document.

3. Click "Create VPC" and use the following settings:

  - "VPC Only"
  - Set `patientping` as the name tag
  - Use an [IPv4 CIDR](https://aws.amazon.com/what-is/cidr/) of `10.0.0.0/22`
  - Leave all other default settings
2. Click the "Create VPC" button at the bottom of the form to finalize it.
  
3. Go back to the "Your VPCs" list and verify that the new `patientping` VPC is there.
## Tip

If you want to use the AWS CLI instead:

```sh
aws ec2 create-vpc \
  --cidr-block '10.0.0.0/22' \
  --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=patientping}]"
```