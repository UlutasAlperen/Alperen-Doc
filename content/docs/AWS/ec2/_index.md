---
title: "ec2"
weight: 20
bookCollapseSection: true
---

# EC2 (Elastic Compute Cloud) kind of vm but in cloud they say

- 001 - [ec2-instance-configuration-keys](ec2-instance-configuration-keys/) = creating of ssh key for aws vm's 
- 002 - [ec2-instances](ec2-instances/) = its just virtual machines
- 003 - [elastic-ips](elastic-ips/) = fancy name of static ipv4
- 004 - [aws-security-groups](aws-security-groups/) = network guvenlik ayarlari ve bunu instanceye baglama

![Security groups diagram](/images/aws/security-groups-diagram.png)

- 005 - [ssh-to-aws-server](ssh-to-aws-server/) = SSH ile aws  uzak sunucuya bağlanma  (klasik ssh)

![SSH to AWS server](/images/aws/ssh-to-aws-server.png)

- 006 - [deploy-site-to-aws-server](deploy-site-to-aws-server/) = siteyi sunucuya deploy etme
- 007 - [open-web-traffic](open-web-traffic/) = aslinda burada yaptigim benim tanimladigim statik ip'ye policy atayarak statik ip'ye gelelecek olan 8080 portunu aciyorum

![Open web traffic](/images/aws/open-web-traffic-diagram.png)

- 008 - [creating-and-using-your-own-ami](creating-and-using-your-own-ami/) = burada aslinda AMI (amazon machine image yani bildigin os iso) kendi istedgim tekrar kullanilabilir ornek vm aws'ye eklemek yani **instance** so you have a reusable image (and a simple backup).
- 009 - [aws-launch-template](aws-launch-template/) = onceden hazirlanmis vm sablon - [launch template](https://docs.aws.amazon.com/autoscaling/ec2/userguide/launch-templates.html) is a pre-baked configuration for EC2 instances.
- 010 - [launch-from-template-aws](launch-from-template-aws/) = onceden hazirlanmis vm sablonunu calistirma

### Ec2 pricing options

![Ec2 pricing alternatives](/images/aws/Ec2_pricing_options.png)

### Alternative full managed server for container

![Ec2 full managed server](/images/aws/Fargate.png)
