---
title: "testing-lambda-function"
weight: 660
---

# Testing Lambda Functions

Your Lambda function is deployed, now let's test it. Lambda provides several ways to [invoke functions](https://docs.aws.amazon.com/lambda/latest/dg/lambda-invocation.html), and we'll explore a few.

When you test a Lambda function directly (via the CLI or AWS Console), you're _not_ making an HTTP request to it. You're telling AWS to execute your code with a specific payload. Take a look at this diagram:

![Lambda cli invocation](https://storage.googleapis.com/qvault-webapp-dynamic-assets/course_assets/WvQZMrd-527x720.png)

## Invoke from the CLI

For our function, we need to simulate what [API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html) _would_ send. When you invoke the function directly via the console, you're providing mock data. The IP address in your test event is not your real IP address; it's example data you're feeding to the function.

## Cold Start vs. Warm Start

When you invoke a Lambda function, you'll notice different performance characteristics:

1. **First invocation (cold start):** Might take 200-500 ms. This includes the time to provision a container, load your code, and start the [Python](https://www.python.org/) runtime.
2. **Subsequent invocations (warm):** Might take just 1-5 ms, if AWS reuses the existing container.

The logs will show an `Init Duration` on the first run, that's the [cold start](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html#cold-start-latency) overhead. On subsequent runs (within a few minutes), you usually won't see `INIT_START` or `Init Duration`.

Cold starts can be frustrating for user-facing APIs. For truly latency-sensitive applications, you might want to keep functions warm (there are techniques like scheduled invocations or provisioned concurrency). But for many use cases, a 200-500 ms cold start is totally acceptable.

## Assignment

**Test your Lambda function with different scenarios.**

**Cost check:** Each Lambda invocation counts toward your free tier (1 million requests/mo). These test invocations will use about 10-20 requests total.

1.  In the AWS Console, navigate to `Lambda` → `Functions` and open `patientping-ip-function`.
2.  Open the **Test** tab and create a test event named `test-ip` with this payload:

```json
{
  "requestContext": {
    "identity": {
      "sourceIp": "8.8.8.8"
    }
  },
  "headers": {}
}
```

3.  Run the test. You should see a nice green "success" box with a JSON response that includes a `200` status code and a "Your IP address is: 8.8.8.8" message in the body.
4.  Create another test event named `test-xff` with this payload:

```json
{
  "requestContext": {
    "identity": {}
  },
  "headers": {
    "X-Forwarded-For": "198.51.100.99"
  }
}
```

5.  Run `test-xff` and confirm the response body is "Your IP address is: 198.51.100.99"