---
layout: post
title: "Web Development Fundamentals — Part 8: React — Building Modern User Interfaces"
date: 2025-07-17 00:00:00 +0000
tags:
  - web
  - react
  - javascript
  - frontend
  - hooks
  - components
series: "Web Development Fundamentals"
series_part: 8
---


## Series Overview

This is an 8-part series covering the core technologies of web development:

1. [HTML — The Structure of the Web](2025-04-web-dev-part1-html.md) — Elements, attributes, block vs inline, and document structure
2. [CSS — Styling the Web](2025-04-web-dev-part2-css.md) — Selectors, properties, the box model, and layout basics
3. [JavaScript — Adding Interactivity](2025-05-web-dev-part3-javascript.md) — Variables, data types, functions, arrays, and control flow
4. [The DOM — Connecting JavaScript to HTML](2025-05-web-dev-part4-dom.md) — Selecting elements, changing styles, and modifying attributes
5. [jQuery — Write Less, Do More](2025-06-web-dev-part5-jquery.md) — Selectors, events, and DOM manipulation with jQuery
6. [Bootstrap — Responsive Layouts Made Easy](2025-06-web-dev-part6-bootstrap.md) — Grid system, responsive breakpoints, and rapid prototyping
7. [HTTP — How the Web Communicates](2025-07-web-dev-part7-http.md) — Requests, responses, methods, and status codes
8. **React — Building Modern User Interfaces** (this article) — Components, JSX, props, state, hooks, and thinking in React


## What Is React?

React is a JavaScript library for building user interfaces, created by Meta (Facebook). Instead of manipulating the DOM directly — as we did with vanilla JavaScript in [Part 4](2025-05-web-dev-part4-dom.md) and jQuery in [Part 5](2025-06-web-dev-part5-jquery.md) — React lets you describe **what** the UI should look like and handles the DOM updates for you.

React's key ideas:

- **Component-based** — Build UIs from small, reusable, self-contained pieces
- **Declarative** — Describe the desired state, React figures out how to update the DOM
- **Unidirectional data flow** — Data flows down from parent to child via props

## Getting Started

The recommended way to start a new React project is with a framework like Next.js or with a build tool like Vite:

```shell
# Using Vite
npm create vite@latest my-app -- --template react
cd my-app
npm install
npm run dev
```

> Note: Create React App (CRA) is deprecated and no longer recommended.

## Components — The Building Blocks

A React component is a JavaScript function that returns **JSX** (markup):

```jsx
function MyButton() {
  return (
    <button>I'm a button</button>
  );
}
```

Components compose into larger UIs by nesting them:

```jsx
export default function MyApp() {
  return (
    <div>
      <h1>Welcome to my app</h1>
      <MyButton />
    </div>
  );
}
```

**Key rule:** React component names must always start with a capital letter. HTML tags must be lowercase. This is how React tells them apart — `<button>` is an HTML element, `<MyButton />` is a React component.

## JSX — HTML in JavaScript

JSX is a syntax extension that looks like HTML but works inside JavaScript. It's stricter than HTML:

- Every tag must be closed: `<br />`, `<img />`
- A component can only return **one root element** — use a Fragment (`<>...</>`) to wrap multiple elements without adding extra DOM nodes

```jsx
function AboutPage() {
  return (
    <>
      <h1>About</h1>
      <p>Hello there.<br />How do you do?</p>
    </>
  );
}
```

### Embedding JavaScript in JSX

Use curly braces `{}` to embed JavaScript expressions inside JSX:

```jsx
const user = { name: "Alice", avatar: "alice.jpg" };

function Profile() {
  return (
    <div>
      <h1>{user.name}</h1>
      <img
        className="avatar"
        src={user.avatar}
        alt={"Photo of " + user.name}
        style={{ width: 100, height: 100 }}
      />
    </div>
  );
}
```

Note: use `className` instead of `class` for CSS classes — `class` is a reserved word in JavaScript.

## Props — Passing Data to Components

Props let you pass data from a parent component to a child component, just like HTML attributes:

```jsx
// Parent passes props
function App() {
  return <Greeting name="Alice" age={30} />;
}

// Child receives props
function Greeting({ name, age }) {
  return (
    <p>Hello, {name}! You are {age} years old.</p>
  );
}
```

Props are **read-only** — a child component should never modify the props it receives. If data needs to change, use state instead.

## Conditional Rendering

Use standard JavaScript conditionals to render different UI based on state:

### if/else

```jsx
function Dashboard({ isLoggedIn }) {
  let content;
  if (isLoggedIn) {
    content = <AdminPanel />;
  } else {
    content = <LoginForm />;
  }
  return <div>{content}</div>;
}
```

### Ternary Operator

```jsx
function Dashboard({ isLoggedIn }) {
  return (
    <div>
      {isLoggedIn ? <AdminPanel /> : <LoginForm />}
    </div>
  );
}
```

### Logical AND (`&&`)

When you only need to render something or nothing:

```jsx
function Notification({ hasMessages }) {
  return (
    <div>
      {hasMessages && <span>You have new messages!</span>}
    </div>
  );
}
```

## Rendering Lists

Use JavaScript's `map()` to transform an array into a list of elements:

```jsx
const products = [
  { title: "Cabbage", id: 1 },
  { title: "Garlic", id: 2 },
  { title: "Apple", id: 3 },
];

function ProductList() {
  const listItems = products.map((product) => (
    <li key={product.id}>{product.title}</li>
  ));

  return <ul>{listItems}</ul>;
}
```

**Important:** Every list item must have a `key` prop with a unique identifier. Keys help React track which items have changed, been added, or removed — this is critical for performance.

## Handling Events

Pass event handler functions (not calls) to JSX elements:

```jsx
function MyButton() {
  function handleClick() {
    alert("You clicked me!");
  }

  return (
    <button onClick={handleClick}>
      Click me
    </button>
  );
}
```

Note: pass `handleClick` (the function), not `handleClick()` (which would call it immediately during render).

Common event handlers: `onClick`, `onChange`, `onSubmit`, `onMouseEnter`, `onKeyDown`.

## State with `useState`

State lets a component "remember" information between renders. Import `useState` from React:

```jsx
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);
  }

  return (
    <button onClick={handleClick}>
      Clicked {count} times
    </button>
  );
}
```

`useState` returns two things:
1. The **current state value** (`count`)
2. A **setter function** to update it (`setCount`)

When you call the setter, React re-renders the component with the new value.

### Sharing State Between Components

If two sibling components need the same state, **lift the state up** to their common parent and pass it down as props:

```jsx
function App() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);
  }

  return (
    <div>
      <CounterButton count={count} onClick={handleClick} />
      <CounterButton count={count} onClick={handleClick} />
    </div>
  );
}

function CounterButton({ count, onClick }) {
  return (
    <button onClick={onClick}>
      Clicked {count} times
    </button>
  );
}
```

Now both buttons share and update the same count.

## Side Effects with `useEffect`

Effects let you synchronise a component with external systems — APIs, browser APIs, timers, third-party libraries, etc.

```jsx
import { useState, useEffect } from "react";

function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.connect();

    // Cleanup: runs when roomId changes or component unmounts
    return () => {
      connection.disconnect();
    };
  }, [roomId]); // Only re-run when roomId changes

  return <h1>Welcome to {roomId}!</h1>;
}
```

### The Dependency Array

The second argument to `useEffect` controls when the effect runs:

| Dependency | When it runs |
|------------|-------------|
| No array | After every render |
| `[]` | Only on mount (once) |
| `[a, b]` | When `a` or `b` changes |

### Cleanup Functions

The function returned from `useEffect` is the **cleanup**. It runs:
- Before the effect re-runs (when dependencies change)
- When the component unmounts

Common patterns:

```jsx
// Event listeners
useEffect(() => {
  function handleScroll() {
    console.log(window.scrollY);
  }
  window.addEventListener("scroll", handleScroll);
  return () => window.removeEventListener("scroll", handleScroll);
}, []);

// Data fetching with race condition prevention
useEffect(() => {
  let ignore = false;

  async function fetchData() {
    const response = await fetch(`/api/users/${userId}`);
    const data = await response.json();
    if (!ignore) {
      setUser(data);
    }
  }

  fetchData();
  return () => { ignore = true; };
}, [userId]);
```

### When NOT to Use Effects

- **Transforming data for rendering** — do it during render instead
- **Handling user events** — use event handlers
- **Resetting state when a prop changes** — use a `key` to force remount

Effects are an "escape hatch" from React's declarative model. Use them only when you need to step outside React to interact with external systems.

## Other Essential Hooks

Beyond `useState` and `useEffect`, React provides several other built-in hooks:

### `useContext` — Avoid Prop Drilling

Read values from a context without passing props through every level:

```jsx
import { useContext } from "react";

function Button() {
  const theme = useContext(ThemeContext);
  return <button className={theme}>Click me</button>;
}
```

### `useRef` — Persist Values Without Re-rendering

Hold a mutable value that doesn't trigger re-renders (commonly used for DOM references):

```jsx
import { useRef } from "react";

function TextInput() {
  const inputRef = useRef(null);

  function handleClick() {
    inputRef.current.focus();
  }

  return (
    <>
      <input ref={inputRef} />
      <button onClick={handleClick}>Focus input</button>
    </>
  );
}
```

### `useMemo` — Cache Expensive Calculations

Skip recalculating a value if its dependencies haven't changed:

```jsx
import { useMemo } from "react";

function TodoList({ todos, filter }) {
  const visibleTodos = useMemo(
    () => filterTodos(todos, filter),
    [todos, filter]
  );
  return <ul>{visibleTodos.map(/* ... */)}</ul>;
}
```

### `useCallback` — Cache Function References

Prevent child components from re-rendering when a callback function hasn't changed:

```jsx
import { useCallback } from "react";

function Parent({ productId }) {
  const handleSubmit = useCallback((data) => {
    post("/product/" + productId, data);
  }, [productId]);

  return <Form onSubmit={handleSubmit} />;
}
```

### Hooks at a Glance

| Hook | Purpose |
|------|---------|
| `useState` | Add state to a component |
| `useEffect` | Synchronise with external systems |
| `useContext` | Read context without prop drilling |
| `useRef` | Reference a DOM node or persist a value |
| `useMemo` | Cache an expensive computation |
| `useCallback` | Cache a function reference |
| `useReducer` | Manage complex state with a reducer |
| `useTransition` | Mark state updates as non-blocking |
| `useId` | Generate unique IDs for accessibility |

### Rules of Hooks

1. Only call hooks at the **top level** of a component or custom hook — never inside loops, conditions, or nested functions
2. Only call hooks from **React function components** or custom hooks — never from regular JavaScript functions

## Building Custom Hooks

Extract reusable logic into custom hooks — functions that start with `use`:

```jsx
function useWindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);

  useEffect(() => {
    function handleResize() {
      setWidth(window.innerWidth);
    }
    window.addEventListener("resize", handleResize);
    return () => window.removeEventListener("resize", handleResize);
  }, []);

  return width;
}

// Use it in any component
function Header() {
  const width = useWindowWidth();
  return <header>{width > 768 ? "Desktop" : "Mobile"}</header>;
}
```

Custom hooks let you share stateful logic between components without duplicating code.

## Thinking in React

React encourages a specific mental model for building UIs:

1. **Break the UI into a component hierarchy** — Each component should do one thing
2. **Build a static version first** — Render the UI from data using props, no state yet
3. **Identify the minimal state** — What's the smallest set of state your app needs?
4. **Decide where state should live** — Find the common parent of components that need the state
5. **Add inverse data flow** — Pass callbacks down so child components can update parent state

## Series Recap

Over this 8-part series, we've traced the complete arc of web development:

1. **[HTML](2025-04-web-dev-part1-html.md)** — The structure and content of web pages
2. **[CSS](2025-04-web-dev-part2-css.md)** — Visual styling, layout, and responsive design
3. **[JavaScript](2025-05-web-dev-part3-javascript.md)** — Programming logic and interactivity
4. **[DOM](2025-05-web-dev-part4-dom.md)** — The bridge between JavaScript and HTML
5. **[jQuery](2025-06-web-dev-part5-jquery.md)** — A concise API for DOM manipulation
6. **[Bootstrap](2025-06-web-dev-part6-bootstrap.md)** — Rapid responsive layouts
7. **[HTTP](2025-07-web-dev-part7-http.md)** — The protocol that connects it all
8. **React** — Component-based UIs with declarative state management

From hand-crafting HTML tags to building reactive component trees — these fundamentals form the foundation of modern web development.

## References

- [React Official Documentation](https://react.dev/learn)
- [React Hooks Reference](https://react.dev/reference/react/hooks)
- [Synchronizing with Effects](https://react.dev/learn/synchronizing-with-effects)
- [React Tutorial (interactive)](https://react-tutorial.app/)
