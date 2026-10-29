---
title: "system-information"
weight: 35
---
**Scope:** Distro-independent; applies to Debian-based and RHEL-based Linux distributions  
**Topics:** os-release fields, scripting with os-release, uname, hostnamectl, release files, and verification.

## 1. Identify the Distribution with /etc/os-release

`/etc/os-release` is the standard machine-readable identification file defined by the os-release specification. Shell scripts, provisioning tools, and package managers source it to detect the distribution. It is present on every current Debian-based and RHEL-based system.

Inspect the file:

```bash
cat /etc/os-release
```

Expected output on Ubuntu 24.04:

```text
PRETTY_NAME="Ubuntu 24.04.1 LTS"
NAME="Ubuntu"
VERSION_ID="24.04"
VERSION="24.04.1 LTS (Noble Numbat)"
VERSION_CODENAME=noble
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=noble
LOGO=ubuntu-logo
```

Read the fields:

| Field | Meaning | Example |
|---|---|---|
| `NAME` | Human-readable distribution name | `Ubuntu` |
| `ID` | Lowercase, machine-safe identifier for the distribution | `ubuntu` |
| `ID_LIKE` | Space-separated identifiers of related distributions | `debian` |
| `VERSION` | Full version string, often with a release codename | `24.04.1 LTS (Noble Numbat)` |
| `VERSION_ID` | Version suitable for scripts and comparisons | `24.04` |
| `VERSION_CODENAME` | Release codename | `noble` |
| `PRETTY_NAME` | Single-line name for user-facing output | `Ubuntu 24.04.1 LTS` |
| `HOME_URL` | Project homepage | `https://www.ubuntu.com/` |

## 2. Use os-release in Scripts

Source the file to load its fields as shell variables:

```bash
source /etc/os-release
echo "$ID" "$VERSION_ID"
```

Expected output:

```text
ubuntu 24.04
```

Branch on the distribution in a script:

```bash
source /etc/os-release

if [ "$ID" = "ubuntu" ] || [ "$ID" = "debian" ]; then
    echo "Debian family: $PRETTY_NAME"
    apt update
elif [ "$ID" = "rocky" ] || [ "$ID" = "rhel" ]; then
    echo "RHEL family: $PRETTY_NAME"
    dnf check-update
else
    echo "Unsupported distribution: $ID"
fi
```

Check `ID_LIKE` for derivative distributions. Values such as `ID_LIKE=debian` on Ubuntu or `ID_LIKE="rhel centos fedora"` on Rocky Linux identify the upstream family, so a script can match the family instead of every derivative:

```bash
source /etc/os-release
case " $ID_LIKE " in
    *debian*) echo "Debian-based" ;;
    *rhel*)   echo "RHEL-based" ;;
    *)        echo "Unknown family" ;;
esac
```

## 3. Inspect the Kernel with uname

Display all kernel and hardware information:

```bash
uname -a
```

Expected output:

```text
Linux srv01 6.1.0-18-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.76-1 (2024-02-01) x86_64 GNU/Linux
```

Query individual fields:

```bash
uname -s
uname -r
uname -m
uname -n
uname -v
```

**Options:**
- `-s`: Kernel name, `Linux`.
- `-r`: Kernel release, for example `6.1.0-18-amd64`. Use this for module and driver compatibility checks.
- `-m`: Hardware architecture, for example `x86_64` or `aarch64`.
- `-n`: Network hostname. Prefer `hostnamectl` for hostname management.
- `-v`: Kernel build version string, including the build compiler and date.

## 4. Inspect the Hostname with hostnamectl

A systemd-based system maintains three hostname values:

- **Static:** The configured hostname stored in `/etc/hostname`. Set by the administrator and applied at boot.
- **Pretty:** A free-form display name that may contain spaces and mixed case, shown in desktop environments.
- **Transient:** A temporary hostname set at runtime, for example by DHCP. It falls back to the static hostname when not supplied.

Display the current host status:

```bash
hostnamectl
```

Expected output:

```text
 Static hostname: srv01
       Icon name: computer-vm
         Chassis: vm 🖴
Machine ID: 8f4a2b1c9d3e4f5a6b7c8d9e0f1a2b3c
       Boot ID: 1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d
Virtualization: kvm
Operating System: Ubuntu 24.04.1 LTS
          Kernel: Linux 6.8.0-45-generic
    Architecture: x86-64
```

Set the static hostname:

```bash
hostnamectl set-hostname <HOSTNAME>
```

Display only the static hostname:

```bash
hostnamectl --static
```

**Options:**
- `set-hostname <HOSTNAME>`: Sets the static hostname and applies it immediately.
- `--static`: Prints the static hostname only.
- `--transient`: Prints the transient hostname only.
- `--pretty`: Prints the pretty hostname only.
- `--pretty`, `--static`, or `--transient` combined with `set-hostname`: Sets only that hostname type.

## 5. Query Additional Release Files

Inspect the login banner and hostname files:

```bash
cat /etc/issue
cat /etc/hostname
```

Check the distribution-specific version file. Debian-based systems provide `/etc/debian_version`:

```bash
cat /etc/debian_version
```

Expected output on Debian 12:

```text
12.7
```

RHEL-based systems provide `/etc/redhat-release`:

```bash
cat /etc/redhat-release
```

Expected output on Rocky 9:

```text
Rocky Linux release 9.4 (Blue Onyx)
```

Query the Linux Standard Base information when available:

```bash
lsb_release -a
```

Expected output:

```text
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 24.04.1 LTS
Release:        24.04
Codename:       noble
```

On Debian-based systems, `lsb_release` may require the `lsb-release` package:

```bash
apt install lsb-release
```

On RHEL-based systems it is available in the `redhat-lsb-core` package. When the tool is absent, `/etc/os-release` provides the same information without an extra dependency.

## 6. Verify the Collected Information

Run the identification commands together and confirm that the values are consistent:

```bash
source /etc/os-release
echo "Distribution: $PRETTY_NAME"
echo "Kernel: $(uname -r)"
echo "Architecture: $(uname -m)"
hostnamectl --static
```

Expected output:

```text
Distribution: Ubuntu 24.04.1 LTS
Kernel: 6.8.0-45-generic
Architecture: x86-64
srv01
```

Cross-check the results: the distribution name must match the release files, the architecture must match the installed package set, and the hostname must match `/etc/hostname` and the entry in `/etc/hosts`.

---

# Part 3: Important Notes

## Debian vs RHEL os-release

The same fields carry different values on each family:

| Field | Debian 12 | Rocky 9 |
|---|---|---|
| `PRETTY_NAME` | `Debian GNU/Linux 12 (bookworm)` | `Rocky Linux 9.4 (Blue Onyx)` |
| `NAME` | `Debian GNU/Linux` | `Rocky Linux` |
| `VERSION` | `12 (bookworm)` | `9.4 (Blue Onyx)` |
| `VERSION_ID` | `12` | `9.4` |
| `VERSION_CODENAME` | `bookworm` | Not set |
| `ID` | `debian` | `rocky` |
| `ID_LIKE` | Not set | `rhel centos fedora` |
| `HOME_URL` | `https://www.debian.org/` | `https://rockylinux.org/` |

Note that `VERSION_CODENAME` is not defined on RHEL-based systems, and `ID_LIKE` is empty on Debian itself because it is the upstream of its family. Scripts must handle missing fields gracefully.

## Command Reference

| Task | Command |
|---|---|
| Show distribution identity | `cat /etc/os-release` |
| Load fields into the shell | `source /etc/os-release` |
| Print distribution ID and version | `echo "$ID" "$VERSION_ID"` |
| Show all kernel information | `uname -a` |
| Show kernel release | `uname -r` |
| Show architecture | `uname -m` |
| Show host status | `hostnamectl` |
| Show static hostname | `hostnamectl --static` |
| Set static hostname | `hostnamectl set-hostname <HOSTNAME>` |
| Show login banner | `cat /etc/issue` |
| Show Debian version file | `cat /etc/debian_version` |
| Show RHEL version file | `cat /etc/redhat-release` |
| Show LSB information | `lsb_release -a` |
| Verify collected information | `source /etc/os-release && echo "$PRETTY_NAME" "$(uname -r)"` |

## Security Guidelines

- Replace placeholders such as `<HOSTNAME>` with actual values. Do not include the angle brackets in commands.
- Treat `/etc/os-release` as trustworthy metadata, not as security input. Do not use its values for access control or authorization decisions without additional verification.
- Do not rely on `/etc/issue` as an access control banner. Configure `Banner /etc/issue.net` in `sshd_config` and the corresponding PAM motd settings for pre-login legal notices.
- Prefer `/etc/os-release` over parsing `lsb_release` output in scripts. The file is always present and its fields follow a stable specification.
- Quote all variables sourced from `/etc/os-release`. Values such as `PRETTY_NAME` contain spaces and parentheses that break unquoted shell expansion.
- Match on `ID` or `ID_LIKE` instead of `PRETTY_NAME` or `NAME`. The human-readable fields change between releases and languages.
- Handle missing fields explicitly. `VERSION_CODENAME` and vendor-specific entries are optional, and `set -u` scripts must tolerate their absence.
- Use lowercase values for `hostnamectl set-hostname` and keep the hostname consistent with `/etc/hosts` to avoid unexpected resolution behavior.
- When logged in as root, the commands above generally do not require the `sudo` prefix.
- Re-verify system identification after major upgrades or migrations, since `os-release` contents and kernel versions change with the distribution release.
