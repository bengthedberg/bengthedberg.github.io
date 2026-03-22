---
layout: post
title: "C# Development with VS Code — Part 2: Developing C# Apps"
date: 2025-09-18 00:00:00 +0000
tags:
  - csharp
  - vscode
  - dotnet
  - developer-tools
  - intellisense
series: "C# Development with VS Code"
series_part: 2
---


## Series Overview

1. [Getting Started](/posts/vscode-csharp-part1-getting-started/) — Installation, UI tour, Git integration
2. **Developing C# Apps** (this article) — Extensions, editing, IntelliSense, NuGet packages
3. [Debugging](/posts/vscode-csharp-part3-debugging/) — Breakpoints, configurations, attach to process
4. [Productivity](/posts/vscode-csharp-part4-productivity/) — Keyboard shortcuts, tasks, workflow optimisation


## Essential Extensions

### C# Dev Kit

The cornerstone of C# development in VS Code is the **C# Dev Kit** extension from Microsoft. It provides:

- IntelliSense and code completion
- Code navigation (Go to Definition, Find References)
- Refactoring support
- Solution Explorer
- Test Explorer integration
- Build and debug assets

Open the Extensions pane (`Ctrl+Shift+X` / `Cmd+Shift+X`), search for "C# Dev Kit", and install it:

![C# extension](/assets/img/posts/vscode-csharp-part2-developing/vscode-series/25a4836a-e4f6-4c1b-9f61-aca0464e8402_2020-05-23_22-28-31.png)

> **Note**: The original "C#" extension (OmniSharp) is still available but C# Dev Kit is now Microsoft's recommended extension. It includes the C# extension as a dependency.

### C# Extensions

Also install **C# Extensions** by JosKreativ. This adds shortcuts for scaffolding new classes and interfaces via right-click in the Explorer — a small but significant time-saver.

## The Terminal-First Workflow

VS Code doesn't have a "File > New Project" wizard. Instead, .NET development relies on the `dotnet` CLI through the integrated terminal (`` Ctrl+` ``).

Create a new console project:

```bash
dotnet new console -o GuideDemoApp
```

Open it in the current VS Code window:

```bash
code -r GuideDemoApp
```

When VS Code detects a C# project, it prompts you to add build and debug assets:

![Build assets prompt](/assets/img/posts/vscode-csharp-part2-developing/vscode-series/f4d9844b-8259-4390-bbb9-6c387865dfd3_01.png)

Click **Yes** — this creates a `.vscode` directory with `launch.json` and `tasks.json` that enable debugging. If you miss the prompt, open the Command Palette and search for `.NET: Generate Assets for Build and Debug`.

The Debug pane (`Ctrl+Shift+D` / `Cmd+Shift+D`) now shows launch configurations:

![Debug pane](/assets/img/posts/vscode-csharp-part2-developing/vscode-series/9e461036-135e-4852-990d-d4104e783b41_02.png)

## Editing C# Code

### Creating Classes

Right-click in the Explorer and select **New C# Class** (from the C# Extensions extension). Enter the filename — the class is scaffolded with the correct namespace automatically.

### Code Snippets

Type snippet shortcuts and press `Tab` to expand them. The most useful ones:

| Snippet | Expands To |
|---------|-----------|
| `prop` | Auto-implemented property |
| `propfull` | Property with backing field |
| `ctor` | Constructor |
| `class` | Class declaration |
| `interface` | Interface declaration |
| `for`, `foreach` | Loop structures |
| `try` | Try/catch block |
| `cw` | `Console.WriteLine()` |

For example, type `prop` and press `Tab`:

![prop snippet](/assets/img/posts/vscode-csharp-part2-developing/vscode-series/6566c151-29b1-40c0-8c76-b03f7d164950_08.png)

The type is highlighted — start typing `string` to change it. Press `Tab` again to jump to the property name.

### IntelliSense

Just like Visual Studio, VS Code shows context-aware completions after typing a dot:

![IntelliSense](/assets/img/posts/vscode-csharp-part2-developing/vscode-series/7fb6a806-cf2b-4e39-aa5a-cce4539644f9_03.png)

**Hover** over any symbol to see its definition. For keyboard access: `Ctrl+K Ctrl+I` / `Cmd+K Cmd+I`.

### Quick Fixes

When VS Code underlines a problem with a wavy line, place your cursor on it and press `Ctrl+.` / `Cmd+.` to see available fixes:

![Quick fix - add using](/assets/img/posts/vscode-csharp-part2-developing/vscode-series/87719b00-7020-4ab4-b362-99958a33d966_07.png)

Common quick fixes include:
- Adding missing `using` statements
- Removing unused usings
- Generating constructor parameters
- Implementing interface members
- Extracting methods or variables

### Quick File Navigation

Press `Ctrl+P` / `Cmd+P` and start typing any filename. VS Code fuzzy-matches across the entire project:

![Quick file open](/assets/img/posts/vscode-csharp-part2-developing/vscode-series/dfe4c881-594b-4b3f-91e7-45cab52aba29_2020-05-23_22-34-04.png)

## Working with NuGet Packages

### Via the Terminal

The most reliable way to add NuGet packages:

```bash
dotnet add package Bogus
```

### Via the NuGet Gallery Extension

For a visual experience, install the **NuGet Gallery** extension. Open it via Command Palette > `NuGet: Open Gallery`:

![NuGet Gallery](/assets/img/posts/vscode-csharp-part2-developing/vscode-series/874aa4ac-02ad-4f50-8102-9e145a0bd54d_06.png)

Search for packages, view details, and click **Install** next to the target `.csproj` file.

## A Complete Example

Let's put it together. Create a `DemoUser` class with the **New C# Class** shortcut, then add properties using the `prop` snippet:

```csharp
namespace GuideDemoApp;

public class DemoUser
{
    public string Username { get; set; } = string.Empty;
    public string Password { get; set; } = string.Empty;

    public override string ToString()
        => $"<User: {Username} / {Password}>";
}
```

In `Program.cs`, use IntelliSense to write the main logic. Use `Ctrl+.` to auto-import `Bogus` when using the `Randomizer` class:

```csharp
using Bogus;

var bogusPerson = new Person();
var user = new DemoUser
{
    Username = (bogusPerson.FirstName[0] + bogusPerson.LastName).ToLower(),
    Password = GeneratePassword()
};

Console.WriteLine(user);

static string GeneratePassword(int numChars = 4, int numDigits = 4)
{
    var randomizer = new Randomizer();
    var letters = randomizer.String(numChars, numChars, 'A', 'Z');
    var digits = randomizer.String(numDigits, numDigits, '0', '9');
    return $"{letters}-{digits}";
}
```

Run it:

```bash
dotnet run
```

![Generated user output](/assets/img/posts/vscode-csharp-part2-developing/vscode-series/548138f6-14d4-429b-9a8f-06d4b3ccb72f_2020-05-23_22-21-41.png)

## Key Takeaways

1. **Install C# Dev Kit** — it's the foundation for everything: IntelliSense, debugging, testing, navigation.
2. **Learn the `dotnet` CLI** — project creation, package management, and builds all happen through the terminal.
3. **Use code snippets** — `prop`, `ctor`, `cw` and others save significant typing.
4. **Use `Ctrl+.` constantly** — quick fixes for missing usings, interface implementation, and refactoring.
5. **Use `Ctrl+P` for file navigation** — faster than clicking through the Explorer tree.

## What's Next

In [Part 3: Debugging](/posts/vscode-csharp-part3-debugging/), we'll explore breakpoints, variable inspection, debug configurations, command-line arguments, web app debugging, and attaching to running processes.
