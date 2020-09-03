---
layout: post
title:  "Visual Studio Tips and Extensions"
date:   2020-01-15 07:21:47 +1000
categories: Visual Studio 
tags:
- Productivity
---

These are the extensions I use in Visual Studio as well as common short cuts, (that I keep forgetting).

## Visual Studio Shortcuts

| Shortcut (All Profiles)                                      | Command                              | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------- | :----------------------------------------------------------- |
| **Ctrl**+**T**                                               | Go To All                            | Navigate to any file, type, member, or symbol declaration    |
| **F12** (also **Ctrl**+**Click**)                            | Go To Definition                     | Navigate to where a symbol is defined                        |
| **Ctrl**+**F12**                                             | Go To Implementation                 | Navigate from a base type or member to its various implementations |
| **Shift**+**F12**                                            | Find All References                  | See all symbol or literal references                         |
| **Alt**+**Home**                                             | Go To Base                           | Navigate up the inheritance chain                            |
| **Ctrl+.**                                                   | Quick Actions and Refactoring        | See what code fixes, code generation actions, refactorings, or other quick actions are available at your cursor position or code selection |
| **Ctrl**+**D**                                               | Duplicate line                       | Duplicates the line of code that the cursor is in (available in **Visual Studio 2017 version 15.6** and later) |
| **Shift**+**Alt**+**+**/**-**                                | Expand/Contract selection            | Expands or contracts the current selection in the editor (available in **Visual Studio 2017 version 15.5** and later) |
| **Shift** + **Alt** + **.**                                  | Insert Next Matching Caret           | Adds a selection and caret at the next location that matches the current selection (available in **Visual Studio 2017 version 15.8** and later) |
| **Ctrl**+**Q**                                               | Search                               | Search all Visual Studio settings                            |
| **F5**                                                       | Start Debugging                      | Start debugging your application                             |
| **Ctrl**+**F5**                                              | Run without Debug                    | Run your application locally without debugging               |
| **Ctrl**+**K**,**D** (Default Profile) or **Ctrl**+**E**,**D** (C# Profile) | Format Document                      | Cleans up formatting violations in your file based on your newline, spacing, and indentation settings |
| **Ctrl**+**\**,**Ctrl**+**E** (Default Profile) or **Ctrl**+**W**,**E** (C# Profile) | View Error List                      | See all errors in your document, project, or solution        |
| **Alt** + **PgUp/PgDn**                                      | Go to Next/Previous Issue            | Jump to the previous/next error, warning, suggestion in your document (available in **Visual Studio 2017 version 15.8** and later) |
| **Ctrl**+**K**,**/**                                         | Toggle single line comment/uncomment | This command adds or removes a single line comment depending on whether your selection is already commented |
| **Ctrl**+**Shift**+**/**                                     | Toggle block comment/uncomment       | This command adds or removes block comments depending on what you have selected |



#### Windows Navigation

| Shortcut (All Profiles) | Command               | Description                               |
| :---------------------- | :-------------------- | :---------------------------------------- |
| **Alt+Shift+Enter**     | Zen Mode              | Windows open in full screen               |
| **Ctrl+Tab**            | Show All Open Windows | Opens a dialog, showing all open windows. |
| **Ctrl+F6**             | Go To Next Window     | Navigate to next window                   |
| **Ctrl+Shift+F6**       | Go To Previous Window | Navigate to previous window               |
| **Ctrl+F4**             | Close current window  | Close the current window                  |
| **Alt**+W+L             | Close all windows     | Close all windows that are opened         |


## Extensions

#### Productivity Power Tools extension

The [Productivity Power Tools extension](https://marketplace.visualstudio.com/items?itemName=VisualStudioProductTeam.ProductivityPowerPack2017)  is now a collection of 15 other extensions, making it easier to manage and update the child extensions with new features without having to re-install the entire Productivity Power Tools collection

Tip: One install you can enforce code formatting when saving document. Go to Tools->Options and select the Productivity Power Tool section.



#### Visual Studio Spell Checker 

[Visual Studio Spell Checker](https://marketplace.visualstudio.com/items?itemName=EWoodruff.VisualStudioSpellCheckerVS2017andLater). An editor extension that checks the spelling of comments, strings, and plain text as you type or interactively with a tool window. It can also spell check an entire solution, project, or selected items. Options are available to define multiple languages to spell check against.



#### Git Diff Margin

[Git Diff Margin](https://marketplace.visualstudio.com/items?itemName=LaurentKempe.GitDiffMargin). Git Diff Margin displays live Git changes of the currently edited file on Visual Studio margin and scroll bar.



#### EditorConfig Language Service

Create code style preferences in an .editorconfig file, and the [EditorConfig Language Service](https://marketplace.visualstudio.com/items?itemName=MadsKristensen.EditorConfig) is the extension that makes writing those files a breeze.

[The EditorConfig Project](http://editorconfig.org/) helps developers define and maintain consistent coding styles between different editors, but it doesn't give language support for editing those files. This extension provides that capability.



#### Web Essentials

No list of extensions for Visual Studio would be complete without mentioning [Web Essentials](https://marketplace.visualstudio.com/items?itemName=MadsKristensen.WebExtensionPack2017). Like the Productivity Power Tools, Web Essentials is now a collection of 25 child extensions that can be updated and maintained separately.



#### GitHub Extensions

GitHub Extension for Visual StudioThe [GitHub Extension for Visual Studio](https://visualstudio.github.com/) makes it easy to connect to and work with your repositories on [GitHub](https://github.com/) and [GitHub Enterprise](https://enterprise.github.com/) from directly within Visual Studio 2015 or newer. Clone existing repositories or create new ones and start collaborating!





## Resources

[Tips and Tricks](https://code.visualstudio.com/docs/getstarted/tips-and-tricks)

[Visual Studio 2019 Productivity](https://devblogs.microsoft.com/dotnet/visual-studio-2019-net-productivity-2/)

[Visual Studio Tips and Tricks](https://devblogs.microsoft.com/visualstudio/visual-studio-tips-and-tricks/)