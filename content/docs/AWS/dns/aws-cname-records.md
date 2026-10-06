---
title: "aws-cname-records"
weight: 50
---

# CNAME Records

What if you want `dev.ulutasalperen.com` to point to the same server as `blog.ulutas.com`? You could create another A record pointing to the same IP address, but there's a better way: **CNAME records**.

A **[CNAME record](https://www.cloudflare.com/learning/dns/dns-records/dns-cname-record/)** (Canonical Name) maps one domain name to another domain name. It's _kind of_ like a DNS-level redirect (or more accurately, an alias).

When someone looks up `dev.ulutasalperen.com`, the CNAME says: "Check `blog.ulutasalperen.com` instead." That domain's A record then provides the IP address.

Sometimes it'll be a pretty simple path:

```
dev.ulutasalperen.com → blog.ulutasalperen.com → 1.2.3.4
CNAME           A                     IP
```

Sometimes it can be a bit more complex:

```text
dev.ulutasalperen.com → blog.boot.com  → blog.east.ulutasalperen.com → 1.2.3.4
CNAME           CNAME                            A records           IP
	                                    → blog.east.ulutasalperen.com → 1.2.3.5
	                                    → blog.east.ulutasalperen.com → 1.2.3.6
	                                    → blog.east.ulutasalperen.com → 1.2.3.7
	                                    → blog.east.ulutasalperen.com → 1.2.3.8
```

A CNAME record has two main parts:

- **Name:** The subdomain you want to alias (e.g., `links`)
- **Value:** Another (sometimes entirely different) domain name (e.g., `www.shortlinks.com`)

## Important Rules

- CNAME records can only point to other domain names, not directly to IP addresses.
- CNAMEs can't overlap. You can't have two rules for the same subdomain (e.g., an A record _and_ a CNAME record for `blog.ulutasalperen.com`).
- If the target domain has multiple A records, the CNAME will point to all of them.

Technically you can't have a CNAME for the root domain (`ulutasalperen.com`), but many DNS providers, including Route 53, let you do essentially the same thing through special handling (ANAME or Aliases). If you're curious, you can [read more here](https://www.networksolutions.com/blog/what-is-apex-domain/).

## Example

PatientPing wants `blog.patientping.internal` to point at the same place as `www.patientping.internal` so they don't have to maintain two A records. Add a `CNAME` that aliases `blog` to `www`.

**Create a CNAME record: `blog.patientping.internal` → `www.patientping.internal`.**

1.  Navigate to `Route 53` → `Hosted zones` in the AWS console.
2.  Select your hosted zone (e.g. `patientping.internal`).
3.  Click `Create record`.
4.  Configure the CNAME record:
    1.  **Record name:** `blog` (this creates `blog.patientping.internal`)
    2.  **Record type:** `CNAME – Routes traffic to another domain name`
    3.  **Value:** Enter `www.patientping.internal`. You must include the full domain name, not just `www`.
    4.  **TTL:** 15 seconds.
5.  Click `Create records`.
6.  Verify that the CNAME record appears in your hosted zone. You should see `blog` pointing to `www.patientping.internal`.
7.  SSH into your app server:

```bash
ssh patientping
```

8.  Run DNS lookup from the server:

```bash
dig blog.patientping.internal
```

9.  Confirm the command returns:
 ```TEXT
 www.patientping.internal.  15  IN  A  10.0.10.50
 ```


## Tip

If you'd prefer to use the CLI:

```sh
aws route53 change-resource-record-sets --hosted-zone-id <zone-id> --change-batch '{"Changes":[{"Action":"CREATE","ResourceRecordSet":{"Name":"<record-name>","Type":"CNAME","TTL":300,"ResourceRecords":[{"Value":"<target-domain>"}]}}]}'
```