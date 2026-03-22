---
layout: post
title: "GitHub DevOps — Part 3: Build and Publish NuGet Packages with GitHub Actions"
date: 2025-11-27 00:00:00 +0000
categories: [DevOps]
tags:
  - github
  - github-actions
  - nuget
  - cicd
  - gitversion
  - devops
series: "GitHub DevOps"
series_part: 3
---


## Series Overview

1. [Multiple GitHub Accounts with SSH](/posts/github-devops-part1-multiple-accounts/) — Configure SSH for personal and work accounts
2. [Semantic Versioning with GitVersion](/posts/github-devops-part2-gitversion/) — Automated versioning using GitFlow
3. **GitHub Actions Workflows** (this article) — Build, version, and publish NuGet packages


## What We're Building

In this article, we'll create a complete CI/CD pipeline that:

1. Builds a .NET class library on every push
2. Calculates the version number automatically using GitVersion (from [Part 2](/posts/github-devops-part2-gitversion/))
3. Packs it as a NuGet package
4. Publishes it to GitHub's NuGet package registry
5. Creates a GitHub Release with the package attached

The entire pipeline runs on GitHub Actions with no manual version management.

## Step 1: Create the Project

```bash
mkdir Greetings.Nuget.Demo
cd Greetings.Nuget.Demo
git init --initial-branch=main
dotnet new classlib -f net10.0 -o src/Greetings.Nuget.Demo
rm src/Greetings.Nuget.Demo/Class1.cs
dotnet new gitignore
dotnet new sln
dotnet sln add $(find . -name "*.csproj")
```

## Step 2: Add the Library Code

Create `src/Greetings.Nuget.Demo/Greetings.cs`:

```csharp
namespace Greetings.Nuget.Demo;

public class Greetings
{
    public void SayHello() => Console.WriteLine("Hello from Greetings.Nuget.Demo");
    public void SayGoodMorning() => Console.WriteLine("Good morning from Greetings.Nuget.Demo");
    public void SayGreetings() => Console.WriteLine("Greetings from Greetings.Nuget.Demo");
}
```

## Step 3: Configure NuGet Package Metadata

Update `src/Greetings.Nuget.Demo/Greetings.Nuget.Demo.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>

    <!-- NuGet metadata -->
    <PackageId>Greetings.Nuget.Demo</PackageId>
    <Authors>Your Name</Authors>
    <Company>Your Company</Company>
    <Description>A demo NuGet package for greeting messages</Description>
    <RepositoryUrl>https://github.com/your-org/Greetings.Nuget.Demo.git</RepositoryUrl>
    <PackageTags>Demo;NuGet;Greetings</PackageTags>
    <GeneratePackageOnBuild>false</GeneratePackageOnBuild>
  </PropertyGroup>

</Project>
```

> Update `Authors`, `Company`, and `RepositoryUrl` to match your GitHub account.

## Step 4: Set Up GitVersion

Initialise GitVersion with GitFlow and continuous delivery mode (see [Part 2](/posts/github-devops-part2-gitversion/) for details):

```bash
dotnet-gitversion init
```

Select: **Getting started wizard** → **GitFlow** → **Continuous delivery mode** → **Save and exit**.

This creates `GitVersion.yml`:

```yaml
mode: ContinuousDelivery
branches: {}
ignore:
  sha: []
merge-message-formats: {}
```

## Step 5: Create the GitHub Actions Workflow

Create `.github/workflows/build-and-publish.yml`:

```yaml
name: Build & Publish NuGet

on:
  push:
    branches: [main]
    paths-ignore:
      - "**/*.md"

env:
  PROJECT_PATH: "src/Greetings.Nuget.Demo/"
  NUGET_REGISTRY: "https://nuget.pkg.github.com/${{ github.repository_owner }}/index.json"

permissions:
  contents: write
  packages: write

jobs:
  build:
    name: Build & Version
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.gitversion.outputs.semVer }}
      commits: ${{ steps.gitversion.outputs.commitsSinceVersionSource }}

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history required for GitVersion

      - name: Install GitVersion
        uses: gittools/actions/gitversion/setup@v3
        with:
          versionSpec: '6.x'

      - name: Calculate version
        uses: gittools/actions/gitversion/execute@v3
        id: gitversion

      - name: Display version
        run: |
          echo "SemVer: ${{ steps.gitversion.outputs.semVer }}"
          echo "Commits since last version: ${{ steps.gitversion.outputs.commitsSinceVersionSource }}"

      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 10.0.x

      - name: Build and pack
        run: >
          dotnet pack ${{ env.PROJECT_PATH }}
          -p:Version='${{ steps.gitversion.outputs.semVer }}'
          -c Release

      - name: Upload package artifact
        uses: actions/upload-artifact@v4
        with:
          name: nuget-package
          path: ${{ env.PROJECT_PATH }}bin/Release/*.nupkg

  release:
    name: Publish & Release
    runs-on: ubuntu-latest
    needs: build
    if: needs.build.outputs.commits > 0

    steps:
      - name: Download package
        uses: actions/download-artifact@v4
        with:
          name: nuget-package
          path: package

      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 10.0.x

      - name: Add GitHub NuGet source
        run: >
          dotnet nuget add source
          --username ${{ github.repository_owner }}
          --password ${{ secrets.GITHUB_TOKEN }}
          --store-password-in-clear-text
          --name github
          "${{ env.NUGET_REGISTRY }}"

      - name: Push to GitHub Packages
        run: >
          dotnet nuget push package/*.nupkg
          --api-key ${{ secrets.GITHUB_TOKEN }}
          --source github

      - name: Create GitHub Release
        uses: ncipollo/release-action@v1
        with:
          tag: ${{ needs.build.outputs.version }}
          name: Release ${{ needs.build.outputs.version }}
          artifacts: "package/*"
          token: ${{ secrets.GITHUB_TOKEN }}
```

## Understanding the Workflow

### Two Jobs: Build and Release

The workflow is split into two jobs for clarity and safety:

**Build job:**
1. Checks out the full Git history (`fetch-depth: 0` — required for GitVersion)
2. Installs and runs GitVersion to calculate the version
3. Builds and packs the NuGet package with the calculated version
4. Uploads the `.nupkg` as an artifact

**Release job:**
1. Only runs if there are new commits (`commitsSinceVersionSource > 0`)
2. Downloads the package artifact
3. Pushes it to GitHub's NuGet package registry
4. Creates a tagged GitHub Release with the package attached

### Authentication with GITHUB_TOKEN

The workflow uses `secrets.GITHUB_TOKEN` — a token that GitHub automatically creates for each workflow run. It requires the `packages: write` and `contents: write` permissions, declared at the top of the workflow.

> **Troubleshooting**: If the token fails to push the package, create a **Personal Access Token (classic)** with the `write:packages` scope. Add it as a repository secret (e.g., `NUGET_TOKEN`) and replace `${{ secrets.GITHUB_TOKEN }}` with `${{ secrets.NUGET_TOKEN }}`.

### Version Flow

The version number flows through the pipeline:

```
Git history → GitVersion → dotnet pack -p:Version=X.Y.Z → NuGet push → GitHub Release tag
```

No manual version bumping anywhere.

## Step 6: Commit and Push

```bash
git add .
git commit -m 'initial setup with CI/CD'
git push -u origin main
```

The workflow triggers automatically. Check the **Actions** tab in your repository to see it run.

## Consuming the Package

Once published, other projects can consume your package from GitHub's registry.

### Add the GitHub NuGet Source

```bash
dotnet nuget add source \
  --username YOUR_USERNAME \
  --password YOUR_GITHUB_TOKEN \
  --store-password-in-clear-text \
  --name github \
  "https://nuget.pkg.github.com/YOUR_ORG/index.json"
```

### Install the Package

```bash
dotnet add package Greetings.Nuget.Demo
```

### Use in a NuGet.config File

For team projects, add a `NuGet.config` at the solution root:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
    <add key="github" value="https://nuget.pkg.github.com/YOUR_ORG/index.json" />
  </packageSources>
  <packageSourceCredentials>
    <github>
      <add key="Username" value="YOUR_USERNAME" />
      <add key="ClearTextPassword" value="%GITHUB_TOKEN%" />
    </github>
  </packageSourceCredentials>
</configuration>
```

## Extending the Workflow

### Add Tests

Insert a test step before packing:

```yaml
- name: Run tests
  run: dotnet test --configuration Release --no-build
```

### Publish to nuget.org

Add a step to push to the public NuGet registry:

```yaml
- name: Push to nuget.org
  run: >
    dotnet nuget push package/*.nupkg
    --api-key ${{ secrets.NUGET_API_KEY }}
    --source https://api.nuget.org/v3/index.json
```

### Build on Pull Requests

Add PR triggers for validation without publishing:

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
```

Then gate the release job:

```yaml
if: github.ref == 'refs/heads/main' && needs.build.outputs.commits > 0
```

## Key Takeaways

1. **GitVersion + GitHub Actions = automatic versioning.** No manual version bumping, ever.
2. **`fetch-depth: 0` is non-negotiable.** GitVersion needs the full Git history to calculate versions correctly.
3. **Split build and release into separate jobs.** Build on every push, release only when there are new commits on main.
4. **`GITHUB_TOKEN` handles authentication.** Declare the required permissions in the workflow file.
5. **The version flows end-to-end.** From Git history → calculated version → package version → release tag.

## Series Conclusion

Over this 3-part series we've built a complete GitHub DevOps foundation:

1. **Multiple accounts** — SSH aliases for seamless multi-account workflows
2. **Automatic versioning** — GitVersion derives version numbers from your branching strategy
3. **CI/CD pipelines** — GitHub Actions builds, versions, and publishes packages with zero manual intervention

Together, these form the backbone of a professional .NET development workflow on GitHub.

## References

- [GitHub Actions Documentation](https://docs.github.com/en/actions/writing-workflows)
- [GitVersion in GitHub Actions](https://github.com/GitTools/actions)
- [GitHub Packages — NuGet Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-nuget-registry)
- [GITHUB_TOKEN Authentication](https://docs.github.com/en/actions/security-for-github-actions/security-guides/automatic-token-authentication)
