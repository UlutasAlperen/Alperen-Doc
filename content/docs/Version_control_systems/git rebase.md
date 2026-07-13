---
title: "git rebase"
weight: 18
---
# Rebase

"[Rebase](https://git-scm.com/docs/git-rebase) vs [Merge](https://git-scm.com/docs/git-merge)" is one of the most hotly debated topics in the Git world. A lot of the discussions you'll see online come down to the fact that many developers (yes, even professionals) don't understand the purpose of rebase and use it incorrectly, causing a bunch of Git havoc, and then blame the rebase command.

_It's not Git's fault, it's a skill issue._

## Visualizing Rebase

Say we have this commit history:

```
A - B - C    main
   \
    D - E    feature_branch
```

We're working on `feature_branch`, and want to bring in the changes our team added to `main` so we're not working with a stale branch. We could merge `main` into `feature_branch`, but that would create an additional merge commit. Rebase avoids a merge commit by replaying the commits from `feature_branch` on top of `main`. After a rebase, the history will look like this:

```
A - B - C         main
         \
          D - E   feature_branch
```

# Run Rebase

To use [rebase](https://git-scm.com/docs/git-rebase) to bring changes from `main` onto a current branch (let's pretend we're on one called `jdsl`), we would run this while on the `jdsl` branch:

```bash
git rebase main
```

This will do the following:

1. Checkout the latest commit from `main` into a temporary location
2. Replay each commit from `jdsl` one at a time onto this temporary location
3. Update the `jdsl` branch to point to the last replayed commit in the temporary location, making this the new permanent `jdsl`.
4. The rebase does _not_ affect the `main` branch; `jdsl` now includes all changes from `main`.

![rebase update_dune branch onto the main branch](https://storage.googleapis.com/qvault-webapp-dynamic-assets/course_assets/tCgqhtr.png)
