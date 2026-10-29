---
title: "file-permissions"
weight: 30
---
**Scope:** Distro-independent; applies to ext4, XFS, and other standard Linux filesystems  
**Topics:** ls -l output, ugo and rwx, chmod notation, ownership with chown and chgrp, umask, SUID, SGID, sticky bit, ACLs, and verification.

# Main Flow

## 1. Read the Permission String

List the file details:

```bash
ls -l <FILE>
```

Expected output:

```text
-rwxr-xr-- 1 alice devops 2048 Mar  5 10:00 report.sh
```

Break down the output:

- `-rwxr-xr--`: Permission string.
- `1`: Hard link count.
- `alice`: Owner (user).
- `devops`: Owning group.
- `2048`: Size in bytes.
- `Mar  5 10:00`: Last modification time.
- `report.sh`: File name.

Read the permission string as follows:

- Character 1: File type. `-` is a regular file, `d` is a directory, `l` is a symbolic link.
- Characters 2-4: Owner permissions, `rwx`.
- Characters 5-7: Group permissions, `rwx`.
- Characters 8-10: Other permissions, `rwx`.

## 2. Understand ugo and rwx

Three classes of users and three permissions combine into nine permission bits:

| Class | Letter | Meaning |
|---|---|---|
| Owner | `u` | The file's owning user |
| Group | `g` | The file's owning group |
| Other | `o` | Everyone else |
| All | `a` | Owner, group, and other combined |

| Permission | Letter | Octal | On a file | On a directory |
|---|---|---|---|---|
| Read | `r` | 4 | Read file contents | List entries with `ls` |
| Write | `w` | 2 | Modify file contents | Create, rename, or delete entries |
| Execute | `x` | 1 | Run the file as a program | Enter the directory with `cd`, access entries by name |

Directory write access without execute is meaningless, and execute without read allows access to known names but no listing.

## 3. Change Permissions with Symbolic Notation

Add execute permission for the owner:

```bash
chmod u+x <FILE>
```

Remove write permission for group and other:

```bash
chmod go-w <FILE>
```

Set exact permissions for all classes:

```bash
chmod u=rwx,g=rx,o=r <FILE>
```

Apply recursively to a directory tree:

```bash
chmod -R g+rw <DIRECTORY>
```

**Options:**
- `u`, `g`, `o`, `a`: Select owner, group, other, or all classes.
- `+`, `-`, `=`: Add, remove, or set permissions exactly.
- `r`, `w`, `x`: Select the read, write, and execute permissions.
- `-R`: Applies the change recursively. Use with care.

## 4. Change Permissions with Octal Notation

Each class receives one octal digit that sums read (4), write (2), and execute (1):

| Octal | Bits | Permissions |
|---|---|---|
| 0 | 000 | `---` |
| 1 | 001 | `--x` |
| 2 | 010 | `-w-` |
| 3 | 011 | `-wx` |
| 4 | 100 | `r--` |
| 5 | 101 | `r-x` |
| 6 | 110 | `rw-` |
| 7 | 111 | `rwx` |

Read the three digits as owner, group, other:

```bash
chmod 750 <FILE>
```

Expected result: `rwxr-x---` for owner `alice`, group `devops`.

Common combinations:

| Octal | Permissions | Typical use |
|---|---|---|
| 600 | `rw-------` | Private files such as SSH keys |
| 644 | `rw-r--r--` | Readable configuration files |
| 700 | `rwx------` | Private scripts and `~/.ssh` |
| 750 | `rwxr-x---` | Group-shared scripts |
| 755 | `rwxr-xr-x` | Public executables |

## 5. Change Ownership

Change the owner and group of a file:

```bash
chown <USERNAME>:<GROUP_NAME> <FILE>
```

Change only the group:

```bash
chgrp <GROUP_NAME> <FILE>
```

Apply recursively to a directory tree:

```bash
chown -R <USERNAME>:<GROUP_NAME> <DIRECTORY>
```

**Options:**
- `<USERNAME>`: New owning user.
- `<GROUP_NAME>`: New owning group.
- `-R`: Applies the change recursively.

Only root can change the owning user. A group owner can change the group to any of their own groups.

## 6. Set the Default umask

The umask subtracts permissions from the defaults of `666` for files and `777` for directories. Display the current value:

```bash
umask
```

Expected output:

```text
0022
```

With `umask 022`, new files are created as `644` and new directories as `755`. With `umask 077`, new files are created as `600` and new directories as `700`.

Set the umask for the current session:

```bash
umask 077
```

Make it persistent for a user in `~/.bashrc`:

```text
umask 077
```

Set the system default for new logins in `/etc/login.defs`:

```ini
UMASK 022
```

## 7. Apply Special Bits

The three special bits occupy the first octal digit or replace the execute letter in the permission string.

| Special bit | Octal | Symbol | Effect |
|---|---|---|---|
| SUID | 4000 | `s` in the owner execute position | The file runs with the effective UID of the file owner |
| SGID | 2000 | `s` in the group execute position | The file runs with the file group; on a directory, new entries inherit the directory group |
| Sticky bit | 1000 | `t` in the other execute position | Only the file owner, directory owner, or root can delete entries |

Set SUID on a script owner:

```bash
chmod u+s <FILE>
chmod 4755 <FILE>
```

Set SGID on a shared project directory so new files keep the group:

```bash
chmod g+s <DIRECTORY>
chmod 2775 <DIRECTORY>
```

Set the sticky bit on `/tmp`:

```bash
chmod 1777 /tmp
```

Expected result: `drwxrwxrwt`. Every user can create files in `/tmp`, but users can only delete their own files. This is the standard protection for world-writable directories.

SUID on scripts is ignored by most kernels for security reasons. Restrict SUID to vetted compiled binaries such as `/usr/bin/passwd`.

## 8. Configure ACLs

Access control lists grant permissions to specific users or groups beyond the `ugo` model. Check the ACL tools first:

```bash
getfacl <FILE>
setfacl --version
```

Grant read access to a single user:

```bash
setfacl -m u:<USERNAME>:r <FILE>
```

Grant read and execute access to a group on a directory:

```bash
setfacl -m g:<GROUP_NAME>:rx <DIRECTORY>
```

Set a default ACL so new entries inherit it:

```bash
setfacl -d -m g:<GROUP_NAME>:rx <DIRECTORY>
```

Remove an ACL entry:

```bash
setfacl -x u:<USERNAME> <FILE>
```

**Options:**
- `-m`: Modifies the ACL.
- `-x`: Removes an ACL entry.
- `-d`: Applies the rule as a default ACL on a directory.
- `-R`: Applies the rule recursively.

Files with ACLs show a `+` at the end of the permission string in `ls -l`. Use `getfacl` to read the full effective permissions.

## 9. Verify the Result

Inspect the effective permissions:

```bash
ls -l <FILE>
getfacl <FILE>
stat -c '%A %U:%G %n' <FILE>
```

Expected output for `stat`:

```text
-rwxr-x--- alice devops report.sh
```

Check directory traversal on the full path:

```bash
namei -l <PATH>
```

Every directory component on the path requires execute permission for the accessing account, otherwise the final file permissions do not matter.

---

# Part 3: Important Notes

## Permission Evaluation Order

- ACLs are checked first. If a matching user or group ACL entry exists, it determines the outcome.
- Otherwise, the `ugo` bits apply: owner, then group, then other.
- The most specific match wins, so an owner ACL entry overrides the group mask.

## Symbolic vs. Octal Notation

| Property | Symbolic | Octal |
|---|---|---|
| Syntax | `chmod u+x <FILE>` | `chmod 750 <FILE>` |
| Change type | Relative or selective | Absolute for all classes at once |
| Readability | States the intent | States the final state |
| Risk | Low | Overwrites all nine bits at once |

Prefer symbolic notation for single-bit changes and octal notation when setting a complete, well-understood permission set.

## Command Reference

| Operation | Command |
|---|---|
| List permissions | `ls -l <FILE>` |
| Add permission | `chmod u+x <FILE>` |
| Remove permission | `chmod go-w <FILE>` |
| Set absolute permissions | `chmod 750 <FILE>` |
| Change owner and group | `chown <USERNAME>:<GROUP_NAME> <FILE>` |
| Change group only | `chgrp <GROUP_NAME> <FILE>` |
| Set SUID | `chmod 4755 <FILE>` |
| Set SGID | `chmod 2775 <DIRECTORY>` |
| Set sticky bit | `chmod 1777 /tmp` |
| Show umask | `umask` |
| Read ACL | `getfacl <FILE>` |
| Grant ACL entry | `setfacl -m u:<USERNAME>:r <FILE>` |
| Remove ACL entry | `setfacl -x u:<USERNAME> <FILE>` |
| Show exact metadata | `stat -c '%A %U:%G %n' <FILE>` |
| Check path traversal | `namei -l <PATH>` |

## Security Guidelines

- Replace placeholders such as `<FILE>`, `<USERNAME>`, and `<GROUP_NAME>` with actual values. Do not include the angle brackets in commands.
- Set a restrictive umask such as `077` on accounts that handle credentials or private keys.
- Never use `chmod 777`. It grants write access to every account and is never a fix for permission problems.
- Avoid `chmod -R` on system paths. Recursive changes are difficult to reverse and can break services.
- Do not set SUID on scripts or interpreters. SUID means the file runs with its owner's privileges, so any flaw becomes a privilege escalation path.
- Keep `/tmp` and other shared directories at `1777` with the sticky bit set.
- Remove ACL entries when they are no longer needed. ACLs override the `ugo` bits and are easy to forget during audits.
- Verify ownership and permissions after every change with `ls -l`, `getfacl`, or `stat`.
- Grant group access through a dedicated group per project instead of widening permissions for other users.
- When logged in as root, the commands above do not require the `sudo` prefix.
