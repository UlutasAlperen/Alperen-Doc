---
title: "aws-connect-app-to-database"
weight: 210
---

# Connect App to Database

We've seen that we can use `psql` from our EC2 instance to connect to the Postgres DB running in RDS. That's no small feat; a bunch of things had to be configured _just right_ for it to work.

The good news is that it will take only one more small step to get the PatientPing app communicating with the database _automatically_. Instead of using `psql` manually, we'll set a `DATABASE_URL` environment variable that the Python app can read when it starts up!

## Assignment

The PatientPing app still can't connect to the Postgres database! **Fix the problem by adding a `.env` file with DB connection info on the EC2 instance.**

1.  SSH into the EC2 instance and navigate to the app directory:
```bash
ssh patientping
cd ~/patientping-web
```
> To keep things simple, make sure the app isn't running at this point.

2.  In that directory, use a terminal text editor (e.g. `nano` or `vim`) to open a new `.env` file:

```bash
nano .env
```

3.  Enter a `DATABASE_URL` environment variable in the following format, with your actual values:

```bash
DATABASE_URL='postgresql://postgres:PASSWORD@patientping-db.RANDOM-ID.us-east-1.rds.amazonaws.com:5432/patientping'
```

4.  Save and exit the `.env` file (Ctrl+X, Y, Enter in `nano`; [good luck](https://stackoverflow.com/questions/11828270/how-do-i-exit-vim) in `vim`).
5.  Confirm that the `.env` file looks correct, and that you're in the right directory:

```bash
pwd       # /home/ec2-user/patientping-web
cat .env  # DATABASE_URL='postgresql://postgres:PASSWORD...'
```

6.  Start the server:
```bash
uv run patientping.py
```

7.  Confirm that things are working properly:
    -  If the app was able to connect to the database on startup, you should see a `DB init ok` message in the terminal.
    -  You should also see PatientPing appointment reminders being generated and inserted into the DB every ~5 seconds.
    -  Open a browser on your local machine and visit `http://YOUR.EC2.INSTANCE.IP:8080`. The app should load; status should be green with `DB connected` and `Internet egress OK`; and you should see a list of reminders (refresh the page to see new ones).
