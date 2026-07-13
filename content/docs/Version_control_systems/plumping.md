---
title: "plumping"
weight: 25
---
# The Plumbing

So far we've been using Git in a "porcelain" manner. But to sate our insatiable curiosity, let's take a look at some of the "plumbing".

## It's Just Files All the Way Down

All the data in a Git repository is stored directly in the (hidden) `.git` directory. That includes all the commits, branches, tags, and other objects we'll learn about later.

Git is made up of [objects](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects) that are stored in the `.git/objects` directory. A commit is just a type of object.

## Assignment

1. Use `git log -n 10` to find your commit hash again.
2. Use `ls -al` to list the contents of the `.git/objects` directory.
3. Look for a directory that suspiciously matches the first two characters of your commit hash.
4. Use `ls -al` to list the contents of that directory and answer the question.

# Cat File

Luckily, Git has a built-in plumbing command, [cat-file](https://git-scm.com/docs/git-cat-file), that allows us to see the contents of a commit without needing to futz around with the object files directly.

```bash
git cat-file -p <hash>
```

## Assignment

1. [ ] Use the `cat-file` command to view the contents of your last commit.
2. [ ] For the CLI to be able to check your output, do it again, but redirect the output to a temporary file:

```bash
[catfile command here] > /tmp/catfileout.txt
```

### Trees and Blobs

Now that we understand some of our plumbing equipment, let's get into the pipes. Here are some terms to know:

- `tree`: git's way of storing a directory
- `blob`: git's way of storing a file

Here's what I got when I inspected my last commit:

```bash
> git cat-file -p 5ba786fcc93e8092831c01e71444b9baa2228a4f

tree 4e507fdc6d9044ccd8a4a3061324c9f711c4667d
author alperen <the.alperen@aol.com> 1705891256 -0700
committer alperen <the.alperen@aol.com> 1705891256 -0700

A: add contents.md
```

Notice that we can see:

- The `tree` object
- The `author`
- The `committer`
- The commit message

However, we _cannot_ see the contents of the `contents.md` file itself! That's because the `blob` object stores it.

# Storing Data

Git stores an entire _snapshot_ of files on a _per-commit_ level. This was a surprise to me! I always assumed each commit only stored the _changes_ made in that commit.

## Optimization

While it's true that Git stores entire snapshots, it _does_ have some performance optimizations so that your `.git` directory doesn't get too unbearably large.

- Git [compresses and packs](https://git-scm.com/book/en/v2/Git-Internals-Packfiles) files to store them more efficiently.
- Git deduplicates files that are the same across different commits. If a file doesn't change between commits, Git will only store it once.
