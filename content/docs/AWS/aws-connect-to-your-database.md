---
title: "aws-connect-to-your-database"
weight: 200
---

# Connect to Your Database

Now that you have a database, we need our application server to connect to it. After all, they're in different subnets (public vs. private) in the same VPC. AWS adds a **local route** for your VPC CIDR by default, so subnets in that VPC can route to one another.

The key is **security groups**. Remember, security groups are like stateful firewalls that control traffic to and from your resources.

"Stateful" here just means we allow the return trip by default. If the firewall lets you out to [https://www.boot.dev/](https://www.boot.dev/), when that server responds to you it will also be allowed through the firewall.

## Connection Path

Here's how traffic will flow from your application server to your database:

1. **Application server initiates connection**
    - Your server needs an **outbound rule** allowing traffic to the database.
    - Port: `5432` (PostgreSQL default port)
    - _If your server has a default outbound rule allowing all traffic, you won't need to add a specific rule here._
2. **Traffic routes through VPC**
    - VPCs allow internal communication between subnets by default.
    - Your `patientping-public-a` subnet can reach your `patientping-private-a` subnet automatically.
3. **Database accepts connection**
    - Your RDS instance needs an **inbound rule** allowing traffic from your application servers.
    - Port: `5432`
    - Source: The security group of your application servers
    - _Inbound connections, unlike outbound, must be managed carefully and only via specific rules._

## Assignment

**Configure security groups so your application server can reach the RDS instance on port `5432`.**

**Cost check:** No new billable resources in this lesson. You're only changing security group rules; your existing RDS instance (~$13/mo. for `db.t3.micro`) and EC2 instance costs are unchanged.

1.  Configure the app server security group – **this is likely already done**.
    1.  Navigate to "EC2" → "Security Groups" (under "Network & Security") in the AWS Console.
    2.  Find your application server security group (`patientping-public`) and select it.
    3.  Go to the "Outbound rules" tab.
    4.  Ensure that there's a single IPV4 outbound rule allowing _all traffic_ for _all protocols_ on _all ports_ to _all destinations_ (`0.0.0.0/0`).
2.  Confirm the RDS DB's security group – **this is likely already done**.
    1.  Navigate to "Aurora and RDS" → "Databases" in the AWS Console.
    2.  Select your database (`patientping-db`).
    3.  In the "Connectivity & security" tab, scroll down to "Security group rules" and confirm that all existing rules are associated with `patientping-rds-sg`.
3.  Add the necessary inbound rule to allow the app server to access the DB instance.
    1.  Navigate back to "EC2" → "Security Groups" in the AWS Console.
    2.  Select the RDS DB's security group (`patientping-rds-sg`).
    3.  Go to the "Inbound rules" tab and click "Edit inbound rules."
    4.  Click "Add rule" and use the following settings:
        -  **Type:** `PostgreSQL` (will auto-select TCP port `5432`)
        -  **Source:** `Custom`
        -  **Source (search bar):** Find and select the application server SG (`patientping-public`)
    5.  Click "Save rules."
    6.  Take a moment to review the rules in the RDS security group. You probably have a default inbound rule allowing access from your own computer's IP address, and a default outbound rule allowing all traffic. This is fine; just make sure the new rule allowing access from the app server SG is also present.
4.  Test the connection between your EC2 instance and your RDS database.
    1.  SSH into your EC2 instance: `ssh patientping`
    2.  Update the system: `sudo dnf upgrade`
    3.  Install the PostgreSQL client: `sudo dnf install postgresql17`
    4.  Verify that `psql` is in your `PATH`: `which psql`
    5.  Test the connection to your RDS instance using `psql`. Get _your_ endpoint from the RDS database's page → "Connectivity & security" tab → "Endpoints":
```bash
psql -h patientping-db.XXXXX.us-east-1.rds.amazonaws.com -U postgres -d patientping
```

1.  Enter the password you set during database creation.
2.  If successful, you should see a PostgreSQL prompt like `patientping=>` which means you're in!
3.  Type `\q` to quit the `psql` prompt and return to the shell in your SSH session.


## Tip

For AWS CLI users, here's the syntax to configure security group rules:

```sh
# Add outbound rule to app server security group (assuming you want a specific rule)
aws ec2 authorize-security-group-egress --group-id APP-SG-ID --ip-permissions IpProtocol=tcp,FromPort=5432,ToPort=5432,UserIdGroupPairs=[{GroupId=RDS-SG-ID}]

# Add inbound rule to DB security group
aws ec2 authorize-security-group-ingress --group-id RDS-SG-ID --ip-permissions IpProtocol=tcp,FromPort=5432,ToPort=5432,UserIdGroupPairs=[{GroupId=APP-SG-ID}]
```