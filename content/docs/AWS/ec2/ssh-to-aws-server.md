---
title: "ssh-to-aws-server"
weight: 50
---

# SSH to Our Server

You may start to understand why people sometimes throw up their hands and use an "easier" provider like Vercel or Heroku. Those providers configure all the same things we've done so far, then sell it to you at a (steep) markup.

But if you can handle the basics on your own (and you can), you'll save money and have more control over your infrastructure.

Seriously, some folks are happy to chew through C code full of `malloc`, `calloc`, and `int *int_list_ptr = (int *)struct_list_of_ints;`, but adding a route is too hard? Not you; onward and upward.

Here's a review of the networking flow for us to SSH into our server:

1. Our computer communicates with the server's public **Elastic IP** address.
2. The server is in a **public subnet** with a **route** to the **internet gateway**.
3. The **security group** allows **inbound** traffic on `22/tcp` from _our_ public IP address.
4. Our **SSH private key** matches the **public key** we uploaded to AWS.

## how connect aws wm with ssh

**Cost check:** No new resources in this lesson. You're using the EC2 instance and Elastic IP you already have; cost remains ~$7.60/mo for the instance plus ~$3.60/mo for the EIP.

**SSH into `patientping-web` and update its message of the day.**

1.  Log into the AWS Console and grab the public IP address of your server:
    -  Navigate to "EC2" → "Instances."
    -  Select the `patientping-web` instance and copy its "Public IPv4 address."
2.  Make sure you can at least _reach_ the server on `22/tcp` from your computer. I prefer to use `nc` (short for [`netcat`](https://www.blackhillsinfosec.com/netcat-cheatsheet/)):

```bash
nc -zv YOUR.INSTANCE.IP.ADDR 22
# Connection to YOUR.INSTANCE.IP.ADDR port 22 [tcp/ssh] succeeded!
```

3.  Add an entry to your local SSH config, to serve as a shortcut for connecting to `patientping-web`. This is not only convenient; it will also allow our CLI tests to verify the connection.
  -  Open your `~/.ssh/config` file in your text editor of choice. If it doesn't exist, create it first: `touch ~/.ssh/config`
  -  Add the following `Host` entry, setting the IP address and the absolute path of your private key as appropriate:

```text
Host patientping
    HostName IP_ADDR # burada aslinda sanal makinamiza atadigimiz elstic ip(ipv4) giriyoruz
    User ec2-user # amazon user ismi genelde bu idk why
    IdentityFile ~/.ssh/patientping-key
```

 -  Save your changes to the config file.
4.  Test your configuration by running `ssh patientping`; it should log you in fully (because you didn't set a password on your private key).
5.  While on the EC2 instance, update the "[Message of the Day](https://en.wikipedia.org/wiki/Message_of_the_day)" (a simple welcome message that prints whenever someone logs into the server) to say `Welcome, PatientPing Server Admin!`:

```sh
echo 'Welcome, PatientPing Server Admin!' | sudo tee /etc/motd > /dev/null
```

6.  Verify the `motd` is working by logging out of the server and back in.

```sh
exit
ssh patientping
```



