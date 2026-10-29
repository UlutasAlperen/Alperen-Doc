---
title: "group-management"
weight: 20
---
**Scope:** Debian-based and RHEL-based Linux distributions  
**Topics:** Primary and supplementary groups, group creation and deletion, membership management, /etc/group and /etc/gshadow, group modification, and verification.

# Part 1: Debian-Based Systems

**Examples:** Debian, Ubuntu

## 1. Create a Group

Run as root:

```bash
addgroup <GROUP_NAME>
```

The GID is allocated automatically from the regular GID range.

## 2. Create a System Group

Create a system group for a service or application:

```bash
addgroup --system <GROUP_NAME>
```

**Options:**
- `--system`: Allocates the GID from the system GID range instead of the regular range.

## 3. Add a User to a Group

Add the user to the group as a supplementary group:

```bash
usermod -aG <GROUP_NAME> <USERNAME>
```

**Options:**
- `-a`: Appends the user to the group list without removing existing supplementary groups.
- `-G`: Specifies one or more supplementary groups.

Alternatively, use the Debian helper:

```bash
adduser <USERNAME> <GROUP_NAME>
```

Verify the membership:

```bash
id <USERNAME>
groups <USERNAME>
```

Confirm that `<GROUP_NAME>` appears in the output. If the user is logged in, start a new login session before testing access.

## 4. Remove a User from a Group

Remove the user from a supplementary group:

```bash
deluser <USERNAME> <GROUP_NAME>
```

Alternatively:

```bash
gpasswd -d <USERNAME> <GROUP_NAME>
```

Start a new login session so the change takes effect in the user's environment.

## 5. Modify a Group

Rename an existing group:

```bash
groupmod -n <NEW_GROUP_NAME> <GROUP_NAME>
```

Change the GID of an existing group:

```bash
groupmod -g <NEW_GID> <GROUP_NAME>
```

**Options:**
- `-n`: Sets a new group name.
- `-g`: Sets a new GID.

Changing a GID does not update file ownership automatically. Run `chgrp` on affected files afterwards.

## 6. Verify Group Membership

Inspect the account database:

```bash
id <USERNAME>
groups <USERNAME>
getent group <GROUP_NAME>
```

Expected output:

```text
uid=1001(alice) gid=1001(alice) groups=1001(alice),27(sudo),1005(devops)
devops:x:1005:alice,bob
```

`getent` reads the configured name service, so it reflects LDAP or other directory-backed groups as well.

## 7. Delete a Group

Delete the group:

```bash
delgroup <GROUP_NAME>
```

The command fails if the group is still the primary group of any user. Change those users first:

```bash
usermod -g <OTHER_GROUP> <USERNAME>
delgroup <GROUP_NAME>
```

---

# Part 2: RHEL-Based Systems

**Examples:** RHEL, Rocky Linux, AlmaLinux

## 1. Create a Group

Run as root:

```bash
groupadd <GROUP_NAME>
```

The command completes silently on success. To assign a specific GID:

```bash
groupadd -g <GID> <GROUP_NAME>
```

**Options:**
- `-g <GID>`: Sets the GID explicitly.

## 2. Create a System Group

Create a system group for a service or application:

```bash
groupadd -r <GROUP_NAME>
```

**Options:**
- `-r`: Allocates the GID from the system GID range.

## 3. Add a User to a Group

Add the user to the group as a supplementary group:

```bash
usermod -aG <GROUP_NAME> <USERNAME>
```

**Options:**
- `-a`: Appends the user to the group list without removing existing supplementary groups.
- `-G`: Specifies one or more supplementary groups.

Verify the membership:

```bash
id <USERNAME>
groups <USERNAME>
```

Confirm that `<GROUP_NAME>` appears in the output. Start a new login session before testing access.

## 4. Remove a User from a Group

Remove the user from a supplementary group:

```bash
gpasswd -d <USERNAME> <GROUP_NAME>
```

RHEL-based systems do not provide the Debian `deluser` helper by default.

## 5. Modify a Group

Rename an existing group:

```bash
groupmod -n <NEW_GROUP_NAME> <GROUP_NAME>
```

Change the GID of an existing group:

```bash
groupmod -g <NEW_GID> <GROUP_NAME>
```

**Options:**
- `-n`: Sets a new group name.
- `-g`: Sets a new GID.

Run `chgrp` on affected files after a GID change.

## 6. Verify Group Membership

Inspect the account database:

```bash
id <USERNAME>
groups <USERNAME>
getent group <GROUP_NAME>
```

Confirm that `wheel` and `<GROUP_NAME>` appear in the output.

## 7. Delete a Group

Delete the group:

```bash
groupdel <GROUP_NAME>
```

The command fails if the group is still the primary group of any user. Change those users first:

```bash
usermod -g <OTHER_GROUP> <USERNAME>
groupdel <GROUP_NAME>
```

---

# Part 3: Important Notes

## Primary vs. Supplementary Groups

| Property | Primary group | Supplementary groups |
|---|---|---|
| Number per user | Exactly one | Zero or more |
| Recorded in | `/etc/passwd` field 4 | `/etc/group` field 4 |
| Effect on new files | New files receive this group | No effect on file ownership |
| Typical use | Personal file ownership | Shared access to projects and resources |
| Changed with | `usermod -g <GROUP_NAME> <USERNAME>` | `usermod -aG <GROUP_NAME> <USERNAME>` |

The `id` output lists the primary group as `gid=` and all groups in `groups=`, with the primary group first.

To use a different group for a single session without changing the account, run:

```bash
newgrp <GROUP_NAME>
```

New files created in that session are owned by the selected group.

## /etc/group Format

One line per group:

```text
devops:x:1005:alice,bob
```

| Field | Content | Example |
|---|---|---|
| 1 | Group name | `devops` |
| 2 | Password field, shadowed | `x` |
| 3 | GID | `1005` |
| 4 | Supplementary members, comma-separated | `alice,bob` |

Do not edit this file directly. Use `groupadd`, `groupdel`, `groupmod`, and `gpasswd`, which keep the file consistent and handle locking.

## /etc/gshadow Format

One line per group:

```text
devops:!::alice
```

| Field | Content | Example |
|---|---|---|
| 1 | Group name | `devops` |
| 2 | Encrypted group password, `!` when none is set | `!` |
| 3 | Group administrators | empty |
| 4 | Group members | `alice` |

Group passwords allow users to switch into a group with `newgrp` without being members. This mechanism is rarely used and weakens access control. Keep the field locked and manage membership explicitly.

## Naming Rules and Reserved System Groups

Naming rules:
- Use lowercase letters, digits, hyphens, and underscores.
- Keep names at or below 32 characters.
- Start with a lowercase letter or underscore.
- Do not reuse names of existing accounts or of vendor-reserved groups.

Reserved or well-known system groups:

| Group | Purpose |
|---|---|
| `root` | Superuser group, GID 0 |
| `sudo` | Administrative access on Debian-based systems |
| `wheel` | Administrative access on RHEL-based systems |
| `adm` | Read access to system logs on Debian-based systems |
| `shadow` | Read access to `/etc/shadow` and `/etc/gshadow` |
| `disk` | Raw block device access |

`sudo` and `wheel` serve the same purpose: their members receive administrative privileges through the sudoers configuration. Debian-based systems grant this through the `sudo` group, RHEL-based systems through the `wheel` group. Both are ordinary groups whose power comes from the sudoers rule that references them. Confirm the sudoers rule before relying on membership, and verify actual GIDs with `getent group <GROUP_NAME>` instead of assuming them.

## Command Reference

| Operation | Debian-based | RHEL-based |
|---|---|---|
| Create group | `addgroup <GROUP_NAME>` | `groupadd <GROUP_NAME>` |
| Create system group | `addgroup --system <GROUP_NAME>` | `groupadd -r <GROUP_NAME>` |
| Add user to group | `usermod -aG <GROUP_NAME> <USERNAME>` | `usermod -aG <GROUP_NAME> <USERNAME>` |
| Remove user from group | `deluser <USERNAME> <GROUP_NAME>` | `gpasswd -d <USERNAME> <GROUP_NAME>` |
| Rename group | `groupmod -n <NEW_GROUP_NAME> <GROUP_NAME>` | `groupmod -n <NEW_GROUP_NAME> <GROUP_NAME>` |
| Change GID | `groupmod -g <NEW_GID> <GROUP_NAME>` | `groupmod -g <NEW_GID> <GROUP_NAME>` |
| Delete group | `delgroup <GROUP_NAME>` | `groupdel <GROUP_NAME>` |
| List groups | `groups <USERNAME>` | `groups <USERNAME>` |
| Inspect membership | `id <USERNAME>` | `id <USERNAME>` |
| Query group entry | `getent group <GROUP_NAME>` | `getent group <GROUP_NAME>` |

## Security Guidelines

- Replace placeholders such as `<GROUP_NAME>` and `<USERNAME>` with actual names. Do not include the angle brackets in commands.
- Always use `usermod -aG`. The variant `usermod -G` replaces all existing supplementary groups.
- Never edit `/etc/group` or `/etc/gshadow` with a text editor. Use the group-management commands instead.
- Keep group passwords disabled. Grant access through explicit membership only.
- Do not add service accounts to `sudo`, `wheel`, `adm`, or `disk`.
- Assign a dedicated group per project or application and grant file access to that group instead of sharing accounts.
- Remove group membership promptly when a user changes role, and start a new login session to enforce the change.
- Delete groups that no longer serve a purpose. Review membership with `getent group` during audits.
- When logged in as root, the commands above do not require the `sudo` prefix.
