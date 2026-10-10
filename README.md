# About This Repo

This repository contains the source of my personal documentation site, built with [Hugo](https://gohugo.io/) and the [hugo-book](https://github.com/alex-shpak/hugo-book) theme (included as a git submodule).

```
.
├── content/          # Documentation content (markdown)
│   └── docs/         # AWS, Docker, Kubernetes, Git, and more
├── themes/           # hugo-book theme (git submodule)
├── layouts/          # Custom layout overrides
├── static/           # Static assets
├── assets/           # Asset pipeline files
├── hugo.toml         # Site configuration
└── .github/
    └── workflows/
        └── deploy.yml  # CI/CD deployment pipeline
```
## Documentation

- [Version Control Systems](https://dev.ulutasalperen.com/docs/version_control_systems) (Git, GitHub workflows, CI/CD concepts)
- [Docker](https://dev.ulutasalperen.com/docs/docker)
- [Kubernetes](https://dev.ulutasalperen.com/docs/kubernetes) (pods, deployments, services, storage, RBAC, scaling, and more)
- [Kubernetes v2](https://dev.ulutasalperen.com/docs/kubernetes_v2)
- [AWS](https://dev.ulutasalperen.com/docs/aws) (EC2, S3, IAM, ECS, EKS, Lambda, RDS, networking, CDN, DNS, monitoring)

## Language Note

Documentation on this site is written in either **English** or **Turkish**, depending on the topic and personal preference. Bu sitedeki dokumanlar, konuya ve kisisel tercihe gore **Ingilizce** veya **Turkce** olarak hazirlanmistir.


## Getting Started (Local)

Prerequisites: [Hugo (extended)](https://gohugo.io/installation/) and Git.

```bash
# Clone with theme submodule
git clone --recursive https://github.com/UlutasAlperen/Alperen-Doc.git
cd Alperen-Doc

# If you already cloned without --recursive
git submodule update --init --recursive

# Start local dev server
hugo server -D

# Build production site
hugo --minify
```

The site will be available at `http://localhost:1313/` during development.

## Deployment

The site is deployed automatically via GitHub Actions (`.github/workflows/deploy.yml`):

1. Push to the `main` branch
2. GitHub Actions checks out the code (including submodules)
3. Hugo (extended) builds and minifies the site
4. The generated `public/` folder is synced to the production VPS over SSH/rsync

Live site: [blog.ulutasalperen.com](https://blog.ulutasalperen.com/)

## Resources I Commonly Use

- [boot.dev](https://boot.dev)
- [kubernetes.io](https://kubernetes.io/docs/home/)
- [docker.com](https://docs.docker.com/)
- [iximiuz.com](https://labs.iximiuz.com/docs)
- [skillbuilder.aws](https://skillbuilder.aws)
