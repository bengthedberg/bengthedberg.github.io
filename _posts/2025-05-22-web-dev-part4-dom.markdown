---
layout: post
title: "Web Development Fundamentals — Part 4: The DOM — Connecting JavaScript to HTML"
date: 2025-05-22 00:00:00 +0000
categories: [Web Development]
tags:
  - web
  - javascript
  - dom
  - frontend
  - beginner
series: "Web Development Fundamentals"
series_part: 4
---


## Series Overview

This is an 8-part series covering the core technologies of web development:

1. [HTML — The Structure of the Web](/posts/web-dev-part1-html/) — Elements, attributes, block vs inline, and document structure
2. [CSS — Styling the Web](/posts/web-dev-part2-css/) — Selectors, properties, the box model, and layout basics
3. [JavaScript — Adding Interactivity](/posts/web-dev-part3-javascript/) — Variables, data types, functions, arrays, and control flow
4. **The DOM — Connecting JavaScript to HTML** (this article) — Selecting elements, changing styles, and modifying attributes
5. [jQuery — Write Less, Do More](/posts/web-dev-part5-jquery/) — Selectors, events, and DOM manipulation with jQuery
6. [Bootstrap — Responsive Layouts Made Easy](/posts/web-dev-part6-bootstrap/) — Grid system, responsive breakpoints, and rapid prototyping
7. [HTTP — How the Web Communicates](/posts/web-dev-part7-http/) — Requests, responses, methods, and status codes
8. [React — Building Modern User Interfaces](/posts/web-dev-part8-react/) — Components, JSX, props, state, hooks, and thinking in React


## What Is the DOM?

The **Document Object Model (DOM)** is a programming interface for HTML and XML documents. It represents the page as a tree of nodes and objects so that programming languages like JavaScript can connect to the page and change its structure, style, and content.

You can explore the DOM of any page in your browser's developer console:

```javascript
console.dir(document);
```

This shows the entire document object with all its properties and methods.

## Selecting Elements

Before you can change anything on the page, you need to select the element you want to modify. The DOM provides several methods for this.

### By ID

Select a single element by its unique `id`:

```javascript
var element = document.getElementById("intro");
```

### By Class Name

Select all elements with a given class (returns a live HTMLCollection):

```javascript
var elements = document.getElementsByClassName("highlight");
```

### By Tag Name

Select all elements of a given type:

```javascript
var paragraphs = document.getElementsByTagName("p");
```

### By CSS Selector (most flexible)

Use `querySelector` for the first match and `querySelectorAll` for all matches:

```javascript
// First element matching the selector
var first = document.querySelector(".card > h2");

// All elements matching the selector
var all = document.querySelectorAll("ul.menu li");
```

`querySelectorAll` uses the same selector syntax as CSS, making it the most powerful and flexible option.

## Styling Elements

The `style` property lets you get and set inline styles on an element. CSS property names are converted to camelCase in JavaScript (`font-size` becomes `fontSize`):

```javascript
var elem = document.getElementById("intro");

// Set styles
elem.style.color = "blue";
elem.style.fontSize = "18px";
elem.style.fontWeight = "bold";

// Get styles
alert(elem.style.color);     // "blue"
alert(elem.style.fontSize);  // "18px"
```

Note that the `style` property only reads **inline styles** — it won't return styles applied via external stylesheets or `<style>` tags. For computed styles, use `window.getComputedStyle(elem)`.

## Working with CSS Classes

Manipulating classes is usually better than setting individual style properties. The `classList` property provides a clean API:

```javascript
var elem = document.getElementById("info");

// Add classes
elem.classList.add("hide");
elem.classList.add("note", "highlight");  // Multiple at once

// Remove classes
elem.classList.remove("hide");
elem.classList.remove("disabled", "note");

// Toggle — adds if absent, removes if present
elem.classList.toggle("visible");

// Check if a class exists
if (elem.classList.contains("highlight")) {
    alert("The element is highlighted.");
}
```

This approach keeps your styling in CSS where it belongs and uses JavaScript only to switch classes on and off.

## Getting and Setting Attributes

### Reading Attributes

Use `getAttribute()` to read any attribute value:

```javascript
var link = document.getElementById("myLink");

var href = link.getAttribute("href");
alert(href);    // "https://www.google.com/"

var target = link.getAttribute("target");
alert(target);  // "_blank"
```

If the attribute doesn't exist, `getAttribute()` returns `null`.

### Setting Attributes

Use `setAttribute()` to add or change an attribute:

```javascript
var btn = document.getElementById("myBtn");

btn.setAttribute("class", "click-btn");
btn.setAttribute("disabled", "");
```

### Removing Attributes

Use `removeAttribute()` to completely remove an attribute:

```javascript
var link = document.getElementById("myLink");
link.removeAttribute("href");
```

## Putting It All Together

Here's a practical example that combines selection, styling, and attribute manipulation:

```html
<button id="toggleBtn">Toggle Panel</button>
<div id="panel" class="panel">
    <p>This is the panel content.</p>
</div>

<script>
var btn = document.getElementById("toggleBtn");
var panel = document.getElementById("panel");

btn.onclick = function() {
    panel.classList.toggle("hidden");

    if (panel.classList.contains("hidden")) {
        btn.setAttribute("aria-expanded", "false");
    } else {
        btn.setAttribute("aria-expanded", "true");
    }
};
</script>
```

This pattern — select elements, attach event handlers, modify classes and attributes — is the foundation of interactive web pages.

## What's Next?

The DOM API is powerful but verbose. Writing `document.getElementById()` and `document.querySelectorAll()` repeatedly gets tedious. In [Part 5](/posts/web-dev-part5-jquery/), we'll introduce **jQuery**, a library that wraps the DOM API in a concise, chainable syntax.

## References

- [MDN — Document Object Model](https://developer.mozilla.org/en-US/docs/Web/API/Document_Object_Model/Introduction)
- [MDN — Element.classList](https://developer.mozilla.org/en-US/docs/Web/API/Element/classList)
- [MDN — querySelectorAll](https://developer.mozilla.org/en-US/docs/Web/API/Document/querySelectorAll)
