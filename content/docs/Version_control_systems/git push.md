---
title: "git push"
weight: 13
---
# Git Push

The `git push` command pushes (sends) local changes to any "remote" - in our case, GitHub. For example, to push our local `main` branch's commits to the remote `origin`'s `main` branch we would run:

```bash
git push origin main
```

_You need to be authenticated with the remote to push changes, which you should have done in the last lesson._

## Alternative Options

You can also push a local branch to a remote with a _different_ name:

```
git push origin <localbranch>:<remotebranch>
```

_It's less common to do this, but nice to know._

You can also _delete_ a remote branch by pushing an empty branch to it:

```
git push origin :<remotebranch>
```