---
title: "aws-s3-bucket"
weight: 20
---

# S3 Buckets

The _bucket_ is the highest level of organization in S3; it's a _container for storing objects_.

Some projects put everything in a single bucket, organizing objects by [prefix](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-prefixes.html) (like folders). Other projects use multiple buckets for different purposes (for example, one for user uploads and another for static assets). There's no single right answer; it depends on your needs.

That said, you don't want to find yourself _searching across multiple buckets_ for a file, so try to keep any collection of related files in the same bucket.

Buckets have _globally unique names_ because they're part of the URL used to access them. If I make a bucket called `bd-vids`, you cannot also have a `bd-vids` bucket, even if you're in a separate AWS account.

Use the pattern suggested in the previous lesson: `patientping-favicon-bucket-${codename}`. Pick a random unique codename so your bucket name doesn't collide with others.

## Example

PatientPing's site is missing a favicon (the little icon in the browser tab). Instead of baking it into the app, we're going to put it in S3 so we can eventually serve it (and other static assets) through AWS' CloudFront CDN.

**Cost check:** S3 storage costs about $0.023 per GB per month. A small file like the one we'll be using is effectively free.

**Create the S3 bucket:**

1.  Navigate in the AWS Console to "S3" → "Buckets" → "General purpose buckets" and click "Create bucket".
2.  Name it `patientping-favicon-bucket-CODENAME`, where `CODENAME` is any random character sequence you like (so that you avoid a naming collision)
3.  Leave ACLs disabled
4.  **Uncheck** "Block all public access". We want to serve an image over the internet after all!
    
    Accidentally publishing sensitive information about your company in S3 buckets has happened [more than a few times](https://github.com/nagwww/s3-leaks). It's led to a comical number of "Are you really sure you want to publish this bucket globally?!" prompts. Just smile at this [Chesterton's fence](https://en.wikipedia.org/wiki/G._K._Chesterton#Chesterton's_fence) and carry on.
    
5.  Leave versioning disabled.
6.  Leave encryption default.
7.  Click "Create Bucket".

## Tip

If you prefer to use the CLI:

```sh
aws s3 mb s3://patientping-favicon-bucket-YOUR_CODENAME --region us-east-1

aws s3api put-public-access-block \
  --bucket patientping-favicon-bucket-YOUR_CODENAME \
  --public-access-block-configuration "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"
```