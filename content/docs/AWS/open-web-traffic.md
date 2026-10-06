---
title: "open-web-traffic"
weight: 140
---

# Open Web Traffic

The PatientPing app is running on the server, but nobody can reach it because our `patientping-public` security group only allows inbound SSH traffic on port `22`. Port `8080`, where the app is listening, is blocked.

This is actually a _good_ default. A firewall that blocks everything except what you explicitly allow is the safest starting point. You saw firsthand in the last lesson what that looks like: the browser just hangs because the packets never make it through.

Now let's poke a hole for web traffic.

## how to enable socket to acces my app in my virtual machine 

**Add an inbound rule to the `patientping-public` security group to allow web traffic on port `8080`.**

**Cost check:** No new resources. Security group rules are free.

1.  Navigate to "EC2" → "Security Groups" (under "Network & Security") in the AWS Console.
2.  Select the `patientping-public` security group ID.
3.  On the "Inbound rules" tab, click "Edit inbound rules."
4.  Click "Add rule" and configure it:
    -  **Type:** `Custom TCP`
    -  **Port range:** `8080`
    -  **Source:** `Anywhere-IPv4` (`0.0.0.0/0`)
    -  **Description:** `Allow web traffic for PatientPing site`
5.  Click "Save rules."
6.  Open a browser on your local machine and go to `http://<patientping-web-public-ip>:8080`. You should now be able to access the PatientPing site.

Security group changes take effect almost immediately. You don't need to restart the instance or the application.

If the site loads (don't worry about pesky "Not Secure" warnings from your browser), you're good to go.

**Run and submit** the CLI tests.

## Tip

If you want to use the CLI instead:

```sh
# Get the security group ID
aws ec2 describe-security-groups --filters 'Name=group-name,Values=patientping-public' --query 'SecurityGroups[0].GroupId' --output text

# Add the inbound rule
aws ec2 authorize-security-group-ingress --group-id SG-ID --ip-permissions IpProtocol=tcp,FromPort=8080,ToPort=8080,IpRanges=[{CidrIp=0.0.0.0/0,Description="Allow web traffic for PatientPing site"}]
```