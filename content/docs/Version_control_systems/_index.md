---
itle: "Version Control Systems"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
bookCollapseSection: true
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---
[Git](https://git-scm.com/) is _the_ distributed [version control system](https://git-scm.com/book/en/v2/Getting-Started-About-Version-Control) (VCS). Nearly every developer in the world uses it to manage their code. It has quite a monopoly on VCS. Developers use Git to:

- Keep a history of their code changes
- Revert mistakes made in their code
- Collaborate with other developers
- Make backups of their code
- And much more
## Git konular

1. [basic_git_knowledge](basic_git_knowledge/)
2. [github_workflow](github_workflow/)
3. [basic_git_conf](basic_git_conf/)
### Porcelain and Plumbing


In Git, commands are divided into high-level ("porcelain") commands and low-level ("plumbing") commands. The porcelain commands are the ones that you will use most often as a developer to interact with your code. Some porcelain commands are:

- [git status](git-status/) = The `git status` command shows you the current state of your repo
- [git add](git-add/)  =  add file to git status
- [git commit](git-commit/) = A commit is a snapshot of the repository at a given point in time. It's a way to save the state of the repository
- [git push](git-push/) =The `git push` command pushes (sends) local changes to any "remote" 
- [git branch](git-branch/)  =A [Git branch](https://git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell) allows you to keep track of different changes separately.
- [git pull](git-pull/) = fetching the all data from remote repo
- [git log](git-log/) = showing logs
- `git init` = create a new Git repository
- [git merge](git-merge/) = merge `example_branch` into `main_branch` by running this while on `main_branch`
- [git rebase](git-rebase/) = turn branch to a main branch
- [git reset](git-reset/) = resetting to a committed hash
- [git remote](git-remote/) =git remote add  name (origin most of time)  uri (yanked repo)
- [git fetch](git-fetch/) =This downloads copies of all the contents of the .git/objects 
- [git switch](git-switch/) = switching branch , using -c flag crate ,if use committed hash back to old repo

[plumping](plumping/) commands are:

- `git apply`
- `git commit-tree`
- `git hash-object`

**[git config](git-config/)**

# Locations

There are several locations where Git can be configured. From more general to more specific, they are:

- **system**: `/etc/gitconfig`, a file that configures Git for all users on the system
- **global**: `~/.gitconfig`, a file that configures Git for all projects of a user
- **local**: `.git/config`, a file that configures Git for a specific project
- **worktree**: `.git/config.worktree`, a file that configures Git for part of a project

In my experience, 90% of the time you will be using `--global` to set things like your username and email. The other 9% of the time you will be using `--local` to set project-specific configurations. The last 1% of the time you _might_ need to futz with system and worktree configurations, but it's extremely rare.

# Overriding

If you set a configuration in a more specific location, it will override the same configuration in a more general location. For example, if you set `user.name` in the local configuration, it will override the `user.name` set in the global configuration.

![overriding](https://storage.googleapis.com/qvault-webapp-dynamic-assets/course_assets/e4S7M9u.png)

# What Is a Branch?

A [Git branch](https://git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell) allows you to keep track of different changes separately.

For example, let's say you have a big web project and you want to experiment with changing the color scheme. Instead of changing the entire project directly (as of right now, our `master` branch), you can create a new branch called `color_scheme` and work on that branch. When you're done, if you like the changes, you can [merge](https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging) the `color_scheme` branch back into the `master` branch to keep the changes. If you don't like the changes, you can simply delete the `color_scheme` branch and go back to the `master` branch.

## Under the Hood

A branch is just a named [pointer](https://en.wikipedia.org/wiki/Pointer_(computer_programming)#:~:text=A%20pointer%20is%20a%20programming,than%20storing%20the%20data%20itself.) to a specific commit. When you create a branch, you are creating a new pointer to a specific commit. The commit that the branch points to is called the tip of the branch.

![commits and branches](https://storage.googleapis.com/qvault-webapp-dynamic-assets/course_assets/iH1kl8l.png)

Because a branch is just a pointer to a commit, they're lightweight and "cheap" resource-wise to create. When you create 10 branches, you're not creating 10 copies of your project on your hard drive.

# Merge

"What's the point of having multiple branches?" you might ask. They're most often used to safely make changes without affecting your (or your _team's_) primary branch. However, once you're happy with your changes, you'll want to [git merge](git-merge/) them back into the main branch so that they make their way into the final product.

# Git Remote

Often our frenemies (read: coworkers) make code changes that we need to begrudgingly accept into our pristine bug-free repos. _/s_

This is where the "distributed" in "distributed version control system" comes from. We can have "remotes", which are just external repos with _mostly_ the same Git history as our local repo.

When it comes to Git (the CLI tool), there really isn't a "central" repo. GitHub is just someone else's repo. Only by convention and convenience have we, as developers, started to use GitHub as a "source of truth" for our code. [git remote](git-remote/)
# Pull Requests

On GitHub, a [pull request](https://docs.github.com/en/github/collaborating-with-issues-and-pull-requests/about-pull-requests) is a way to propose changes, typically to the rest of your team, or to the maintainer of a project you're contributing to.

Pull requests allow team members to see what changes are being proposed and to discuss them before they are merged into the main codebase.


# My Team Workflow

When you're working with a _team_, Git gets a bit more involved (and we'll cover more of this in part 2 of this course). Here's what I do:

- Update my local `main` branch with `git pull origin main`
- Checkout a new branch for the changes I want to make with `git switch -c <branchname>`
- Make changes to files
- `git add .`
- `git commit -m "a message describing the changes"`
- `git push origin <branchname>` (I push to the _new_ branch name, not `main`)
- Open a [pull request](https://docs.github.com/en/github/collaborating-with-issues-and-pull-requests/about-pull-requests) on GitHub to merge my changes into `main`
- Ask a team member to review my pull request
- Once approved, click the "Merge" button on GitHub to merge my changes into `main`
- Delete my feature branch, and repeat with a new branch for the next set of changes

# Merge Pull Request

In a typical team workflow, you would ask a team mate to review your pull request. If they approve of the changes, they would approve the pull request, and you'd be clear to merge.

We're the dictators of our own repo however, so no need to wait for approval!

# Gitignore

As you've seen, it's _pretty normal_ to use the following workflow from the top level of your repo:

1. `git add .`
2. `git commit -m "some message here"`
3. `git push origin main`

A problem arises when we want to put files in our project's directory, but we _don't_ want to track them with Git. **[gitignore](gitignore/) file solves this.** For example, if you work with Python, you probably want to ignore automatically generated files like `.pyc` and `__pycache__`. If you are building a server, you probably want to ignore `.env` files that might hold private keys. If you (I'm sorry) work with JavaScript, you might want to ignore the `node_modules` directory.

![node_modules](https://storage.googleapis.com/qvault-webapp-dynamic-assets/course_assets/Mn7FERh.png)

Here's an example `.gitignore` file, which exists at the root of a repo:

```
node_modules
```

This will ignore every path containing `node_modules` as a "section" (directory name or file name). It ignores:

- `node_modules/code.js`
- `src/node_modules/code.js`
- `src/node_modules`

It does _not_ ignore:

- `src/node_modules_2/code.js`
- `env/node_modules_3`
