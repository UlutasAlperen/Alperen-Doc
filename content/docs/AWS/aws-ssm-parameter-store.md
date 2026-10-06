---
title: "aws-ssm-parameter-store"
weight: 310
---

# SSM Parameters

A ubiquitous problem with **running applications in production** is _configuring_ servers and applications. For example:

- Should the app run in debug mode?
- What domain name should it expect?
- Where should it send logs?

On our own machines, we often use _environment variables_: key-value pairs that are available to everything running in the local environment (e.g. `EDITOR=nvim`). In Kubernetes, we use `ConfigMaps` or `Secrets` to store configuration values.

AWS provides a vendor-specific solution in [AWS Systems Manager Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html) (SSM parameters). It's a centralized safe where you can keep configuration data, secrets, and parameters that your applications need. It provides:

- **Key-value storage:** Store configuration values, secrets, and parameters
- **Organization by path:** Use hierarchical paths like `/DB_ENDPOINT` or `/SECRETS`
- **Security:** Can encrypt sensitive values (`SecureString` type)
- **Accessibility:** Applications and EC2 instances can retrieve parameters via IAM permissions
- **Versioning:** Track changes to parameter values over time

I often encounter applications where part of the config is "computed" at startup or during deployment (stuff you can't know ahead of time, or that depends on the environment). SSM parameters make it easy to adjust those magic values in a central place, and let your application pull its exact configuration whenever it needs.

## Assignment

Setting a `DATABASE_URL` env var got the Pinger server up and running, but PatientPing wants to do things right and start saving app config in SSM parameters.

**Create parameters `/DATABASE_URL` (an RDS instance connection URL) and `/CMO_NAME` (a value your app will use) in Parameter Store.**

**Cost check:** Standard SSM parameters are free for up to 10,000 parameters. Advanced parameters (with encryption) cost $0.05 per parameter per month.

1.  Navigate to "Systems Manager" (SSM) → "Parameter Store" (under "Application Tools") in the AWS Console.
2.  Ensure your region is set to "N. Virginia (us-east-1)," as the Pinger app code is configured to use SSM parameters from this region.
3.  Click the orange "Create parameter" button.
4.  Add a parameter for your database endpoint:
    -  **Name:** `/DATABASE_URL`
    -  **Tier:** `Standard`
    -  **Type:** `String`
    -  **Data type:** `text`
    -  **Value:** `postgresql://postgres:PASSWORD@patientping-db.RANDOM-ID.us-east-1.rds.amazonaws.com:5432/patientping` (replace with _your_ RDS instance ID and password)
    -  Click "Create parameter."
5.  Add another parameter: the name of PatientPing's Chief Medical Officer.
    -  **Name:** `/CMO_NAME`
    -  **Tier:** `Standard`
    -  **Type:** `String`
    -  **Data type:** `text`
    -  **Value:** `Dr. Strangelove`
    -  Click "Create parameter."

You should now see `DATABASE_URL` and `CMO_NAME` in the "My parameters" list.
## Tip

You can also create SSM parameters via the AWS CLI:

```sh
aws ssm put-parameter --name /DATABASE_URL --value postgresql://postgres:PASSWORD@patientping-db.RANDOM-ID.us-east-1.rds.amazonaws.com:5432/patientping --type String
aws ssm put-parameter --name /CMO_NAME --value 'Dr. Strangelove' --type String
```