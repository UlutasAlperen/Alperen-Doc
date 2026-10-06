---
title: "elastic-ips"
weight: 100
---

# Elastic IPs

Our PatientPing server _does_ need a public IP address to be reachable on the internet, but we're going to use an "[Elastic IP address](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html)" (EIP). This is an address that we can attach to any of our servers and move around as needed. You might be thinking:

> "But Alperen, why not just use the nice automatic public IP address?"

Well, the "auto-provisioned IP address" that we opted _not_ to use earlier has a few downsides:

- It can change any time your server is stopped and restarted.
- You can't attach it to a different server.
- Those types **cost the same as an EIP**! If they're less convenient, shouldn't they be cheaper?!

Most residential Internet Service Providers (ISPs) don't make any guarantees about the public IP address that you get assigned. ISPs often change and rotate IP addresses at their convenience. Which gives folks trying to run a little server out of their house [wonderful hoops to jump through](https://en.wikipedia.org/wiki/Dynamic_DNS).

## how to create elestic ip (statik ipv4 olusturma)

**Allocate and associate an Elastic IP to `patientping-web`.**

**Cost check:** Elastic IPs used to be free, but that's changed. All public IP addresses now cost **$0.005/hour**, or about **$3.60/month**.

1.  Navigate to "EC2" using the search bar in the AWS Console.
2.  Click "Elastic IPs" in the left-hand menu under "Network & Security."
3.  Click the orange "Allocate Elastic IP address" button at the top right.
4.  Leave the default settings. It should be an address from "Amazon's pool of IPv4 addresses," and the "Network border group" should be `us-east-1`.
5.  Click "Allocate."
6.  Back in the list of Elastic IPs, select the one you just allocated by clicking on its checkbox.
7.  Click the "Actions" dropdown button on the top right, then "Associate Elastic IP address."
8.  Use the "Instance" resource type.
9.  Click on the "Instance" field and select your `patientping-web` EC2 instance.
10. Click "Associate."
11. Verify that the Elastic IP is now associated with your instance by checking the instance details.


## Tip

If you want to use the CLI instead, here's the command structure:

```sh
# Allocate an Elastic IP
aws ec2 allocate-address --domain vpc

# Associate it with an instance
aws ec2 associate-address --instance-id INSTANCE-ID --allocation-id ALLOCATION-ID
```