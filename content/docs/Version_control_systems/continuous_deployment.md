---
title: "continuous_deployment"
weight: 5
---
Continuous Deployment (CD) is the process of automatically deploying code changes to a production environment after the code has been built and tested. Let's set up CD for Notely!

We'll be using GitHub Actions again, but we'll create a new workflow for CD: `cd.yml`

Click to hide video

Your browser does not support playing HTML5 video. You can instead. Here is a description of the content: continuous deployment

## Assignment

1. [ ] Create a new workflow `.github/workflows/cd.yml`
2. [ ] It should trigger on `push` into the `main` branch.

```yaml
on:
  push:
    branches: [main]
```

3. It should have a single job called `Deploy`
4. It should checkout the code
5. It should set up the Go toolchain
6. It should build the app using the `scripts/buildprod.sh` script

To be clear, the `ci` workflow runs when a pull request is _opened_, but the `cd` workflow runs when a pull request is _merged_ (or when code is pushed directly to the `main` branch).

Run the `cd` workflow and ensure it works by merging your PR into `main` (or pushing directly to `main`).

_Paste the URL of your GitHub repo into the box and run the GitHub checks._

```yaml
name: cd

on:
  push:
    branches: [main]

jobs:
  deploy:
    name: Deploy
    runs-on: ubuntu-latest

    steps:
      - name: Check out code
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.26.0"

      - name: Build app
        run: ./scripts/buildprod.sh
```

