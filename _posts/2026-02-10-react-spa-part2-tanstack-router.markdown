---
layout: post
title: "Type-Safe Routing with TanStack Router"
date: 2026-02-10 00:00:00 +0000
tags:
  - react
  - tanstack-router
  - routing
  - typescript
series: "Building Modern React SPAs"
series_part: 2
---


## Series Overview

1. [TypeScript Patterns for React](/posts/react-spa-part1-typescript-patterns/)
2. **Type-Safe Routing with TanStack Router** (this post)
3. [Server State with TanStack Query](/posts/react-spa-part3-tanstack-query/)
4. [Styling with Tailwind CSS v4 & shadcn/ui](/posts/react-spa-part4-tailwind-shadcn/)
5. [Modular Feature Architecture](/posts/react-spa-part5-architecture/)
6. [Deploy to AWS with S3, CloudFront & CDK](/posts/react-spa-part6-deploy-aws/)

## Introduction

Routing is the backbone of any single-page application. TanStack Router was designed from the ground up with TypeScript as a first-class concern. Unlike React Router, which adds type definitions on top of runtime logic, TanStack Router designs its API so that types flow through generics from route definitions all the way to `<Link>` components. Mistyped paths, missing params, and invalid search params are caught at compile time -- not in production.

This post covers everything you need to build a fully type-safe routing layer: file-based route generation, dynamic parameters, search params as first-class state, layout routes, data loading, error handling, code splitting, and Vite plugin configuration.

## Why TanStack Router Over React Router

Key advantages for SPAs:

- **100% type-safe navigation** -- `<Link to="/posts/$postId" params={{ postId: '42' }}>` autocompletes paths and enforces required params. A typo in the path or a missing param is a TypeScript error.
- **Search params as first-class state** -- validated with schemas (Zod, Valibot), type-safe, JSON-serializable, and reactive. TanStack describes this as "useState, but in the URL."
- **Built-in data loading** -- route `loader` functions run before rendering, avoiding render-then-fetch waterfalls. Loaders integrate natively with TanStack Query.
- **Automatic code splitting** -- the Vite plugin wraps each route in `React.lazy()` automatically when `autoCodeSplitting: true` is set.
- **File-based route generation** -- a Vite plugin watches `src/routes/` and generates a typed route tree. No manual route registration.

## File-Based Routing

TanStack Router supports three styles of route organization: directory-based, flat (dot-notation), and mixed. The Vite plugin generates `routeTree.gen.ts` automatically from your `src/routes/` directory.

### Core File Naming Rules

| File/Folder | Route | Purpose |
|-------------|-------|---------|
| `__root.tsx` | -- | Root layout, wraps ALL routes |
| `index.tsx` | `/` | Index (home) route |
| `about.tsx` | `/about` | Static route |
| `posts.tsx` | `/posts` | Layout route for `/posts/*` children |
| `posts/index.tsx` | `/posts` | Index page within the posts layout |
| `posts/$postId.tsx` | `/posts/:postId` | Dynamic param route |
| `_auth.tsx` | -- | Pathless layout (no URL segment) |
| `_auth/login.tsx` | `/login` | Child of pathless layout |
| `posts_.$postId.edit.tsx` | `/posts/:postId/edit` | Non-nested route (breaks out of parent layout) |
| `-components/Button.tsx` | -- | Excluded from routing (prefix `-`) |

### Directory-Based Example

```
src/routes/
├── __root.tsx
├── index.tsx
├── about.tsx
├── posts/
│   ├── route.tsx          # /posts layout (renders <Outlet/>)
│   ├── index.tsx          # /posts (list page)
│   └── $postId.tsx        # /posts/:postId (detail page)
├── _authenticated/
│   ├── route.tsx          # Pathless layout (auth guard)
│   ├── dashboard.tsx      # /dashboard
│   └── settings.tsx       # /settings
└── _authenticated.tsx     # Pathless layout config
```

### Flat (Dot-Notation) Example

The same structure using dots instead of directories:

```
src/routes/
├── __root.tsx
├── index.tsx
├── about.tsx
├── posts.tsx
├── posts.index.tsx
├── posts.$postId.tsx
├── _authenticated.tsx
├── _authenticated.dashboard.tsx
└── _authenticated.settings.tsx
```

Use **directories** when a route has multiple children -- it groups related files together and keeps filenames short. Use **flat dot-notation** for leaf routes or when you have only one or two children. Most projects end up with a mixed approach.

## The Root Route

The root route (`__root.tsx`) wraps every other route and is always rendered. Use it for global layout elements, provider wrappers, and devtools.

```tsx
import {
  createRootRouteWithContext,
  Link,
  Outlet,
} from '@tanstack/react-router';
import { TanStackRouterDevtools } from '@tanstack/react-router-devtools';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import type { QueryClient } from '@tanstack/react-query';

interface RouterContext {
  queryClient: QueryClient;
}

export const Route = createRootRouteWithContext<RouterContext>()({
  component: RootLayout,
  errorComponent: ({ error }) => (
    <div>
      <h1>Something went wrong</h1>
      <pre>{error.message}</pre>
    </div>
  ),
  notFoundComponent: () => (
    <div>
      <h1>404 -- Page Not Found</h1>
      <Link to="/">Go Home</Link>
    </div>
  ),
});

function RootLayout() {
  return (
    <>
      <header>
        <nav>
          <Link to="/" activeProps={{ className: 'active' }}>Home</Link>
          <Link to="/about" activeProps={{ className: 'active' }}>About</Link>
          <Link to="/posts" activeProps={{ className: 'active' }}>Posts</Link>
        </nav>
      </header>
      <main>
        <Outlet />
      </main>
      <TanStackRouterDevtools position="bottom-right" />
      <ReactQueryDevtools initialIsOpen={false} />
    </>
  );
}
```

Use `createRootRouteWithContext<T>()` when you need to pass dependencies (like `QueryClient`, auth state, or feature flags) through the router context. Every route's `loader`, `beforeLoad`, and component can access this context with full type safety.

## Dynamic Routes and Path Parameters

Dynamic segments use the `$` prefix in filenames:

```tsx
// src/routes/posts/$postId.tsx
import { createFileRoute } from '@tanstack/react-router';

export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params, context: { queryClient } }) => {
    return queryClient.ensureQueryData(postQueryOptions(params.postId));
  },
  component: PostPage,
});

function PostPage() {
  const { postId } = Route.useParams();
  // postId is typed as string, guaranteed by the router
}
```

### Catch-All (Splat) Routes

Use `$.tsx` to capture the rest of the URL:

```tsx
// src/routes/files/$.tsx
export const Route = createFileRoute('/files/$')({
  component: FileBrowser,
});

function FileBrowser() {
  const { _splat } = Route.useParams();
  // e.g., /files/documents/report.pdf -> _splat = "documents/report.pdf"
}
```

## Search Params: Type-Safe URL State

This is TanStack Router's most distinctive feature. Search params are validated, typed, and reactive -- not raw strings.

### Validation with Zod (Recommended)

```tsx
import { z } from 'zod';

const productSearchSchema = z.object({
  page: z.number().catch(1),
  sort: z.enum(['newest', 'oldest', 'price']).catch('newest'),
  category: z.string().optional(),
  inStock: z.boolean().catch(false),
});

export const Route = createFileRoute('/products')({
  validateSearch: productSearchSchema,
  component: ProductsPage,
});

function ProductsPage() {
  const { page, sort, category } = Route.useSearch();
  // All fully typed
}
```

### Writing Search Params from Components

```tsx
import { Link, useNavigate } from '@tanstack/react-router';

// Via Link: type-safe, autocompleted
<Link to="/products" search={{ page: 2, sort: 'price' }}>
  Page 2
</Link>

// Via navigate: imperative
const navigate = useNavigate();
navigate({
  to: '/products',
  search: (prev) => ({ ...prev, page: prev.page + 1 }),
});
```

Declaring the schema in the route definition creates a single source of truth. Every `<Link>` in the app is type-checked against that schema. Parent route schemas are inherited by children, so shared params like `locale` or `theme` can be defined once on a parent layout.

## Layout Routes and Pathless Layouts

### Layout Routes (With a URL Segment)

A layout route renders at a specific path and wraps its children with shared UI:

```tsx
// src/routes/posts/route.tsx
export const Route = createFileRoute('/posts')({
  component: PostsLayout,
});

function PostsLayout() {
  return (
    <div>
      <h1>Blog Posts</h1>
      <aside>Sidebar content here</aside>
      <Outlet />
    </div>
  );
}
```

### Pathless Layout Routes (No URL Segment)

Prefix the filename with `_` to create a layout that wraps children without adding a path segment. This is ideal for auth guards, theme wrappers, or shared sidebars:

```tsx
// src/routes/_authenticated.tsx
export const Route = createFileRoute('/_authenticated')({
  beforeLoad: async ({ context }) => {
    if (!context.auth?.isLoggedIn) {
      throw redirect({ to: '/login' });
    }
  },
  component: () => (
    <div className="authenticated-layout">
      <Sidebar />
      <Outlet />
    </div>
  ),
});
```

Children of this layout live in `src/routes/_authenticated/` and their URLs have no `_authenticated` segment -- it is purely a logical grouping.

### Non-Nested Routes

Sometimes a child route should not inherit the parent's layout. Use `_` after the parent segment to break out:

```tsx
// src/routes/posts_.$postId.edit.tsx
// URL: /posts/:postId/edit
// Does NOT render inside the /posts layout
```

## Route Loaders and beforeLoad

### Loader: Fetch Data Before Rendering

Loaders run in parallel for all matched routes, avoiding serial waterfall requests:

```tsx
export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params, context: { queryClient } }) => {
    const post = await queryClient.ensureQueryData(
      postQueryOptions(params.postId)
    );
    return { post };
  },
  component: PostPage,
});

function PostPage() {
  const { post } = Route.useLoaderData();
}
```

### beforeLoad: Guards, Redirects, and Context Enrichment

`beforeLoad` runs before the loader and is the right place for auth checks:

```tsx
export const Route = createFileRoute('/admin')({
  beforeLoad: async ({ context }) => {
    if (!context.auth?.isAdmin) {
      throw redirect({ to: '/login', search: { redirect: '/admin' } });
    }
    return { adminPermissions: await fetchPermissions() };
  },
  loader: async ({ context }) => {
    return fetchAdminDashboard(context.adminPermissions);
  },
});
```

### Loader vs TanStack Query

Use **loaders** to prefetch data via `queryClient.ensureQueryData()`. Use **`useQuery`** in the component to read from the cache and get reactive updates (background refetching, stale indicators). This "prefetch in loader, read in component" pattern gives you instant navigation AND live data updates.

## Error Handling

Each route can define its own error boundary:

```tsx
export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params }) => {
    const post = await fetchPost(params.postId);
    if (!post) throw new Error('Post not found');
    return post;
  },
  errorComponent: ({ error, reset }) => (
    <div>
      <h2>Failed to load post</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  ),
  pendingComponent: () => <div>Loading post...</div>,
});
```

## Code Splitting

With `autoCodeSplitting: true` in the Vite plugin config, each route's component is automatically lazy-loaded. No manual `React.lazy()` calls needed.

For manual control, split components from route configuration using lazy routes:

```tsx
// src/routes/about.tsx -- just the config
export const Route = createFileRoute('/about')({
  // loader, validateSearch, etc. stay here (small bundle)
});

// src/routes/about.lazy.tsx -- the component (loaded on demand)
import { createLazyFileRoute } from '@tanstack/react-router';

export const Route = createLazyFileRoute('/about')({
  component: AboutPage,
});

function AboutPage() {
  return <div>About us</div>;
}
```

This keeps route configuration in the main bundle while deferring heavy component code until the route is visited.

## Vite Plugin Configuration

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { tanstackRouter } from '@tanstack/router-plugin/vite';

export default defineConfig({
  plugins: [
    // MUST come before react()
    tanstackRouter({
      target: 'react',
      autoCodeSplitting: true,
      routesDirectory: './src/routes',
      generatedRouteTree: './src/routeTree.gen.ts',
      routeFileIgnorePrefix: '-',
      quoteStyle: 'single',
    }),
    react(),
  ],
});
```

Remember to add `src/routeTree.gen.ts` to `.prettierignore`, `.eslintignore`, and optionally `.gitignore`.

## Complete Entry Point

```tsx
// src/main.tsx
import { StrictMode } from 'react';
import ReactDOM from 'react-dom/client';
import { RouterProvider, createRouter } from '@tanstack/react-router';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { routeTree } from './routeTree.gen';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60,
      gcTime: 1000 * 60 * 5,
      refetchOnWindowFocus: false,
    },
  },
});

const router = createRouter({
  routeTree,
  context: { queryClient },
  defaultPendingComponent: () => <div>Loading...</div>,
  defaultErrorComponent: ({ error }) => <div>Error: {error.message}</div>,
  defaultPreload: 'intent',
});

declare module '@tanstack/react-router' {
  interface Register {
    router: typeof router;
  }
}

const rootElement = document.getElementById('root')!;
if (!rootElement.innerHTML) {
  ReactDOM.createRoot(rootElement).render(
    <StrictMode>
      <QueryClientProvider client={queryClient}>
        <RouterProvider router={router} />
      </QueryClientProvider>
    </StrictMode>,
  );
}
```

Setting `defaultPreload: 'intent'` preloads route data and code when a user hovers over or focuses a `<Link>`. Combined with TanStack Query caching, this makes navigations feel instant.

## Conclusion

TanStack Router brings compile-time safety to every aspect of client-side routing: paths, parameters, search state, loaders, and navigation. Combined with file-based route generation and automatic code splitting, it eliminates an entire class of runtime errors while reducing boilerplate.

In the next post, we will explore TanStack Query -- the server-state management layer that pairs perfectly with TanStack Router's loader system to deliver instant navigations with live data updates.

## References

- [TanStack Router Documentation](https://tanstack.com/router/latest/docs/overview)
- [File-Based Routing](https://tanstack.com/router/v1/docs/framework/react/routing/file-based-routing)
- [Search Params Guide](https://tanstack.com/router/v1/docs/framework/react/guide/search-params)
- [Code Splitting](https://tanstack.com/router/v1/docs/framework/react/guide/code-splitting)
- [Data Loading](https://tanstack.com/router/v1/docs/framework/react/guide/data-loading)
- [Vite Installation](https://tanstack.com/router/v1/docs/framework/react/installation/with-vite)
