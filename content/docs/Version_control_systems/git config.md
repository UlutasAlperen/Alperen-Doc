---
title: "git config"
weight: 23
---
We need to configure Git to contain _your_ information. Whenever code changes, Git tracks _who_ made the change. To ensure you get proper credit (or more likely, blame) for all the code you write, you need to set your name and email.

Git comes with a [configuration](https://git-scm.com/docs/git-config) both at the global and the repo (project) level. Most of the time, you'll just use the global config.

## Assignment

Let's set your identity. Check if your `user.name` and `user.email` are already set:

```bash
git config --get user.name
```

```bash
git config --get user.email
```

If they aren't, set them. I recommend using your GitHub username and email.

```bash
git config --add --global user.name "github_username_here"
git config --add --global user.email "email@example.com"
```

Finally, let's set a default branch (we'll talk more about configs and branches later) so that we're all on the same page. Run:

```bash
git config --global init.defaultBranch master
```

> We're using `master` for now because it is Git's default, but later we'll change it to `main`, which is GitHub's default. Just bear with us for a second.

## Make Sure It Worked

Your `~/.gitconfig` file is the file that stores your global Git configuration. View it:

```bash
cat ~/.gitconfig
```

# Git Config

Git stores author information so that when you're making a commit it can track _who_ made the change. Here's how you might update your global [Git configuration](https://git-scm.com/docs/git-config) (don't do this yet):

```bash
git config --add --global user.name "alperen"
git config --add --global user.email "alperen@cartcurt.com"
```

Let's take the command apart:

- `git config`: The command to interact with your Git configuration.
- `--add`: Flag stating you want to _add_ a configuration.
- `--global`: Flag stating you want this configuration to be stored globally in your `~/.gitconfig`. The opposite is "local", which stores the configuration in the current repository only.
- `user`: The section.
- `name`: The key within the section.
- `"ThePrimeagen"`: The value you want to set for the key.
