---
title: "git remote"
weight: 20
---
# Git Remote

Often our frenemies (read: coworkers) make code changes that we need to begrudgingly accept into our pristine bug-free repos. _/s_

This is where the "distributed" in "distributed version control system" comes from. We can have "remotes", which are just external repos with _mostly_ the same Git history as our local repo.

When it comes to Git (the CLI tool), there really isn't a "central" repo. GitHub is just someone else's repo. Only by convention and convenience have we, as developers, started to use GitHub as a "source of truth" for our code.


# Adding a Remote

In git, another repo is called a "remote." The standard convention is that when you're treating the remote as the "authoritative source of truth" (such as GitHub) you would name it the "origin".

By "authoritative source of truth" we mean that it's the one you and your team treat as the "true" repo. It's the one that contains the most up-to-date version of the accepted code.

# Command Syntax

```bash
git remote add <name>(origin most of time) <uri>(yanked repo)
```
 
 Inside our new repo, add `webflyx` as a remote. Give it the name `origin`. The `uri` should just be a relative path to the `webflyx` directory, in our case, `../webflyx`.


# Log Remote

The `git log` command isn't only useful for your _local_ repo. You can log the commits of a _remote_ repo as well!

```bash
git log remote/branch
```

For example, if you wanted to see the commits of a branch named `primeagen` from a remote named `origin` you would run:

```bash
git log origin/primeagen
```

# Merge

Just as we merged branches within a single local repo, we can also merge branches between local and remote repos.

## Syntax

```bash
git merge remote/branch
```

For example, if you wanted to merge the `primeagen` branch of the remote `origin` into your local `main` branch, you would run this inside the local repo while on the `main` branch:

```bash
git merge origin/alperen
```

# GitHub Repository

Just like we created a `webflyx-local` repo and used `webflyx` as a remote, GitHub makes it easy to create "remote" repos that are hosted on their servers.

```bash
git remote add origin https://github.com/your-username/webflyx.git
```
