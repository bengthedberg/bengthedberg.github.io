---
layout: post
title: "Node.js Fundamentals Part 1: Core Concepts and Building Blocks"
date: 2026-01-05 00:00:00 +0000
tags:
  - nodejs
  - javascript
  - typescript
  - backend
series: "Node.js Fundamentals"
series_part: 1
---


## Series Overview

- **Part 1: Node.js Core Concepts and Building Blocks** (this post)
- [Part 2: Node.js on AWS](/posts/nodejs-fundamentals-part2-aws/)

## Introduction

Node.js transformed JavaScript from a browser-only language into a full-stack powerhouse. Created by Ryan Dahl in 2009, it wraps Google's V8 engine in a C++ runtime that provides access to the filesystem, networking, and operating system APIs. Today, Node.js powers everything from lightweight microservices to large-scale enterprise platforms.

This post covers the foundational concepts you need to understand Node.js deeply: how it works under the hood, its module system, core APIs, asynchronous patterns, and how to build production-grade applications with Express, TypeScript, and Docker.

## How Node.js Works Under the Hood

### Architecture

Node.js sits on two pillars: the **V8 engine** for JavaScript execution and **libuv** for asynchronous I/O.

```
┌─────────────────────────────────────────────────────────┐
│                    YOUR JAVASCRIPT CODE                  │
├─────────────────────────────────────────────────────────┤
│                       NODE.JS APIs                       │
│          (fs, http, crypto, stream, net, etc.)           │
├──────────────────────┬──────────────────────────────────┤
│       V8 ENGINE      │           LIBUV                   │
│  (JS compilation &   │  (Event loop, async I/O,         │
│   execution)         │   thread pool, networking)       │
├──────────────────────┴──────────────────────────────────┤
│               OPERATING SYSTEM                           │
│    (epoll/kqueue/IOCP, TCP/UDP, file descriptors)       │
└─────────────────────────────────────────────────────────┘
```

V8 compiles JavaScript directly to machine code using Just-In-Time (JIT) compilation. Your code is first parsed into an Abstract Syntax Tree, compiled to bytecode by the Ignition interpreter for fast startup, and then optimized into native machine code by TurboFan for hot code paths.

### The Event Loop

The event loop is the heart of Node.js. It continuously checks for pending work and executes callbacks when operations complete, cycling through six phases:

1. **Timers** -- executes `setTimeout` and `setInterval` callbacks.
2. **Pending callbacks** -- runs deferred I/O callbacks.
3. **Idle, prepare** -- internal use only.
4. **Poll** -- retrieves new I/O events and executes their callbacks. This is where Node spends most of its time.
5. **Check** -- runs `setImmediate()` callbacks.
6. **Close callbacks** -- handles close events like `socket.on('close')`.

Between every phase, Node drains the **microtask queues**: `process.nextTick()` runs first, followed by Promise callbacks.

```js
console.log("1 — Sync");

setTimeout(() => console.log("2 — Timer"), 0);
setImmediate(() => console.log("3 — Check"));
process.nextTick(() => console.log("4 — nextTick"));
Promise.resolve().then(() => console.log("5 — Promise"));

console.log("6 — Sync");

// Output:
// 1 — Sync
// 6 — Sync
// 4 — nextTick
// 5 — Promise
// 2 — Timer
// 3 — Check
```

### libuv and the Thread Pool

Node delegates I/O to libuv, a C library that provides a thread pool (default 4 threads) for file system operations, DNS lookups, and crypto, plus OS-level async primitives (epoll/kqueue/IOCP) for network I/O.

```bash
# Increase the thread pool for I/O-heavy workloads
UV_THREADPOOL_SIZE=16 node app.js
```

## The Module System

Node supports two module systems: CommonJS (the original) and ES Modules (the modern standard).

### CommonJS

```js
// math.js
function add(a, b) { return a + b; }
function multiply(a, b) { return a * b; }
module.exports = { add, multiply };

// app.js
const { add, multiply } = require("./math");
```

Modules are cached after first load, making them effectively singletons. The `require()` function resolves paths, wraps your code in a function for scope isolation, evaluates it, and caches the result.

### ES Modules

Use `.mjs` extension or set `"type": "module"` in `package.json`:

```js
// math.mjs
export function add(a, b) { return a + b; }
export default function multiply(a, b) { return a * b; }

// app.mjs
import multiply, { add } from "./math.mjs";

// Dynamic import (works in both CJS and ESM)
const { readFile } = await import("fs/promises");
```

| Feature | CommonJS | ES Modules |
|---------|----------|------------|
| Syntax | `require()` / `module.exports` | `import` / `export` |
| Loading | Synchronous | Asynchronous |
| Top-level await | No | Yes |
| `__dirname` | Available | Use `import.meta.url` |

## Core Modules Deep Dive

### File System (`fs`)

Always prefer the promise-based API (`fs/promises`) over synchronous methods:

```js
const fsp = require("fs/promises");

// Read and write files
const data = await fsp.readFile("config.json", "utf8");
await fsp.writeFile("output.txt", "Hello, Node!", "utf8");

// Directory operations
await fsp.mkdir("path/to/nested/dir", { recursive: true });
const entries = await fsp.readdir("./src", { withFileTypes: true });

// Watch for changes
const watcher = fsp.watch("./src", { recursive: true });
for await (const event of watcher) {
  console.log(`${event.eventType}: ${event.filename}`);
}
```

### Streams

Streams process data piece by piece, which is critical for handling large files without loading everything into memory:

```js
const { pipeline } = require("stream/promises");
const zlib = require("zlib");
const fs = require("fs");

// Compress a file using streams
async function compressFile(input, output) {
  await pipeline(
    fs.createReadStream(input),
    zlib.createGzip(),
    fs.createWriteStream(output)
  );
}
```

Node provides four stream types: **Readable** (data source), **Writable** (data destination), **Duplex** (both), and **Transform** (modifies data in transit). The `pipeline` function handles error propagation and cleanup automatically.

## Asynchronous Patterns

### Async/Await (Recommended)

```js
async function combineFiles() {
  const [d1, d2] = await Promise.all([
    fsp.readFile("file1.txt", "utf8"),
    fsp.readFile("file2.txt", "utf8"),
  ]);
  await fsp.writeFile("combined.txt", d1 + d2);
}
```

### Concurrency Utilities

```js
// Parallel execution
const [users, posts] = await Promise.all([fetchUsers(), fetchPosts()]);

// First to finish wins
const fastest = await Promise.race([fetchFromUS(), fetchFromEU()]);

// Wait for all, don't fail on any
const results = await Promise.allSettled([serviceA(), serviceB()]);

// Cancellable operations
const controller = new AbortController();
setTimeout(() => controller.abort(), 5000);
const res = await fetch(url, { signal: controller.signal });
```

## Building REST APIs with Express

### Full CRUD Example

```js
const express = require("express");
const app = express();
app.use(express.json());

let books = [
  { id: 1, title: "Dune", author: "Frank Herbert", year: 1965 },
];
let nextId = 2;

app.get("/api/books", (req, res) => {
  let result = [...books];
  if (req.query.author) {
    result = result.filter((b) =>
      b.author.toLowerCase().includes(req.query.author.toLowerCase())
    );
  }
  res.json({ count: result.length, books: result });
});

app.post("/api/books", (req, res) => {
  const { title, author, year } = req.body;
  if (!title || !author) {
    return res.status(400).json({ error: "Title and author required" });
  }
  const book = { id: nextId++, title, author, year: year || null };
  books.push(book);
  res.status(201).json(book);
});

app.listen(3000, () => console.log("API on http://localhost:3000"));
```

### Middleware

Middleware functions execute in order, forming a chain of responsibility:

```
Request → [Logger] → [CORS] → [Auth] → [Route] → Response
                                  ↓ (error)
                         [Error Handler]
```

```js
function logger(req, res, next) {
  const start = Date.now();
  res.on("finish", () => {
    console.log(`${req.method} ${req.url} → ${res.statusCode} (${Date.now() - start}ms)`);
  });
  next();
}

function authenticate(req, res, next) {
  const token = req.headers.authorization?.replace("Bearer ", "");
  if (!token) return res.status(401).json({ error: "No token" });
  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET);
    next();
  } catch {
    res.status(401).json({ error: "Invalid token" });
  }
}
```

### Input Validation with Zod

```js
const { z } = require("zod");

const createBookSchema = z.object({
  title: z.string().min(1, "Required").max(200),
  author: z.string().min(1, "Required").max(100),
  year: z.number().int().min(1000).max(new Date().getFullYear()).optional(),
  tags: z.array(z.string()).max(10).default([]),
});

function validate(schema) {
  return (req, res, next) => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      return res.status(400).json({
        error: "Validation failed",
        details: result.error.issues,
      });
    }
    req.body = result.data;
    next();
  };
}
```

## Working with Databases

Node.js works well with both SQL and NoSQL databases. Here are three common choices:

### SQLite with better-sqlite3

A great option for small to medium applications and prototyping:

```js
const Database = require("better-sqlite3");
const db = new Database("app.db");
db.pragma("journal_mode = WAL");

const insertUser = db.prepare(
  "INSERT INTO users (name, email) VALUES (?, ?)"
);
const findByEmail = db.prepare(
  "SELECT * FROM users WHERE email = ?"
);

// Transactions for atomic operations
const createWithPosts = db.transaction((user, posts) => {
  const { lastInsertRowid } = insertUser.run(user.name, user.email);
  const insertPost = db.prepare(
    "INSERT INTO posts (title, body, user_id) VALUES (?, ?, ?)"
  );
  for (const p of posts) insertPost.run(p.title, p.body, lastInsertRowid);
});
```

### PostgreSQL with pg

For production relational databases, use connection pooling:

```js
const { Pool } = require("pg");
const pool = new Pool({ host: "localhost", database: "myapp", max: 20 });

const { rows } = await pool.query(
  "SELECT * FROM users WHERE id = $1", [userId]
);
```

### MongoDB with Mongoose

For document-oriented data:

```js
const mongoose = require("mongoose");
await mongoose.connect("mongodb://localhost:27017/myapp");

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  role: { type: String, enum: ["user", "admin"], default: "user" },
}, { timestamps: true });

const User = mongoose.model("User", userSchema);
```

## TypeScript with Node.js

TypeScript adds static type checking to Node.js, catching errors at compile time:

```bash
npm install -D typescript @types/node
npx tsc --init
```

```ts
// src/types.ts
export interface User {
  id: number;
  name: string;
  email: string;
  role: "user" | "admin";
}

export interface ApiResponse<T> {
  success: boolean;
  data?: T;
  error?: string;
}

// src/services/userService.ts
export class UserService {
  private users: User[] = [];

  async create(dto: CreateUserDTO): Promise<ApiResponse<User>> {
    if (this.users.find((u) => u.email === dto.email)) {
      return { success: false, error: "Email exists" };
    }
    const user: User = { id: this.nextId++, ...dto, role: "user" };
    this.users.push(user);
    return { success: true, data: user };
  }
}
```

## Dockerizing Node.js Applications

A production-ready Dockerfile for Node.js:

```dockerfile
FROM node:22-alpine AS base
WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci --only=production

COPY . .

RUN addgroup -S app && adduser -S app -G app
USER app

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "src/index.js"]
```

Key practices: use Alpine images for smaller size, run as a non-root user, include health checks, and use multi-stage builds for applications that need a build step.

## Error Handling and Security

### Custom Error Classes

```js
class AppError extends Error {
  constructor(message, statusCode, code) {
    super(message);
    this.statusCode = statusCode;
    this.code = code;
    this.isOperational = true;
  }
}

class NotFoundError extends AppError {
  constructor(resource = "Resource") {
    super(`${resource} not found`, 404, "NOT_FOUND");
  }
}
```

### Graceful Shutdown

```js
function gracefulShutdown(signal) {
  console.log(`${signal} received`);
  server.close(() => {
    db.close();
    process.exit(0);
  });
  setTimeout(() => process.exit(1), 10000); // force after 10s
}

process.on("SIGTERM", () => gracefulShutdown("SIGTERM"));
process.on("SIGINT", () => gracefulShutdown("SIGINT"));
```

### Security Essentials

```js
const helmet = require("helmet");
const rateLimit = require("express-rate-limit");

app.use(helmet());
app.use(express.json({ limit: "10kb" }));
app.use("/api/", rateLimit({ windowMs: 15 * 60 * 1000, max: 100 }));
```

Always use parameterized queries, hash passwords with bcrypt or scrypt, validate all input server-side, and never hardcode secrets.

## Recommended Project Structure

```
my-app/
├── src/
│   ├── config/          # env validation, database config
│   ├── controllers/     # HTTP request handlers
│   ├── middleware/       # auth, logging, validation, errors
│   ├── models/          # database schemas/models
│   ├── repositories/    # database access layer
│   ├── routes/          # route definitions
│   ├── services/        # business logic
│   ├── utils/           # helpers
│   ├── app.js           # express setup (no listen)
│   └── server.js        # starts the server
├── tests/
│   ├── unit/
│   └── integration/
├── Dockerfile
├── docker-compose.yml
└── package.json
```

Separate `app.js` (creates Express app) from `server.js` (calls `app.listen`) so that tests can import the app and start their own server on a random port.

## Conclusion

Node.js is a mature, high-performance platform for building server-side applications with JavaScript or TypeScript. Understanding its event loop, module system, and asynchronous patterns is the foundation for writing efficient, production-ready code. Combined with Express for HTTP APIs, Zod for validation, and Docker for deployment, Node.js provides a complete toolkit for modern backend development.

In the next part of this series, we will take these fundamentals to the cloud and explore how to deploy and run Node.js applications on AWS using Lambda, DynamoDB, S3, SQS, and more.

## References

- [Node.js Official Documentation](https://nodejs.org/docs/latest/api/)
- [Express.js Guide](https://expressjs.com/en/guide/routing.html)
- [Node.js Release Schedule](https://github.com/nodejs/release#release-schedule)
- [Zod Documentation](https://zod.dev/)
- [Pino Logger](https://getpino.io/)
