---
title: "AWS"
weight: 5
bookCollapseSection: true
---

> Claimer: most of the content in this section i use boot.dev for recourses

![AWS full architecture diagram](/images/aws/aws-full-diagram.png)

> full diagram (without ecs i will update soon)

Here is the link to view the diagram I drew on Excalidraw: [Diagram link(excalidraw)](https://excalidraw.com/#json=R8A21JFry9HFb3Ypjmy5V,MtI6udd_9j8RKWi9BWEZlw) 

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

### [For start](for-start/)

1. [create-iam-user-and-group](for-start/create-iam-user-and-group/)
2. [regions-and-availability-zones](for-start/regions-and-availability-zones/)

### [Networking - VPC's](networking/)

1. [virtual-private-cloud-vpc](networking/virtual-private-cloud-vpc/)
2. [aws-subnetting](networking/aws-subnetting/)
3. [internet-gateways-igw](networking/internet-gateways-igw/)
4. [route-tables](networking/route-tables/)
5. [private-subnets](networking/private-subnets/)

### [EC2 (Elastic Compute Cloud)](ec2/)

1. [ec2-instance-configuration-keys](ec2/ec2-instance-configuration-keys/)
2. [ec2-instances](ec2/ec2-instances/)
3. [elastic-ips](ec2/elastic-ips/)
4. [aws-security-groups](ec2/aws-security-groups/)
5. [ssh-to-aws-server](ec2/ssh-to-aws-server/)
6. [deploy-site-to-aws-server](ec2/deploy-site-to-aws-server/)
7. [open-web-traffic](ec2/open-web-traffic/)
8. [creating-and-using-your-own-ami](ec2/creating-and-using-your-own-ami/)
9. [aws-launch-template](ec2/aws-launch-template/)
10. [launch-from-template-aws](ec2/launch-from-template-aws/)

### [RDS: Relational Database Service](rds/)

1. [relational-database-service](rds/relational-database-service/)
2. [create-a-postgresql-database-in-aws](rds/create-a-postgresql-database-in-aws/)
3. [aws-connect-to-your-database](rds/aws-connect-to-your-database/)
4. [aws-connect-app-to-database](rds/aws-connect-app-to-database/)
5. [rds-backups](rds/rds-backups/)
6. [create-read-replicas-for-scalability](rds/create-read-replicas-for-scalability/)
7. [rds-general-knowledge](rds/rds-general-knowledge/)

### [Identity and Access Management (IAM)](iam/)

1. [identity-and-access-management](iam/identity-and-access-management/)
2. [create-iam-user](iam/create-iam-user/)
3. [inline-policies-iam-user](iam/inline-policies-iam-user/)
4. [iam-groups](iam/iam-groups/)
5. [iam-roles](iam/iam-roles/)
6. [aws-iam-deny-policies](iam/aws-iam-deny-policies/)
7. [aws-ssm-parameter-store](iam/aws-ssm-parameter-store/)
8. [accessing-ssm-parameters-from-ec2](iam/accessing-ssm-parameters-from-ec2/)
9. [use-ssm-from-ec2](iam/use-ssm-from-ec2/)

### [Monitoring - AWS](monitoring/)

1. [aws-external-monitoring](monitoring/aws-external-monitoring/)
2. [aws-internal-monitoring](monitoring/aws-internal-monitoring/)
3. [cloud-watch-alarms](monitoring/cloud-watch-alarms/)
4. [cloudtrail](monitoring/cloudtrail/)

### [Amazon S3 - Simple Storage Service](s3/)

1. [aws-s3](s3/aws-s3/)
2. [aws-s3-bucket](s3/aws-s3-bucket/)
3. [aws-s3-objects](s3/aws-s3-objects/)

### [AWS - DNS - Route 53](dns/)

1. [route-53-hosted-zones](dns/route-53-hosted-zones/)
2. [aws-a-records](dns/aws-a-records/)
3. [dns-ttl-and-caching](dns/dns-ttl-and-caching/)
4. [aws-verifying-dns](dns/aws-verifying-dns/)
5. [aws-cname-records](dns/aws-cname-records/)

### [AWS - CDN - CloudFront](cdn/)

1. [cloudfront-cdn](cdn/cloudfront-cdn/)
2. [cloudfront-distributions](cdn/cloudfront-distributions/)
3. [cloudfront-invalidation](cdn/cloudfront-invalidation/)
4. [presigned-urls](cdn/presigned-urls/)
5. [why-use-presigned-urls](cdn/why-use-presigned-urls/)
6. [cloudfront-with-dns](cdn/cloudfront-with-dns/)

### [AWS - ECS - Elastic Container Service](ecs/)

1. [vpc-and-networking-setup-for-ecs](ecs/vpc-and-networking-setup-for-ecs/)
2. [why-ecs](ecs/why-ecs/)
3. [elastic-container-registry](ecs/elastic-container-registry/)
4. [ecr-repo](ecs/ecr-repo/)
5. [ecs-clusters](ecs/ecs-clusters/)
6. [ecs-permissions](ecs/ecs-permissions/)
7. [ecs-task-definitions](ecs/ecs-task-definitions/)
8. [ecs-security-groups](ecs/ecs-security-groups/)
9. [application-load-balancer](ecs/application-load-balancer/)
10. [ecs-target-groups](ecs/ecs-target-groups/)
11. [cloudwatch-log-groups](ecs/cloudwatch-log-groups/)
12. [ecs-services](ecs/ecs-services/)

### [AWS Lambda](lambda/)

1. [aws-lambda](lambda/aws-lambda/)
2. [deploy-lambda](lambda/deploy-lambda/)
3. [testing-lambda-function](lambda/testing-lambda-function/)
4. [lambda-api-gateway](lambda/lambda-api-gateway/)
5. [cloudwatch-log-groups-and-viewing-lambda-logs](lambda/cloudwatch-log-groups-and-viewing-lambda-logs/)
6. [other-lambda-use-cases](lambda/other-lambda-use-cases/)

### [General knowledge](general-knowledge/)

1. [reserved-instances-savings-plans](general-knowledge/reserved-instances-savings-plans/)
2. [spot-instances](general-knowledge/spot-instances/)
3. [auto-scaling-groups](general-knowledge/auto-scaling-groups/)
