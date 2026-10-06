---
title: "launch-from-template-aws"
weight: 100
---

# Launch from Template

The point of creating our launch template was so we could treat servers _like cattle, not pets_. Let's do that.

We'll terminate our `patientping-web` instance, and launch a brand new one from the `patientping-web-launcher` template to get the site back up. Because the AMI already has the app baked in, we'll basically just need to SSH back in and start the app.

## Assignment

**Terminate the current PatientPing instance and launch a replacement from your launch template.**

**Cost check:** You're replacing one `t3.micro` with another. No change in cost.

1.  **Terminate the old instance:**
    -  In the AWS Console, go to "EC2" → "Instances."
    -  Select the `patientping-web` instance.
    -  Click "Instance state" → "Terminate (delete) instance" and confirm.
    -  Wait until the instance shows `Terminated` (may take a minute or two).
2.  **Launch a new instance from the template:**
    -  Go to "EC2" → "Launch Templates."
    -  Select the `patientping-web-launcher` template.
    -  Click "Actions" → "Launch instance from template."
    -  On the launch page, everything should be pre-filled from the template. The only change: under **Resource tags**, add a `Name` tag with the value `patientping-web-v2`.
    -  Click "Launch instance."
3.  **Re-associate your Elastic IP:**
    -  Go to "EC2" → "Elastic IPs."
    -  Select your existing EIP (it should be unassociated after the old instance was terminated).
    -  Click "Actions" → "Associate Elastic IP address."
    -  Select the new `patientping-web-v2` instance and click "Associate."
4.  **Update your SSH config** to remove the old host key (the new instance has a different host key even though the IP is the same):

```sh
ssh-keygen -R YOUR_ELASTIC_IP
```

5.  **SSH into the new instance:**

```sh
ssh patientping
```

6.  **Start the application.** The app code is already on the server (baked into the AMI), so you just need to start it:

```sh
cd ~/patientping-web
uv run patientping.py
```

7.  **Verify again that the site loads** at `http://<your-elastic-ip>:8080` in a browser on your local machine.

Notice how much faster this was than the first time. No installing `git`, no cloning, no installing `uv`. The AMI had it all. That's the power of baking your app into a custom image.

## Tip

If you want to use the CLI instead:

```sh
# Terminate the old instance
aws ec2 terminate-instances --instance-ids OLD-INSTANCE-ID

# Launch from template
aws ec2 run-instances --launch-template LaunchTemplateName=patientping-web-launcher --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=patientping-web-v2}]'

# Re-associate the Elastic IP
aws ec2 associate-address --instance-id NEW-INSTANCE-ID --allocation-id EIP-ALLOCATION-ID
```