---
layout: post
title: "Modular Feature Architecture"
date: 2026-03-10 00:00:00 +0000
tags:
  - react
  - architecture
  - tanstack-router
  - tanstack-query
  - typescript
series: "Building Modern React SPAs"
series_part: 5
---


## Series Overview

1. [TypeScript Patterns for React](/posts/react-spa-part1-typescript-patterns/)
2. [Type-Safe Routing with TanStack Router](/posts/react-spa-part2-tanstack-router/)
3. [Server State with TanStack Query](/posts/react-spa-part3-tanstack-query/)
4. [Styling with Tailwind CSS v4 & shadcn/ui](/posts/react-spa-part4-tailwind-shadcn/)
5. **Modular Feature Architecture** (this post)
6. [Deploy to AWS with S3, CloudFront & CDK](/posts/react-spa-part6-deploy-aws/)

## Introduction

With TypeScript patterns, routing, server state, and styling in place, the remaining challenge is organizing it all so the codebase scales. A small project can get away with a flat structure, but once you have multiple developers and a dozen features, code organization becomes a make-or-break concern.

This post presents a modular feature architecture built on four principles: feature cohesion, explicit boundaries, route-driven code splitting, and router-as-authority. We will cover the complete folder structure, authentication architecture, protected route patterns, feature module anatomy, shared code rules, environment configuration, dependency rules, and testing strategy.

## Guiding Principles

1. **Feature cohesion** -- Code is organized by business domain (auth, dashboard, posts), not by technical type (components, hooks, utils). When you work on the auth feature, everything you need is in one folder.
2. **Explicit boundaries** -- Features expose a public API via an `index.ts` barrel file. Other features import only through that API, never reaching into internal files.
3. **Route-driven code splitting** -- Each route loads only the feature code it needs. TanStack Router's `autoCodeSplitting` handles this automatically.
4. **Router-as-authority** -- Authentication, authorization, and data prefetching happen in the router layer (via `beforeLoad` and `loader`), not scattered across components.

## Complete Folder Structure

```
src/
├── routes/                          # TanStack Router file-based routes
│   ├── __root.tsx                   #   Root layout (nav, providers, devtools)
│   ├── index.tsx                    #   Landing page (/)
│   ├── _public/                     #   Public routes (no auth required)
│   │   ├── route.tsx                #     Public layout
│   │   ├── about.tsx                #     /about
│   │   ├── pricing.tsx              #     /pricing
│   │   └── contact.tsx              #     /contact
│   ├── _auth/                       #   Auth routes (redirect if logged in)
│   │   ├── route.tsx                #     Auth layout (centered card)
│   │   ├── login.tsx                #     /login
│   │   ├── register.tsx             #     /register
│   │   └── forgot-password.tsx      #     /forgot-password
│   ├── _authenticated/              #   Protected routes (require login)
│   │   ├── route.tsx                #     Auth guard + app shell layout
│   │   ├── dashboard.tsx            #     /dashboard
│   │   ├── settings/
│   │   │   ├── route.tsx            #     /settings layout (tabs)
│   │   │   ├── index.tsx            #     /settings (profile tab)
│   │   │   └── security.tsx         #     /settings/security
│   │   └── posts/
│   │       ├── route.tsx            #     /posts layout
│   │       ├── index.tsx            #     /posts (list)
│   │       ├── new.tsx              #     /posts/new (create)
│   │       └── $postId.tsx          #     /posts/:postId (detail)
│   └── _admin/                      #   Admin routes (require admin role)
│       ├── route.tsx                #     Admin guard + admin layout
│       ├── users.tsx                #     /admin/users
│       └── analytics.tsx            #     /admin/analytics
│
├── features/                        # Feature modules (business domains)
│   ├── auth/
│   │   ├── api/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── queries/
│   │   ├── types/
│   │   ├── utils/
│   │   └── index.ts                 # Public API
│   ├── posts/
│   │   ├── api/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── queries/
│   │   ├── types/
│   │   └── index.ts
│   ├── dashboard/
│   │   ├── components/
│   │   ├── queries/
│   │   └── index.ts
│   └── admin/
│       ├── api/
│       ├── components/
│       ├── queries/
│       └── index.ts
│
├── shared/                          # Shared code (used by 2+ features)
│   ├── components/
│   │   ├── ui/                      #   shadcn/ui primitives
│   │   ├── layout/                  #   AppShell, Sidebar, Header
│   │   └── feedback/                #   ErrorFallback, EmptyState, LoadingSkeleton
│   ├── hooks/
│   ├── lib/                         #   api-client, query-client, zod-schemas
│   ├── types/
│   └── utils/
│
├── config/                          # App configuration
│   ├── env.ts
│   ├── routes.ts
│   └── constants.ts
│
├── main.tsx                         # Entry point
└── routeTree.gen.ts                 # Auto-generated (DO NOT EDIT)
```

## Why This Structure?

### Features Instead of Type-Based Folders

The traditional structure groups all components together, all hooks together, all services together. This works for small apps but breaks down as the codebase grows. To work on the "posts" feature, you jump between `components/PostCard.tsx`, `hooks/usePosts.ts`, `services/postsApi.ts`, and `types/post.ts` across four different directories.

Feature-based organization puts everything related to "posts" in `features/posts/`. This mirrors how teams think about work ("I'm building the posts feature") and how code changes cluster ("this PR modifies the posts feature").

### Routes Separate from Features

Routes are the **composition layer** -- they import from features, wire them together, and define the URL structure. Features are the **business logic layer** -- they contain components, hooks, API calls, and types that are domain-specific but route-agnostic.

This separation means a feature's components can be reused across multiple routes, routes stay thin (mostly imports + layout), and you can reorganize URL structure without touching feature code.

### Shared Code Rules

Code lives in `shared/` only when it is used by two or more features. If a component is only used within one feature, it stays in that feature's folder.

| Put in `shared/` | Keep in `features/` |
|-------------------|---------------------|
| Used by 2+ features | Used by 1 feature only |
| Design system components (Button, Input) | Feature-specific forms (LoginForm) |
| API client configuration | Feature-specific API calls |
| Generic hooks (useDebounce) | Feature-specific hooks (useAuth) |
| Common types (PaginatedResponse) | Feature types (Post, User) |

## Authentication Architecture

### Step 1: Auth Context Type in the Router

Pass auth state through the router context so every route can access it:

```tsx
// src/features/auth/types/auth.types.ts
export interface AuthUser {
  id: string;
  email: string;
  name: string;
  role: 'user' | 'admin';
}

export interface AuthContext {
  user: AuthUser | null;
  isAuthenticated: boolean;
  isAdmin: boolean;
  login: (credentials: LoginCredentials) => Promise<void>;
  logout: () => Promise<void>;
}
```

### Step 2: Auth Hook

```tsx
// src/features/auth/hooks/useAuth.ts
export function useAuth(): AuthContext {
  const [user, setUser] = useState<AuthUser | null>(null);

  const login = useCallback(async (credentials) => {
    const { token, user } = await loginApi(credentials);
    setToken(token);
    setUser(user);
  }, []);

  const logout = useCallback(async () => {
    await logoutApi();
    removeToken();
    setUser(null);
  }, []);

  return useMemo(() => ({
    user,
    isAuthenticated: !!user,
    isAdmin: user?.role === 'admin',
    login,
    logout,
  }), [user, login, logout]);
}
```

### Step 3: Root Route with Auth Context

```tsx
// src/routes/__root.tsx
interface RouterContext {
  queryClient: QueryClient;
  auth: AuthContext;
}

export const Route = createRootRouteWithContext<RouterContext>()({
  component: () => <Outlet />,
});
```

### Step 4: Entry Point Wiring

```tsx
// src/main.tsx
const router = createRouter({
  routeTree,
  context: {
    queryClient,
    auth: undefined!,  // Will be provided by InnerApp
  },
  defaultPreload: 'intent',
});

declare module '@tanstack/react-router' {
  interface Register { router: typeof router }
}

function InnerApp() {
  const auth = useAuth();
  return <RouterProvider router={router} context={{ auth }} />;
}

ReactDOM.createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <QueryClientProvider client={queryClient}>
      <InnerApp />
    </QueryClientProvider>
  </StrictMode>,
);
```

The `context: { auth: undefined! }` pattern tells TypeScript "I promise this will be provided at runtime." The actual value is passed via `<RouterProvider context={{ auth }}>`.

## Protected Route Patterns

### Pattern 1: Authenticated Route Group

All routes under `_authenticated/` require login:

```tsx
// src/routes/_authenticated/route.tsx
export const Route = createFileRoute('/_authenticated')({
  beforeLoad: ({ context }) => {
    if (!context.auth.isAuthenticated) {
      throw redirect({
        to: '/login',
        search: { redirect: location.pathname },
      });
    }
  },
  component: () => (
    <AppShell>
      <Outlet />
    </AppShell>
  ),
});
```

Every child route inherits this guard automatically. No need to repeat the auth check on each route.

### Pattern 2: Admin Route Group

```tsx
// src/routes/_admin/route.tsx
export const Route = createFileRoute('/_admin')({
  beforeLoad: ({ context }) => {
    if (!context.auth.isAuthenticated) {
      throw redirect({ to: '/login' });
    }
    if (!context.auth.isAdmin) {
      throw redirect({ to: '/dashboard' });
    }
  },
  component: () => (
    <AdminLayout>
      <Outlet />
    </AdminLayout>
  ),
});
```

### Pattern 3: Auth Routes (Redirect if Already Logged In)

```tsx
// src/routes/_auth/route.tsx
export const Route = createFileRoute('/_auth')({
  beforeLoad: ({ context }) => {
    if (context.auth.isAuthenticated) {
      throw redirect({ to: '/dashboard' });
    }
  },
  component: () => (
    <div className="auth-layout">
      <div className="auth-card">
        <Outlet />
      </div>
    </div>
  ),
});
```

### Pattern 4: Invalidate on Auth Change

When the user logs in or out, invalidate the router so guards re-run:

```tsx
const login = async (credentials) => {
  await loginApi(credentials);
  queryClient.invalidateQueries();
  router.invalidate();
};
```

## Feature Module Anatomy

Each feature follows the same internal structure:

```
features/posts/
├── api/
│   └── posts.api.ts          # fetch/axios calls, returns typed data
├── components/
│   ├── PostCard.tsx
│   ├── PostList.tsx
│   ├── PostForm.tsx
│   └── PostDetail.tsx
├── hooks/
│   └── usePosts.ts           # Wraps useQuery/useMutation if needed
├── queries/
│   └── posts.queries.ts      # queryOptions factories + query keys
├── types/
│   └── posts.types.ts        # Post, CreatePostInput, PostFilters
├── utils/
│   └── formatPost.ts
└── index.ts                  # Public API
```

### The Public API (index.ts)

```tsx
// features/posts/index.ts
export { PostCard } from './components/PostCard';
export { PostList } from './components/PostList';
export { PostForm } from './components/PostForm';
export { PostDetail } from './components/PostDetail';
export { postListOptions, postDetailOptions } from './queries/posts.queries';
export type { Post, CreatePostInput } from './types/posts.types';
```

Routes and other features import from `@/features/posts`, never from `@/features/posts/components/PostCard` directly. This lets you refactor internal structure without breaking imports.

### Query Options Factory

```tsx
// features/posts/queries/posts.queries.ts
import { queryOptions } from '@tanstack/react-query';

export const postKeys = {
  all:     ['posts'] as const,
  lists:   () => [...postKeys.all, 'list'] as const,
  list:    (filters: PostFilters) => [...postKeys.lists(), filters] as const,
  details: () => [...postKeys.all, 'detail'] as const,
  detail:  (id: string) => [...postKeys.details(), id] as const,
};

export function postListOptions(filters: PostFilters = {}) {
  return queryOptions({
    queryKey: postKeys.list(filters),
    queryFn: () => fetchPosts(filters),
    staleTime: 60_000,
  });
}
```

### Route Consuming the Feature

```tsx
// src/routes/_authenticated/posts/index.tsx
import { createFileRoute } from '@tanstack/react-router';
import { useQuery } from '@tanstack/react-query';
import { PostList, postListOptions } from '@/features/posts';

export const Route = createFileRoute('/_authenticated/posts/')({
  loader: ({ context: { queryClient } }) =>
    queryClient.ensureQueryData(postListOptions()),
  component: PostsPage,
});

function PostsPage() {
  const { data: posts } = useQuery(postListOptions());
  return <PostList posts={posts ?? []} />;
}
```

The route is thin -- it imports the feature's components and query options, wires them to the loader, and renders. All business logic lives in the feature.

## Environment Configuration

```tsx
// src/config/env.ts
import { z } from 'zod';

const envSchema = z.object({
  API_BASE_URL: z.string().url(),
  APP_NAME: z.string().default('My App'),
  ENABLE_DEVTOOLS: z.boolean().default(true),
});

export const env = envSchema.parse({
  API_BASE_URL: import.meta.env.VITE_API_BASE_URL,
  APP_NAME: import.meta.env.VITE_APP_NAME,
  ENABLE_DEVTOOLS: import.meta.env.DEV,
});
```

## Dependency Rules

To prevent spaghetti imports, enforce these rules (manually or via ESLint):

```
routes/       -> can import from features/, shared/, config/
features/X    -> can import from shared/, config/
features/X    -> CANNOT import from features/Y
features/X    -> CANNOT import from routes/
shared/       -> can import from config/
shared/       -> CANNOT import from features/ or routes/
config/       -> CANNOT import from anything in src/
```

If two features need to share data, lift the shared code to `shared/` or communicate through the router context or query cache.

## Testing Strategy

### Colocate Tests with Source

```
features/posts/
├── components/
│   ├── PostCard.tsx
│   └── PostCard.test.tsx
├── hooks/
│   ├── usePosts.ts
│   └── usePosts.test.ts
```

### Test Layers

| Layer | Tool | What to Test |
|-------|------|-------------|
| Components | Vitest + React Testing Library | Renders correctly, user interactions |
| Hooks | Vitest + `renderHook` | Returns correct data, handles states |
| API functions | Vitest + MSW | Correct requests, error handling |
| Routes | Vitest + Router testing utils | Guards redirect correctly, loaders prefetch |
| E2E | Playwright | Full user flows (login -> dashboard -> create post) |

## Adding a New Feature: Checklist

1. Create the feature directory under `src/features/my-feature/`
2. Define types in `types/my-feature.types.ts`
3. Build the API layer in `api/my-feature.api.ts`
4. Create query options in `queries/my-feature.queries.ts` with key factory + `queryOptions`
5. Build components in `components/`
6. Export the public API in `index.ts`
7. Create the route under `src/routes/` in the appropriate group
8. Wire the loader with `queryClient.ensureQueryData()`
9. Colocate tests next to the files they test

## Scaling Patterns

### Domain Grouping (10+ Features)

```
src/features/
├── @content/           # Content domain
│   ├── posts/
│   ├── comments/
│   └── categories/
├── @identity/          # Identity domain
│   ├── auth/
│   ├── profile/
│   └── settings/
└── @admin/             # Admin domain
    ├── users/
    └── analytics/
```

### Team Boundaries

Feature boundaries become team boundaries. Each team owns their feature directories and the routes that consume them. The `shared/` directory becomes a shared library maintained by a platform team.

## Conclusion

This architecture gives you a clear place for every piece of code: features own business logic, routes handle composition and navigation, and shared code serves everyone. Combined with TanStack Router's pathless layout routes for auth guards and TanStack Query's `queryOptions` pattern for data management, the result is a codebase that scales from solo developer to multi-team organization.

In the final post of this series, we will deploy our SPA to AWS using S3, CloudFront, and CDK -- the infrastructure layer that takes our application from localhost to production.

## References

- [TanStack Router Authenticated Routes](https://tanstack.com/router/v1/docs/guide/authenticated-routes)
- [TanStack Router Auth Setup](https://tanstack.com/router/v1/docs/framework/react/how-to/setup-authentication)
- [TanStack Router File-Based Routing](https://tanstack.com/router/v1/docs/framework/react/routing/file-based-routing)
- [TanStack Query queryOptions](https://tanstack.com/query/latest/docs/framework/react/guides/query-options)
- [React Folder Structure (Robin Wieruch)](https://www.robinwieruch.de/react-folder-structure/)
- [Feature-Based Structure (DEV Community)](https://dev.to/itswillt/folder-structures-in-react-projects-3dp8)
