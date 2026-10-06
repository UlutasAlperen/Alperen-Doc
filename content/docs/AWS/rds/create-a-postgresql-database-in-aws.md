---
title: "create-a-postgresql-database-in-aws"
weight: 20
---

# Create a PostgreSQL Database

Alright, let's get a database set up and ready to use.

## Assignment

PatientPing's data needs a proper home. Everyone is in constant fear that the next time Zach reboots things, we're going to lose data.

**Create a PostgreSQL RDS instance** (`patientping-db`) in your `patientping` VPC's private subnets (no public access!).

**Cost check:** A `db.t3.micro` instance costs about $0.018 per hour (~$13/month). **Be sure to delete this** when you're done. AWS only allows you to "pause" an RDS instance for a time, then it starts it up again.

1. Navigate to "Aurora and RDS" using the search bar at the top of the AWS Console.
2. Click "Subnet groups" in the left-hand menu. _We're going to start by defining which subnets our DB instance can be placed in, to ensure it doesn't end up in a public subnet._
3. Click the orange "Create DB subnet group" button at the top right.
4. Configure the subnet group:
    -  **Name:** `patientping-private-subnet-group`
    -  **Description:** `Private subnets for RDS DB instances`
    -  **VPC:** Select `patientping`
    -  **Availability Zones:** Select `us-east-1a` and `us-east-1b`
    -  **Subnets:** Select `patientping-private-a` and `patientping-private-b`
5. Click "Create"; it should complete immediately.
6. _Still in the "Aurora and RDS" area_, click "Databases" in the left-hand menu.
7. Click the orange "Create database" button at the top right.
8. You'll be greeted with a large number of options. Start by selecting "Full configuration" (_not_ "Express configuration").
9. Choose PostgreSQL as the engine.
10. Under "Engine version," select the latest available version of `PostgreSQL 17`.
11. Under "Templates," select "Free Tier" if available, otherwise "Sandbox" or "Dev/Test." Regardless, make sure "Availability and durability" is set to **"Single-AZ" with 1 instance** (to keep things simple and minimize cost).
12. Configure database settings:
    -  **DB instance identifier:** `patientping-db`
    -  **Master username:** `postgres` (default)
    -  **Credentials management:** "Self managed" (default)
    -  **Master password:** Create a strong password _using only alphanumeric characters_ (to avoid encoding issues), and save it
	
        Make sure you save your database password securely! You won't be able to connect to the DB from your application server without it.
		
    -  **Database authentication options:** "Password authentication" (default)
	
---
1. Configure instance:
    
    -  **Instance type:** `db.t3.micro`
    -  **Storage:** Leave default (`20 GiB`, `gp2`)
    -  **Storage autoscaling:** Disable
    
    **AWS Free Tier issue:** If the instance type dropdown is disabled, or all the instance types are greyed out, you'll need to create a database using the AWS CLI instead.
    
    Go directly to the Tips section below and use the provided CLI commands.
    
2. Configure connectivity:
    
    -  Select "Don't connect to an EC2 compute resource" (we'll do that later)
    -  **VPC:** Select `patientping`
    -  **DB subnet group:** Select `patientping-private-subnet-group`
    -  **Public access:** `No` (critical: database should not be publicly accessible!)
    -  **VPC security group:** Select "Create new," and name it `patientping-rds-sg`
3. **Availability Zone:** `us-east-1a`
    
4. Configure monitoring:
    
    -  Select "Database Insights – Standard" (rather than "Advanced")
    -  Disable "Performance Insights" and everything else in this section
5. Set "Additional configuration":
    
    -  **Initial database name:** `patientping`
    -  **DB parameter group:** Leave default
    -  **Backup:** Leave "Enable automated backup" enabled
    -  **Backup retention period:** `1 day`
    -  **Encryption:** Enable if possible
    -  **Maintenance:** Leave default
6. Finally, click the orange "Create database" button at the bottom.
    
7. Wait for the database to come online; this can take 5–10 minutes.

Once the database status shows `Available`

![PostgreSQL database available in the RDS console](/images/aws/create-postgresql-database.png)
## Tips

If you want (or need) to use the AWS CLI instead of the web console, you can use the following commands.

-  Check if you already created `patientping-private-subnet-group`:
```bash
aws rds describe-db-subnet-groups \
  --db-subnet-group-name patientping-private-subnet-group
```

-  If you need to create the subnet group, get the subnet IDs of `patientping-private-a` and `patientping-private-b`:

```bash
aws ec2 describe-subnets \
  --filters Name=tag:Name,Values=patientping-private-a --query 'Subnets[0].SubnetId'
aws ec2 describe-subnets \
  --filters Name=tag:Name,Values=patientping-private-b --query 'Subnets[0].SubnetId'
```

-  Create the RDS subnet group:
```bash
aws rds create-db-subnet-group \
  --db-subnet-group-name patientping-private-subnet-group \
  --db-subnet-group-description 'Private subnets for RDS DB instances' \
  --subnet-ids PRIVATE_A_ID PRIVATE_B_ID
```

-  Before creating the database itself, make a security group for it. You'll need your VPC ID:
   
```bash
aws ec2 describe-vpcs \
  --filters 'Name=tag:Name,Values=patientping' --query 'Vpcs[0].VpcId'
```

-  Create the security group, and _note its `GroupId`_:

```bash
aws ec2 create-security-group \
  --group-name patientping-rds-sg \
  --description 'Security group for PatientPing RDS' \
  --vpc-id VPC_ID
```

-  Create the actual database instance:
```bash
aws rds create-db-instance \
  --db-instance-identifier patientping-db \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --master-username postgres \
  --master-user-password YOUR_ALPHANUMERIC_PASSWORD \
  --allocated-storage 20 \
  --storage-type gp2 \
  --db-name patientping \
  --db-subnet-group-name patientping-private-subnet-group \
  --vpc-security-group-ids SECURITY_GROUP_ID \
  --availability-zone us-east-1a \
  --no-publicly-accessible \
  --no-multi-az \
  --backup-retention-period 1 \
  --no-enable-performance-insights \
  --no-deletion-protection
```

-  Wait until the database becomes `Available` (allow 5–10 minutes for this command to return):

```bash
aws rds wait db-instance-available --db-instance-identifier patientping-db
```