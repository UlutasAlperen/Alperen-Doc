---
title: "rds-backups"
weight: 220
---

# RDS Backups

> One is none, two is one.

Backups are important. We're going to break things in creative, irreparable ways from time to time, and RDS can automatically back up your database.

- **Daily backups:** Taken during your preferred backup window (you can set this)
- **Transaction logs:** Continuously backed up (enables point-in-time recovery)
- **Retention period:** Configurable (1-35 days for most engines)
- **Storage:** Backups stored in S3, separate from your database instance
- **Cost:** Storage for backups is charged separately (about `$0.095` per GB per month)

**Cost check:** Backups cost money. A 7-day retention period means you're storing 7 days of backups (at ~`$0.095/GB/month`). For a 20 GB database that can be ~140 GB of backup storage. Monitor backup costs, and reduce retention or delete snapshots when done.

## What's in a Backup?

RDS backups include:

1. **Full database snapshot:** Complete copy of your database at a point in time
2. **Transaction logs:** All changes since the last backup (enables point-in-time recovery)

The combination of full backups plus transaction logs lets you restore to _any_ specific time within your retention period.

## Restoring from Backups

You can restore your database in several ways.

### Point-in-Time Recovery

Restore to any specific time within the retention period:

- Uses full backup + transaction logs
- Can restore to the exact second before a data loss event
- Useful when you know exactly when something went wrong

### Restore from Snapshot

Restore to a specific backup snapshot:

- Faster and simpler than point-in-time recovery
- Useful for restoring to a known good state
- You can also create manual snapshots at any time

### Restore to New Instance

All restores create a **new database instance**:

- Original instance remains unchanged
- This means "restoring" is also useful for testing, cloning, or migration
- You can restore to test changes before applying to production

Backups are the preferred method for cloning databases. Instead of exporting and importing data manually, just restore from a backup. It's faster, more reliable, and creates an exact copy. You don't even need the DB's credentials to migrate it.