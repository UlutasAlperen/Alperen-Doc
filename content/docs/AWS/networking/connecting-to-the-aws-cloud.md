---
title: "connecting-to-the-aws-cloud"
weight: 70
aliases:
  - /docs/aws/networking/Connecting-to-the-AWS-Cloud/
---

# Connecting to the AWS Cloud

With so many different types of networks, on-premises datacenters, and remote workers, companies need a wide range of ways to connect to the AWS Cloud. In the following section, you will learn four ways to connect to the AWS Cloud:

- AWS Client VPN

- AWS Site-to-Site VPN

- AWS PrivateLink

- AWS Direct Connect

![Internet gateway traffic flow](/images/aws/connect_vpc.png)


## Securely connect a remote workforce to AWS Cloud resources

Imagine a company with a recent acquisition needing to securely connect their new remote workforce to their AWS Cloud resources. Even the largest companies with worldwide remote workers can quickly scale up and connect to the AWS Cloud. That's where AWS Client VPN can help.

![aws client vpn](/images/aws/aws_connect_vpn.png)


## Securely connect sites to other sites

Some companies might want to establish secure, encrypted connections between their on-premises networks like data centers or branch offices and their resources in their Amazon VPC. That's where Site-to-Site VPN can help.

![aws-site-to-site-vpn](/images/aws/aws-site-to-site-vpn.png)

## Securely connect resources, even in other VPCs

Other companies sometimes need the flexibility to privately connect to resources in other cloud providers as though they were in their own VPC. They need a way to communicate with these resources and don't want the hassle of setting up gateways or site-to-site VPNs. That's where AWS PrivateLink can help.

![aws_private_link](/images/aws/aws_private_link.png)

## Dedicated private connections for increased bandwidth

![aws_direct_connect](/images/aws/aws_direct_connect.png)

![aws_direct_connect](/images/aws/aws_direct_connection_2.png)
