---
title: "gitignore"
weight: 24
---
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

# Nested Gitignore

Your `.gitignore` file does _not_ necessarily need to be at the root of your project.

It's fairly common to have multiple `.gitignore` files in different directories throughout a project. A nested `.gitignore` file only applies to the directory it's in and its subdirectories.

Let's say you have the following setup:

```
src/
├── assets/
│   ├── .gitignore
|   ├── cover_art.jpg
│   └── onlydevs.png
├── main.py
├── tests.py
├── venv/
│   └── bin/
|       ├── activate
│       └── python
.gitignore
```

Here's the contents of `src/assets/.gitignore`:

```
onlydevs.png
main.py
```

Here's the contents of the root `.gitignore`:

```
venv/bin/activate
```

# Patterns

It would be _rough_ if `.gitignore` files only accepted exact filepath section names. Luckily, they don't!

Let's go over some of the most common patterns.

## Wildcards

The `*` character matches any number of characters except for a slash (`/`). For example, to ignore _all_ `.txt` files, you could use the following pattern:

```
*.txt
```
## Rooted Patterns

Patterns starting with a `/` are anchored to the directory containing the `.gitignore` file. For example, this would ignore a `main.py` in the root directory, but not in any subdirectories:

```
/main.py
```
## Negation

You can negate a pattern by prefixing it with an exclamation mark (`!`). For example, to ignore all `.txt` files _except_ for `important.txt`, you could use the following pattern:

```
*.txt
!important.txt
```
## Comments

You can add comments to your `.gitignore` file by starting a line with a `#`. For example:

```
# Ignore all .txt files
*.txt
```
## Order Matters

The order of patterns in a `.gitignore` file determines their effect, and patterns can override each other. For example:

```
temp/*
!temp/instructions.md
```

Everything in the `temp/` directory would be ignored _except_ for `instructions.md`. If the order were reversed, `instructions.md` would be ignored.

> You can read more about the patterns that are available in the [official documentation](https://git-scm.com/docs/gitignore#_pattern_format) if you'd like.

# What to Ignore

We've talked about _how_ to ignore files, but the deeper question is _what_ should you ignore? Here are some rules of thumb for coding projects:

1. Ignore things that can be _generated_ (e.g. compiled code, minified files, etc.)
2. Ignore dependencies (e.g. `node_modules`, `venv`, `packages`, etc.)
3. Ignore things that are personal or specific to how you like to work (e.g. editor settings)
4. Ignore things that are sensitive or dangerous (e.g. `.env` files, passwords, API keys, etc.)