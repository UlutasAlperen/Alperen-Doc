---
title: "create-read-replicas-for-scalability"
weight: 230
---

# Read Replicas

Think about how **Discord** or **Slack** works for a moment. There are chat channels filled with tons of users. Do you, as a user, _read_ more or _write_ more to the server?

Unless you're an incredibly chatty person, you're going to read waaaaaay more messages than you send. Wouldn't it be nice to offload those reads from an expensive primary database to a cheaper server?

Enter **read replicas**.

## How Read Replicas Work

- **Asynchronous replication:** Data is copied from the primary database to replicas continually
- **Read-only:** Replicas handle read queries (`SELECT` statements)
- **Write operations:** Still go to the primary database (`INSERT`, `UPDATE`, `UPSERT`, `DELETE`)
- **Automatic updates:** Replicas stay in sync with the primary, give or take a few seconds

## Why Use Read Replicas?

1. **Read performance:** Distribute read traffic across multiple instances
2. **Reduce load:** Take read load off the primary database
3. **Scalability:** Add more replicas as traffic grows
4. **Geographic distribution:** Place replicas closer to users in different regions, so they at least get fast reads
5. **High availability:** Use replicas for read-only failover if the primary fails

Read replicas can lag behind the primary by a few seconds due to asynchronous replication. This is usually fine for most use cases, but if you need real-time consistency, you'll need to read from the primary.

## Assignment

Traffic is growing and the database is starting to sweat. Engineering wants to offload read traffic to a replica so the main DB can focus on writes.

**Create a read replica (`patientping-replica`) from your existing `patientping-db` instance.**

**Cost check:** Read replicas cost the same as the primary instance (given the same instance class). A `db.t3.micro` replica costs about $0.018 per hour (~$13 per month). We'll delete it after testing.

1.  Navigate to "RDS" → "Databases" in the AWS Console.
2.  Select your primary database (`patientping-db`).
3.  Click "Actions" → "Create read replica."
4.  Configure the replica:
    -  **DB instance identifier:** `patientping-replica`
    -  **DB instance class:** Leave default (`db.t3.micro`)
    -  **AWS Region:** Leave default (`us-east-1`)
    -  **Storage:** Leave default (same as primary)
    -  **Availability:** `Single-AZ DB instance deployment (1 instance)`
    -  **Network type:** `IPv4`
    -  **DB subnet group:** Same as primary (`patientping-private-subnet-group`)
    -  **Public access:** `No` (keep it private like the primary)
    -  **VPC security groups:** Select your existing RDS security group (`patientping-rds-sg`)
    -  **Availability Zone:** `No preference`
    -  **Database authentication:** `Password authentication` (credentials will be the same as the primary)
    -  **Monitoring:** Select `Database Insights – Standard` and disable everything else
5.  Click "Create read replica."
6.  Wait for the replica to come online (this can take 5–10 minutes).

Once the replica status shows `Available`, check its domain name. You'll see that it's _different_ from the primary DB.


Your applications (or data analysts?) can connect to the read replica just like the primary database, but remember: **it's read-only**. Any write operations (`INSERT`, `UPDATE`, `DELETE`) will fail. Only `SELECT` queries will work.

## Tip

For CLI users, create a read replica with the following command:

```bash
aws rds create-db-instance-read-replica --db-instance-identifier patientping-replica --source-db-instance-identifier patientping-db --db-instance-class db.t3.micro --vpc-security-group-ids YOUR-SG-ID --no-publicly-accessible
```