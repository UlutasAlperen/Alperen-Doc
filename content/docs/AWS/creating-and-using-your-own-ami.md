---
title: "creating-and-using-your-own-ami"
weight: 150
---

# Creating and Using Your Own AMI

When we configured our server, we chose one of the out-of-the-box "base AMIs" that AWS provides. However, it's also possible to make your own!

## Why Create an AMI?

1. **Simple backup:** You can snapshot a running server so that if something goes wrong, you can launch a new instance from that image and get back to a known good state.
2. **Cold store and save on costs:** Instead of leaving a server running 24/7 when you don't need it, you can create an AMI, terminate the instance, and later launch a new instance from the AMI when you need it again. You pay only for the (cheaper) storage of the AMI/snapshot until you spin the server back up.
3. **Transfer to a different region:** AMIs are region-specific. If you want to run the same setup in another region, you can copy your AMI to that region and launch instances there. Handy for disaster recovery or moving closer to your users.
4. **Cookie cutter for new servers:** Once you have a server configured the way you want (OS, app, config), you can create an AMI from it and use that AMI as the basis for **new instances** (we'll talk more about this soon).

## Assignment

**Create an AMI from your `patientping-web` instance** so you have a reusable image (and a simple backup).

**Cost check:** AMIs are stored as EBS snapshots; you pay a small amount for snapshot storage.

1.  In the AWS Console, go to "EC2" → "Instances."
2.  Select your `patientping-web` instance.
3.  Click "Actions" → "Image and templates" → "Create image."
4.  Give it a name: `patientping-web-base`
5.  Click "Create image."
6.  Navigate to "Images" → "AMIs" in the left side bar. AWS will create the AMI in the background; its status is `Pending` until it's ready to use. It will take a few minutes to become `Available`, but you're free to move forward in the meantime.