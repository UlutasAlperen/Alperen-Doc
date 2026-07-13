---
title: "git fetch"
weight: 21
---
# Fetch

Adding a remote to our Git repo does _not_ mean that we automagically have all the contents of the remote. First, we need to [fetch](https://git-scm.com/docs/git-fetch) the contents (but not yet!).

## Command

```bash
git fetch
```

This downloads copies of all the contents of the `.git/objects` directory (and other book-keeping information) from the remote repository into your current one.

