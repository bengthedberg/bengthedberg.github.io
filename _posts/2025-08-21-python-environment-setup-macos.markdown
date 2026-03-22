---
layout: post
title: "Setting Up Python the Right Way on macOS with pyenv"
date: 2025-08-21 00:00:00 +0000
categories: [Python, Developer Tools]
tags:
  - python
  - pyenv
  - macos
  - virtual-environments
  - developer-tools
---


## The Problem with Python on macOS

macOS ships with a system Python, but you should never use it for development. It's there for the operating system's own tools, and modifying it can break things. Installing Python from python.org or Homebrew directly creates a different problem: you get one global version, and switching between projects that need different Python versions becomes a manual headache.

The solution is **pyenv** — a tool that lets you install multiple Python versions side by side and pin a specific version to each project.


## Why pyenv?

- **Multiple Python versions** on the same machine, installed in isolation
- **Per-project version pinning** via a `.python-version` file — collaborators get the same version automatically
- **Virtual environments** (via `pyenv-virtualenv`) keep each project's dependencies separate
- **No sudo required** — everything installs under your home directory
- **Shell integration** — the active version switches automatically when you `cd` into a project


## Installation

### Prerequisites

#### 1. Xcode Command Line Tools

These provide the compilers and headers needed to build Python from source:

```bash
xcode-select --install
```

#### 2. Homebrew

If you don't have Homebrew yet:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### Install pyenv and pyenv-virtualenv

```bash
brew install pyenv pyenv-virtualenv
```

`pyenv` manages Python versions. `pyenv-virtualenv` adds virtual environment support on top.

### Configure Your Shell

Add the following to your `~/.zshrc` (or `~/.bashrc` if using Bash):

```bash
# pyenv
export PYENV_ROOT="$HOME/.pyenv"
[[ -d $PYENV_ROOT/bin ]] && export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"
```

Reload your shell:

```bash
source ~/.zshrc
```

### Install a Python Version

List available versions:

```bash
pyenv install --list | grep '^\s*3\.'
```

Install one (e.g., Python 3.13):

```bash
pyenv install 3.13.2
```

> pyenv builds Python from source, so this takes a minute or two. If the build fails, you may need additional dependencies — see the [pyenv wiki](https://github.com/pyenv/pyenv/wiki#suggested-build-environment) for your platform.

Set a global default (used when no project-specific version is configured):

```bash
pyenv global 3.13.2
```

Verify:

```bash
python --version
# Python 3.13.2
```


## Per-Project Setup

The real power of pyenv is **per-project version pinning** with isolated virtual environments. Here's the recommended workflow.

### 1. Create a Virtual Environment

Navigate to your project directory and create a virtual environment tied to a specific Python version:

```bash
cd ~/projects/my-api
pyenv virtualenv 3.13.2 my-api
```

This creates a virtual environment named `my-api` using Python 3.13.2. A good convention is to **name the virtual environment the same as the project/repository**.

### 2. Pin It to the Project

```bash
pyenv local my-api
```

This creates a `.python-version` file in the project root containing the name `my-api`. From now on, whenever you `cd` into this directory, pyenv automatically activates the correct virtual environment.

> **Add `.python-version` to your `.gitignore`** — this file is a local machine concern, not a repository concern. Collaborators should create their own virtual environment with the same name or their own preferred name.

### 3. Verify

```bash
python --version
# Python 3.13.2

which python
# /Users/you/.pyenv/shims/python
```

If your shell prompt is configured to show the active pyenv environment, you'll see the environment name displayed.

### 4. Install Dependencies

With the virtual environment active, `pip install` goes into the isolated environment:

```bash
pip install -r requirements.txt
```

Dependencies are completely isolated from other projects and other Python versions on your machine.


## Managing Environments

### List everything

```bash
# All installed Python versions
pyenv versions

# All virtual environments
pyenv virtualenvs
```

### Delete a virtual environment

```bash
pyenv virtualenv-delete my-api
```

This removes the virtual environment and all its installed packages. The `.python-version` file in the project directory will become stale — delete it or create a new environment.

### Install additional Python versions

You can have as many Python versions installed simultaneously as you need:

```bash
pyenv install 3.12.8
pyenv install 3.11.11
```

Different projects can pin different versions without conflict.


## Workflow Summary

Here's the complete flow for starting a new Python project:

```bash
# Create the project
mkdir my-new-project && cd my-new-project
git init

# Create and pin a virtual environment
pyenv virtualenv 3.13.2 my-new-project
pyenv local my-new-project

# Add .python-version to .gitignore
echo '.python-version' >> .gitignore

# Install dependencies
pip install --upgrade pip
pip install <your-packages>
pip freeze > requirements.txt

# Verify
python --version
pip list
```

Every time you return to this directory, pyenv activates `my-new-project` automatically. No manual `source venv/bin/activate` needed.


## pyenv vs Other Tools

| Tool | Manages versions? | Virtual environments? | Per-project pinning? |
|------|-------------------|----------------------|---------------------|
| **pyenv + pyenv-virtualenv** | Yes | Yes | Yes (`.python-version`) |
| **venv** (built-in) | No | Yes | No (manual activation) |
| **conda** | Yes | Yes | Yes (but heavier, aimed at data science) |
| **asdf** | Yes (multi-language) | Via plugin | Yes (`.tool-versions`) |
| **uv** | Yes | Yes | Yes (emerging Rust-based alternative) |

pyenv is the best fit if you want a lightweight, Unix-philosophy tool focused purely on Python version and environment management.

> **Note on uv**: [uv](https://github.com/astral-sh/uv) is a fast, Rust-based Python package manager from Astral (the creators of Ruff). It can manage Python versions, virtual environments, and dependencies in a single tool. It's worth watching as a potential successor to the pyenv + pip workflow.


## Troubleshooting

**`pyenv: command not found`** — Your shell config isn't loaded. Make sure the pyenv init lines are in `~/.zshrc` and run `source ~/.zshrc`.

**Build fails during `pyenv install`** — Install build dependencies:
```bash
brew install openssl readline sqlite3 xz zlib tcl-tk
```

**`python` still points to system Python** — Check that pyenv's shims directory is first in your `PATH`:
```bash
echo $PATH | tr ':' '\n' | head -5
```
The `.pyenv/shims` directory should appear before `/usr/bin`.

**Virtual environment not auto-activating** — Ensure `eval "$(pyenv virtualenv-init -)"` is in your shell config and the `.python-version` file exists in the project root.


## Key Takeaways

1. **Never use the system Python for development** — use pyenv to install and manage your own versions
2. **One virtual environment per project** — keeps dependencies isolated and reproducible
3. **Name virtual environments after the project** — makes it easy to identify what belongs where
4. **Pin with `pyenv local`** — automatic activation when you enter the project directory
5. **Add `.python-version` to `.gitignore`** — it's a local configuration, not a shared one

## References

- [pyenv — GitHub](https://github.com/pyenv/pyenv)
- [pyenv-virtualenv — GitHub](https://github.com/pyenv/pyenv-virtualenv)
- [pyenv Installation Guide](https://github.com/pyenv/pyenv#installation)
- [uv — Astral](https://github.com/astral-sh/uv)
