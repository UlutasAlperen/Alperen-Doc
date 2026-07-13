---
title: "git switch"
weight: 22
---
# New Branch

You should already be on the `main` branch: your "default" branch. You can always check with `git branch`.

## Two Ways to Create a Branch

```bash
git branch my_new_branch
```

This creates a new branch called `my_new_branch`. The thing is, I rarely use this command because usually I want to create a branch and switch to it immediately. So I use this command instead:

```bash
git switch -c my_new_branch
```

The [switch](https://git-scm.com/docs/git-switch) command allows you to switch branches, and the `-c` flag tells Git to create a new branch if it doesn't already exist.

When you create a new branch, it uses the _current commit_ you are on as the branch base. For example, if you're on your `main` branch with 3 commits, `A`, `B`, and `C`, and then you run `git switch -c my_new_branch`, your new branch will look like this:

![branch diagram same commits](https://storage.googleapis.com/qvault-webapp-dynamic-assets/course_assets/oah2FRD.png)# New Branch

We need a new feature branch, but to be able to practice a [rebase](https://git-scm.com/docs/git-rebase), we want it to _not_ include some of the recent commits on `main`.

## Assignment

1. [ ] Use the [git switch](https://git-scm.com/docs/git-switch) command to create and switch to a new branch called `update_dune`, but branch off of the `D` commit. You can supply the commit hash directly to the `git switch` command:

```bash
git switch -c update_dune COMMITHASH
```