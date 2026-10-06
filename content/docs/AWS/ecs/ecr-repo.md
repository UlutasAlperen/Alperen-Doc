---
title: "ecr-repo"
weight: 40
---

# ECR Repo

Now that we have a Docker image, let's host it on ECR!

## Assignment

**Create an ECR repository `patientping-ecs`, and push the image.**

**Cost check:** ECR storage costs about $0.10 per GB per month. For a small image like ours, which we'll keep on ECR only long enough for learning/testing, this is essentially free. Data transfer costs also apply when pulling images, but again, for learning purposes it's minimal.

1. Navigate to "ECR" using the search bar in the AWS console
2. Click "Create repository"
3. Ensure **Private** is selected, name it `patientping-ecs`, and click "Create"
4. Click into your repository and click the "View push commands" button:
  ![ecr-push-commands](https://storage.googleapis.com/qvault-webapp-dynamic-assets/course_assets/wMpmDHG-477x312.png)
5. Copy and paste the first command into your terminal to login to ECR.
6. Build and push a linux/amd64 image so ECS can run it:
```bash
docker buildx build \
  --platform linux/amd64 \
  --tag "YOUR-ACCT-ID.dkr.ecr.us-east-1.amazonaws.com/patientping-ecs:latest" \
  --push .
```
7. Navigate to your `patientping-ecs` repository in the AWS console and confirm that your `latest` image is there.
## Tip

If you prefer the CLI:

```sh
aws ecr create-repository --repository-name patientping-ecs

my_account_id=$(aws sts get-caller-identity --query Account --output text)
my_region=$(aws configure get region)

aws ecr get-login-password --region "${my_region}" | docker login \
  --username AWS \
  --password-stdin "${my_account_id}.dkr.ecr.${my_region}.amazonaws.com"

docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag "${my_account_id}.dkr.ecr.${my_region}.amazonaws.com/patientping-ecs:latest" \
  --push .

# Fallback if multi-platform is unsupported on your Docker setup:
docker buildx build \
  --platform linux/amd64 \
  --tag "${my_account_id}.dkr.ecr.${my_region}.amazonaws.com/patientping-ecs:latest" \
  --push .
```