---
title: "package-management"
weight: 200
---
**Scope:** Debian-based and RHEL-based Linux distributions  
**Topics:** Package installation, removal, upgrades, repository management, version locking, and low-level package tools.

# Part 1: Debian-Based Systems

**Examples:** Debian, Ubuntu

## 1. Refresh the Package Index

Run as root before installing or upgrading:

```bash
apt update
```

`apt update` downloads repository metadata only. It does not install or upgrade packages.

## 2. Install Packages

```bash
apt install <PACKAGE_NAME>
```

**Options:**
- `-y`: Confirms all prompts automatically. Use only in scripts and after reviewing the transaction.
- `--no-install-recommends`: Installs hard dependencies only, skipping recommended packages.
- `--dry-run`: Simulates the transaction without changing the system.

## 3. Remove Packages

Remove package binaries but keep configuration files:

```bash
apt remove <PACKAGE_NAME>
```

Remove binaries and configuration files:

```bash
apt purge <PACKAGE_NAME>
```

**Options:**
- `remove`: Uninstalls the package and preserves its configuration files.
- `purge`: Uninstalls the package and deletes its configuration files.

## 4. Upgrade Packages

Upgrade all installed packages without removing any:

```bash
apt upgrade
```

Upgrade packages and allow dependency resolution to remove others when required:

```bash
apt full-upgrade
```

**Options:**
- `upgrade`: Keeps every installed package. Held-back packages stay at their current version.
- `full-upgrade`: Resolves dependencies aggressively and may remove packages.

Review the transaction before confirming. Held-back packages usually indicate that `apt upgrade` is insufficient for that release transition.

## 5. Clean Up

Remove automatically installed packages that are no longer required:

```bash
apt autoremove
```

Remove outdated package files from the local cache:

```bash
apt autoclean
```

**Options:**
- `autoremove`: Deletes orphaned dependencies.
- `autoclean`: Deletes cached package files that are no longer downloadable.
- `clean`: Deletes the entire package cache. Reinstalls then require a new download.

## 6. Search and Inspect Packages

Search the repository index:

```bash
apt search <KEYWORD>
```

Show package details:

```bash
apt show <PACKAGE_NAME>
```

List installed packages with `apt list --installed`.

## 7. Manage Repositories

Repository entries live in `/etc/apt/sources.list.d/`. Add a custom repository as a `.list` file with a `deb` line such as `deb https://deb.example.org/debian bookworm main`:

```bash
vim /etc/apt/sources.list.d/<REPO_NAME>.list
```

Debian 12 and Ubuntu 24.04 use the deb822 format in `.sources` files in the same directory:

```text
Types: deb
URIs: https://deb.example.org/debian
Suites: bookworm
Components: main
Signed-By: /etc/apt/keyrings/<REPO_NAME>.gpg
```

Apply the changes:

```bash
apt update
```

## 8. Hold Package Versions

Prevent a package from being upgraded:

```bash
apt-mark hold <PACKAGE_NAME>
```

List and remove holds:

```bash
apt-mark showhold
apt-mark unhold <PACKAGE_NAME>
```

Use holds to protect critical packages during routine upgrades.

---

# Part 2: RHEL-Based Systems

**Examples:** RHEL, Rocky Linux, AlmaLinux

## 1. Install Packages

Run as root:

```bash
dnf install <PACKAGE_NAME>
```

**Options:**
- `-y`: Confirms all prompts automatically.
- `--assumeno`: Simulates the transaction without changing the system. Check the reported transaction summary.

`dnf` refreshes repository metadata automatically when it is outdated. No separate update step is required.

## 2. Remove Packages

```bash
dnf remove <PACKAGE_NAME>
```

Review the transaction summary: `dnf remove` also removes dependent packages that would otherwise be broken.

## 3. Upgrade Packages

Upgrade all installed packages:

```bash
dnf upgrade
```

Upgrade a single package:

```bash
dnf upgrade <PACKAGE_NAME>
```

**Options:**
- `upgrade`: Updates packages to the latest available version and resolves dependencies.
- `--security`: Limits the transaction to security updates. Requires the `dnf-plugin-security` plugin.
- `update`: Alias of `upgrade` on dnf4. Prefer `upgrade` in scripts.

## 4. Search and Inspect Packages

Search the repository metadata:

```bash
dnf search <KEYWORD>
```

Show package details:

```bash
dnf info <PACKAGE_NAME>
```

List installed packages with `dnf list --installed`.

## 5. List Repositories

```bash
dnf repolist
```

Include disabled repositories:

```bash
dnf repolist --all
```

**Options:**
- `--all`: Lists enabled and disabled repositories.

Repository definitions live in `/etc/yum.repos.d/*.repo`. Disable a repository for a single transaction with `--disablerepo=<REPO_ID>`.

## 6. Clean the Cache

Remove cached metadata and packages:

```bash
dnf clean all
```

Rebuild the metadata cache:

```bash
dnf makecache
```

## 7. Lock Package Versions

Install the versionlock plugin:

```bash
dnf install python3-dnf-plugin-versionlock
```

Lock a package to its installed version:

```bash
dnf versionlock add <PACKAGE_NAME>
```

List and remove locks:

```bash
dnf versionlock list
dnf versionlock delete <PACKAGE_NAME>
```

Use version locks to protect critical packages during routine upgrades.

## 8. Enable EPEL

EPEL provides additional packages that are not in the base repositories. On Rocky Linux and AlmaLinux:

```bash
dnf install epel-release
```

On RHEL 9, enable the CodeReady Builder repository first, then install the EPEL release package published by the Fedora project:

```bash
subscription-manager repos --enable codeready-builder-for-rhel-9-$(arch)-rpms
dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
```

Verify the repository with `dnf repolist`.

---

# Part 3: Important Notes

## .deb vs. .rpm

| Property | .deb (Debian-based) | .rpm (RHEL-based) |
|---|---|---|
| Package format | Debian binary package | RPM package |
| Low-level tool | `dpkg` | `rpm` |
| High-level tool | `apt` | `dnf` |
| Metadata refresh | `apt update` | Automatic in `dnf` |
| File name pattern | `<PACKAGE_NAME>_<VERSION>_<ARCH>.deb` | `<PACKAGE_NAME>-<VERSION>.<ARCH>.rpm` |
| Repository files | `/etc/apt/sources.list.d/` | `/etc/yum.repos.d/` |
| Version locking | `apt-mark hold` | `dnf versionlock` |

## Low-Level Package Tools

Query the local package database on Debian-based systems:

```bash
dpkg -l
```

Expected output:

```text
ii  curl 7.88.1-10+deb12u12 amd64 command line tool for transferring data
```

Query the local package database on RHEL-based systems:

```bash
rpm -qa
```

**Options:**
- `dpkg -l`: Lists installed packages. The first column shows the package state, for example `ii` for installed.
- `dpkg -s <PACKAGE_NAME>`: Shows the status of one package.
- `rpm -qa`: Lists all installed packages.
- `rpm -qf <FILE_PATH>`: Shows which package owns a file.

`dpkg` and `rpm` do not resolve dependencies from repositories. Use `apt` and `dnf` for normal operations.

## Command Reference

| Operation | Debian-based | RHEL-based |
|---|---|---|
| Refresh metadata | `apt update` | `dnf makecache` |
| Install | `apt install <PACKAGE_NAME>` | `dnf install <PACKAGE_NAME>` |
| Remove | `apt remove <PACKAGE_NAME>` | `dnf remove <PACKAGE_NAME>` |
| Remove with config | `apt purge <PACKAGE_NAME>` | `dnf remove <PACKAGE_NAME>` |
| Upgrade all | `apt upgrade` | `dnf upgrade` |
| Search | `apt search <KEYWORD>` | `dnf search <KEYWORD>` |
| Package info | `apt show <PACKAGE_NAME>` | `dnf info <PACKAGE_NAME>` |
| List installed | `dpkg -l` | `rpm -qa` |
| Clean cache | `apt autoclean` | `dnf clean all` |
| Lock version | `apt-mark hold <PACKAGE_NAME>` | `dnf versionlock add <PACKAGE_NAME>` |
| List repositories | `apt-cache policy` | `dnf repolist` |

## Security Guidelines

- Replace placeholders such as `<PACKAGE_NAME>` and `<REPO_NAME>` with real names. Do not include the angle brackets in commands.
- Use only distribution repositories or repositories from trusted vendors. Every repository can install arbitrary code as root.
- Verify package signatures. Debian-based systems use `Signed-By=` with a key in `/etc/apt/keyrings/`; RHEL-based systems use `gpgcheck=1` and imported keys in the `.repo` file.
- Never disable GPG verification to make an installation succeed. Fix or replace the repository instead.
- Review the transaction summary before confirming. Do not use `-y` blindly in production.
- Test upgrades on a staging host first. Do not run `apt full-upgrade` or `dnf upgrade` on production systems without a rollback plan.
- Pin critical packages with `apt-mark hold` or `dnf versionlock` so routine upgrades cannot replace them unexpectedly.
- Prefer repository packages over manually installed `.deb` and `.rpm` files. Manual files receive no security updates.
- Run package operations as root or through `sudo`, and keep the package index current with `apt update` before installing.
- Audit `/etc/apt/sources.list.d/` and `/etc/yum.repos.d/` regularly and remove stale third-party repositories.
