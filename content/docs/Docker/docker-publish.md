---
title: "docker-publish"
weight: 12
---
# Publishing to Docker Hub

Let's publish the Go server we Dockerized up to Docker Hub.

1. Rebuild the Go binary:

```bash
GOOS=linux GOARCH=amd64 go build
```

2. Rebuild the image. You'll need to use a name that corresponds to _your_ namespace on Docker Hub. Swap out `USERNAME` for _your_ Docker Hub username.

```bash
docker build . -t USERNAME/goserver
```

3. Run your image in a container to make sure it still works:

```bash
docker run -p 8991:8991 USERNAME/goserver
```

4. Push the image to Docker Hub:

```bash
docker push USERNAME/goserver
```

# Delete and Pull

Let's delete our local copy of the image, then pull it back down from Docker Hub. Just like with GitHub, the nice thing about having images in the cloud is that if something happens to your computer, or you're working on another machine, you can always pull down your images.

1. Remove your local `USERNAME/goserver` image:

```bash
docker image rm USERNAME/goserver
```

2. Pull it back down from Docker Hub:

```bash
docker pull USERNAME/goserver
```

3. Run it to make sure it works:

```bash
docker run -p 8991:8991 USERNAME/goserver
```

# Tags

Let's publish a new _version_ of our web server. With Docker, a tag is a label that you can assign to a specific version of an image, similar to a tag in Git.

The `latest` tag is the default tag that Docker uses when you don't specify one. It's a convention to use `latest` for the most recent version of an image, but it's also common to include other tags, often [semantic versioning](https://semver.org/) tags like `0.1.0`, `0.2.0`, etc.

## Deployment Pipelines

Publishing new versions of Docker images is a _very common_ method of deploying cloud-native back-end servers. Here's a diagram describing the deployment pipeline of many production systems (including the server that powers the Boot.dev site you're on currently).

![](https://storage.googleapis.com/qvault-webapp-dynamic-assets/course_assets/P1cZo5d-1086x720.png)

# Latest

If you look closely, you'll notice that your _old_ version is tagged "latest"... that's a bit confusing. As it turns out, the `latest` tag doesn't always indicate that a specific tag is the latest version of an image. In reality, `latest` is just the _default_ tag that's used if you don't explicitly supply one. We didn't use a tag on our first version, that's why it was tagged with "latest".

## Should I Use “latest”?

The convention I'm familiar with is to use [semantic versioning](https://semver.org/) on all your images, but to _also_ push to the "latest" tag on your most recent image. That way you can keep all of your old versions around, but the `latest` tag still always points to the latest version.

So, for example, if I were updating an application to version 5.4.6, I would probably do it like this:

```bash
docker build -t ulutasalperen/awesomeimage:5.4.6 -t ulutasalperen/awesomeimage:latest .
docker push ulutasalperen/awesomeimage --all-tags
```
# The Bigger Picture

This is the last lesson, but before we go, I want to reiterate how Docker fits into the software development lifecycle, particularly at modern "DevOpsy" tech companies, because it's _really important_ to understand.

## The Deployment Process

1. The developer (you) writes some new code
2. The developer commits the code to Git
3. The developer pushes a new branch to GitHub
4. The developer opens a [pull request](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/about-pull-requests) to the `main` branch
5. A teammate reviews the PR and approves it (if it looks good)
6. The developer merges the pull request
7. Upon merging, an automated script, perhaps a [GitHub action](https://docs.github.com/en/actions), is started
8. The script builds the code (if it's a compiled language)
9. The script builds a new docker image with the latest program
10. The script pushes the new image to Docker Hub
11. The server that runs the containers, perhaps a [Kubernetes](https://kubernetes.io/) cluster, is told there is a new version
12. The k8s cluster pulls down the latest image
13. The k8s cluster shuts down old containers as it spins up new containers of the latest image

## It's Never the Same

While the deployment process I've outlined above is a common one, especially at newer "cloud native" companies, two companies rarely have identical processes. Instead of GitHub, it might be GitLab. Instead of Docker Hub, it might be ECR. Instead of Kubernetes, it might be Docker Swarm or a more managed service.

That said, I hope this helps give you an idea of what to expect in the wild.
