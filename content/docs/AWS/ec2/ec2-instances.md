---
title: "ec2-instances"
weight: 20
---

# EC2 Instances

With a VPC, subnets, and a key pair in hand, we're ready to launch our first AWS **server instance**.

**Elastic Cloud Compute** (**EC2**) is AWS' flagship service for running servers on their infrastructure. It's their answer to questions like:

- How do I get a Linux or Windows box in Amazon's cloud?
- How do I provision dedicated hardware?
- How do I build a server with a GPU attached?

Let's create an instance and look at the choices we have to make along the way.

## Assignment

**Launch a `t3.micro` EC2 instance named `patientping-web` in `patientping-public-a`.**

**Cost check:** The Linux server we provision here will cost $0.0104 per hour (in `us-east-1`). That means about $0.25 per day, or $7.60 per month. If you're on the free tier, it will be covered by that.

**Remember to delete this server when you're done with it!**

1.  Navigate to "EC2" using the search bar at the top of the AWS Console.
2.  Click on "Instances" in the left-hand menu.
3.  Click the orange "Launch instances" button at the top of the table.
4.  Set up the server with the following options:
    1.  Add a `Name` tag with a value of `patientping-web`. Note that this is **case-sensitive**.
    2.  Use the default "Amazon Machine Image" (AMI; we'll talk about this later). It should be "Amazon Linux."
    3.  Use `t3.micro` as the instance type. You may have to type the first few characters into search to get it to pop up.
    4.  Select our `patientping-key` key pair.
5.  In the "Networking" section, click "Edit" in order to update a few things.
    1.  Select the `patientping` VPC.
    2.  Select the `patientping-public-a` subnet we created earlier.
    3.  For "Auto-assign public IP," set `Disable`.
    4.  For the "Firewall (security groups)" section, select "Create security group."
    5.  Name the security group (SG) `patientping-empty` (we'll add rules later).
    6.  Set the SG description to `Placeholder security group for PatientPing server` (this is arbitrary, but a description is required).
    7.  Click "Remove" next to any rules that were created by default – there should be an SSH rule to remove.
    8.  Don't bother with "Advanced network configuration."
6.  Storage: leave the default configuration.
7.  Advanced details: leave the default configuration.
8.  Click "Launch instance."
9.  You should see a message like `Success: Successfully initiated launch of instance (i-123123123)`.

Go back to the "Instances" list and confirm the new `patientping-web` server is there.

## Tip

If you want to use the CLI instead, here's the command structure:

```sh
aws ec2 run-instances --image-id AMI-ID --instance-type TYPE --key-name KEY-NAME --subnet-id SUBNET-ID --security-group-ids SG-ID --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=INSTANCE-NAME}]'
```


# AMIs and Instance Types

Let's talk a bit more about the choices we made when launching that EC2 instance.

We chose the ["Amazon Linux"](https://aws.amazon.com/amazon-linux-ami/) AMI ([Amazon Machine Image](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html)) to boot our virtual machine. This contains the baseline software for the server. Most AMIs just include an operating system like Ubuntu Linux or Windows Server 20XX, while others include full software stacks from a vendor.

You can also make your own AMIs, which is helpful if you're going to be repeating the same configuration over and over again (we'll talk about autoscaling later) or moving a server to a different region. It's a full VM image, not just a container (like Docker)!

We then chose an **instance type** to use. This just defines what hardware resources are attached to our server. Think CPU, memory, GPU, the type of processor/networking card, etc.