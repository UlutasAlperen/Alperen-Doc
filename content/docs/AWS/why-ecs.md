---
title: "why-ecs"
weight: 530
---

# Why ECS?

Containers are a big part of how software is typically built and deployed today.

If you've never used them before, you should take our [course on Docker](https://www.boot.dev/courses/learn-docker) before proceeding! You'll also need `docker` installed locally to complete this chapter.

If you're running just a container or two on a single machine, `docker` is a great way to get up running, but When your needs grow beyond a single machine, [Kubernetes](https://kubernetes.io/), a powerful container orchestration platform, tends to be the go-to solution.

However, Kubernetes requires quite a bit of knowledge and setup to use effectively. A lot of the details and concepts require a deep understanding of how it operates to use well ([check this out for a good primer](https://github.com/kelseyhightower/kubernetes-the-hard-way)). Amazon will happily help host and manage Kubernetes environments with their Elastic Kubernetes Service (EKS), but AWS (and many other cloud platforms) have seen the need for a simpler model to run containers that trades flexibility and control for simplicity: enter the [**Elastic Container Service** (ECS)](https://docs.aws.amazon.com/ecs/).

[ECS](https://aws.amazon.com/pm/ecs/) is a fully managed container orchestration service that allows you to run and scale containerized applications without having to manage the underlying infrastructure. It abstracts away much of the complexity of Kubernetes while still providing powerful features for running containers in production.

I've reached for ECS quite a few times when working with smaller development teams. It, combined with a healthy CI/CD pipeline, is enough for many projects.