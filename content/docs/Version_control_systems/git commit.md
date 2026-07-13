---
title: "git commit"
weight: 12
---
# Commit

After staging a file, we can [commit](https://docs.github.com/en/pull-requests/committing-changes-to-your-project/creating-and-editing-commits/about-commits) it.

A commit is a snapshot of the repository at a given point in time. It's a way to save the state of the repository, and it's how Git keeps track of changes to the project. A commit comes with a message that describes the changes made in the commit.

Here's how to [commit](https://git-scm.com/docs/git-commit) all of your staged files:

```bash
git commit -m "your message here"
```
####  Change the last commit message
```bash
git commit --amend -m "A: add contents.md"
```