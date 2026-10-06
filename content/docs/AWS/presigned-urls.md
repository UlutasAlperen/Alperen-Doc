---
title: "presigned-urls"
weight: 490
---

# Presigned URLs

Serving files is nice, but we often want CDN speed while still limiting access to specific users. To do that, we can use **presigned URLs**.

[**Presigned URLs**](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html) are temporary, time-limited URLs that grant access to private S3 objects or CloudFront content. They're like tickets that expire:

- **Time-limited:** The URL expires after a set duration (e.g., 1 hour, or 24 hours).
- **Secure:** The content itself remains private; only those with the presigned URL can access it.
- **Controlled:** You control who gets the URL and when it expires.

Here's what a presigned URL looks like:

```text
https://YOUR_BUCKET_NAME.s3.amazonaws.com/favicon.ico?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=...&X-Amz-Date=20240101T120000Z&X-Amz-Expires=3600&X-Amz-SignedHeaders=host&X-Amz-Signature=...
```

That long query string contains the cryptographic signature and expiration time. Without it, the cloudfront servers will reject the request.

Presigned URLs are powerful, but remember: anyone with the URL can access the content until it expires! Don't share these links publicly if you want to keep content private. Once someone has the URL, they can use it until it expires.

## Assignment

**Create a presigned URL for your favicon using the AWS CLI**.

**Cost check:** Presigned URLs don't add any additional cost. You're still paying for S3 storage and CloudFront data transfer, but the URL generation itself is free. The security benefit is worth it.

1. [ ] Run the following command, replacing `YOUR_BUCKET_NAME` with your S3 bucket name (e.g. `patientping-favicon-bucket-x7k9m2`), and adjusting the expiration time if needed:
    
    ```sh
    aws s3 presign s3://YOUR_BUCKET_NAME/favicon.ico --expires-in 15
    ```
    
2. [ ] Copy the URL that is returned, and open it in a browser to ensure it works.
3. [ ] Wait at least 15 seconds, then try opening the URL again. _It should no longer work since it has expired_!