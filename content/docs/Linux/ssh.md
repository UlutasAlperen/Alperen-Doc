---
title: "ssh"
weight: 50
---
**Scope:** Debian-based and RHEL-based Linux distributions  
**Topics:** ed25519 key pairs, public key installation, authorized_keys permissions, sshd hardening, service validation and restart, and ssh-agent.

# Part 1: Debian-Based Systems

**Examples:** Debian, Ubuntu

## 1. Generate an ed25519 Key Pair

Run on the client machine as the user who will log in:

```bash
ssh-keygen -t ed25519 -C "<USERNAME>@<CLIENT_HOST>" -f ~/.ssh/id_ed25519
```

**Options:**
- `-t ed25519`: Uses the Ed25519 algorithm. It is fast, small, and current.
- `-C`: Attaches a comment to the public key, typically the account and host.
- `-f`: Sets the output path for the private key.

Enter a passphrase when prompted. The private key is stored encrypted on disk.

Expected output:

```text
Your identification has been saved in /home/alice/.ssh/id_ed25519
Your public key has been saved in /home/alice/.ssh/id_ed25519.pub
```

Do not overwrite an existing key unless the old key is no longer needed.

## 2. Install the Public Key on the Server

Copy the public key to the target account:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub <USERNAME>@<SERVER>
```

Enter the account password one last time. If `ssh-copy-id` is unavailable, install the key manually:

```bash
cat ~/.ssh/id_ed25519.pub | ssh <USERNAME>@<SERVER> "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

The public key is appended to `~/.ssh/authorized_keys` on the server. One key per line.

## 3. Set the Required File Permissions

On the server, verify the ownership and modes:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

On the client, protect the private key:

```bash
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
```

The files must be owned by the login user. If `~/.ssh` or `authorized_keys` is writable by group or other, sshd ignores the key and the login silently falls back to password authentication.

## 4. Test Public Key Login

Log in with the key from the client:

```bash
ssh -i ~/.ssh/id_ed25519 <USERNAME>@<SERVER>
```

Verify the effective user:

```bash
whoami
```

Expected output:

```text
alice
```

For connection diagnostics, run the client in verbose mode:

```bash
ssh -v -i ~/.ssh/id_ed25519 <USERNAME>@<SERVER>
```

Keep the current session open until a second successful login confirms that access works.

## 5. Harden the SSH Daemon

Create a drop-in configuration on the server:

```bash
mkdir -p /etc/ssh/sshd_config.d
vim /etc/ssh/sshd_config.d/99-hardening.conf
```

Example content:

```text
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
MaxAuthTries 3
AllowUsers alice bob
ClientAliveInterval 300
ClientAliveCountMax 2
```

**Options:**
- `PermitRootLogin no`: Blocks direct root login.
- `PasswordAuthentication no`: Accepts public key authentication only.
- `PubkeyAuthentication yes`: Enables public key authentication explicitly.
- `MaxAuthTries 3`: Limits authentication attempts per connection.
- `AllowUsers alice bob`: Restricts SSH access to the listed accounts.
- `ClientAliveInterval` and `ClientAliveCountMax`: Close idle sessions after the stated interval.

Verify that the file is actually included:

```bash
grep -i '^Include' /etc/ssh/sshd_config
```

## 6. Validate and Restart the SSH Service

Validate the configuration first:

```bash
sshd -t
```

The command prints nothing on success. Fix any reported error before continuing. Restart the service:

```bash
systemctl restart ssh
systemctl status ssh
```

The service unit is named `ssh` on Debian-based systems. Keep the existing session open and confirm that a new login succeeds before closing it.

## 7. Cache the Private Key with ssh-agent

Start the agent and load the key:

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

List and manage loaded keys:

```bash
ssh-add -l
ssh-add -D
```

The agent holds the decrypted key in memory, so the passphrase is requested once per session instead of per connection.

---

# Part 2: RHEL-Based Systems

**Examples:** RHEL, Rocky Linux, AlmaLinux

## 1. Generate and Install a Key Pair

Run on the client machine:

```bash
ssh-keygen -t ed25519 -C "<USERNAME>@<CLIENT_HOST>" -f ~/.ssh/id_ed25519
ssh-copy-id -i ~/.ssh/id_ed25519.pub <USERNAME>@<SERVER>
```

`ssh-copy-id` is provided by the `openssh-clients` package. To install the key manually:

```bash
cat ~/.ssh/id_ed25519.pub | ssh <USERNAME>@<SERVER> "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

## 2. Set Permissions and Test Login

On the server:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

On the client:

```bash
chmod 600 ~/.ssh/id_ed25519
ssh -i ~/.ssh/id_ed25519 <USERNAME>@<SERVER>
```

sshd ignores `authorized_keys` if the file or `~/.ssh` is writable by group or other. Keep the current session open until a second login succeeds.

## 3. Harden the SSH Daemon

Create a drop-in configuration on the server:

```bash
mkdir -p /etc/ssh/sshd_config.d
vim /etc/ssh/sshd_config.d/99-hardening.conf
```

Example content:

```text
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
MaxAuthTries 3
AllowUsers alice bob
ClientAliveInterval 300
ClientAliveCountMax 2
```

RHEL-based systems manage algorithms and ciphers through system-wide crypto policies. Do not override `Ciphers`, `MACs`, or `KexAlgorithms` manually unless a policy requirement demands it:

```bash
update-crypto-policies --show
```

## 4. Validate and Restart the SSH Service

Validate the configuration first:

```bash
sshd -t
```

The command prints nothing on success. Restart the service and enable it at boot:

```bash
systemctl restart sshd
systemctl enable --now sshd
systemctl status sshd
```

The service unit is named `sshd` on RHEL-based systems. If SELinux reports context errors on moved files, restore the labels:

```bash
restorecon -Rv /etc/ssh
```

Keep the existing session open and confirm that a new login succeeds before closing it.

## 5. Cache the Private Key with ssh-agent

Start the agent and load the key:

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
ssh-add -l
```

The agent holds the decrypted key in memory for the duration of the session.

---

# Part 3: Important Notes

## Required File Permissions

| Path | Owner | Mode | Notes |
|---|---|---|---|
| `~/.ssh` | Login user | 700 | Ignored by sshd if writable by group or other |
| `~/.ssh/authorized_keys` | Login user | 600 | One public key per line |
| `~/.ssh/id_ed25519` | Login user | 600 | Private key; never share it |
| `~/.ssh/id_ed25519.pub` | Login user | 644 | Public key; safe to distribute |
| `/etc/ssh/sshd_config.d/99-hardening.conf` | root | 600 | Drop-in hardening configuration |

If key-based login fails, check these modes first. Verbose client output names the rejected file:

```bash
ssh -v -i ~/.ssh/id_ed25519 <USERNAME>@<SERVER>
```

## sshd Configuration Directives

| Directive | Recommended value | Effect |
|---|---|---|
| `PermitRootLogin` | `no` | Blocks direct root login |
| `PasswordAuthentication` | `no` | Disables password login entirely |
| `PubkeyAuthentication` | `yes` | Enables public key login |
| `PermitEmptyPasswords` | `no` | Blocks accounts with empty passwords |
| `MaxAuthTries` | `3` | Limits attempts per connection |
| `AllowUsers` | Explicit list | Restricts which accounts may log in |
| `ClientAliveInterval` | `300` | Sends keepalives every 300 seconds |
| `ClientAliveCountMax` | `2` | Drops the connection after missed keepalives |

Inspect the effective configuration after changes:

```bash
sshd -T
```

Apply `AllowUsers` or `AllowGroups` carefully. A typo locks out every account and requires console access to fix.

## Command Reference

| Operation | Debian-based | RHEL-based |
|---|---|---|
| Generate key pair | `ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519` | `ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519` |
| Install public key | `ssh-copy-id -i ~/.ssh/id_ed25519.pub <USERNAME>@<SERVER>` | `ssh-copy-id -i ~/.ssh/id_ed25519.pub <USERNAME>@<SERVER>` |
| Set key permissions | `chmod 600 ~/.ssh/id_ed25519` | `chmod 600 ~/.ssh/id_ed25519` |
| Set authorized_keys mode | `chmod 600 ~/.ssh/authorized_keys` | `chmod 600 ~/.ssh/authorized_keys` |
| Test login | `ssh -i ~/.ssh/id_ed25519 <USERNAME>@<SERVER>` | `ssh -i ~/.ssh/id_ed25519 <USERNAME>@<SERVER>` |
| Validate sshd config | `sshd -t` | `sshd -t` |
| Show effective config | `sshd -T` | `sshd -T` |
| Restart SSH service | `systemctl restart ssh` | `systemctl restart sshd` |
| Check service status | `systemctl status ssh` | `systemctl status sshd` |
| Load key into agent | `ssh-add ~/.ssh/id_ed25519` | `ssh-add ~/.ssh/id_ed25519` |

## Security Guidelines

- Replace placeholders such as `<USERNAME>`, `<SERVER>`, and `<CLIENT_HOST>` with actual values. Do not include the angle brackets in commands.
- Use ed25519 keys. Do not generate new RSA keys for SSH unless a legacy system requires them.
- Protect every private key with a passphrase and store it with mode 600.
- Never disable `PasswordAuthentication` before a key-based login is confirmed in a second session.
- Keep one session open while testing. A bad `sshd_config` or a locked account otherwise requires console access.
- Always run `sshd -t` before restarting the SSH service.
- Block root login with `PermitRootLogin no` and administer through a regular account with `sudo`.
- Restrict SSH access with `AllowUsers` or `AllowGroups` on hosts with a stable account list.
- Remove unused public keys from `authorized_keys` and delete retired key pairs from the client.
- Avoid agent forwarding (`ssh -A`) to hosts you do not control. A compromised host can use a forwarded agent to authenticate as you.
- Keep the `openssh` packages updated and monitor `/var/log/auth.log` or the journal for repeated failed authentication attempts.
