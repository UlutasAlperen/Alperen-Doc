---
title: "cloudfront-invalidation"
weight: 480
---

# CloudFront Invalidation

A CDN is a geographically distributed [cache](https://en.wikipedia.org/wiki/Cache_\(computing\)), which means it's one of the hardest things to deal with...

> "There are 2 hard problems in computer science: cache invalidation, naming things, and off-by-1 errors."
> 
> – Leon Bambrick

If you update the favicon in your S3 bucket, it won't immediately update for users accessing it through CloudFront.

Similarly, if you use CloudFront to serve content like HTML, CSS, or JavaScript files for a website, users might not see the latest version of your site immediately after you make changes.

For that reason, you need to know how to [invalidate](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Invalidation.html) cached content in CloudFront. This forces CloudFront to fetch the latest version from the origin (your S3 bucket) instead of serving the cached version. It basically says:

> "Hey! There's new content, go update the cache!"

## Assignment

PatientPing has gone _stylish_. Marketing wants to use a new black and white version of the logo.

**Upload the new logo to S3 and invalidate the CloudFront distribution cache so that users see the new logo immediately.**

1.  Download the new black and white favicon:
    
```bash
curl -o favicon.ico https://storage.googleapis.com/qvault-webapp-dynamic-assets/course_assets/patientping-bw-favicon.ico
```
    
2.  In the AWS Console, navigate to `S3` and open your `patientping-favicon-bucket-YOUR_SUFFIX` bucket.
3.  Upload `favicon.ico` to the bucket to _replace_ the existing file.
4.  Load the favicon via CloudFront in the browser (e.g. `https://YOUR-ID.cloudfront.net/favicon.ico`) and hard-refresh the page (Ctrl/Cmd+Shift+R). _Notice you should still see the old version of the favicon!_
5.  In the AWS Console, navigate to `CloudFront` and open your distribution.
6.  Open the `Invalidations` tab and create a new invalidation:
    -  **Object paths:** `/favicon.ico` (we _could_ use a `*` to invalidate everything, but it's best practice to be specific and only invalidate what you need to).
    -  Submit the invalidation request.
7.  Wait for invalidation status to become `Completed` (it may be `InProgress` for a moment).
8.  Verify the update by opening your CloudFront URL (for example, `https://YOUR_DISTRIBUTION_DOMAIN/favicon.ico`) and hard-refreshing (Ctrl/Cmd+Shift+R) the page. _You should now see the new black and white version of the favicon_!

The CLI tests check [CloudTrail](https://aws.amazon.com/cloudtrail/) for your invalidation event. CloudTrail events can take a few minutes to appear, so wait 2-5 minutes after the invalidation completes. 
## Tip

If you prefer to use the CLI, here is the command structure:

```sh
aws cloudfront create-invalidation \
  --distribution-id YOUR_DISTRIBUTION_ID \
  --paths "/favicon.ico"
```