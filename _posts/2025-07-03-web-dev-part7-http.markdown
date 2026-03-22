---
layout: post
title: "Web Development Fundamentals — Part 7: HTTP — How the Web Communicates"
date: 2025-07-03 00:00:00 +0000
tags:
  - web
  - http
  - networking
  - backend
  - beginner
series: "Web Development Fundamentals"
series_part: 7
---


## Series Overview

This is an 8-part series covering the core technologies of web development:

1. [HTML — The Structure of the Web](2025-04-web-dev-part1-html.md) — Elements, attributes, block vs inline, and document structure
2. [CSS — Styling the Web](2025-04-web-dev-part2-css.md) — Selectors, properties, the box model, and layout basics
3. [JavaScript — Adding Interactivity](2025-05-web-dev-part3-javascript.md) — Variables, data types, functions, arrays, and control flow
4. [The DOM — Connecting JavaScript to HTML](2025-05-web-dev-part4-dom.md) — Selecting elements, changing styles, and modifying attributes
5. [jQuery — Write Less, Do More](2025-06-web-dev-part5-jquery.md) — Selectors, events, and DOM manipulation with jQuery
6. [Bootstrap — Responsive Layouts Made Easy](2025-06-web-dev-part6-bootstrap.md) — Grid system, responsive breakpoints, and rapid prototyping
7. **HTTP — How the Web Communicates** (this article) — Requests, responses, methods, and status codes
8. [React — Building Modern User Interfaces](2025-07-web-dev-part8-react.md) — Components, JSX, props, state, hooks, and thinking in React


## What Is HTTP?

HTTP (HyperText Transfer Protocol) is a **client-server protocol**. Requests are sent by one entity — the **user-agent** (usually a web browser, but it can be anything, like a search engine crawler or a mobile app) — to a **server**, which handles the request and provides a **response**.

Every time you load a web page, your browser is making HTTP requests and receiving HTTP responses. Understanding this protocol is essential for web development, whether you're building frontend applications that consume APIs or backend services that serve them.

## The Request

An HTTP request consists of:

1. **HTTP method** — A verb like `GET` or `POST` that defines what the client wants to do
2. **Path** — The URL of the resource (e.g., `/api/users`)
3. **HTTP version** — e.g., `HTTP/1.1` or `HTTP/2`
4. **Headers** — Optional metadata (e.g., `Content-Type`, `Authorization`)
5. **Body** — Optional data sent with the request (used with `POST`, `PUT`, etc.)

Example raw request:

```
GET /index.html HTTP/1.1
Host: www.example.com
Accept: text/html
```

## HTTP Methods

HTTP methods indicate the desired action to be performed on a resource. Each method has different semantics, but they can be grouped by properties like safety (read-only) and idempotency (same result regardless of how many times called).

### The Five Core Methods

| Method | Purpose | Safe | Idempotent |
|--------|---------|------|------------|
| `GET` | Retrieve a resource | Yes | Yes |
| `POST` | Create a resource or submit data | No | No |
| `PUT` | Replace a resource entirely | No | Yes |
| `DELETE` | Remove a resource | No | Yes |
| `HEAD` | Same as GET but without the response body | Yes | Yes |

**GET** — Requests a representation of a resource. Should only retrieve data, never modify it.

**POST** — Submits data to a resource, often causing a change in state or side effects on the server (creating a record, submitting a form, triggering a process).

**PUT** — Replaces the entire target resource with the request payload. If you send a PUT to `/users/1`, you're saying "here is the complete new state of user 1."

**DELETE** — Removes the specified resource.

**HEAD** — Identical to GET but returns only headers, no body. Useful for checking if a resource exists or reading metadata without downloading content.

## The Response

An HTTP response consists of:

1. **HTTP version** — e.g., `HTTP/1.1`
2. **Status code** — A three-digit number indicating the result
3. **Status message** — A short description (e.g., "OK", "Not Found")
4. **Headers** — Metadata about the response
5. **Body** (optional) — The content of the response (HTML, JSON, an image, etc.)

Example raw response:

```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1234

<!DOCTYPE html>
<html>...
```

## Status Codes

Status codes tell the client what happened with their request. They're grouped into five categories by their first digit.

### 1xx — Informational

The server received the request and is continuing to process it.

| Code | Message | Meaning |
|------|---------|---------|
| 100 | Continue | Initial part of request received, client should continue |
| 101 | Switching Protocols | Server is changing protocols as requested |

### 2xx — Success

The request was successfully received, understood, and accepted.

| Code | Message | Meaning |
|------|---------|---------|
| **200** | **OK** | Standard success response |
| **201** | **Created** | New resource was created (typically after POST) |
| 202 | Accepted | Request accepted but processing not complete |
| 204 | No Content | Success but no content to return (common for DELETE) |

### 3xx — Redirection

The client needs to take additional action to complete the request.

| Code | Message | Meaning |
|------|---------|---------|
| **301** | **Moved Permanently** | Resource has a new permanent URL — update your bookmarks |
| **302** | **Found** | Resource temporarily at a different URL |
| **304** | **Not Modified** | Cached version is still valid, no need to re-download |
| 307 | Temporary Redirect | Like 302, but the HTTP method must not change |
| 308 | Permanent Redirect | Like 301, but the HTTP method must not change |

### 4xx — Client Error

The request contains an error on the client's side.

| Code | Message | Meaning |
|------|---------|---------|
| **400** | **Bad Request** | Malformed syntax or invalid parameters |
| **401** | **Unauthorized** | Authentication is required (login needed) |
| **403** | **Forbidden** | Authenticated but not authorized (no permission) |
| **404** | **Not Found** | The resource does not exist |
| **405** | **Method Not Allowed** | The HTTP method is not supported for this resource |
| 408 | Request Timeout | The server timed out waiting for the request |
| 409 | Conflict | Request conflicts with the current state of the resource |
| 410 | Gone | Resource was intentionally removed and won't be back |
| 413 | Request Entity Too Large | The request body exceeds the server's size limit |
| 415 | Unsupported Media Type | The request format is not supported |

### 5xx — Server Error

The server failed to fulfil a valid request.

| Code | Message | Meaning |
|------|---------|---------|
| **500** | **Internal Server Error** | Generic server-side error |
| 501 | Not Implemented | The server doesn't support the functionality required |
| **502** | **Bad Gateway** | The server (acting as gateway/proxy) got an invalid response from upstream |
| **503** | **Service Unavailable** | Server is overloaded or down for maintenance (usually temporary) |
| 504 | Gateway Timeout | The gateway/proxy didn't get a timely response from upstream |

### Status Codes You'll See Most Often

In day-to-day web development, these are the ones that matter most:

- **200** — Everything worked
- **201** — Resource created (after a POST)
- **204** — Success, no content to return (after a DELETE)
- **301/302** — Redirects
- **400** — Bad input from the client
- **401** — Not logged in
- **403** — Logged in but not allowed
- **404** — Resource not found
- **500** — Server error (check the logs)
- **503** — Service temporarily down

## What's Next?

With the fundamentals in place — HTML, CSS, JavaScript, the DOM, and HTTP — you're ready to build modern user interfaces. In [Part 8](2025-07-web-dev-part8-react.md), we'll learn **React**, the most popular JavaScript library for building component-based, declarative UIs.

## References

- [MDN — HTTP Overview](https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview)
- [MDN — HTTP Status Codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)
- [HTTP Specification (RFC 7231)](https://tools.ietf.org/html/rfc7231)
