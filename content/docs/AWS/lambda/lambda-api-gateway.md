---
title: "lambda-api-gateway"
weight: 40
---

# API Gateway

Your Lambda function works, but it can only be invoked from the AWS CLI or AWS Console. To make it accessible over HTTP (so you can call it from a browser, mobile app, or any HTTP client), we need [**Amazon API Gateway**](https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html).

## What Is API Gateway?

**API Gateway** is a managed service that creates, publishes, and manages APIs. It sits in front of your Lambda function and handles much of what a web server or proxy would do. Have a look at this diagram:

![API Gateway + Lambda Architecture](https://storage.googleapis.com/qvault-webapp-dynamic-assets/course_assets/2QaTlEh-880x760.png)

Another critical concept: we can connect **multiple Lambda functions to the same API Gateway**. This lets us hook up arbitrary Lambda functions into a single application. It can be a wonderful convenience, or indecipherable spaghetti.

When you make an HTTP request through API Gateway, it captures your actual IP address and includes it in the event it sends to Lambda. Unlike CLI and AWS Console tests where you provide mock data, this is a real HTTP request traveling over the internet, so you'll see your real IP in the response!

I love [prototype-driven development](https://www.machow.ski/posts/galls-law-and-prototype-driven-development/); it's a powerful tool to build a toy version of the app or feature you're thinking about to see what it would take to deploy it in production. API gateways and lambdas can be a great way to get your proof of concept online.

## Assignment

**Create an API Gateway HTTP API named `patientping-ip-api` that exposes your Lambda function over HTTP.** (Optional: repeat with the CLI.)

**Cost check:** [API Gateway](https://aws.amazon.com/api-gateway/pricing/) charges for **HTTP requests** and **data transfer** (pay-as-you-go). HTTP APIs run about **`$1.00` per million requests**. This should put our testing in the less than $.01 category.

1.  Create the HTTP API from the console:
    1.  Open [**API Gateway**](https://console.aws.amazon.com/apigateway) in the AWS Console.
    2.  Click **Create an API** and under **HTTP API**, click **Build**.
    3.  Enter the API name **`patientping-ip-api`**.
    4.  Under **Integrations**, click "Add" and use **Lambda** as the integration type.
    5.  Select **`patientping-ip-function`**
    6.  Click **Next**.
    7.  **Configure routes:** add a route with **Method** `GET`, **Resource path** `/`, and **Integration target** the same Lambda (`patientping-ip-function`).
    8.  Click **Next**.
    9.  Leave **Stage name** as **`$default`**, leave **Auto-deploy** enabled.
    10. lick **Next**, review, then **Create**.
2.  Test the API:
    1.  Navigate to "Deploy" → "Stages" in the left sidebar in the API Gateway console, and click on the stage named `$default`.
    2.  Copy the "invoke URL" and paste it into your browser. **You should get a live response with your real IP address!**