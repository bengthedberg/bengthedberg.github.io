---
layout: post
title: "GitHub DevOps — Part 1: Managing Multiple GitHub Accounts with SSH"
date: 2025-11-06 00:00:00 +0000
categories: [DevOps]
tags:
  - github
  - ssh
  - git
  - devops
  - multi-account
series: "GitHub DevOps"
series_part: 1
---


## Series Overview

1. **Multiple GitHub Accounts with SSH** (this article) — Configure SSH for personal and work accounts
2. [Semantic Versioning with GitVersion](/posts/github-devops-part2-gitversion/) — Automated versioning using GitFlow
3. [GitHub Actions Workflows](/posts/github-devops-part3-workflows/) — Build, version, and publish NuGet packages


## The Problem

GitHub offers two ways to authenticate with repositories: HTTPS and SSH.

- **HTTPS** requires an access token every time you push
- **SSH** lets you push without re-entering credentials

SSH works perfectly with a single account. But if you have a **personal** GitHub account and a **work** account (or multiple client accounts as a freelancer), things get complicated. By default, SSH uses a single identity — so how do you tell Git which account to use for which repository?

The solution is **SSH config aliases** — one identity file per account, mapped to different hostnames.

## Prerequisites

### macOS / Linux

The `ssh-agent` is typically running by default. Verify:

```bash
ssh-add -l
```

If you get an error, start the agent:

```bash
eval "$(ssh-agent -s)"
```

### Windows

On Windows, the `ssh-agent` service may be disabled by default. Check and enable it in an **administrator** terminal:

```powershell
# Check status
Get-Service ssh-agent

# If not running, enable and start it
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent

# Verify
Get-Service ssh-agent
```

## Step 1: Generate SSH Keys

You likely already have a key for your primary account (`~/.ssh/id_ed25519` or `~/.ssh/id_rsa`). Generate a new key for the second account:

```bash
ssh-keygen -t ed25519 -C "your-work-email@company.com"
```

> **Note**: Use `ed25519` instead of `rsa` — it's more secure and produces shorter keys. Only fall back to RSA if your system doesn't support Ed25519.

When prompted for a file location, give it a distinct name:

```
Enter file in which to save the key (~/.ssh/id_ed25519): ~/.ssh/id_ed25519_work
```

Enter a passphrase (recommended — use something memorable but hard to guess).

You now have two files:

```
~/.ssh/id_ed25519_work       # Private key (never share this)
~/.ssh/id_ed25519_work.pub   # Public key (add to GitHub)
```

## Step 2: Add the Public Key to GitHub

Copy your public key:

```bash
cat ~/.ssh/id_ed25519_work.pub
```

In your **work** GitHub account:
1. Go to **Settings** → **SSH and GPG keys**

![GitHub SSH settings](/assets/img/posts/github-devops-part1-multiple-accounts/github-series/image-20240326095200789.png)

2. Click **New SSH key**

![Add SSH key](/assets/img/posts/github-devops-part1-multiple-accounts/github-series/image-20240326095245056.png)

3. Paste the public key content and click **Add SSH key**

## Step 3: Register the Key with ssh-agent

```bash
ssh-add ~/.ssh/id_ed25519_work
```

You'll be prompted for the passphrase. On success:

```
Identity added: ~/.ssh/id_ed25519_work (your-work-email@company.com)
```

## Step 4: Configure SSH Aliases

This is the key step. Create or edit `~/.ssh/config`:

```bash
# Default GitHub (personal account)
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519

# Work GitHub account
Host github.com-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_work
```

The trick is the `Host` value. For your work account, we use `github.com-work` instead of `github.com`. SSH uses this alias to select the correct identity file, while `HostName` still points to the real GitHub server.

## Step 5: Clone and Use Repositories

### Personal Account (unchanged)

```bash
git clone git@github.com:your-personal/repo.git
```

### Work Account (use the alias)

```bash
git clone git@github.com-work:your-org/repo.git
```

The only difference is `github.com-work` instead of `github.com`. SSH resolves the alias, connects to GitHub, and authenticates with the work key.

### Existing Repositories

For repositories you've already cloned, update the remote URL:

```bash
# Check current remote
git remote -v

# Update to use work alias
git remote set-url origin git@github.com-work:your-org/repo.git
```

## Step 6: Set Git User per Repository

SSH handles authentication, but you also want the correct name and email on your commits:

```bash
# Inside a work repository
git config user.name "Your Name"
git config user.email "your-work-email@company.com"
```

This sets the identity for that repository only. Your global `~/.gitconfig` remains unchanged for personal projects.

### Automating with Conditional Includes

If all your work repos live under a common directory (e.g., `~/work/`), you can automate this in `~/.gitconfig`:

```ini
[includeIf "gitdir:~/work/"]
    path = ~/.gitconfig-work
```

Then create `~/.gitconfig-work`:

```ini
[user]
    name = Your Name
    email = your-work-email@company.com
```

Every repository under `~/work/` automatically uses the work identity.

## Verifying the Setup

Test each connection:

```bash
# Personal account
ssh -T git@github.com
# → Hi personal-username! You've successfully authenticated...

# Work account
ssh -T git@github.com-work
# → Hi work-username! You've successfully authenticated...
```

## Adding More Accounts

The pattern scales to any number of accounts. For a third client:

```bash
# Generate key
ssh-keygen -t ed25519 -C "email@client.com" -f ~/.ssh/id_ed25519_client

# Add to ssh-agent
ssh-add ~/.ssh/id_ed25519_client

# Add to SSH config
Host github.com-client
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_client

# Clone using the alias
git clone git@github.com-client:client-org/repo.git
```

## Key Takeaways

1. **One SSH key per GitHub account** — never share keys between accounts
2. **SSH config aliases** (`github.com-work`) select the right identity automatically
3. **Use `ed25519`** over RSA for new keys — shorter, faster, more secure
4. **Set `user.email` per repo** or use conditional includes to keep commits attributed correctly
5. **Test with `ssh -T`** to verify each alias resolves to the expected account

## What's Next

In [Part 2: Semantic Versioning with GitVersion](/posts/github-devops-part2-gitversion/), we'll automate version number generation using GitVersion and the GitFlow branching strategy.
