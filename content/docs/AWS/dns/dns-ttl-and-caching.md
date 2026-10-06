---
title: "dns-ttl-and-caching"
weight: 30
---

# DNS TTL and Caching

When you update a DNS record, the change doesn't appear everywhere immediately. DNS is [eventually consistent](https://en.wikipedia.org/wiki/Eventual_consistency) and _heavily_ cached.

"Eventually consistent" means changes are applied, but not everywhere at once. It takes time to propagate.

When creating DNS records, you'll notice a field called [TTL](https://www.cloudflare.com/learning/cdn/glossary/time-to-live-ttl/) (Time To Live). This tells resolvers how often to re-check a record. If our records change often, clients need a lower TTL.

The trade-off is always the same: lower TTL means faster updates but more queries (and more cost). Higher TTL means slower updates but fewer queries.

**Cost check:** Route 53 charges about `$0.40` per million queries for the first 1 billion. Lower TTLs mean more queries; for learning, cost is negligible.

When the client has to double-check DNS (due to lower TTL) the site will load a bit slower for them, and you have to pay for the extra queries.

When you update a DNS record in Route 53:

1. Route 53 immediately has the new record.
2. Clients won't ask Route 53 again until the TTL expires.
3. Eventually, all caches expire and everyone sees the new record.

This is part of why the meme "[It's always DNS](https://isitdns.com/)" exists. If you submit a typo in your DNS record, you'll have to wait for the TTL to expire before your customers start getting the correct address. I once watched a team spend hours debugging why their new server wasn't getting traffic, only to realize they had a typo in the IP address, and even after fixing it, they still had to wait for TTL expiry.