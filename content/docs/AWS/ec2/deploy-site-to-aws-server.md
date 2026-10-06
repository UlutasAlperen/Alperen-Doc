---
title: "deploy-site-to-aws-server"
weight: 60
---

# Deploy the PatientPing Site

Now that you can SSH into the `patientping-web` server, let's get the PatientPing application running on it. We'll install `git`, clone the app, install its dependencies, and start the web server.

**Cost check:** No new resources. You're using the same EC2 instance; cost is unchanged.

## Assignment

**Deploy the PatientPing application on the `patientping-web` instance.**

1.  SSH into your `patientping-web` instance (e.g. `ssh patientping` if you configured that in the previous lesson).
2.  **Install `git`** on the instance (the base AMI may not include it). On Amazon Linux, run:
```sh

sudo dnf upgrade
sudo dnf install git
```

3.  Install [`uv`](https://github.com/astral-sh/uv) to be able to run the Python server:

```sh
curl -LsSf https://astral.sh/uv/install.sh | sh
```

4.  Verify that you have both `git` and `uv` in `PATH`:
```sh
which git
which uv
```

5.  **Clone the GitHub repo** of the PatientPing app to the EC2 instance:

```sh
cd ~
git clone https://github.com/bootdotdev/patientping-web.git
cd patientping-web
uv sync
uv run patientping.py
```

> You should see the server start up on port `8080`!
6.  On your local machine, try opening `http://<patientping-web-public-ip>:8080` in your browser; **it should hang and fail**. Why? Because our security group only allows inbound traffic on port `22` (SSH). There's no rule permitting traffic on port `8080`, so the firewall blocks it.

This is the security group doing its job. By default, all inbound traffic is denied unless you explicitly allow it. Right now the only hole in the firewall is port `22` for SSH.

Don't worry, we'll fix this soon. In a new terminal, leave the server running in the original session.