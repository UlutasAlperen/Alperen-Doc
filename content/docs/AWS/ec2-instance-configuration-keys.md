---
title: "ec2-instance-configuration-keys"
weight: 80
---

# EC2 Instance Configuration: Keys

We need to get the PatientPing application running on a server, but first we need a secure way to _connect_ to that server. Usually, this is done with `ssh` and a key pair.

Windows servers have extra hoops to jump through (some of them also requiring a key pair), but Linux systems make up [the majority](https://sqmagazine.co.uk/linux-statistics/#:~:text=Linux%20runs%2049.2%25%20of%20all,among%20active%20subscriptions%20in%202025.) of cloud servers, even on Microsoft's [Azure](https://build5nines.com/linux-is-most-used-os-in-microsoft-azure-over-50-percent-fo-vm-cores/), which gives me joy. We'll focus on Linux for this document.

A cryptographic key pair has two parts that work together:

- The **public key** is a string of gibberish that you can share with anyone.
- The **private key** is another string of gibberish that you **must keep secret**.

If you've ever used `ssh` to connect to a server, you probably had to generate a key pair for it. You'll do the same thing to access your AWS server instances.

## Assignment

**All PatientPing employees need to have their "own" key pair. Let's make one!**

**Cost check:** AWS would probably love to charge by the key pair, but they don't; it's free.

1.  Generate a new `patientping-key` key pair. On any modern Unix-like system, `ssh-keygen` should be available, and you can use it as follows (note, we recommend the [Ed25519 algorithm](https://en.wikipedia.org/wiki/EdDSA#Ed25519)):

```sh
ssh-keygen -t ed25519 -C "patientping-key" -f ~/.ssh/patientping-key
```

>  **U can  set a password** on the SSH key. Passwords are good IRL to harden security.

2.  Upload your **PUBLIC** key to AWS using the CLI. Make sure to include the `fileb://` prefix with the path to your `.pub` file:

```sh
aws ec2 import-key-pair --key-name "patientping-key" --public-key-material fileb://$HOME/.ssh/patientping-key.pub
```

3.  Check your key pairs on AWS. You should now see one with a `KeyName` of `patientping-key`.

```sh
aws ec2 describe-key-pairs
```

### Tip

When the AWS CLI takes a parameter like `--public-key-material`, it can accept the value in a few forms: a literal string typed directly, or a reference to file contents. AWS uses special prefixes to tell the CLI how to interpret the value:

- `file://` — reads the file's contents as **text**
- `fileb://` — reads the file's contents as **binary** (raw bytes)

Your public key file (`patientping-key.pub`) technically contains text (it's base64-encoded), but AWS specifically requires the `import-key-pair` command's `--public-key-material` to be passed as binary data rather than a text string. If you used `file://` instead, the CLI would try to process it as a UTF-8 string, which can cause encoding mismatches or errors when AWS tries to validate the key material.

So the command:

```sh
aws ec2 import-key-pair --key-name "patientping-key" --public-key-material fileb://$HOME/.ssh/patientping-key.pub
```

is telling the CLI: "Don't interpret this file as text, just read the raw bytes and send them along."

It's a small detail, but it's a common gotcha with the AWS CLI. Any time you're passing certificates, keys, or binary blobs (like Lambda zip files) to AWS CLI commands, keep an eye out for whether it wants `file://` or `fileb://`.