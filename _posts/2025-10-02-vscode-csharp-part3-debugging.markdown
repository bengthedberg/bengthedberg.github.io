---
layout: post
title: "C# Development with VS Code — Part 3: Debugging"
date: 2025-10-02 00:00:00 +0000
tags:
  - csharp
  - vscode
  - dotnet
  - debugging
  - developer-tools
series: "C# Development with VS Code"
series_part: 3
---


## Series Overview

1. [Getting Started](/posts/vscode-csharp-part1-getting-started/) — Installation, UI tour, Git integration
2. [Developing C# Apps](/posts/vscode-csharp-part2-developing/) — Extensions, editing, IntelliSense, NuGet
3. **Debugging** (this article) — Breakpoints, configurations, web app debugging, attach to process
4. [Productivity](/posts/vscode-csharp-part4-productivity/) — Keyboard shortcuts, tasks, workflow optimisation


## Setup

Debugging in VS Code requires the C# Dev Kit extension (see [Part 2](/posts/vscode-csharp-part2-developing/)) and a `.vscode` directory with debug assets.

When you first open a C# project, VS Code prompts you to generate these assets:

![Build and debug prompt](/assets/img/posts/vscode-csharp-part3-debugging/vscode-series/df436f55-679e-4240-b2d3-c9b0912b6d41_01.png)

Click **Yes** to create the `.vscode` directory:

![.vscode directory](/assets/img/posts/vscode-csharp-part3-debugging/vscode-series/34789606-8447-44dc-bd5c-b9a6f1820412_02.png)

If you missed the prompt, use the Command Palette (`Ctrl+Shift+P`) and search for `.NET: Generate Assets for Build and Debug`:

![Generate assets command](/assets/img/posts/vscode-csharp-part3-debugging/vscode-series/bbe6c2b9-c689-48e9-b2ab-826546fb3326_03.png)

## The Debug Pane

Open the Debug pane with `Ctrl+Shift+D` / `Cmd+Shift+D`:

![Debug pane](/assets/img/posts/vscode-csharp-part3-debugging/vscode-series/257cc59c-5e64-49cc-87a7-3ef03cb73dda_04.png)

At the top is a dropdown with configurations:
- **.NET Core Launch (console)** — builds and runs your app with the debugger attached
- **.NET Core Attach** — attaches the debugger to an already-running process

## Catching Exceptions

Click the green **Run** button to start debugging. If your code throws an unhandled exception, the debugger pauses on the exact line:

![Exception caught](/assets/img/posts/vscode-csharp-part3-debugging/vscode-series/3f65a018-f509-485e-a17b-ee33a1e2b855_05.png)

The **VARIABLES** section in the Debug pane lets you inspect the state of all objects:

![Inspecting variables](/assets/img/posts/vscode-csharp-part3-debugging/vscode-series/b9bbad3e-38c8-445c-a333-91960226930e_06.png)

In this example, expanding the `Ticket` object reveals that `Attendee` is `null` — the cause of the `NullReferenceException`. Stop the debugger with the **Stop** button:

![Stop debugger](/assets/img/posts/vscode-csharp-part3-debugging/vscode-series/1014936b-e4eb-4c23-827a-187286c68318_07.png)

## Breakpoints

Breakpoints pause execution at a specific line — no exception needed. Click to the left of any line number to set one (a red circle appears):

![Setting a breakpoint](/assets/img/posts/vscode-csharp-part3-debugging/vscode-series/44fad24f-1184-42cb-8fc4-971660b5c94b_08.png)

Run the app. Execution pauses at the breakpoint and you can inspect all variables:

![Breakpoint hit](/assets/img/posts/vscode-csharp-part3-debugging/vscode-series/6daf320f-153c-4eed-a61c-08611b166f71_09.png)

### Inline Variable Inspection

Hover over any variable in the editor while paused to see its current value — no need to look at the Debug pane:

![Inline inspection](/assets/img/posts/vscode-csharp-part3-debugging/vscode-series/e2da1c45-eba1-454c-987c-2f864b8e1d7f_10.png)

### Continuing Execution

Click **Continue** (`F5`) to resume execution until the next breakpoint or the end of the program:

![Continue button](/assets/img/posts/vscode-csharp-part3-debugging/vscode-series/1e4d50ea-ab2b-4c42-bb43-df7e4ddc925b_11.png)

Output appears in the Debug Console:

![App output](/assets/img/posts/vscode-csharp-part3-debugging/vscode-series/0b693f85-76cd-427e-8ac5-075af116a720_12.png)

### Debug Toolbar Actions

| Button | Shortcut | Action |
|--------|----------|--------|
| Continue | `F5` | Resume execution |
| Step Over | `F10` | Execute current line, skip into methods |
| Step Into | `F11` | Execute current line, enter method calls |
| Step Out | `Shift+F11` | Finish current method, return to caller |
| Restart | `Ctrl+Shift+F5` | Stop and restart debugging |
| Stop | `Shift+F5` | Stop debugging |

## Debug Configurations

The `launch.json` file in `.vscode` defines how the debugger runs your application.

### Passing Command-Line Arguments

If your app reads from `args`, you need to pass arguments through the debug configuration. Edit `.vscode/launch.json` and add values to the `args` array:

```json
{
    "name": ".NET Core Launch (console)",
    "type": "coreclr",
    "request": "launch",
    "program": "${workspaceFolder}/bin/Debug/net10.0/MyApp.dll",
    "args": ["John", "Doe"],
    "cwd": "${workspaceFolder}",
    "stopAtEntry": false,
    "console": "internalConsole"
}
```

![Configuration with args](/assets/img/posts/vscode-csharp-part3-debugging/vscode-series/61969dbd-3c72-41d1-a2c3-05ee8281b7c8_14.png)

Without this, `args` is an empty array and accessing `args[0]` throws an `IndexOutOfRangeException`:

![Empty args](/assets/img/posts/vscode-csharp-part3-debugging/vscode-series/e98ef5d7-cc74-40e5-aee2-478debdb874e_13.png)

### Environment Variables

Add environment variables to your configuration:

```json
{
    "name": ".NET Core Launch (console)",
    "env": {
        "ASPNETCORE_ENVIRONMENT": "Development",
        "MY_API_KEY": "test-key-123"
    }
}
```

## Debugging Web Applications

For ASP.NET Core web apps, the generated configuration automatically handles launching the browser. The first configuration targets web apps instead of console apps.

### Debugging Razor Pages

You can set breakpoints in both C# code-behind files (`.cshtml.cs`) and Razor markup files (`.cshtml`):

**Code-behind breakpoint** — step through your `OnGet` method:

![Stepping through code](/assets/img/posts/vscode-csharp-part3-debugging/vscode-series/07f92467-3abc-4f24-9d8c-ed503f5faa0c_15.png)

**Razor markup breakpoint** — inspect variables directly in the template:

![Debugging markup](/assets/img/posts/vscode-csharp-part3-debugging/vscode-series/a36a6266-5768-40d6-8340-0cee935a8a9a_16.png)

This is powerful — you can debug the Razor template rendering and inspect the values of variables as they're written to HTML.

## Attaching to a Running Process

Sometimes you want to use `dotnet watch run` for hot-reload during development but still have debugging available. The **Attach** configuration handles this.

### Workflow

1. Start your app with `dotnet watch run` in the terminal
2. Select the **.NET Core Attach** configuration in the Debug pane
3. Click **Run** — VS Code shows a list of processes:

![Attach to process](/assets/img/posts/vscode-csharp-part3-debugging/vscode-series/16c94775-7e30-44d0-8883-4d5d9cd89bae_17.png)

4. Search for your project name and select the `dotnet` process running your `.dll`

Now you have the debugger **and** hot-reload working together.

The debug toolbar shows a **Disconnect** button instead of **Stop** — clicking it detaches the debugger but leaves the app running:

![Disconnect button](/assets/img/posts/vscode-csharp-part3-debugging/vscode-series/01e362e7-2ecb-4993-8853-6938e6e7511b_18.png)

## Debugging Tips

### Conditional Breakpoints
Right-click a breakpoint and select **Edit Breakpoint**. Add a condition like `items.Count > 5` — the breakpoint only triggers when the condition is true.

### Logpoints
Instead of a breakpoint, add a **Logpoint** (right-click the gutter > **Add Logpoint**). It logs a message to the Debug Console without pausing execution — useful for tracing without modifying code.

### Watch Expressions
In the **WATCH** section of the Debug pane, add expressions like `user.Username.Length` or `items.Where(x => x.Status == "Active").Count()` to monitor computed values as you step through code.

### Debug Console
The Debug Console (`` Ctrl+Shift+Y ``) lets you evaluate expressions while paused at a breakpoint. Type any valid C# expression to inspect or compute values on the fly.

## Key Takeaways

1. **Generate debug assets early** — accept the prompt or use `.NET: Generate Assets for Build and Debug`.
2. **Use breakpoints, not `Console.WriteLine`** — the Debug pane shows you everything, including nested object properties.
3. **Configure `args` in `launch.json`** for command-line applications.
4. **Attach to `dotnet watch run`** for the best of both worlds: hot-reload and debugging.
5. **Learn the toolbar shortcuts** — `F5` (continue), `F10` (step over), `F11` (step into) keep you in flow.

## What's Next

In [Part 4: Productivity](/posts/vscode-csharp-part4-productivity/), we'll cover the keyboard shortcuts and task configurations that make VS Code a productivity powerhouse for C# development.
