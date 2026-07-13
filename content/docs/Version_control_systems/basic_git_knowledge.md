---
title: "basic_git_knowledge"
weight: 2
---
# Simplified Git Commands Guide

## Initialize a Git Repository

1. Navigate to the directory you want to track.
2. Run the following command:
  ```bash
  git init
  ```
## Adding Files or Directories

- To add files or directories to the staging area, use:

```bash
git add <file_or_directory_name>
```

[git add](git-add/)
## Checking the Current State

- To see the current state of your repository:

```bash
git status
```

[git status](git-status/)

## Committing Changes

- To save changes to the repository:
```bash
git commit -m "Your commit message"
```

[git commit](git-commit/)
## Branch Management

### Creating and Switching to a New Branch

- Use the following command to create and switch to a new branch:

```bash
git switch -c my_new_branch
```
_This is easier than using `git branch` followed by `git switch`._ [git branch](git-branch/)
### Switching Between Branches

- To switch to an existing branch:

```bash
git switch branch_name
```

### Viewing Existing Branches

- To list all branches:

```bash
git branch
```

## Viewing Commit Logs

- To see commit history:

```bash
git log
```

[git log](git-log/)

- Optionally, you can specify a branch name:
```bash
git log branch_name
```

- For a compact, graphical log, create an alias:

```bash
git config --global alias.gitmap "log --oneline --graph --all --decorate --parents"
```

Then use:

```bash
git gitmap
```

## Fetching Updates from Remote Repositories

- To download objects and refs from a remote repository without merging them into your working directory:

```bash
git fetch
```

[git fetch](git-fetch/)
## Working with Remote Repositories

- To add a remote repository:

```bash
git remote add origin <repository_url>
```


- To list all remote repositories:

```bash
git remote -v
```

- To remove a remote repository:

```bash
git remote remove <name>
```


```bash
git remote add origin https://github.com/your-username/repo_name
```
[git remote](git-remote/)
## Merging Branches

- To merge a branch into the current branch:

```bash
git merge example_branch
```

Example: To merge `example_branch` into `main`,***ensure you are on `main` and run the above command.***

## Creating a New Branch from an Older Commit

- To create a new branch from a specific commit:

```bash
git switch -c new_branch_name old_commit_hash
```

## Rebasing Branches

- To reapply commits from `main` onto the current branch:

```bash
git rebase main
```

***Example: If you're on branch `example_brach`, this command brings changes from `main` onto `example_brach`.***
[git rebase](git-rebase/)
## Resetting Commits

- To undo commits:
- Use `--soft` to keep changes in the staging area:

```bash
git reset --soft commit
```

- Use `--hard` to discard changes:

```bash
git reset --hard commit
```
[git reset](git-reset/)

# Git Push

The `git push` command pushes (sends) local changes to any "remote" - in our case, GitHub. For example, to push our local `main` branch's commits to the remote `origin`'s `main` branch we would run:

```bash
git push origin <branch_name>
```

[git push](git-push/)

# Git pull

- To pull changes from a remote repository and update your local repository:
 
```bash
git pull origin <branch_name>
```
  
 [git pull](git-pull/)


### What happens when you `git clone`?

1. It creates a directory for the repository.
2. It initializes the repository and fetches its content.
3. It automatically sets up the **`origin`** remote to point to the repository URL you cloned.

Use the `git clone` command followed by the repository URL:

```bash
git clone <repository-url> <directory-name>
```

Example:

```bash
git clone https://github.com/user/repository.git
```

Clone a Specific Branch:

```bash
git clone --branch <branch-name> <repository-url>
```