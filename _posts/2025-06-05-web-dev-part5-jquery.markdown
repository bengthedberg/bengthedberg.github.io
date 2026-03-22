---
layout: post
title: "Web Development Fundamentals — Part 5: jQuery — Write Less, Do More"
date: 2025-06-05 00:00:00 +0000
categories: [Web Development]
tags:
  - web
  - javascript
  - jquery
  - frontend
  - beginner
series: "Web Development Fundamentals"
series_part: 5
---


## Series Overview

This is an 8-part series covering the core technologies of web development:

1. [HTML — The Structure of the Web](/posts/web-dev-part1-html/) — Elements, attributes, block vs inline, and document structure
2. [CSS — Styling the Web](/posts/web-dev-part2-css/) — Selectors, properties, the box model, and layout basics
3. [JavaScript — Adding Interactivity](/posts/web-dev-part3-javascript/) — Variables, data types, functions, arrays, and control flow
4. [The DOM — Connecting JavaScript to HTML](/posts/web-dev-part4-dom/) — Selecting elements, changing styles, and modifying attributes
5. **jQuery — Write Less, Do More** (this article) — Selectors, events, and DOM manipulation with jQuery
6. [Bootstrap — Responsive Layouts Made Easy](/posts/web-dev-part6-bootstrap/) — Grid system, responsive breakpoints, and rapid prototyping
7. [HTTP — How the Web Communicates](/posts/web-dev-part7-http/) — Requests, responses, methods, and status codes
8. [React — Building Modern User Interfaces](/posts/web-dev-part8-react/) — Components, JSX, props, state, hooks, and thinking in React


## What Is jQuery?

jQuery is a fast, small, and feature-rich JavaScript library. It simplifies HTML document traversal and manipulation, event handling, and animation with an easy-to-use API that works across browsers.

Its motto is **"Write less, do more"** — and it delivers. What takes several lines of vanilla DOM code often takes one line with jQuery.

## Setup

### Download and Include

Download jQuery from [jquery.com/download](https://jquery.com/download/) and include it before your scripts:

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>jQuery Example</title>
    <link rel="stylesheet" href="css/style.css">
    <script src="js/jquery-3.x.min.js"></script>
</head>
<body>
    <h1>Hello, World!</h1>
    <script src="js/app.js"></script>
</body>
</html>
```

Always include jQuery **before** your custom scripts — otherwise jQuery's API won't be available when your code runs.

Use the **minified** version for production (smaller file, faster load) and the uncompressed version for development (readable, easier to debug).

## Basic Syntax

Every jQuery statement follows this pattern:

```javascript
$(selector).action();
```

- `$` is an alias for `jQuery`
- `selector` finds HTML elements (uses CSS selector syntax)
- `action()` performs something on the selected elements

### Document Ready

Always wrap your code in the document ready handler to ensure the DOM is fully loaded:

```javascript
$(document).ready(function() {
    // Your code here — DOM is ready
    alert("Hello World!");
});
```

## Selectors

jQuery uses CSS selector syntax, making it instantly familiar.

### By ID

```javascript
$("#mark").css("background", "yellow");
```

### By Class

```javascript
$(".mark").css("background", "yellow");
```

### By Element

```javascript
$("p").css("background", "yellow");
```

### By Attribute

```javascript
$('input[type="text"]').css("background", "yellow");
```

### Compound Selectors

Combine selectors for precise targeting:

```javascript
// Only <p> elements with class "mark"
$("p.mark").css("background", "yellow");

// <span> elements inside #mark
$("#mark span").css("background", "yellow");

// <li> elements inside <ul>
$("ul li").css("background", "red");

// <li> inside <ul> with id "mark"
$("ul#mark li").css("background", "yellow");

// Anchors with target="_blank"
$('a[target="_blank"]').css("background", "yellow");
```

### Custom jQuery Selectors

jQuery extends CSS selectors with its own pseudo-selectors:

```javascript
// Odd and even table rows
$("tr:odd").css("background", "yellow");
$("tr:even").css("background", "orange");

// First and last paragraph
$("p:first").css("background", "red");
$("p:last").css("background", "green");

// Form input types
$("form :text").css("background", "purple");
$("form :password").css("background", "blue");
$("form :submit").css("background", "violet");
```

## Events

Events respond to user interactions — clicks, key presses, form changes, and more.

### Mouse Events

```javascript
$(document).ready(function() {
    // Click
    $("button").click(function() {
        alert("Button clicked!");
    });

    // Double click
    $("p").dblclick(function() {
        $(this).css("color", "red");
    });

    // Hover (mouseenter + mouseleave)
    $("p").hover(
        function() {
            $(this).addClass("highlight");
        },
        function() {
            $(this).removeClass("highlight");
        }
    );
});
```

### Keyboard Events

```javascript
// keypress — fires when a character key is pressed
$("input").keypress(function() {
    console.log("Key pressed");
});

// keydown — fires when any key is first pressed
$("input").keydown(function() {
    console.log("Key down");
});

// keyup — fires when a key is released
$("input").keyup(function() {
    console.log("Key up");
});
```

### Form Events

```javascript
// Change — select box, checkbox, or radio changes
$("select").change(function() {
    var selected = $(this).find(":selected").val();
    alert("You selected: " + selected);
});

// Focus — element gains focus
$("input").focus(function() {
    $(this).css("border-color", "blue");
});

// Blur — element loses focus
$("input").blur(function() {
    $(this).css("border-color", "");
});

// Submit — form submission
$("form").submit(function() {
    alert("Form submitted!");
});
```

### Document and Window Events

```javascript
// Page loaded
$(document).ready(function() {
    console.log("DOM ready");
});

// Window resized
$(window).resize(function() {
    console.log("Window width: " + $(window).width());
});

// Page scrolled
$(window).scroll(function() {
    console.log("Scroll position: " + $(window).scrollTop());
});
```

## jQuery vs Vanilla JavaScript

Here's a comparison to show what jQuery simplifies:

| Task | Vanilla JS | jQuery |
|------|-----------|--------|
| Select by ID | `document.getElementById("x")` | `$("#x")` |
| Select by class | `document.querySelectorAll(".x")` | `$(".x")` |
| Set style | `elem.style.color = "red"` | `$(".x").css("color", "red")` |
| Add class | `elem.classList.add("active")` | `$(".x").addClass("active")` |
| Click handler | `elem.addEventListener("click", fn)` | `$(".x").click(fn)` |
| Ready event | `document.addEventListener("DOMContentLoaded", fn)` | `$(document).ready(fn)` |

jQuery also automatically handles iteration — `$("p").css("color", "red")` applies to **all** matching paragraphs, while vanilla JS requires a loop.

## What's Next?

jQuery makes DOM manipulation concise, but building responsive layouts from scratch is still tedious. In [Part 6](/posts/web-dev-part6-bootstrap/), we'll use **Bootstrap** — a CSS framework that gives you a responsive grid system and pre-built components out of the box.

## References

- [jQuery Official Site](https://jquery.com/)
- [jQuery API Documentation](https://api.jquery.com/)
- [jQuery Tutorial](https://www.tutorialrepublic.com/jquery-tutorial/)
