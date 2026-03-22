---
layout: post
title: "Web Development Fundamentals — Part 2: CSS — Styling the Web"
date: 2025-04-24 00:00:00 +0000
tags:
  - web
  - css
  - frontend
  - beginner
series: "Web Development Fundamentals"
series_part: 2
---


## Series Overview

This is an 8-part series covering the core technologies of web development:

1. [HTML — The Structure of the Web](/posts/web-dev-part1-html/) — Elements, attributes, block vs inline, and document structure
2. **CSS — Styling the Web** (this article) — Selectors, properties, the box model, and layout basics
3. [JavaScript — Adding Interactivity](/posts/web-dev-part3-javascript/) — Variables, data types, functions, arrays, and control flow
4. [The DOM — Connecting JavaScript to HTML](/posts/web-dev-part4-dom/) — Selecting elements, changing styles, and modifying attributes
5. [jQuery — Write Less, Do More](/posts/web-dev-part5-jquery/) — Selectors, events, and DOM manipulation with jQuery
6. [Bootstrap — Responsive Layouts Made Easy](/posts/web-dev-part6-bootstrap/) — Grid system, responsive breakpoints, and rapid prototyping
7. [HTTP — How the Web Communicates](/posts/web-dev-part7-http/) — Requests, responses, methods, and status codes
8. [React — Building Modern User Interfaces](/posts/web-dev-part8-react/) — Components, JSX, props, state, hooks, and thinking in React


## What Is CSS?

CSS (Cascading Style Sheets) controls how HTML elements are displayed on screen. While HTML defines the **structure** of a page, CSS defines its **presentation** — colours, fonts, spacing, layout, and responsive behaviour.

The "cascading" part means that styles can be inherited and overridden in a predictable order: browser defaults, external stylesheets, internal styles, and inline styles — with the most specific rule winning.

## Three Ways to Add CSS

### 1. Inline Styles (avoid)

Applied directly to an element using the `style` attribute:

```html
<p style="color: blue; font-size: 18px;">This is blue text.</p>
```

This works but mixes presentation with structure — hard to maintain.

### 2. Internal Stylesheet

Placed inside a `<style>` tag in the `<head>`:

```html
<head>
  <style>
    p {
      color: blue;
      font-size: 18px;
    }
  </style>
</head>
```

Useful for single-page demos, but doesn't scale across multiple pages.

### 3. External Stylesheet (recommended)

A separate `.css` file linked in the `<head>`:

```html
<head>
  <link rel="stylesheet" href="css/style.css">
</head>
```

```css
/* style.css */
p {
  color: blue;
  font-size: 18px;
}
```

This keeps content and presentation separate and allows the same stylesheet to be shared across multiple pages.

## CSS Selectors

Selectors determine which HTML elements a rule applies to.

### Basic Selectors

```css
/* Element selector — targets all <p> elements */
p {
  color: navy;
}

/* Class selector — targets elements with class="highlight" */
.highlight {
  background-color: yellow;
}

/* ID selector — targets the element with id="header" */
#header {
  font-size: 24px;
}

/* Universal selector — targets all elements */
* {
  margin: 0;
  padding: 0;
}
```

### Combining Selectors

```css
/* Descendant — any <a> inside a <nav> */
nav a {
  text-decoration: none;
}

/* Direct child — only <li> directly inside <ul> */
ul > li {
  list-style: square;
}

/* Multiple selectors — same rules for h1, h2, and h3 */
h1, h2, h3 {
  font-family: Arial, sans-serif;
}

/* Class + element — only <p> elements with class "intro" */
p.intro {
  font-weight: bold;
}
```

### Pseudo-classes

```css
/* Link states */
a:hover {
  color: red;
}

a:visited {
  color: purple;
}

/* First and last child */
li:first-child {
  font-weight: bold;
}

li:last-child {
  border-bottom: none;
}

/* Odd/even rows */
tr:nth-child(odd) {
  background-color: #f2f2f2;
}
```

## The Box Model

Every HTML element is a rectangular box. The CSS box model describes the space an element takes up:

```
+---------------------------+
|         margin            |
|  +---------------------+ |
|  |      border          | |
|  |  +-----------------+ | |
|  |  |    padding       | | |
|  |  |  +----------+   | | |
|  |  |  |  content  |   | | |
|  |  |  +----------+   | | |
|  |  +-----------------+ | |
|  +---------------------+ |
+---------------------------+
```

```css
.box {
  width: 300px;          /* Content width */
  padding: 20px;         /* Space inside the border */
  border: 1px solid #ccc; /* The border itself */
  margin: 10px;          /* Space outside the border */
}
```

By default, `width` only applies to the content area. Add `box-sizing: border-box` to make `width` include padding and border — a common best practice:

```css
*, *::before, *::after {
  box-sizing: border-box;
}
```

## Common CSS Properties

### Text and Font

```css
p {
  color: #333;                    /* Text colour */
  font-family: Arial, sans-serif; /* Font stack */
  font-size: 16px;                /* Size */
  font-weight: bold;              /* Weight: normal, bold, 100-900 */
  line-height: 1.5;               /* Line spacing */
  text-align: center;             /* Alignment: left, center, right, justify */
  text-decoration: underline;     /* Underline, overline, line-through, none */
  text-transform: uppercase;      /* uppercase, lowercase, capitalize */
}
```

### Backgrounds

```css
.hero {
  background-color: #f0f0f0;
  background-image: url('bg.jpg');
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
}
```

### Display and Visibility

```css
.hidden {
  display: none;       /* Removed from layout entirely */
}

.invisible {
  visibility: hidden;  /* Hidden but still takes up space */
}

.inline-block {
  display: inline-block; /* Inline but respects width/height */
}
```

## Layout with Flexbox

Flexbox is the modern way to create one-dimensional layouts (rows or columns):

```css
.container {
  display: flex;
  justify-content: space-between; /* Horizontal distribution */
  align-items: center;            /* Vertical alignment */
  gap: 20px;                      /* Space between items */
}

.item {
  flex: 1; /* Each item takes equal space */
}
```

## Layout with CSS Grid

For two-dimensional layouts (rows and columns together):

```css
.grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr); /* Three equal columns */
  grid-gap: 20px;
}
```

## Responsive Design with Media Queries

Media queries let you apply different styles based on screen size:

```css
/* Mobile first — base styles for small screens */
.container {
  width: 100%;
  padding: 10px;
}

/* Tablets and above */
@media (min-width: 768px) {
  .container {
    width: 750px;
    margin: 0 auto;
  }
}

/* Desktops */
@media (min-width: 1200px) {
  .container {
    width: 1140px;
  }
}
```

## CSS Units

| Unit | Type | Description |
|------|------|-------------|
| `px` | Absolute | Fixed pixels |
| `em` | Relative | Relative to parent font-size |
| `rem` | Relative | Relative to root font-size |
| `%` | Relative | Percentage of parent |
| `vw` / `vh` | Relative | Viewport width / height |
| `fr` | Relative | Fraction of available space (Grid) |

## What's Next?

With HTML for structure and CSS for style, the missing piece is **behaviour**. In [Part 3](/posts/web-dev-part3-javascript/), we'll learn JavaScript — the programming language that makes web pages interactive.

## References

- [MDN CSS Reference](https://developer.mozilla.org/en-US/docs/Web/CSS)
- [CSS-Tricks — A Complete Guide to Flexbox](https://css-tricks.com/snippets/css/a-guide-to-flexbox/)
- [CSS-Tricks — A Complete Guide to Grid](https://css-tricks.com/snippets/css/complete-guide-grid/)
