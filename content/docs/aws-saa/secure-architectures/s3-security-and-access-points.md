---
title: "s3-security-and-access-points"
weight: 50
---

# S3 (Simple Storage Service) Security and Access Points

S3 security questions usually start with "an object was accidentally made public" or "only this VPC (Virtual Private Cloud) may access the bucket". The answer set is small: **Block Public Access**, bucket policies, IAM (Identity and Access Management), and **Access Points**.

## The access control toolkit

| Mechanism | What it does | Exam note |
|---|---|---|
| **Block Public Access** | Master switch; overrides everything | Turn on account-wide, "prevent any public access" |
| **Bucket policy** | Resource-based JSON on the bucket | **Cross-account access**, VPC-only access |
| **IAM policies** | Identity-based permissions | Same-account access for users/roles |
| **ACLs** | Legacy per-object permissions | "ACL disabled / Object Ownership: Bucket owner enforced" is the modern answer |
| **Access Points** | Per-workload entry point with its own policy | Huge buckets, many teams/VPCs |
| **Pre-signed URLs** | Time-limited URL signed with credentials | "Share a private object for 1 hour" |

Handy labs: [aws-s3-bucket](../../../aws/s3/aws-s3-bucket/), [presigned-urls](../../../aws/cdn/presigned-urls/).

## The classic "public website" question

Two patterns appear over and over:

1. **Static website hosting** - bucket policy allows `s3:GetObject` for `*`, Block Public Access off, website endpoint
2. **Private content via CloudFront** - bucket stays private; CloudFront accesses it with **OAC (Origin Access Control)** so users can never hit S3 directly

> [!NOTE]
> **Not:** Sınavda 2023'ten beri "en güvenli" statik site cevabı **CloudFront + OAC (Origin Access Control)** (eski adıyla OAI)'dir. Bucket'ı public yapmak hâlâ çalışır ama "MOST secure" sorularında kaybeder. Birden fazla özel dosyaya (örn. HLS video parçaları) URL değiştirmeden süreli erişim verilecekse **CloudFront Signed Cookies**, tek bir dosyaya verilecekse **CloudFront Signed URL / S3 Pre-Signed URL** seçilir.

## S3 Object Lock (WORM), Glacier Vault Lock & MFA Delete

Compliance questions ("files must never be deleted or modified for 7 years") test these mechanisms:

| Feature | How it works | Exam keyword |
|---|---|---|
| **Object Lock - Compliance mode** | Object version cannot be overwritten or deleted by **anyone — including the root user** — until retention expires | "Strict regulatory compliance, even root cannot delete" |
| **Object Lock - Governance mode** | Protected by default, but users with `s3:BypassGovernanceRetention` IAM permission can override | "Prevent accidental deletion, allow admin override" |
| **Object Lock - Legal Hold** | On/off switch with **no expiration date**; stays locked until explicitly removed (`s3:PutObjectLegalHold`) | "Indefinite hold during a legal investigation" |
| **Glacier Vault Lock** | Lock a Glacier vault policy with a 24-hour confirmation code; **immutable forever** once locked | "Archive vault compliance policy that can never be changed" |
| **MFA Delete** | Requires MFA code to permanently delete an object version or suspend Versioning (**only the Root user** can enable/disable via CLI/API) | "Extra authentication before deleting versioned objects" |

*Prerequisite:* **S3 Versioning must be enabled** before you can use S3 Object Lock or MFA Delete.

## CORS (Cross-Origin Resource Sharing)

If a web page hosted at `https://app.example.com` makes browser `fetch`/`XHR` requests (or loads fonts/images via JS) from `https://assets.otherdomain.com` (another S3 bucket), you must configure a **CORS rule on the target (assets) S3 bucket** allowing the `Origin: https://app.example.com`.

## Restricting access to a VPC

A bucket policy can say "only requests coming through my VPC endpoint":

```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": [
    "arn:aws:s3:::my-bucket",
    "arn:aws:s3:::my-bucket/*"
  ],
  "Condition": {
    "StringNotEquals": {
      "aws:sourceVpce": "vpce-0123456789abcdef0"
    }
  }
}
```

Note the combination: **Deny + StringNotEquals** = "deny everyone not coming from my endpoint". This `aws:sourceVpce` pattern is an exam favorite.

## Access Points

Instead of one giant bucket policy, give each use case an endpoint:

```mermaid
flowchart TB
    b[(Shared bucket)]
    ap1[Access point: finance-app] --> b
    ap2[Access point: analytics, VPC-only] --> b
    app1[Finance app] --> ap1
    app2[Analytics jobs] --> ap2
```

- Each access point has its **own DNS name and policy**
- Can be locked to a **VPC** (`network origin: VPC`) - behaves like a private endpoint
- **Object Ownership: Bucket owner enforced** disables ACLs entirely (the modern default)

## Exam check

**Q1: A security audit found that an S3 bucket allows public ACLs. The requirement is that no object in any bucket of the account can ever be made public. What is the fastest fix?**

- **A)** Delete the bucket policy
- **B)** Enable S3 Block Public Access at the **account** level
- **C)** Enable versioning on the bucket
- **D)** Move objects to Glacier

{{% details "Cevabı göster" %}}
**Answer: B.** Account-level Block Public Access overrides every bucket policy and ACL.
{{% /details %}}

**Q2: Only instances inside a specific VPC should be able to read a bucket. What should the architect configure?**

- **A)** An IAM role with `s3:GetObject`
- **B)** A bucket policy denying requests where `aws:sourceVpce` is not the VPC endpoint
- **C)** A security group attached to the bucket
- **D)** A pre-signed URL on each object

{{% details "Cevabı göster" %}}
**Answer: B.** The Deny + `aws:sourceVpce` condition pattern. A doesn't restrict network origin, C is nonsense (SGs (Security Group) don't attach to buckets), D is per-object and temporary.
{{% /details %}}

## Ozet

- Block Public Access = her seyi bastiran ana anahtar, hesap seviyesinde ac
- Bucket policy = cross-account ve VPC-only erisim icin
- ACL = eski teknoloji, "Bucket owner enforced" = ACL'ler kapali
- CloudFront + OAC = en guvenli statik site
- Access Point = kendi politikasi olan is-load endpoint'i, VPC'ye kilitlenebilir
- Pre-signed URL = zaman sinirli paylasim
- aws:sourceVpce ile "sadece bu VPC" sarti kurulur
