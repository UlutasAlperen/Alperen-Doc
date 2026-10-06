---
title: "AWS"
weight: 5
bookCollapseSection: true
---

> Claimer: most of the content in this section i use boot.dev for recourses

![AWS full architecture diagram](/images/aws/aws-full-diagram.png)

> full diagram (without ecs i will update soon)

[Amazon Web Services](https://aws.amazon.com/) is _the_ cloud platform. Nearly every DevOps workflow runs on top of it in production. I use AWS to:

- Build isolated networks with VPCs, subnets, route tables and internet gateways
- Run virtual servers (EC2) with security groups, AMIs and launch templates
- Store data in managed databases (RDS) with backups and read replicas
- Control access with IAM users, groups, roles and policies
- Store files in S3 and serve them worldwide with CloudFront
- Route traffic with Route 53 DNS records
- Run containers with ECS and serverless functions with Lambda
- Monitor everything with CloudWatch and CloudTrail
- And much more

## AWS konular

1. [create-iam-user-and-group](create-iam-user-and-group/)
2. [regions-and-availability-zones](regions-and-availability-zones/)
3. [virtual-private-cloud-vpc](virtual-private-cloud-vpc/)
4. [aws-subnetting](aws-subnetting/)
5. [internet-gateways-igw](internet-gateways-igw/)
6. [route-tables](route-tables/)
7. [private-subnets](private-subnets/)
8. [ec2-instance-configuration-keys](ec2-instance-configuration-keys/)
9. [ec2-instances](ec2-instances/)
10. [elastic-ips](elastic-ips/)
11. [aws-security-groups](aws-security-groups/)
12. [ssh-to-aws-server](ssh-to-aws-server/)
13. [deploy-site-to-aws-server](deploy-site-to-aws-server/)
14. [open-web-traffic](open-web-traffic/)
15. [creating-and-using-your-own-ami](creating-and-using-your-own-ami/)
16. [aws-launch-template](aws-launch-template/)
17. [launch-from-template-aws](launch-from-template-aws/)
18. [relational-database-service](relational-database-service/)
19. [create-a-postgresql-database-in-aws](create-a-postgresql-database-in-aws/)
20. [aws-connect-to-your-database](aws-connect-to-your-database/)
21. [aws-connect-app-to-database](aws-connect-app-to-database/)
22. [rds-backups](rds-backups/)
23. [create-read-replicas-for-scalability](create-read-replicas-for-scalability/)
24. [rds-general-knowledge](rds-general-knowledge/)
25. [identity-and-access-management](identity-and-access-management/)
26. [create-iam-user](create-iam-user/)
27. [inline-policies-iam-user](inline-policies-iam-user/)
28. [iam-groups](iam-groups/)
29. [iam-roles](iam-roles/)
30. [aws-iam-deny-policies](aws-iam-deny-policies/)
31. [aws-ssm-parameter-store](aws-ssm-parameter-store/)
32. [accessing-ssm-parameters-from-ec2](accessing-ssm-parameters-from-ec2/)
33. [use-ssm-from-ec2](use-ssm-from-ec2/)
34. [aws-external-monitoring](aws-external-monitoring/)
35. [aws-internal-monitoring](aws-internal-monitoring/)
36. [cloud-watch-alarms](cloud-watch-alarms/)
37. [cloudtrail](cloudtrail/)
38. [aws-s3](aws-s3/)
39. [aws-s3-bucket](aws-s3-bucket/)
40. [aws-s3-objects](aws-s3-objects/)
41. [route-53-hosted-zones](route-53-hosted-zones/)
42. [aws-a-records](aws-a-records/)
43. [dns-ttl-and-caching](dns-ttl-and-caching/)
44. [aws-verifying-dns](aws-verifying-dns/)
45. [aws-cname-records](aws-cname-records/)
46. [cloudfront-cdn](cloudfront-cdn/)
47. [cloudfront-distributions](cloudfront-distributions/)
48. [cloudfront-invalidation](cloudfront-invalidation/)
49. [presigned-urls](presigned-urls/)
50. [why-use-presigned-urls](why-use-presigned-urls/)
51. [cloudfront-with-dns](cloudfront-with-dns/)
52. [vpc-and-networking-setup-for-ecs](vpc-and-networking-setup-for-ecs/)
53. [why-ecs](why-ecs/)
54. [elastic-container-registry](elastic-container-registry/)
55. [ecr-repo](ecr-repo/)
56. [ecs-clusters](ecs-clusters/)
57. [ecs-permissions](ecs-permissions/)
58. [ecs-task-definitions](ecs-task-definitions/)
59. [ecs-security-groups](ecs-security-groups/)
60. [application-load-balancer](application-load-balancer/)
61. [ecs-target-groups](ecs-target-groups/)
62. [cloudwatch-log-groups](cloudwatch-log-groups/)
63. [ecs-services](ecs-services/)
64. [aws-lambda](aws-lambda/)
65. [deploy-lambda](deploy-lambda/)
66. [testing-lambda-function](testing-lambda-function/)
67. [lambda-api-gateway](lambda-api-gateway/)
68. [cloudwatch-log-groups-and-viewing-lambda-logs](cloudwatch-log-groups-and-viewing-lambda-logs/)
69. [other-lambda-use-cases](other-lambda-use-cases/)
70. [reserved-instances-savings-plans](reserved-instances-savings-plans/)
71. [spot-instances](spot-instances/)
72. [auto-scaling-groups](auto-scaling-groups/)
73. [genel-temizlik](genel-temizlik/)

### For start

001 - [create-iam-user-and-group](create-iam-user-and-group/) = ilk admin IAM kullanıcısı ve grubunu oluşturma
002 - [regions-and-availability-zones](regions-and-availability-zones/) = region ve AZ kavramları

### Networking - VPC's

001 - [virtual-private-cloud-vpc](virtual-private-cloud-vpc/) = izole sanal ağ (VPC) ve CIDR blokları

![VPC diagram](/images/aws/vpc-diagram.png)

002 - [aws-subnetting](aws-subnetting/) = VPC adres alanını private/public subnet'lere bölme

![Subnetting diagram](/images/aws/subnetting-diagram.png)

003 - [internet-gateways-igw](internet-gateways-igw/) = public subnet'leri internet'e bağlayan IGW

![Internet gateway diagram](/images/aws/internet-gateway-diagram.png)

004 - [route-tables](route-tables/) = trafiğin nereye gideceğini belirleyen route tabloları

![Route tables diagram](/images/aws/route-tables-diagram.png)

![Route tables diagram 2](/images/aws/route-tables-diagram-2.png)

005 - [private-subnets](private-subnets/) = internet'e doğrudan erişimi olmayan subnet'ler

![Private subnets diagram](/images/aws/private-subnets-diagram.png)

### EC2 (Elastic Compute Cloud) kind of vm but in cloud they say

001 - [ec2-instance-configuration-keys](ec2-instance-configuration-keys/) = creatin of ssh key for vms
002 - [ec2-instances](ec2-instances/) = its just virtual machines
003 - [elastic-ips](elastic-ips/) = fancy name of static ipv4
004 - [aws-security-groups](aws-security-groups/) = network guvenlik ayarlari ve bunu instanceye baglama

![Security groups diagram](/images/aws/security-groups-diagram.png)

005 - [ssh-to-aws-server](ssh-to-aws-server/) = SSH ile sunucuya bağlanma

![SSH to AWS server](/images/aws/ssh-to-aws-server.png)

006 - [deploy-site-to-aws-server](deploy-site-to-aws-server/) = siteyi sunucuya deploy etme
007 - [open-web-traffic](open-web-traffic/) = aslinda burada yaptigim benim tanimladigim statik ip'ye policy atayarak statik ip'ye gelelecek olan 8080 portunu aciyorum

![Open web traffic](/images/aws/open-web-traffic-diagram.png)

008 - [creating-and-using-your-own-ami](creating-and-using-your-own-ami/) = burada aslinda AMI (amazon machine image yani bildigin os iso) kendi istedgim tekrar kullanilabilir ornek vm aws'ye eklemek yani **instance** so you have a reusable image (and a simple backup).
009 - [aws-launch-template](aws-launch-template/) = onceden hazirlanmis vm - [launch template](https://docs.aws.amazon.com/autoscaling/ec2/userguide/launch-templates.html) is a pre-baked configuration for EC2 instances.
010 - [launch-from-template-aws](launch-from-template-aws/) = onceden hazirlanmis vm calistirma

### RDS: Relational Database Service

001 - [relational-database-service](relational-database-service/)
002 - [create-a-postgresql-database-in-aws](create-a-postgresql-database-in-aws/)

![RDS overview diagram](/images/aws/rds-overview-diagram.png)

003 - [aws-connect-to-your-database](aws-connect-to-your-database/)
004 - [aws-connect-app-to-database](aws-connect-app-to-database/)

![Connect app to database](/images/aws/connect-app-to-database.png)

005 - [rds-backups](rds-backups/)
006 - [create-read-replicas-for-scalability](create-read-replicas-for-scalability/)

![Read replicas diagram](/images/aws/read-replicas-diagram.png)

007 - [rds-general-knowledge](rds-general-knowledge/)

### Identity and Access Management (IAM)

001 - [identity-and-access-management](identity-and-access-management/)

At some stage in building a project on AWS, you'll need to give other people (or perhaps automated systems) access to your cloud resources as well. [IAM](https://aws.amazon.com/iam/) is how AWS controls access to resources: who can log in, and what they can do once they're in.

![IAM diagram](/images/aws/iam-diagram.png)

002 - [create-iam-user](create-iam-user/) = Each user gets a **username**/**password** or **API keys** to access AWS. Users are **long-lived identities**: they stick around until you delete them. Once you create a user and give them credentials, those credentials will work until they expire or you revoke them.

003 - [inline-policies-iam-user](inline-policies-iam-user/) = [Policies](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html) **allow** or **deny** actions on specific resources.

004 - [iam-groups](iam-groups/) = While inline policies are attached directly to a single user, the _preferred_ way to manage permissions in AWS is to create **groups** and give _them_ the policies (sometimes called [customer-managed policies](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_managed-vs-inline.html#customer-managed-policies)).

005 - [iam-roles](iam-roles/) = IAM has [roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html). A role is a **temporary identity** that can be used or **assumed** by trusted parties.

![IAM roles diagram](/images/aws/iam-roles-diagram.png)

006 - [aws-iam-deny-policies](aws-iam-deny-policies/) = explicit Deny her zaman kazanır
007 - [aws-ssm-parameter-store](aws-ssm-parameter-store/) = centralized safe where you can keep configuration data, secrets, and parameters that your applications need
008 - [accessing-ssm-parameters-from-ec2](accessing-ssm-parameters-from-ec2/) = uzak sunucumun ssm parametlerine iznini vermek
009 - [use-ssm-from-ec2](use-ssm-from-ec2/)

### Monitoring - AWS

001 - [aws-external-monitoring](aws-external-monitoring/)
002 - [aws-internal-monitoring](aws-internal-monitoring/)
003 - [cloud-watch-alarms](cloud-watch-alarms/)
004 - [cloudtrail](cloudtrail/) = logs for the AWS account itself

### Amazon S3 - Simple Storage Service

001 - [aws-s3](aws-s3/)

![S3 diagram](/images/aws/s3-diagram.png)

002 - [aws-s3-bucket](aws-s3-bucket/) = The bucket is the highest level of organization in S3; it's a container for storing objects.
003 - [aws-s3-objects](aws-s3-objects/)

![S3 objects diagram](/images/aws/s3-objects-diagram.png)

### AWS - DNS - Route 53

001 - [route-53-hosted-zones](route-53-hosted-zones/)
002 - [aws-a-records](aws-a-records/) = DNS records tell the world where your domain points to. The most fundamental type of record is the **A record**.

![DNS A records diagram](/images/aws/dns-a-records-diagram.png)

003 - [dns-ttl-and-caching](dns-ttl-and-caching/)
004 - [aws-verifying-dns](aws-verifying-dns/)
005 - [aws-cname-records](aws-cname-records/)

### AWS - CDN - CloudFront

001 - [cloudfront-cdn](cloudfront-cdn/) = [CloudFront](https://aws.amazon.com/cloudfront/) is AWS's [content delivery network](https://www.cloudflare.com/learning/cdn/what-is-a-cdn/) (CDN). A CDN is a network of servers distributed around the world that cache and serve content in locations that are _physically close_ to users. In simple terms, they cache static content like images, videos, and scripts from an [origin server](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistS3AndCustomOrigins.html) and serve it from the nearest edge location.

![CloudFront CDN diagram](/images/aws/cloudfront-cdn-diagram.png)

002 - [cloudfront-distributions](cloudfront-distributions/)

![CloudFront distributions diagram](/images/aws/cloudfront-distributions-diagram.png)

003 - [cloudfront-invalidation](cloudfront-invalidation/)
004 - [presigned-urls](presigned-urls/)
005 - [why-use-presigned-urls](why-use-presigned-urls/) = http web sunucu servisi gibi file serve icin kullaniliabilir
006 - [cloudfront-with-dns](cloudfront-with-dns/)

### AWS - ECS - Elastic Container Service

001 - [vpc-and-networking-setup-for-ecs](vpc-and-networking-setup-for-ecs/)
002 - [why-ecs](why-ecs/)
003 - [elastic-container-registry](elastic-container-registry/) = Before we can deploy any containers, we need somewhere to _store their images_. Those images need to be available all the time because containers are regularly replaced and may need to pull a fresh copy of the software.
004 - [ecr-repo](ecr-repo/)
005 - [ecs-clusters](ecs-clusters/)
006 - [ecs-permissions](ecs-permissions/)
007 - [ecs-task-definitions](ecs-task-definitions/)
008 - [ecs-security-groups](ecs-security-groups/)

![ECS security groups diagram](/images/aws/ecs-security-groups-diagram.png)

009 - [application-load-balancer](application-load-balancer/) = A [load balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/what-is-load-balancing.html) sits in front of our service and distributes incoming traffic across your task(s). It handles things like health checks, SSL termination, and routing traffic to healthy containers. We'll place the load balancer in public subnets so it can receive traffic from the internet.

![Application load balancer diagram](/images/aws/application-load-balancer-diagram.png)

010 - [ecs-target-groups](ecs-target-groups/) = A **target group** tells the load balancer where to send traffic. It defines:

- **Protocol and port:** What protocol (HTTP/HTTPS) and which port your tasks are listening on
- **Health checks:** How the load balancer determines if a target is healthy
- **Target type:** Whether targets are IP addresses (for Fargate) or instance IDs (for EC2)

011 - [cloudwatch-log-groups](cloudwatch-log-groups/)
012 - [ecs-services](ecs-services/)

### AWS Lambda

001 - [aws-lambda](aws-lambda/)
002 - [deploy-lambda](deploy-lambda/)
003 - [testing-lambda-function](testing-lambda-function/)

![Testing Lambda function](/images/aws/testing-lambda-function.png)

004 - [lambda-api-gateway](lambda-api-gateway/)

![Lambda API gateway diagram](/images/aws/lambda-api-gateway-diagram.png)

005 - [cloudwatch-log-groups-and-viewing-lambda-logs](cloudwatch-log-groups-and-viewing-lambda-logs/)
006 - [other-lambda-use-cases](other-lambda-use-cases/)

### General knowledge

001 - [reserved-instances-savings-plans](reserved-instances-savings-plans/) = Reserved Instances ve Savings Plans ile compute maliyetini düşürme
002 - [spot-instances](spot-instances/) = ~%90 indirimli, AWS'in geri alabileceği kapasite
003 - [auto-scaling-groups](auto-scaling-groups/) = ASG ile otomatik ölçekleme ve "cattle not pets" yaklaşımı (Stateful/Stateless uygulamalar)

### Cleanup

001 - [genel-temizlik](genel-temizlik/) = kullanılmayan kaynakları silme ve maliyet kontrolü
