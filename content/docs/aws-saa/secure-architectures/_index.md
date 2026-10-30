---
title: "secure-architectures"
weight: 10
bookCollapseSection: true
---

# Domain 1 - Design Secure Architectures (24%)

- 001 - [iam-policies-roles-cross-account](iam-policies-roles-cross-account/) = policy evaluation logic, roles vs users, cross-account access
- 002 - [identity-federation-and-cognito](identity-federation-and-cognito/) = SAML (Security Assertion Markup Language)/OIDC federation, IAM (Identity and Access Management) Identity Center, Cognito User Pools vs Identity Pools
- 003 - [kms-encryption-deep-dive](kms-encryption-deep-dive/) = CMK (customer managed key), envelope encryption, SSE-S3 vs SSE-KMS vs SSE-C
- 004 - [secrets-manager-vs-parameter-store](secrets-manager-vs-parameter-store/) = secret kasasi secimi - rotasyon var mi, kac para
- 005 - [s3-security-and-access-points](s3-security-and-access-points/) = bucket policy vs ACL, Block Public Access, Access Points
- 006 - [network-security-sg-nacl-vpc-endpoints](network-security-sg-nacl-vpc-endpoints/) = stateful vs stateless, gateway vs interface endpoint
- 007 - [security-services-waf-guardduty-inspector](security-services-waf-guardduty-inspector/) = WAF (Web Application Firewall), Shield, GuardDuty, Inspector, Macie - hangisi hangi is icin
- 008 - [governance-monitoring-organizations-config-ssm](governance-monitoring-organizations-config-ssm/) = Organizations & SCP, Control Tower, CloudTrail vs Config vs CloudWatch, SSM Session Manager

Bu domain sinavda "bu mimaride guvenlik acigi ne, en guvenli secenek hangisi" tipi sorularla gelir. Temel fikir su: **en az yetki (least privilege)**, **varsayilan olarak kapali (deny by default)** ve **her yerde sifreleme**.

> Not: hands-on IAM kurulumlari icin [AWS - IAM](../../aws/iam/) notlarima bak. Burasi sinav teorisi.
