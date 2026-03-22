---
layout: post
title: "Web Development Fundamentals — Part 3: JavaScript — Adding Interactivity"
date: 2025-05-08 00:00:00 +0000
tags:
  - web
  - javascript
  - frontend
  - beginner
series: "Web Development Fundamentals"
series_part: 3
---


## Series Overview

This is an 8-part series covering the core technologies of web development:

1. [HTML — The Structure of the Web](2025-04-web-dev-part1-html.md) — Elements, attributes, block vs inline, and document structure
2. [CSS — Styling the Web](2025-04-web-dev-part2-css.md) — Selectors, properties, the box model, and layout basics
3. **JavaScript — Adding Interactivity** (this article) — Variables, data types, functions, arrays, and control flow
4. [The DOM — Connecting JavaScript to HTML](2025-05-web-dev-part4-dom.md) — Selecting elements, changing styles, and modifying attributes
5. [jQuery — Write Less, Do More](2025-06-web-dev-part5-jquery.md) — Selectors, events, and DOM manipulation with jQuery
6. [Bootstrap — Responsive Layouts Made Easy](2025-06-web-dev-part6-bootstrap.md) — Grid system, responsive breakpoints, and rapid prototyping
7. [HTTP — How the Web Communicates](2025-07-web-dev-part7-http.md) — Requests, responses, methods, and status codes
8. [React — Building Modern User Interfaces](2025-07-web-dev-part8-react.md) — Components, JSX, props, state, hooks, and thinking in React


## What Is JavaScript?

JavaScript is the most popular and widely used **client-side scripting language**. Client-side scripting refers to scripts that run within your web browser. JavaScript is designed to add interactivity and dynamic effects to web pages by manipulating the content returned from a web server.

While HTML provides structure and CSS provides style, JavaScript provides **behaviour** — responding to clicks, validating forms, animating elements, and fetching data.

## Adding JavaScript to a Web Page

There are three ways to include JavaScript:

### 1. External File (recommended)

Create a `.js` file and link it with `<script>`:

```javascript
// hello.js
function sayHello() {
    alert("Hello World!");
}

document.getElementById("myBtn").onclick = sayHello;
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>External JavaScript</title>
</head>
<body>
    <button type="button" id="myBtn">Click Me</button>
    <script src="js/hello.js"></script>
</body>
</html>
```

### 2. Embedded in `<script>` Tags

```html
<script>
    alert("Hello World!");
</script>
```

### 3. Inline Event Handlers (avoid)

```html
<button onclick="alert('Hello!')">Click Me</button>
```

Always keep content (HTML), presentation (CSS), and behaviour (JavaScript) separate.

### Script Placement

Place `<script>` tags at the **end of the body**, just before `</body>`. Each `<script>` tag blocks page rendering until the JavaScript is fully downloaded and executed. Placing scripts at the bottom prevents this from slowing down the initial page load.

## Variables

Declare variables with `var` (or `let` and `const` in modern JavaScript):

```javascript
var name = "Peter Parker";
var age = 21;
var isMarried = false;

// Multiple declarations
var name = "Peter Parker",
    age = 21,
    isMarried = false;
```

Variable naming rules:
- Must start with a letter, underscore (`_`), or dollar sign (`$`)
- Cannot start with a number
- Can only contain alphanumeric characters, underscores, and dollar signs
- Cannot be a JavaScript reserved keyword
- Are case-sensitive

## Data Types

JavaScript has six basic data types in three categories:

| Category | Types |
|----------|-------|
| **Primitive** | String, Number, Boolean |
| **Composite** | Object, Array, Function |
| **Special** | Undefined, Null |

### Strings

Text data enclosed in single or double quotes:

```javascript
var a = "Let's have a cup of coffee.";  // single quote inside double
var b = 'He said "Hello" and left.';    // double quotes inside single
var c = 'We\'ll never give up.';        // escaping with backslash
```

### Numbers

Positive or negative, with or without decimals:

```javascript
var a = 25;         // integer
var b = 80.5;       // floating-point
var c = 4.25e+6;    // exponential (4,250,000)
```

Special numeric values:

```javascript
16 / 0;           // Infinity
-16 / 0;          // -Infinity
"text" / 2;       // NaN (Not a Number)
Math.sqrt(-1);    // NaN
```

### Booleans

Only two values: `true` or `false`.

### Undefined and Null

```javascript
var a;           // undefined — declared but no value assigned
var b = null;    // null — explicitly set to "nothing"
```

### Objects

Collections of key-value pairs:

```javascript
var car = {
    model: "BMW X3",
    color: "white",
    doors: 5
};
```

### Arrays

Ordered lists of values:

```javascript
var colors = ["Red", "Yellow", "Green", "Orange"];
colors[0];   // "Red"
colors[2];   // "Green"
```

### Functions

Callable objects that execute a block of code:

```javascript
var greeting = function() {
    return "Hello World!";
};

typeof greeting;  // "function"
greeting();       // "Hello World!"
```

### The `typeof` Operator

Check the type of any value:

```javascript
typeof "Hello";     // "string"
typeof 42;          // "number"
typeof true;        // "boolean"
typeof undefined;   // "undefined"
typeof null;        // "object" (a known JavaScript quirk)
typeof {};          // "object"
typeof [];          // "object"
typeof function(){};// "function"
```

## Useful String Methods

```javascript
var str = "Hello World!";

str.length;                    // 12
str.indexOf("World");          // 6
str.lastIndexOf("l");          // 9
str.slice(0, 5);               // "Hello"
str.toUpperCase();             // "HELLO WORLD!"
str.toLowerCase();             // "hello world!"
str.replace("World", "JS");   // "Hello JS!"
str.charAt(0);                 // "H"
str.split(" ");                // ["Hello", "World!"]

// Search with regex (case-insensitive)
"Color red".search(/color/i);  // 0
```

## Useful Number Methods

```javascript
parseInt("50px");       // 50
parseFloat("3.14em");   // 3.14
parseInt("Year 2048");  // NaN

var x = 72.635;
x.toFixed(2);           // "72.64" (rounds and pads)
x.toPrecision(3);       // "72.6"
(255).toString(16);     // "ff" (hex)
```

## Control Flow

### if / else if / else

```javascript
if (condition1) {
    // runs if condition1 is true
} else if (condition2) {
    // runs if condition2 is true
} else {
    // runs if all conditions are false
}
```

### Ternary Operator

```javascript
var result = (age >= 18) ? "Adult" : "Minor";
```

### switch

```javascript
switch (day) {
    case "Monday":
        console.log("Start of the week");
        break;
    case "Friday":
        console.log("Almost weekend!");
        break;
    default:
        console.log("Just another day");
}
```

## Arrays In Depth

### Basic Operations

```javascript
var fruits = ["Apple", "Banana", "Mango"];

fruits.length;           // 3
fruits.push("Orange");   // Add to end → ["Apple", "Banana", "Mango", "Orange"]
fruits.pop();            // Remove from end → returns "Orange"
fruits.unshift("Kiwi");  // Add to start → ["Kiwi", "Apple", "Banana", "Mango"]
fruits.shift();          // Remove from start → returns "Kiwi"
```

`push()` and `pop()` are faster than `unshift()` and `shift()` because they don't require re-indexing the entire array.

### splice — Insert or Remove at Any Position

```javascript
var colors = ["Red", "Green", "Blue"];

// Remove 1 element at index 0
colors.splice(0, 1);  // colors = ["Green", "Blue"], returns ["Red"]

// Insert at index 1 without removing
colors.splice(1, 0, "Pink", "Yellow");
// colors = ["Green", "Pink", "Yellow", "Blue"]

// Replace 1 element at index 1
colors.splice(1, 1, "Purple");
// colors = ["Green", "Purple", "Yellow", "Blue"]
```

### Searching

```javascript
var arr = ["Apple", "Banana", "Mango"];

arr.indexOf("Banana");       // 1
arr.indexOf("Pineapple");    // -1 (not found)
arr.includes("Mango");       // true
```

### Sorting

```javascript
// Alphabetical
["Banana", "Apple", "Mango"].sort();  // ["Apple", "Banana", "Mango"]

// Numeric (requires compare function)
[5, 20, 10, 75].sort((a, b) => a - b);  // [5, 10, 20, 75]

// Sort objects by property
persons.sort((a, b) => a.age - b.age);
```

### Other Useful Methods

```javascript
["Red", "Green", "Blue"].join(", ");    // "Red, Green, Blue"
["Red", "Green", "Blue"].reverse();     // ["Blue", "Green", "Red"]
[1, 2, 3, 4, 5].slice(1, 3);           // [2, 3] (non-destructive)

// Min and max
Math.max.apply(null, [3, -7, 10, 8]);   // 10
Math.min.apply(null, [3, -7, 10, 8]);   // -7
```

## Loops

```javascript
// for loop
for (var i = 0; i < 5; i++) {
    console.log(i);
}

// while loop
while (condition) {
    // code
}

// do...while — runs at least once
do {
    // code
} while (condition);

// for...in — iterate over object properties
var person = { name: "Clark", age: 36 };
for (var prop in person) {
    console.log(prop + " = " + person[prop]);
}

// for...of (ES6) — iterate over iterable values
let letters = ["a", "b", "c"];
for (let letter of letters) {
    console.log(letter);
}
```

## Functions

```javascript
// Declaration
function add(a, b) {
    return a + b;
}

add(6, 20);  // 26

// Default parameters (ES6)
function greet(name = "Guest") {
    return "Hello, " + name;
}

greet();       // "Hello, Guest"
greet("John"); // "Hello, John"
```

## Objects

Objects are collections of named values (properties):

```javascript
var person = {
    name: "Peter",
    age: 28,
    gender: "Male",
    greet: function() {
        alert(this.name);
    }
};

// Access properties
person.name;          // "Peter"
person["age"];        // 28
person.greet();       // alerts "Peter"

// Loop through properties
for (var key in person) {
    console.log(key + ": " + person[key]);
}
```

## Client-Side vs Server-Side

JavaScript runs in the **browser** (client-side), while languages like PHP, Python, and Java run on the **web server** (server-side) and send HTML back to the browser. This distinction matters because client-side code can respond instantly to user actions without making a round trip to the server.

## What's Next?

JavaScript on its own can't change what's on the page. To do that, it needs to interact with the HTML structure through the **Document Object Model (DOM)**. In [Part 4](2025-05-web-dev-part4-dom.md), we'll learn how JavaScript selects, modifies, and styles HTML elements.

## References

- [MDN JavaScript Reference](https://developer.mozilla.org/en-US/docs/Web/JavaScript)
- [JavaScript Tutorial](https://www.tutorialrepublic.com/javascript-tutorial/)
