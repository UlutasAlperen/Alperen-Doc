---
title: "internet-gateways-igw"
weight: 40
---

# Internet Gateways (IGW)

The eagle-eyed among you may have noticed that there is currently no difference between our "public" and "private" subnets.

Anything we put in _either_ type of subnet **cannot** access the internet, and nothing on the internet can access _it_. Our environment looks like this:

![](https://storage.googleapis.com/qvault-webapp-dynamic-assets/course_assets/rJrDW3Z-921x650.png)

There is a gap between our network and the internet. Usually when folks refer to a subnet as _public_, they mean the servers inside can access the internet (egress), and with limitations, the internet can access them (ingress).

Our PatientPing public subnets need:

- An [Internet Gateway](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html) (IGW) that can connect them to the internet
- A [route](https://docs.aws.amazon.com/vpc/latest/userguide/RouteTables.html) that tells servers in these subnets to use that IGW

The route tells outgoing traffic, "Hey, this gateway exists; if you're trying to reach a server that's not in our network, look here."

![Internet gateway traffic flow](/images/aws/internet-gateway-flow.png)

## how to crate IGW to vpc(sanal ag)

**PatientPing servers need access to the internet. Start by attaching an IGW to our VPC.**

**Cost check:** Internet gateways are free. You only pay for data transfer.

1.  Navigate to "VPC" using the search bar in the AWS Console.
2.  Click on "Internet gateways" in the left-hand menu.
3.  Click the orange "Create internet gateway" button at the top right.
4.  Name the new gateway `patientping-igw`, then click "Create internet gateway."
5.  Go back to the list of gateways and select `patientping-igw` by clicking on its checkbox.
6.  Click the "Actions" dropdown button at the top right, then click "Attach to VPC."
7.  Select the `patientping` VPC and click "Attach internet gateway."

We don't have a _route_ yet (we'll take care of that next), but we're one step closer.

## Tip

If you want to use the CLI instead, here's the command structure:

```sh
# Create an internet gateway
aws ec2 create-internet-gateway --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=<name>}]'
# Attach the IGW to a VPC
aws ec2 attach-internet-gateway --internet-gateway-id <igw-id> --vpc-id <vpc-id>
```
