---
title: "why-use-presigned-urls"
weight: 500
---

# Why Use Presigned URLs?

You just learned how to create presigned URLs, but you might be wondering: _why go through all this trouble_? Well, imagine you're building a course platform where students can download PDFs of their course materials. If you make the PDFs public in S3 then _all the content would be accessible for free_!

...and unless you're as cool as Boot.dev, you probably don't want that.

- Anyone with the URL can access the files (even people who aren't enrolled)
- Search engines and bots can crawl and index your content
- You have no control over who downloads what
- Once someone has the URL, they can share it forever

This is where **presigned URLs** shine. Instead of making your S3 bucket or CloudFront content publicly accessible, you can use presigned URLs to get the best of both worlds: CDN speed with a private-content access control. You dish out presigned URLs to paying users, they download the content before the URL expires, and once it does, their URL becomes useless. By setting _extremely short_ expiry times, and generating a new URL when a paying user requests it, they can effectively keep access forever, but will only be able to get the content if they're going through your link generator.

**Cost check:** Presigned URL generation is free. You still pay for S3 and CloudFront when users access content via the URL; no extra charge for the URL itself.

Presigned URLs are perfect for:

- **Static course platforms:** Students download materials they've purchased
- **Document sharing:** Share sensitive documents with clients temporarily
- **Media libraries:** Let users download their purchased music/videos
- **Backup downloads:** Allow users to download their data exports securely
- **Software distribution:** Distribute licensed software to paying customers

We share a lot of image files for one of our clients, but we'd really prefer not to be their "free CDN" service. So we've started sending them with presigned URLs so they're forced to store the files on their end if they want to serve them to their users.