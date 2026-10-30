---
title: "resilient-architectures"
weight: 20
bookCollapseSection: true
---

# Domain 2 - Design Resilient Architectures (30%)

- 001 - [ha-vs-fault-tolerance](ha-vs-fault-tolerance/) = high availability mi fault tolerance mi, SPOF (single point of failure) avciligi
- 002 - [disaster-recovery-strategies](disaster-recovery-strategies/) = RPO (Recovery Point Objective)/RTO (Recovery Time Objective) + backup-restore, pilot light, warm standby, multi-site
- 003 - [elb-and-auto-scaling-resilience](elb-and-auto-scaling-resilience/) = ALB (Application Load Balancer)/NLB (Network Load Balancer)/GWLB (Gateway Load Balancer) + ASG (Auto Scaling Group) scaling politikalari
- 004 - [decoupling-with-sqs-sns](decoupling-with-sqs-sns/) = kuyruk + pub/sub, FIFO, DLQ (dead-letter queue), fan-out
- 005 - [event-driven-eventbridge-step-functions](event-driven-eventbridge-step-functions/) = olay yonlendirme ve is akisi orkestrasyonu
- 006 - [rds-and-aurora-failover](rds-and-aurora-failover/) = Multi-AZ vs read replica, Aurora failover
- 007 - [route53-routing-policies-and-failover](route53-routing-policies-and-failover/) = DNS routing politikalari ve health check ile failover
- 008 - [vpc-connectivity-peering-tgw-direct-connect](vpc-connectivity-peering-tgw-direct-connect/) = VPC (Virtual Private Cloud)'ler arasi ve hibrit baglanti secenekleri

Bu domain sinavin en buyuk dilimi (%30). Sorular genelde "sunucu olurse / bolge cokerse / trafik patlarsa ne olur" seklinde gelir. Cekirdek fikir: **tek nokta arizasi (SPOF) birakma, her seyi cogul (multi-AZ) yap, servisleri birbirinden kopart (decouple)**.

> Not: pratik kurulumlar [AWS](../../aws/) notlarinda - ornek ECS (Elastic Container Service)/ELB (Elastic Load Balancing) kurulumlari icin [application-load-balancer](../../aws/ecs/application-load-balancer/).
