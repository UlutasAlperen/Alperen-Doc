---
title: "git log"
weight: 16
---
# Git Log

A Git repo is a (potentially very long) list of commits, where each commit represents the _full state of the repository_ at a given point in time.

The [git log](https://git-scm.com/docs/git-log) command shows a history of the commits in a repository. This is what makes Git a version control system. You can see:

- Who made a commit
- When the commit was made
- What was changed

## A Commit Hash

Each commit has a unique identifier called a "commit hash". This is a long string of characters that uniquely identifies the commit. Here's an example of mine:

```
5ba786fcc93e8092831c01e71444b9baa2228a4f
```

For convenience, you can refer to any commit or change within Git by using the first `7` characters of its hash. For mine, that's `5ba786f`.

# Log Flags

As you know, `git log` shows you the history of commits in your repo. There are a few flags I like to use from time to time to make the output easier to read.

The first is [--decorate](https://git-scm.com/docs/git-log#Documentation/git-log.txt---decorateshortfullautono). It can be one of:

- `short` (the default)
- `full` (shows the full ref name)
- `no` (no decoration)

Run `git log --decorate=full`. You should see that instead of just using your branch name, it will show the full ref name. A [ref](https://git-scm.com/book/en/v2/Git-Internals-Git-References) is just a pointer to a commit. All branches are refs, but not all refs are branches.

Run `git log --decorate=no`. You should see that the branch names are no longer shown at all.

The second is [--oneline](https://git-scm.com/docs/git-log#Documentation/git-log.txt---oneline). This flag will show you a more compact view of the log. I use this one all the time, it just makes it so much easier to see what's going on.

```bash
git log --oneline
```

# Log Remote

The `git log` command isn't only useful for your _local_ repo. You can log the commits of a _remote_ repo as well!

```bash
git log remote/branch
```

For example, if you wanted to see the commits of a branch named `alperen` from a remote named `origin` you would run:

```bash
git log origin/alperen
```
