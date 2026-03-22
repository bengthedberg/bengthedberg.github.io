---
layout: post
title: "Web Development Fundamentals — Part 1: HTML — The Structure of the Web"
date: 2025-04-10 00:00:00 +0000
categories: [Web Development]
tags:
  - web
  - html
  - frontend
  - beginner
series: "Web Development Fundamentals"
series_part: 1
---


## Series Overview

This is an 8-part series covering the core technologies of web development:

1. **HTML — The Structure of the Web** (this article) — Elements, attributes, block vs inline, and document structure
2. [CSS — Styling the Web](/posts/web-dev-part2-css/) — Selectors, properties, the box model, and layout basics
3. [JavaScript — Adding Interactivity](/posts/web-dev-part3-javascript/) — Variables, data types, functions, arrays, and control flow
4. [The DOM — Connecting JavaScript to HTML](/posts/web-dev-part4-dom/) — Selecting elements, changing styles, and modifying attributes
5. [jQuery — Write Less, Do More](/posts/web-dev-part5-jquery/) — Selectors, events, and DOM manipulation with jQuery
6. [Bootstrap — Responsive Layouts Made Easy](/posts/web-dev-part6-bootstrap/) — Grid system, responsive breakpoints, and rapid prototyping
7. [HTTP — How the Web Communicates](/posts/web-dev-part7-http/) — Requests, responses, methods, and status codes
8. [React — Building Modern User Interfaces](/posts/web-dev-part8-react/) — Components, JSX, props, state, hooks, and thinking in React


## What Is HTML?

HTML stands for **HyperText Markup Language**. It is the basic building block of the World Wide Web.

**Hypertext** is text displayed on a computer or other electronic device with references to other text that the user can immediately access, usually by a mouse click or key press. Apart from text, hypertext may contain tables, lists, forms, images, and other presentational elements. It is an easy-to-use and flexible format to share information over the Internet.

## Your First HTML Document

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <title>A simple HTML document</title>
  </head>
  <body>
    <p>Hello World!</p>
  </body>
</html>
```

Let's break this down:

- `<!DOCTYPE html>` — The document type declaration. It tells the browser this is an HTML5 document. It is case-insensitive.
- `<head>` — A container for metadata tags that provide information about the document. For example, `<title>` defines the page title shown in the browser tab.
- `<body>` — Contains the document's actual content (paragraphs, links, images, tables, etc.) that is rendered in the browser and displayed to the user.

## Tags and Elements

HTML is written using **markup tags**. Every tag is a keyword surrounded by angle brackets, such as `<html>`, `<head>`, `<body>`, `<title>`, `<p>`, and so on.

Tags normally come in pairs:

- **Opening tag** (start tag): `<p>`
- **Closing tag** (end tag): `</p>`

The closing tag is identical to the opening tag except for the slash (`/`) after the opening angle bracket, telling the browser the command has been completed.

## Empty (Void) Elements

Some elements are **self-closing** — they don't wrap content and have no closing tag. You cannot write `<br>some content</br>`.

Common empty elements:

| Element | Purpose |
|---------|---------|
| `<br>` | Line break |
| `<hr>` | Horizontal rule |
| `<img>` | Image |
| `<input>` | Form input |
| `<link>` | External resource link |
| `<meta>` | Metadata |

## Block-Level vs Inline Elements

HTML elements fall into two categories that control how they flow in the page:

### Block-Level Elements

Block elements take up the **full width available** and are rendered with a line break before and after. They form the document's structure.

Common block elements: `<div>`, `<p>`, `<h1>` through `<h6>`, `<form>`, `<ol>`, `<ul>`, `<li>`

Block elements may contain both inline elements and other block elements.

### Inline Elements

Inline elements take up **only as much space as they need** and flow within a line of text. They dress up the contents of a block.

Common inline elements: `<a>`, `<span>`, `<strong>`, `<em>`, `<img>`, `<code>`, `<input>`, `<button>`

Inline elements typically may only contain text and other inline elements.

## Nesting Elements

Most HTML elements can contain any number of further elements (except empty elements), which are in turn made up of tags, attributes, and content or other elements. Proper nesting means closing tags in the reverse order they were opened:

```html
<!-- Correct nesting -->
<p>This is <strong>bold and <em>italic</em></strong> text.</p>

<!-- Incorrect nesting -->
<p>This is <strong>bold and <em>italic</strong></em> text.</p>
```

## Comments

HTML comments are invisible to the user but useful for developers:

```html
<!-- This is a comment -->
<!--
  This is a
  multi-line comment
-->
```

## Attributes

Attributes define additional characteristics of an element, such as the width and height of an image. They are always specified in the **opening tag** and usually consist of name/value pairs:

```html
<img src="photo.jpg" alt="A photo" width="300" height="200">
```

Attribute values should always be enclosed in quotation marks.

### The `id` Attribute

The `id` attribute gives an element a **unique identifier** within the document. No two elements can share the same ID. This makes it easy to target the element with CSS or JavaScript:

```html
<input type="text" id="firstName" />
<div id="container">Some content</div>
<p id="infoText">This is a paragraph.</p>
```

### The `class` Attribute

Unlike `id`, the `class` attribute **does not have to be unique**. You can apply the same class to multiple elements, and an element can have multiple classes:

```html
<input type="text" class="highlight" />
<div class="box highlight">Some content</div>
<p class="highlight">This is a paragraph.</p>
```

### The `title` Attribute

The `title` attribute provides **advisory text** displayed as a tooltip when the user hovers over the element:

```html
<abbr title="World Wide Web Consortium">W3C</abbr>
<a href="images/kites.jpg" title="Click to view a larger image">
  <img src="images/kites-thumb.jpg" alt="kites" />
</a>
```

### The `style` Attribute

The `style` attribute allows you to specify CSS styling rules directly on an element. However, best practice is to keep CSS separate from HTML — which we'll cover in [Part 2](/posts/web-dev-part2-css/).

## What's Next?

HTML provides the structure of a web page, but on its own it looks plain. In [Part 2](/posts/web-dev-part2-css/), we'll add **CSS** to control how HTML elements look — colours, fonts, spacing, and layout.

## References

- [MDN HTML Reference](https://developer.mozilla.org/en-US/docs/Web/HTML)
- [HTML Living Standard](https://html.spec.whatwg.org/)
