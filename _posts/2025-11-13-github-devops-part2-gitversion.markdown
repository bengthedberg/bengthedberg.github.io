---
layout: post
title: "GitHub DevOps — Part 2: Semantic Versioning with GitVersion"
date: 2025-11-13 00:00:00 +0000
categories: [DevOps]
tags:
  - github
  - gitversion
  - semver
  - gitflow
  - versioning
  - devops
series: "GitHub DevOps"
series_part: 2
---


## Series Overview

1. [Multiple GitHub Accounts with SSH](/posts/github-devops-part1-multiple-accounts/) — Configure SSH for personal and work accounts
2. **Semantic Versioning with GitVersion** (this article) — Automated versioning using GitFlow
3. [GitHub Actions Workflows](/posts/github-devops-part3-workflows/) — Build, version, and publish NuGet packages


## The Versioning Problem

Versioning software artifacts — assemblies, NuGet packages, npm packages, Docker images — is deceptively hard. Teams often resort to manual version bumping, which leads to forgotten updates, inconsistent versions, and "what version is deployed?" confusion.

[Semantic Versioning](https://semver.org/) (SemVer) provides a standard: `MAJOR.MINOR.PATCH` where:
- **MAJOR** = breaking changes
- **MINOR** = new features (backwards-compatible)
- **PATCH** = bug fixes

But how do you automatically calculate the right version number from your Git history? That's where [GitVersion](https://gitversion.net/) comes in.

## What GitVersion Does

GitVersion analyses your Git history — branches, merges, and tags — and generates a semantic version number automatically. It can:

1. Stamp version numbers on build artifacts (NuGet packages, assemblies)
2. Expose version numbers to CI/CD pipelines
3. Patch `AssemblyInfo.cs` files so binaries contain the version

The version number is **derived from your branching strategy**, not manually maintained.

## Setup

Install GitVersion as a .NET global tool:

```bash
dotnet tool install --global GitVersion.Tool
```

## Walkthrough: GitFlow Versioning

This walkthrough uses the [GitFlow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow) branching strategy. We'll create a sample repository and watch how version numbers change as we work through feature branches, releases, and hotfixes.

### Initialise the Repository

```bash
mkdir gitversion-demo
cd gitversion-demo
git init
```

Run the interactive GitVersion setup:

```bash
dotnet-gitversion init
```

Select these options:
1. **Run getting started wizard** (option 2)
2. **GitFlow (or similar)** (option 1)
3. **Follow SemVer — continuous delivery mode** (option 1)
4. **Save changes and exit** (option 0)

This creates a `GitVersion.yml`:

```yaml
mode: ContinuousDelivery
branches: {}
ignore:
  sha: []
merge-message-formats: {}
```

Commit:

```bash
git add .
git commit -m 'initial commit'
```

### Check the Initial Version

```bash
dotnet-gitversion /showvariable FullSemVer
```

Output: `0.1.0`

GitVersion defaults to `0.1.0` when there are no tags.

### Create the Develop Branch

```bash
git checkout -b develop
```

### Feature Branch: myFeatureA

```bash
git checkout -b feature/myFeatureA
```

Check the version:

```
0.1.0-myFeatureA.1+0
```

The version now includes the feature branch name as a pre-release label. Add a commit:

```bash
git commit -m 'feature A: change 1' --allow-empty
```

Version: `0.1.0-myFeatureA.1+1` — the build metadata increments.

### Merge the Feature

```bash
git checkout develop
git merge --no-ff feature/myFeatureA
git branch -d feature/myFeatureA
```

Version on develop: `0.1.0-alpha.2`

The feature name is replaced with `alpha` — this is the develop branch pre-release label.

### Another Feature Branch

Repeat the process with `feature/myFeatureB`. After merging:

Version on develop: `0.1.0-alpha.4`

The Git graph now looks like:

```
*   fe48f23 (HEAD -> develop) Merge branch 'feature/myFeatureB' into develop
|\
| * 4ad9aa8 feature B: change 1
|/
*   20481b3 Merge branch 'feature/myFeatureA' into develop
|\
| * df2170f feature A: change 1
|/
* 17de78f (master) initial commit
```

### Release Branch

Ready to release:

```bash
git checkout -b release/1.0.0 develop
```

Version: `1.0.0-beta.1+0`

The release branch automatically gets a `beta` pre-release label with the target version number.

Fix a bug on the release branch:

```bash
git commit -m 'bug fix' --allow-empty
```

Version: `1.0.0-beta.1+1`

### Merge to Main and Tag

```bash
git checkout main
git merge --no-ff release/1.0.0
git tag '1.0.0'

git checkout develop
git merge --no-ff release/1.0.0
git branch -d release/1.0.0
```

Now:
- **main**: `1.0.0` (the tag tells GitVersion this is the released version)
- **develop**: `1.1.0-alpha.2` (automatically bumps to the next minor)

### Hotfix Branch

A critical bug in production:

```bash
git checkout main
git checkout -b hotfix/1.0.1
git commit -m 'critical fix' --allow-empty
```

Version: `1.0.1-beta.1+7`

Merge the hotfix:

```bash
git checkout main
git merge --no-ff hotfix/1.0.1
git tag '1.0.1'

git checkout develop
git merge --no-ff hotfix/1.0.1
git branch -d hotfix/1.0.1
```

Main is now at `1.0.1`.

## Version Number Summary

| Branch | Version Format | Example |
|--------|---------------|---------|
| `main` (tagged) | `MAJOR.MINOR.PATCH` | `1.0.0` |
| `develop` | `X.Y.Z-alpha.N` | `1.1.0-alpha.2` |
| `feature/*` | `X.Y.Z-featureName.N+M` | `1.1.0-myFeature.1+3` |
| `release/*` | `X.Y.Z-beta.N+M` | `1.0.0-beta.1+1` |
| `hotfix/*` | `X.Y.Z-beta.N+M` | `1.0.1-beta.1+7` |

## Using in CI/CD

GitVersion integrates directly with GitHub Actions:

```yaml
steps:
  - uses: actions/checkout@v4
    with:
      fetch-depth: 0  # Required — GitVersion needs full history

  - name: Install GitVersion
    uses: gittools/actions/gitversion/setup@v3
    with:
      versionSpec: '6.x'

  - name: Determine Version
    uses: gittools/actions/gitversion/execute@v3
    id: gitversion

  - name: Use version
    run: echo "Version is ${{ steps.gitversion.outputs.semVer }}"
```

> **Critical**: `fetch-depth: 0` is required. Without full Git history, GitVersion cannot calculate the correct version.

## Configuration Options

The `GitVersion.yml` supports many options:

```yaml
mode: ContinuousDelivery
next-version: 2.0.0           # Override the next version
branches:
  main:
    increment: Patch
  develop:
    increment: Minor
  feature:
    increment: Inherit
ignore:
  sha: []
```

See the [full configuration reference](https://gitversion.net/docs/reference/configuration) for all options.

## Key Takeaways

1. **Version numbers should be derived, not maintained.** GitVersion calculates them from your Git history.
2. **Tags drive releases.** Tagging `main` with `1.0.0` tells GitVersion "this is the version" — everything else is pre-release.
3. **Branch names become pre-release labels.** Feature branches get the feature name, develop gets `alpha`, release gets `beta`.
4. **Full Git history is required.** Always use `fetch-depth: 0` in CI/CD.
5. **GitFlow maps naturally to SemVer.** Feature → minor, hotfix → patch, release branch → target version.

## What's Next

In [Part 3: GitHub Actions Workflows](/posts/github-devops-part3-workflows/), we'll use GitVersion inside a GitHub Actions workflow to automatically build, version, and publish NuGet packages to GitHub's package registry.

## References

- [Semantic Versioning](https://semver.org/)
- [GitVersion Documentation](https://gitversion.net/)
- [GitFlow Examples](https://gitversion.net/docs/learn/branching-strategies/gitflow/examples)
- [GitFlow Workflow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow)
