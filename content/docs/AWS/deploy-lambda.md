---
title: "deploy-lambda"
weight: 650
---

# Deploy Lambda

[AWS Lambda handlers in Python](https://docs.aws.amazon.com/lambda/latest/dg/python-handler.html) are just functions that:

- Accept an `event` (input data)
- Return a response object

In other words, we don't worry about the runtime, the server, or even any sort of HTTP library. We just write a function that accepts an event object and returns a response object.

The process to create a new Lambda function is simple:

1. Create an [IAM role](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html) that gives Lambda permission to write logs to [CloudWatch](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html)
2. Create the Lambda function itself
3. Paste the handler code and deploy

## Assignment

**Create and deploy `patientping-ip-function` with role `patientping-lambda-role` by pasting Python code in the AWS Console.**

**Cost check:** Lambda functions themselves don't cost anything when not running. Storage for the deployment package costs about $0.0000000034 per GB per hour (basically free for a small function like this).

1.  In the AWS Console, navigate to `IAM` → `Roles`, then create a role:
    -  Trusted entity type: `AWS service`
    -  Use case: `Lambda`
    -  Permissions policy: `AWSLambdaBasicExecutionRole`
    -  Role name: `patientping-lambda-role`
2.  In the AWS Console, navigate to `Lambda` and click **Create function**.
3.  Choose **Author from scratch**, then set:
    -  Function name: `patientping-ip-function`
    -  Runtime: `Python 3.14` (or latest available Python 3.x runtime)
    -  Under **Permissions**, select the existing `patientping-lambda-role` role (you may need to expand **More settings** or look for a **Custom execution role** option)
    -  Leave all additional configurations as default and click **Create function**.
4.  In the function's **Code** tab, replace the default code in `lambda_function.py` with:
```python
def lambda_handler(event, context):
    request_context = event.get("requestContext", {})
    identity = request_context.get("identity", {})
    ip_address = identity.get("sourceIp")
    if not ip_address:
        headers = event.get("headers", {})
        ip_address = headers.get("X-Forwarded-For") or headers.get("x-forwarded-for")
    if not ip_address:
        ip_address = "unknown"
    print(f"Received IP: {ip_address}")
    return {
        "statusCode": 200,
        "headers": {
            "Content-Type": "text/plain",
        },
        "body": f"Your IP address is: {ip_address}",
    }
```

5.  Keep the default handler `lambda_function.lambda_handler`, then click **Deploy**.

If you prefer CLI + local zip:

```bash
cat > lambda_function.py <<'PY'
def lambda_handler(event, context):
    request_context = event.get("requestContext", {})
    identity = request_context.get("identity", {})
    ip_address = identity.get("sourceIp")
    if not ip_address:
        headers = event.get("headers", {})
        ip_address = headers.get("X-Forwarded-For") or headers.get("x-forwarded-for")
    if not ip_address:
        ip_address = "unknown"
    print(f"Received IP: {ip_address}")
    return {
        "statusCode": 200,
        "headers": {
            "Content-Type": "text/plain",
        },
        "body": ip_address,
    }
PY

zip lambda-function.zip lambda_function.py
aws iam create-role --role-name patientping-lambda-role --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
aws iam attach-role-policy --role-name patientping-lambda-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws lambda create-function --function-name patientping-ip-function --runtime python3.12 --role arn:aws:iam::YOUR_ACCOUNT_ID:role/patientping-lambda-role --handler lambda_function.lambda_handler --zip-file fileb://lambda-function.zip
```

In the next lesson, we'll test this function and learn how to update it.