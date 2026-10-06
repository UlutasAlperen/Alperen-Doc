---
title: "private-subnets"
weight: 60
---

# Private Subnets

A subnet becomes _public_ by connecting it to a route table and an internet gateway, but what about our _private subnets_?

An IGW would make the subnet non-private and expose it to the internet. That is too risky for some resources.

On the other hand, we don't want to _completely_ cut off private infrastructure from internet access. Using a floppy disk to download updates is not exactly an option in the cloud.

A private subnet is a subnet whose resources are not accessible from the internet _directly_, but can still have _some_ internet connectivity. AWS gives us a few options to do that.

We "could" leave everything in a public subnet and regulate access with a firewall. However, I prefer a [belt and suspenders](https://en.wiktionary.org/wiki/belt_and_suspenders) approach to security for multiple layers of protection. One good option is to use a NAT ([Network Address Translation](https://en.wikipedia.org/wiki/Network_address_translation)), which lets us keep a subnet private **but still allow outbound access** when we need it.

![700](https://storage.googleapis.com/qvault-webapp-dynamic-assets/course_assets/BcflIGg-883x630.png)

Almost certainly, your computer right now is using a NAT. Home, coffee shop, office, whatever. Your router is "lending" you its public IP address when you talk to resources outside the local network.

## Assignment

**Cost check:** [AWS NATs can be expensive](https://www.lastweekinaws.com/blog/the-aws-managed-nat-gateway-is-unpleasant-and-not-recommended/). We won't deploy one fully, but we'll do everything up to that point.

**PatientPing doesn't have the budget for a NAT right now, but we do want a private route table. Create one and attach it to our private subnets.**

1.  Navigate to the VPC dashboard in the AWS Console.
2.  Click on "Route tables" in the left-hand menu.
3.  Click the orange "Create route table" button at the top right.
4.  Name the new table `patientping-private-rt`, select the `patientping` VPC, and click "Create route table."
5.  Click the "Actions" dropdown button at the top right and select "Edit subnet associations."
6.  Select both `patientping-private-a` and `patientping-private-b`, then click "Save associations."

We can add internal routes later; we just needed to make sure the private subnets **don't use the public route table**.
