---
title: "aws-verifying-dns"
weight: 440
---

# Verifying DNS

We have a `www.patientping.internal` domain name, but it's _only accessible within our VPC_.

In other words, this command won't produce a meaningful IP address locally:

```sh
dig www.patientping.internal +short
```

but if you run that command from the `patientping-web-v2` EC2 instance (that's inside the VPC), you should be able to resolve the domain name!

```sh
ssh patientping
dig www.patientping.internal +short
# 10.0.10.50
```

## Example

**Verify that `www.patientping.internal` resolves from inside `patientping-web-v2` over SSH.**

1.  SSH into your app server:
```bash
ssh patientping
```

2.  Install `dig` (if not already installed):
```bash
sudo dnf install bind-utils
```

3.  Run DNS lookup from the server:

```bash
dig www.patientping.internal +short

```
4.  Confirm the command returns `10.0.10.50`.
5.  Exit the server:

```bash
exit
```