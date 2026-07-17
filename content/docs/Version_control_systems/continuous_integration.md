---
title: "continuous_integration"
weight: 4
---
# Continuous Integration

Continuous Integration (CI) is where developers regularly push code changes into a central repository, and by doing so, automated builds and tests are automatically run.

Those tests can include unit tests, integration tests, styling checks, linting checks, security checks or any other type of automated test. If _any_ of the tests fail, the build is considered "broken" and the developer is notified so they can fix it.


## I was forked boot.dev cli for testing CI at Boot.dev Cli 

Here at Boot.dev  CI tests that run each time a new pull request is opened. The reviewer doesn't need to manually check for formatting issues or run tests locally before approving the changes. It automates _part_ of the code review process.

_CI is all about automating as much of the testing and review process as possible._

## Assignment

1.  While still on the `addtests` branch, create a new `.github` directory in the root of your repository
2.  Create a new directory inside `.github` called `workflows`
3.  Create a new workflow by adding a file called `ci.yml` inside `.github/workflows`:

>Make sure you use the `.yml` extension. `.yaml` is valid, but we test for `.yml`.

>GitHub Actions workflows are written in [YAML](https://en.wikipedia.org/wiki/YAML), and GitHub automatically checks for and runs workflows in the `.github/workflows` directory of your repository.

4. [ ] Open `ci.yml` in your editor and add the following:

```yaml
name: ci

on:
  pull_request:
    branches: [main]

jobs:
  tests:
    name: Tests
    runs-on: ubuntu-latest

    steps:
      - name: Check out code
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.26.0"

      - name: Force Failure
        run: (exit 1)
```

[By default](https://docs.github.com/en/actions/creating-actions/setting-exit-codes-for-actions), a step "succeeds" if it exits with a status code of `0` and "fails" if it exits with a status code other than `0`.

Every (good) CLI tool that I'm aware of follows the convention of exit code `0` = pass, anything else = fail. For example, if a test case fails, `go test` will exit with a status code of `1`.

_Don't worry, I'll explain each line of the workflow file in detail soon._

Let's make sure that our CI tests fail when our tests fail. The last step of our workflow file is:

```yaml
- name: Force Failure
  run: (exit 1)
```

This step always fails because it runs the command `exit 1`, which exits with a status code of `1`.

5.  Commit and push your changes up to your remote branch. Then, on your pull requests page, you should see that the "tests" workflow runs and fails. Great! That means that our (very basic) CI is working.

_Paste the URL of your GitHub repo into the box and run the GitHub checks._


# Breakdown of GitHub Actions

Let's break down some of the concepts from the workflow file we created.

```yaml
name: ci

on:
  pull_request:
    branches: [main]

jobs:
  tests:
    name: Tests
    runs-on: ubuntu-latest

    steps:
      - name: Check out code
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.26.0"

      - name: Force Failure
        run: (exit 1)
```

## Workflows

A [workflow](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions#workflows) is triggered when an [event](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions#events) occurs in your GitHub repository. For example, we'll trigger our "tests" workflow when we open a pull request into the `main` branch.

In our case, the `ci.yml` file contains a single workflow called "ci", but we could have named it anything.

## Jobs

A workflow is made up of one or more [jobs](https://docs.github.com/en/actions/about-github-actions/understanding-github-actions#jobs). A job is a set of steps that run on the same [runner](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions#runners), which is just a virtual machine that runs your job on GitHub's servers.

For now, we just have one job. You would typically have multiple jobs if you wanted to run your tests in parallel, or if you wanted to run your tests on multiple operating systems.

In our case, the `ci.yml` workflow contains a single job called "Tests".

## Steps

A job is made up of one or more steps. A step is a single task that can run commands, a script, or an action. For example, the steps of a job might include:

- Checking out the code
- Installing dependencies
- Running tests

In our case, the "Tests" job contains 3 steps:

1. Check out the code
2. Set up Go
3. Force failure of the CI job

## Assignment

Update the last step of the job to run `go version` instead of `(exit 1)`. That command should just print the current version of Go and exit with code `0`. Give the step a new name. Commit and push the changes, and your CI job should pass!

A workflow that triggers on pull requests will re-run when the branch to be merged is updated.

**Paste the URL of your GitHub repo into the box and run the GitHub checks**.


# GitHub Actions

Let's dissect this workflow file and understand what each line is doing.

```yaml
name: ci

on:
  pull_request:
    branches: [main]

jobs:
  tests:
    name: Tests
    runs-on: ubuntu-latest

    steps:
      - name: Check out code
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.26.0"

      - name: Echo Go version
        run: go version
```


## Workflow Name

```yaml
name: ci
```

The first line simply assigns a human-readable name to the workflow.

## Triggering the Workflow

```yaml
on:
  pull_request:
    branches: [main]
```

The `on` key specifies when the workflow should run. In our case, we want to run the workflow when a pull request is opened to the `main` branch.

## Jobs

```yaml
jobs:
  tests:
    name: Tests
    runs-on: ubuntu-latest
```

## Job Steps

```yaml
- name: Check out code
uses: actions/checkout@v4
```

Each job has a list of steps that make up the job. In our case, we have three steps. The first step checks out the code by using the pre-built [checkout](https://github.com/actions/checkout) action to clone the repository into the runner. You'll almost always want to include this step in your workflows. The `uses` key specifies the action to use, and the `with` key specifies the inputs to the action.

An [action](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions#actions) is a reusable custom application that helps reduce the complexity of creating workflows. The [checkout](https://github.com/actions/checkout) action is a publicly available action that checks-out your repository so your workflow can access it.

```yaml
- name: Set up Go
uses: actions/setup-go@v5
with:
go-version: '1.26.0'
```

The second step sets up Go by using the pre-built [setup-go](https://github.com/actions/setup-go) action to configure a Go environment for use in workflows.

```yaml
- name: Echo Go version
  run: go version
```

The third step runs the `go version` command to print the version of Go installed in the runner. The `run` key specifies the command to run. The `run` key is used to run arbitrary command-line commands in the runner.

# Running Tests

Our current CI doesn't do anything interesting.

A good CI pipeline typically includes:

- Unit tests
- Integration tests
- Styling checks
- Linting checks
- Security checks
- Any other kind of automated test

If _any_ of the tests fail, the build is considered "broken" and the developer is notified (in our case by GitHub) so they can fix it.

## Assignment

The Notely repo has, _gasp_, zero unit tests!

1. [ ] Create `internal/auth/get_api_key_test.go`.
2. [ ] Add a couple unit tests for `GetAPIKey`.
    
    If you're still a little fuzzy on how to write unit tests in Go, I'd highly recommend Dave Cheney's [excellent blog post](https://dave.cheney.net/2019/05/07/prefer-table-driven-tests) on the subject.
    
3. [ ] Run them locally to make sure they work:
    
    ```bash
    go test ./...
    ```
    
    The `./...` tells Go to run all tests in the current directory and all subdirectories.

**Run and submit** the CLI tests from the **root of your repo**.

# Formatting CI

Now that we understand how to check for formatting issues, let's add a formatting check to the CI workflow.

## Check for Formatting Issues

```bash
test -z $(go fmt ./...)
```


# CI Linting

To get linting on our CI runner, we can't just call `staticcheck`: it's not installed on the runner!

Before any steps that _use_ the `staticcheck` command, you'll need to install it:

```yaml
- name: Install staticcheck
  run: go install honnef.co/go/tools/cmd/staticcheck@latest
```



# Security Checks

Another common use case for continuous integration is static security checks. These are checks that can be run on your code to find potential security vulnerabilities.

There are _many_ products out there that do this sort of thing. One of the most popular open-source ones is [Gosec](https://github.com/securego/gosec).

## Install Gosec

```bash
go install github.com/securego/gosec/v2/cmd/gosec@latest
```


# CI Security

Like `staticcheck`, `gosec` is _not_ part of the Go toolchain. We'll need to install it in our remote runner before we can use it.

## Assignment

Security isn't a _style_ concern, so let's add these next steps after `go test` in the "Tests" job.

Add the install step.

```yaml
- name: Install gosec
  run: go install github.com/securego/gosec/v2/cmd/gosec@latest
```

Genel son CI

```yml
name: ci

on:
  pull_request:
    branches: [main]

jobs:
  tests:
    name: Tests
    runs-on: ubuntu-latest

    steps:
      - name: Check out code
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.25.1"

      - name: Run unit tests
        run: go test ./... -cover

      - name: Install gosec
        run: go install github.com/securego/gosec/v2/cmd/gosec@latest

      - name: Run gosec
        run: gosec ./...

  style:
    name: Style
    runs-on: ubuntu-latest

    steps:
      - name: Check out code
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.25.1"

      - name: Check formatting
        run: test -z $(go fmt ./...)

      - name: Install staticcheck
        run: go install honnef.co/go/tools/cmd/staticcheck@latest

      - name: Run staticcheck
        run: staticcheck ./...

```
