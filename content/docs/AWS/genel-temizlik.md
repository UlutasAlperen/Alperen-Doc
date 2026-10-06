---
title: "genel-temizlik"
weight: 730
---

# Cleanup

Delete the read replica now, then optionally clean up the rest if you're taking a break.

**Cost check:** RDS instances, EC2 instances, and Elastic IPs cost money while they exist. Delete what you don't need when you're not actively working.

## Assignment

**Delete the read replica, then optionally clean up additional resources.**

1. [ ] Delete the RDS read replica.
    1. [ ] Navigate to "RDS" → "Databases" in the AWS Console.
    2. [ ] Select your read replica (`patientping-replica`).
    3. [ ] Click "Actions" → "Delete."
    4. [ ] Type `delete me` to confirm, then click "Delete."
    5. [ ] Wait for deletion to complete.

### Optional: Only Do This If You're Taking an Extended Break

1. [ ] Optional: delete the RDS primary database.
    1. [ ] Navigate to "RDS" → "Databases" in the AWS Console.
    2. [ ] Select your primary database (`patientping-db`).
    3. [ ] Click "Actions" → "Delete."
    4. [ ] Configure deletion:
        - [ ] **Create final snapshot:** Uncheck
        - [ ] **Retain automated backups:** Uncheck
    5. [ ] Click the "I acknowledge..." checkbox if required, then type `delete me` to confirm.
    6. [ ] Click "Delete."
2. [ ] Optional: delete the DB subnet group (`patientping-private-subnet-group`).
    1. [ ] Navigate to "RDS" → "Subnet groups" in the AWS Console.
    2. [ ] Select `patientping-private-subnet-group`.
    3. [ ] Click "Delete" and confirm.
3. [ ] Optional: delete the RDS security group (`patientping-rds-sg`).
    1. [ ] Navigate to "EC2" → "Security Groups" (under "Network & Security") in the AWS Console.
    2. [ ] Select `patientping-rds-sg`.
    3. [ ] Click "Actions" → "Delete security groups."
    4. [ ] Click "Delete" to confirm.
4. [ ] Optional: terminate your app server EC2 instance (`patientping-web-v2`).
    1. [ ] Navigate to "EC2" → "Instances" in the AWS Console.
    2. [ ] Select `patientping-web-v2`.
    3. [ ] Click "Instance state" → "Terminate (delete) instance."
    4. [ ] Confirm termination and wait for the state to become `Terminated`.
5. [ ] Optional: release the app server Elastic IP.
    1. [ ] Navigate to "EC2" → "Elastic IPs" in the AWS Console.
    2. [ ] Select the Elastic IP you used for the app server.
    3. [ ] Click "Actions" → "Release Elastic IP addresses," then confirm.