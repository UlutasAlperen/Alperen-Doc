---
title: "aws-launch-template"
weight: 90
---

# Launch Template

A [launch template](https://docs.aws.amazon.com/autoscaling/ec2/userguide/launch-templates.html) is a pre-baked configuration for EC2 instances.

As you know, an AMI is an image of the software on the server, but a launch template goes one step further: it also captures the "hardware" (instance type), the network configuration (subnet, security groups), and more. It's a way to say, "This is the _exact_ configuration I want for new servers."

## Assignment

**PatientPing is eventually going to need a bunch of servers. Create a launch template that uses the AMI you created**.

**Cost check:** Launch templates are free. AMIs store as EBS snapshots; you pay a small amount for snapshot storage. You only pay for instances launched from the template.

1.  Go to "EC2" → "Launch Templates" in the AWS Console.
2.  Click "Create launch template."
3.  Configure the template:
    -  **Name:** `patientping-web-launcher`
    -  **Description:** `Launch template for t3.micro with PatientPing app preinstalled`
    -  **AMI:** under "My AMIs," select your `patientping-web-base` AMI (the one you created a few lessons back)
    -  **Instance type:** `t3.micro`
    -  **Key pair:** select your existing `patientping-key`
    -  **Network settings:**
        -  **Subnet:** select `patientping-public-a`
        -  **Availability Zone:** leave unset (the subnet determines the AZ)
        -  **Common security groups:** select `patientping-public`
    -  **Storage:** leave default (`8 GiB`, `gp3`)
4.  Click "Create launch template."
5.  Return to the list of launch templates and confirm `patientping-web-launcher` is there.