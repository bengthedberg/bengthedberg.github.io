---
layout: post
title: "TypeScript Patterns for React"
date: 2026-02-01 00:00:00 +0000
tags:
  - react
  - typescript
  - patterns
  - frontend
series: "Building Modern React SPAs"
series_part: 1
---


## Series Overview

1. **TypeScript Patterns for React** (this post)
2. [Type-Safe Routing with TanStack Router](/posts/react-spa-part2-tanstack-router/)
3. [Server State with TanStack Query](/posts/react-spa-part3-tanstack-query/)
4. [Styling with Tailwind CSS v4 & shadcn/ui](/posts/react-spa-part4-tailwind-shadcn/)
5. [Modular Feature Architecture](/posts/react-spa-part5-architecture/)
6. [Deploy to AWS with S3, CloudFront & CDK](/posts/react-spa-part6-deploy-aws/)

## Introduction

TypeScript and React are a powerful combination, but writing truly type-safe React code requires more than just adding `.tsx` to your file extensions. This post covers the patterns, conventions, and practical techniques that make TypeScript work for you in a React codebase rather than against you.

We will walk through component typing, props patterns, discriminated unions, generic components, hooks, event handling, context, custom hooks, utility types, API response typing, performance patterns, and file naming conventions. Each section includes real code examples you can adapt to your own projects.

## Component Typing

### Function Components: The Standard Pattern

Use regular function declarations with an inline props type. The `React.FC` type is no longer recommended because it implicitly includes `children`, does not support generics cleanly, and obscures the return type.

```tsx
// Preferred: explicit props, clear return
interface UserCardProps {
  name: string;
  email: string;
  avatarUrl?: string;
}

function UserCard({ name, email, avatarUrl }: UserCardProps) {
  return (
    <div>
      {avatarUrl && <img src={avatarUrl} alt={name} />}
      <h3>{name}</h3>
      <p>{email}</p>
    </div>
  );
}
```

### Children as an Explicit Prop

```tsx
interface PageContainerProps {
  title: string;
  children: React.ReactNode;
}

function PageContainer({ title, children }: PageContainerProps) {
  return (
    <main>
      <h1>{title}</h1>
      {children}
    </main>
  );
}
```

Use `React.ReactNode` for children that can be anything renderable (elements, strings, numbers, fragments, null). Use `React.ReactElement` when you specifically need a JSX element.

## Props Patterns

### Extending HTML Element Props

When wrapping a native element, extend its props so consumers get autocomplete for all standard attributes:

```tsx
import { ComponentProps } from 'react';

interface ButtonProps extends ComponentProps<'button'> {
  variant?: 'primary' | 'secondary' | 'ghost';
  isLoading?: boolean;
}

function Button({
  variant = 'primary',
  isLoading,
  disabled,
  children,
  className,
  ...rest
}: ButtonProps) {
  return (
    <button
      disabled={disabled || isLoading}
      className={cn(
        'px-4 py-2 rounded font-medium',
        variant === 'primary' && 'bg-primary text-primary-foreground',
        variant === 'secondary' && 'bg-secondary text-secondary-foreground',
        variant === 'ghost' && 'bg-transparent hover:bg-accent',
        className,
      )}
      {...rest}
    >
      {isLoading ? <Spinner /> : children}
    </button>
  );
}
```

`ComponentProps<'button'>` gives you `onClick`, `type`, `disabled`, `aria-*`, and every other valid `<button>` attribute -- fully typed.

### Spreading Props to Wrapped Components

You can also extend the props of third-party or internal components:

```tsx
import { ComponentProps } from 'react';
import { Input } from '@/components/ui/input';

interface SearchInputProps extends ComponentProps<typeof Input> {
  onSearch: (query: string) => void;
}

function SearchInput({ onSearch, onChange, ...rest }: SearchInputProps) {
  return (
    <Input
      onChange={(e) => {
        onChange?.(e);
        onSearch(e.target.value);
      }}
      {...rest}
    />
  );
}
```

### Optional vs Required Props

```tsx
interface NotificationProps {
  message: string;              // Required
  type?: 'info' | 'error';     // Optional with default
  onDismiss?: () => void;       // Optional callback
}

function Notification({
  message,
  type = 'info',
  onDismiss,
}: NotificationProps) {
  return (
    <div className={type === 'error' ? 'bg-destructive' : 'bg-muted'}>
      <p>{message}</p>
      {onDismiss && <button onClick={onDismiss}>Dismiss</button>}
    </div>
  );
}
```

## Discriminated Unions: Conditional Props

When certain props should only exist based on the value of another prop, use a discriminated union instead of making everything optional. This prevents invalid combinations at compile time.

### The Problem with Optional Props

```tsx
// Bad: nothing prevents passing cardNumber with method="paypal"
interface PaymentProps {
  method: 'credit-card' | 'paypal';
  cardNumber?: string;
  cvv?: string;
  email?: string;
}
```

### The Solution: Discriminated Union

```tsx
// TypeScript enforces valid combinations
type PaymentProps =
  | { method: 'credit-card'; cardNumber: string; cvv: string }
  | { method: 'paypal'; email: string };

function PaymentForm(props: PaymentProps) {
  switch (props.method) {
    case 'credit-card':
      return <CreditCardForm card={props.cardNumber} cvv={props.cvv} />;
    case 'paypal':
      return <PayPalForm email={props.email} />;
  }
}
```

### Async State Modeling

Discriminated unions are also excellent for modeling async state:

```tsx
type AsyncState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };

function UserProfile({ state }: { state: AsyncState<User> }) {
  switch (state.status) {
    case 'idle':
      return null;
    case 'loading':
      return <Spinner />;
    case 'success':
      return <div>{state.data.name}</div>;
    case 'error':
      return <div>Error: {state.error.message}</div>;
  }
}
```

### Mutually Exclusive Props Without a Discriminant

```tsx
type ExclusiveProps<T, U> =
  | (T & { [K in keyof U]?: never })
  | (U & { [K in keyof T]?: never });

type IconButtonProps = ExclusiveProps<
  { icon: string; 'aria-label': string },
  { children: React.ReactNode }
>;
```

## Generic Components

Use generics when a component works with any data type but needs to preserve type information for consumers.

```tsx
interface SelectOption<T> {
  value: T;
  label: string;
}

interface SelectProps<T> {
  options: SelectOption<T>[];
  value: T;
  onChange: (value: T) => void;
  placeholder?: string;
}

function Select<T extends string | number>({
  options,
  value,
  onChange,
  placeholder,
}: SelectProps<T>) {
  return (
    <select
      value={String(value)}
      onChange={(e) => {
        const selected = options.find((o) => String(o.value) === e.target.value);
        if (selected) onChange(selected.value);
      }}
    >
      {placeholder && <option value="">{placeholder}</option>}
      {options.map((option) => (
        <option key={String(option.value)} value={String(option.value)}>
          {option.label}
        </option>
      ))}
    </select>
  );
}
```

When used, the generic type is inferred automatically:

```tsx
<Select
  options={[
    { value: 'active' as const, label: 'Active' },
    { value: 'inactive' as const, label: 'Inactive' },
    { value: 'pending' as const, label: 'Pending' },
  ]}
  value={status}
  onChange={(val) => setStatus(val)}  // val is 'active' | 'inactive' | 'pending'
/>
```

### Generic List Component

```tsx
interface ListProps<T> {
  items: T[];
  renderItem: (item: T, index: number) => React.ReactNode;
  keyExtractor: (item: T) => string;
  emptyMessage?: string;
}

function List<T>({ items, renderItem, keyExtractor, emptyMessage }: ListProps<T>) {
  if (items.length === 0) {
    return <p className="text-muted-foreground">{emptyMessage ?? 'No items'}</p>;
  }
  return (
    <ul>
      {items.map((item, index) => (
        <li key={keyExtractor(item)}>{renderItem(item, index)}</li>
      ))}
    </ul>
  );
}
```

## Hooks Typing

### useState

TypeScript infers the type from the initial value. Specify the generic when the initial value does not contain all possible types:

```tsx
const [name, setName] = useState('');                    // Inferred as string
const [user, setUser] = useState<User | null>(null);     // Explicit generic
const [items, setItems] = useState<Item[]>([]);           // Array that starts empty
```

### useReducer

```tsx
type State = { count: number; step: number };

type Action =
  | { type: 'increment' }
  | { type: 'decrement' }
  | { type: 'setStep'; payload: number }
  | { type: 'reset' };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'increment':
      return { ...state, count: state.count + state.step };
    case 'decrement':
      return { ...state, count: state.count - state.step };
    case 'setStep':
      return { ...state, step: action.payload };
    case 'reset':
      return { count: 0, step: 1 };
  }
}
```

### useRef

```tsx
// DOM ref: starts as null, React manages the value
const inputRef = useRef<HTMLInputElement>(null);

// Mutable ref: stores a value between renders
const intervalRef = useRef<number | null>(null);
```

The distinction: `useRef<T>(null)` returns `RefObject<T>` (read-only `.current`). `useRef<T | null>(null)` returns `MutableRefObject<T | null>` (writable `.current`).

## Event Handling

### Common Event Types

| Handler | Event Type |
|---------|-----------|
| `onClick` | `React.MouseEvent<HTMLButtonElement>` |
| `onChange` (input) | `React.ChangeEvent<HTMLInputElement>` |
| `onChange` (select) | `React.ChangeEvent<HTMLSelectElement>` |
| `onSubmit` | `React.FormEvent<HTMLFormElement>` |
| `onKeyDown` | `React.KeyboardEvent<HTMLInputElement>` |
| `onFocus` / `onBlur` | `React.FocusEvent<HTMLInputElement>` |

### Form Events Example

```tsx
function LoginForm() {
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    const email = formData.get('email') as string;
  };

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    console.log(e.target.value);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="email" onChange={handleChange} />
      <button type="submit">Submit</button>
    </form>
  );
}
```

## Context with Type Safety

The standard pattern uses an `undefined` default combined with a guard in the hook, ensuring you never silently get `undefined`:

```tsx
import { createContext, useContext, useState, type ReactNode } from 'react';

interface AuthContextValue {
  user: User | null;
  login: (credentials: LoginCredentials) => Promise<void>;
  logout: () => void;
  isAuthenticated: boolean;
}

const AuthContext = createContext<AuthContextValue | undefined>(undefined);

function useAuth(): AuthContextValue {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
}

function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);

  const login = async (credentials: LoginCredentials) => {
    const user = await loginApi(credentials);
    setUser(user);
  };

  const logout = () => setUser(null);

  return (
    <AuthContext.Provider value={{ user, login, logout, isAuthenticated: !!user }}>
      {children}
    </AuthContext.Provider>
  );
}

export { AuthProvider, useAuth };
```

## Custom Hooks

### Naming and Return Type Conventions

Always prefix hooks with `use`. Choose the return shape based on complexity:

```tsx
// Single value: return directly
function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState(false);
  // ... effect logic
  return matches;
}

// Value + setter pair: return a tuple (like useState)
function useLocalStorage<T>(key: string, initialValue: T): [T, (value: T) => void] {
  const [stored, setStored] = useState<T>(() => {
    const item = window.localStorage.getItem(key);
    return item ? (JSON.parse(item) as T) : initialValue;
  });

  const setValue = (value: T) => {
    setStored(value);
    window.localStorage.setItem(key, JSON.stringify(value));
  };

  return [stored, setValue];
}

// Complex state: return an object (named properties)
function useToggle(initial = false) {
  const [value, setValue] = useState(initial);
  const toggle = useCallback(() => setValue((v) => !v), []);
  const setTrue = useCallback(() => setValue(true), []);
  const setFalse = useCallback(() => setValue(false), []);
  return { value, toggle, setTrue, setFalse } as const;
}
```

Using `as const` on tuple returns preserves literal types, which is especially important when returning from hooks.

## Type Utilities

### Common Utility Types for React

```tsx
type ButtonSize = Pick<ButtonProps, 'size'>;
type CardBodyProps = Omit<CardProps, 'header' | 'footer'>;
type PartialUser = Partial<User>;
type RequiredConfig = Required<AppConfig>;
type InputProps = ComponentProps<typeof Input>;
type QueryResult = ReturnType<typeof useQuery>;
type UserData = Awaited<ReturnType<typeof fetchUser>>;
```

### Branded Types for Domain Safety

Branded types prevent accidentally passing one kind of identifier where another is expected:

```tsx
type UserId = string & { __brand: 'UserId' };
type PostId = string & { __brand: 'PostId' };

function fetchUser(id: UserId): Promise<User> { ... }
function fetchPost(id: PostId): Promise<Post> { ... }

const userId = '123' as UserId;
const postId = '456' as PostId;

fetchUser(userId);   // OK
fetchUser(postId);   // Type error
```

## API Response Typing

### Typed API Client

```tsx
interface ApiResponse<T> {
  data: T;
  message: string;
  status: number;
}

interface PaginatedResponse<T> {
  data: T[];
  total: number;
  page: number;
  pageSize: number;
  totalPages: number;
}

async function apiGet<T>(endpoint: string): Promise<ApiResponse<T>> {
  const response = await fetch(`/api${endpoint}`);
  if (!response.ok) throw new ApiError(response.status, await response.text());
  return response.json();
}
```

### Error Types and Type Guards

```tsx
class ApiError extends Error {
  constructor(
    public status: number,
    public body: string,
  ) {
    super(`API Error ${status}: ${body}`);
    this.name = 'ApiError';
  }
}

function isApiError(error: unknown): error is ApiError {
  return error instanceof ApiError;
}
```

## Performance Patterns

### React.memo and useCallback

Use `React.memo` to skip re-renders for unchanged props, and `useCallback` to stabilize function references passed to memoized children:

```tsx
const ExpensiveList = React.memo(function ExpensiveList({
  items,
  onSelect,
}: ExpensiveListProps) {
  return (
    <ul>
      {items.map((item) => (
        <li key={item.id} onClick={() => onSelect(item)}>{item.name}</li>
      ))}
    </ul>
  );
});

function ParentComponent() {
  const handleSelect = useCallback((item: Item) => {
    console.log(item.name);
  }, []);

  return <ExpensiveList items={items} onSelect={handleSelect} />;
}
```

### When NOT to Memoize

Do not wrap everything in `useMemo` or `useCallback`. The overhead of memoization itself is not free. Use them when you are passing callbacks to memoized children, computing expensive derived values, or creating objects/arrays used in dependency arrays. Skip them for simple primitives, event handlers on leaf DOM elements, or values that change on every render anyway.

## File and Naming Conventions

| Convention | Example | When |
|-----------|---------|------|
| PascalCase file = PascalCase component | `UserCard.tsx` | Always for components |
| camelCase for hooks | `useAuth.ts` | Always for hooks |
| camelCase for utilities | `formatDate.ts` | Always for utils |
| Suffix props with `Props` | `UserCardProps` | Always |
| Suffix context values with `ContextValue` | `AuthContextValue` | Always |
| `interface` for objects | `interface User { ... }` | Object shapes |
| `type` for unions/intersections | `type Status = 'active' \| 'inactive'` | Unions, computed types |

### Import Organization

Keep imports consistent: React first, then third-party libraries, then internal aliases, then feature-local imports, then styles. Use the `type` keyword for type-only imports to help bundlers tree-shake.

```tsx
import { useState, useCallback, type ReactNode } from 'react';
import { useQuery } from '@tanstack/react-query';
import { cn } from '@/lib/utils';
import { UserCard } from './UserCard';
import type { User } from '../types/user.types';
```

## Strict TypeScript Configuration

Enable strict mode and additional safety checks in your `tsconfig.json`:

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true
  }
}
```

`"strict": true` enables `strictNullChecks`, `strictFunctionTypes`, `strictBindCallApply`, `noImplicitAny`, `noImplicitThis`, and `alwaysStrict`. Never disable these individually.

## Anti-Patterns to Avoid

| Anti-Pattern | Why | Instead |
|-------------|-----|---------|
| `any` type | Disables type checking | Use `unknown`, generics, or specific types |
| `as` type assertion | Bypasses the type system | Use type guards or discriminated unions |
| Optional props for conditional behavior | Allows invalid states | Use discriminated unions |
| `useEffect` for data fetching | Missing caching, race conditions | Use TanStack Query |
| Prop drilling through many levels | Brittle, hard to refactor | Use Context or state management |
| `// @ts-ignore` | Hides real type errors | Fix the underlying type issue |
| Memoizing everything | Adds overhead without benefit | Memoize only when profiling shows a problem |

## Conclusion

TypeScript in React is not about adding types after the fact -- it is about designing your component APIs so that the compiler catches mistakes before they reach production. The patterns covered here (explicit props, discriminated unions, generics, typed context, and strict configuration) form a solid foundation for any React TypeScript codebase.

In the next post, we will build on these TypeScript foundations by exploring type-safe routing with TanStack Router, where every path, parameter, and search query is validated at compile time.

## References

- [React TypeScript Cheatsheet](https://react-typescript-cheatsheet.netlify.app/)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html)
- [React Documentation: TypeScript](https://react.dev/learn/typescript)
