---
layout: post
title: "Styling with Tailwind CSS v4 & shadcn/ui"
date: 2026-03-01 00:00:00 +0000
categories: [React]
tags:
  - react
  - tailwind
  - shadcn-ui
  - css
  - frontend
series: "Building Modern React SPAs"
series_part: 4
---


## Series Overview

1. [TypeScript Patterns for React](/posts/react-spa-part1-typescript-patterns/)
2. [Type-Safe Routing with TanStack Router](/posts/react-spa-part2-tanstack-router/)
3. [Server State with TanStack Query](/posts/react-spa-part3-tanstack-query/)
4. **Styling with Tailwind CSS v4 & shadcn/ui** (this post)
5. [Modular Feature Architecture](/posts/react-spa-part5-architecture/)
6. [Deploy to AWS with S3, CloudFront & CDK](/posts/react-spa-part6-deploy-aws/)

## Introduction

A modern React SPA needs a design system that is fast, themeable, and accessible without requiring you to build every component from scratch. Tailwind CSS v4 and shadcn/ui together provide exactly that.

**Tailwind CSS v4** is a ground-up rewrite with a new high-performance engine (5x faster full builds), zero-configuration setup, a first-party Vite plugin, automatic content detection, and CSS-first configuration -- no more `tailwind.config.js` required by default.

**shadcn/ui** is not a component library you install as a dependency. Instead, it is a collection of beautifully designed, accessible components built on Radix UI primitives that you copy directly into your project. You own the code -- no version lock-in, no hidden abstractions.

This post walks through the complete setup: installing Tailwind v4, integrating shadcn/ui, theming with CSS variables, implementing dark mode, building custom components, and organizing your component files.

## Installing Tailwind CSS v4

### Step 1: Install Packages

From your Vite + React + TypeScript project root:

```bash
npm install tailwindcss @tailwindcss/vite
```

Tailwind v4 has zero peer dependencies -- no PostCSS, no autoprefixer, no `tailwind.config.js` needed for standard setups.

### Step 2: Add the Vite Plugin

```ts
// vite.config.ts
import path from 'path';
import tailwindcss from '@tailwindcss/vite';
import react from '@vitejs/plugin-react';
import { defineConfig } from 'vite';

export default defineConfig({
  plugins: [
    react(),
    tailwindcss(),
  ],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
});
```

### Step 3: Add the CSS Import

Replace the contents of `src/index.css` with a single line:

```css
@import "tailwindcss";
```

That is the entire Tailwind setup. The v3 directives (`@tailwind base; @tailwind components; @tailwind utilities;`) are no longer needed. In v4, `@import "tailwindcss"` does everything.

### Step 4: Verify

```tsx
function App() {
  return (
    <h1 className="text-3xl font-bold text-blue-600 p-8">
      Tailwind is working!
    </h1>
  );
}
```

The Vite plugin is the recommended path for Vite projects. It provides tighter integration and better performance than the PostCSS plugin. Only use `@tailwindcss/postcss` if your framework requires PostCSS.

## Installing shadcn/ui

### Step 1: Configure Path Aliases

shadcn/ui requires the `@/` import alias. Update both tsconfig files:

**`tsconfig.json`:**
```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

**`tsconfig.app.json`:**
```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

Install `@types/node` for the `path` import in vite config:

```bash
npm install -D @types/node
```

### Step 2: Run the shadcn CLI

```bash
npx shadcn@latest init
```

The CLI will ask you to choose a base color (Neutral, Stone, Zinc, etc.). It creates several files:

1. **`components.json`** -- Configuration file at the project root
2. **`src/lib/utils.ts`** -- The `cn()` helper function
3. **Updated `src/index.css`** -- CSS variables for the theme
4. **Dependencies installed** -- `clsx`, `tailwind-merge`, `class-variance-authority`, `lucide-react`

### The cn() Helper

```ts
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

`cn()` combines `clsx` (conditional class names) and `tailwind-merge` (deduplicates conflicting Tailwind classes). Without it, `className="p-4 p-8"` would apply both padding values. With `twMerge`, the last one wins. This is essential for components that accept `className` props.

## Adding Components

shadcn/ui components are added one at a time. Each command copies the component source code into your project:

```bash
# Add individual components
npx shadcn@latest add button
npx shadcn@latest add card
npx shadcn@latest add input
npx shadcn@latest add dialog
npx shadcn@latest add dropdown-menu
npx shadcn@latest add toast
npx shadcn@latest add form

# Add all components at once
npx shadcn@latest add --all
```

Components are placed in `src/components/ui/`. Using them is straightforward:

```tsx
import { Button } from '@/components/ui/button';
import {
  Card, CardContent, CardDescription, CardHeader, CardTitle,
} from '@/components/ui/card';

function Dashboard() {
  return (
    <Card className="w-96">
      <CardHeader>
        <CardTitle>Welcome</CardTitle>
        <CardDescription>Your dashboard overview</CardDescription>
      </CardHeader>
      <CardContent>
        <p className="mb-4 text-muted-foreground">
          Everything is running smoothly.
        </p>
        <Button>Get Started</Button>
        <Button variant="outline" className="ml-2">Learn More</Button>
      </CardContent>
    </Card>
  );
}
```

Because shadcn/ui copies actual source code into your project, you can modify any component freely. The tradeoff is that you maintain the code yourself.

### Available Component Categories

| Category | Components |
|----------|-----------|
| **Layout** | Card, Separator, Aspect Ratio, Resizable, Collapsible |
| **Navigation** | Navigation Menu, Menubar, Breadcrumb, Pagination, Sidebar, Tabs |
| **Forms** | Button, Input, Textarea, Checkbox, Radio Group, Select, Switch, Slider, Form, Label |
| **Data Display** | Table, Data Table, Badge, Avatar, Calendar, Chart |
| **Feedback** | Alert, Alert Dialog, Dialog, Drawer, Sheet, Toast (Sonner), Tooltip, Popover |
| **Overlay** | Dialog, Sheet, Drawer, Dropdown Menu, Context Menu, Command |

## Theming with CSS Variables

shadcn/ui uses semantic CSS variables that map to Tailwind utility classes. The `:root` selector defines light theme values, `.dark` defines dark theme values, and `@theme inline` registers them with Tailwind v4.

### The Token Convention

Every color has a surface and foreground pair:

| Token | Tailwind Class | Purpose |
|-------|----------------|---------|
| `background` / `foreground` | `bg-background` / `text-foreground` | Page background and default text |
| `primary` / `primary-foreground` | `bg-primary` / `text-primary-foreground` | Brand/CTA buttons |
| `secondary` / `secondary-foreground` | `bg-secondary` / `text-secondary-foreground` | Secondary actions |
| `muted` / `muted-foreground` | `bg-muted` / `text-muted-foreground` | Subtle backgrounds, de-emphasized text |
| `accent` / `accent-foreground` | `bg-accent` / `text-accent-foreground` | Hover/active states |
| `card` / `card-foreground` | `bg-card` / `text-card-foreground` | Card surfaces |
| `destructive` | `bg-destructive` | Danger actions |
| `border` / `input` / `ring` | `border-border` / `border-input` / `ring-ring` | Borders and focus rings |

### OKLCH Colors

Tailwind v4 + shadcn/ui now use OKLCH colors by default. OKLCH is perceptually uniform -- two colors with the same lightness value actually look equally light, unlike HSL where perceived brightness varies wildly between hues. This makes generating consistent color scales much easier.

### Customizing the Theme

To change your brand color, modify the `--primary` variable:

```css
:root {
  --primary: oklch(0.55 0.2 250);          /* Blue brand */
  --primary-foreground: oklch(0.985 0 0);  /* White text on blue */
}
.dark {
  --primary: oklch(0.7 0.18 250);          /* Lighter blue for dark mode */
  --primary-foreground: oklch(0.15 0 0);   /* Dark text on lighter blue */
}
```

### Adding a Custom Token

```css
:root {
  --warning: oklch(0.84 0.16 84);
  --warning-foreground: oklch(0.28 0.07 46);
}
.dark {
  --warning: oklch(0.41 0.11 46);
  --warning-foreground: oklch(0.99 0.02 95);
}

@theme inline {
  --color-warning: var(--warning);
  --color-warning-foreground: var(--warning-foreground);
}
```

Now `bg-warning` and `text-warning-foreground` work in your components. Use the [shadcn/ui theme generator](https://ui.shadcn.com/themes) to create a complete palette from a single brand color.

## Dark Mode

### Step 1: Install a Theme Provider

For SPAs, use `next-themes` (which works with any React framework despite the name):

```bash
npm install next-themes
```

### Step 2: Create a ThemeProvider Wrapper

```tsx
// src/components/theme-provider.tsx
import { ThemeProvider as NextThemesProvider } from 'next-themes';
import type { ComponentProps } from 'react';

export function ThemeProvider({
  children,
  ...props
}: ComponentProps<typeof NextThemesProvider>) {
  return <NextThemesProvider {...props}>{children}</NextThemesProvider>;
}
```

### Step 3: Wrap Your App

```tsx
<ThemeProvider attribute="class" defaultTheme="system" enableSystem>
  <App />
</ThemeProvider>
```

### Step 4: Create a Theme Toggle

```tsx
import { Moon, Sun } from 'lucide-react';
import { useTheme } from 'next-themes';
import { Button } from '@/components/ui/button';
import {
  DropdownMenu, DropdownMenuContent,
  DropdownMenuItem, DropdownMenuTrigger,
} from '@/components/ui/dropdown-menu';

export function ThemeToggle() {
  const { setTheme } = useTheme();

  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <Button variant="outline" size="icon">
          <Sun className="h-4 w-4 rotate-0 scale-100 transition-all dark:-rotate-90 dark:scale-0" />
          <Moon className="absolute h-4 w-4 rotate-90 scale-0 transition-all dark:rotate-0 dark:scale-100" />
          <span className="sr-only">Toggle theme</span>
        </Button>
      </DropdownMenuTrigger>
      <DropdownMenuContent align="end">
        <DropdownMenuItem onClick={() => setTheme('light')}>Light</DropdownMenuItem>
        <DropdownMenuItem onClick={() => setTheme('dark')}>Dark</DropdownMenuItem>
        <DropdownMenuItem onClick={() => setTheme('system')}>System</DropdownMenuItem>
      </DropdownMenuContent>
    </DropdownMenu>
  );
}
```

### How Dark Mode Works

1. `next-themes` adds/removes the `.dark` class on `<html>`
2. CSS variables under `.dark { ... }` override `:root` values
3. `@custom-variant dark (&:is(.dark *))` tells Tailwind v4 that `dark:` utilities should match when `.dark` is on an ancestor
4. All components automatically re-theme -- no per-component changes needed

## Building Custom Components

### Wrapping shadcn/ui Components

Do not modify files in `src/components/ui/` directly for app-specific behavior. Instead, create wrapper components:

```tsx
// src/shared/components/SubmitButton.tsx
import { Button, type ButtonProps } from '@/components/ui/button';
import { Loader2 } from 'lucide-react';
import { cn } from '@/lib/utils';

interface SubmitButtonProps extends ButtonProps {
  isLoading?: boolean;
}

export function SubmitButton({
  children, isLoading, disabled, className, ...props
}: SubmitButtonProps) {
  return (
    <Button
      disabled={disabled || isLoading}
      className={cn('min-w-24', className)}
      {...props}
    >
      {isLoading ? (
        <>
          <Loader2 className="mr-2 h-4 w-4 animate-spin" />
          Loading...
        </>
      ) : (
        children
      )}
    </Button>
  );
}
```

### Forms with shadcn/ui + react-hook-form + Zod

```bash
npx shadcn@latest add form input
npm install zod
```

```tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { Button } from '@/components/ui/button';
import {
  Form, FormControl, FormField, FormItem, FormLabel, FormMessage,
} from '@/components/ui/form';
import { Input } from '@/components/ui/input';

const loginSchema = z.object({
  email: z.string().email('Invalid email address'),
  password: z.string().min(8, 'Must be at least 8 characters'),
});

type LoginValues = z.infer<typeof loginSchema>;

export function LoginForm({ onSubmit }: { onSubmit: (data: LoginValues) => void }) {
  const form = useForm<LoginValues>({
    resolver: zodResolver(loginSchema),
    defaultValues: { email: '', password: '' },
  });

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
        <FormField
          control={form.control}
          name="email"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Email</FormLabel>
              <FormControl>
                <Input placeholder="you@example.com" {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        <FormField
          control={form.control}
          name="password"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Password</FormLabel>
              <FormControl>
                <Input type="password" {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        <Button type="submit" className="w-full">Sign In</Button>
      </form>
    </Form>
  );
}
```

## File Organization

```
src/
├── components/
│   └── ui/                    # shadcn/ui primitives (rarely modified)
│       ├── button.tsx
│       ├── card.tsx
│       └── ...
├── shared/
│   └── components/            # App-specific shared components (wrappers)
│       ├── SubmitButton.tsx
│       ├── StatCard.tsx
│       └── ThemeToggle.tsx
├── features/
│   └── auth/
│       └── components/        # Feature-specific components
│           ├── LoginForm.tsx
│           └── RegisterForm.tsx
└── lib/
    └── utils.ts               # cn() helper
```

Treat `src/components/ui/` as a pseudo-library. When you need to customize behavior, create a wrapper in `shared/components/`. This way, you can re-run `npx shadcn@latest add button --overwrite` to pick up upstream improvements without losing your customizations.

## Common Patterns

### Data Tables

```bash
npx shadcn@latest add table
npm install @tanstack/react-table
```

shadcn/ui provides a Data Table recipe integrating `@tanstack/react-table` with sorting, filtering, pagination, and row selection.

### Toast Notifications

```bash
npx shadcn@latest add sonner
```

```tsx
import { toast } from 'sonner';
toast.success('Post created successfully');
toast.error('Failed to save changes');
```

Add `<Toaster />` once in your root layout.

### Command Palette

```bash
npx shadcn@latest add command dialog
```

The Command component provides a searchable command palette similar to VS Code's command palette.

## Conclusion

Tailwind CSS v4 and shadcn/ui together provide a complete design system for React SPAs. Tailwind handles the utility layer with zero configuration, shadcn/ui gives you accessible component patterns you fully own, and CSS variables control the entire theme including dark mode. The setup is remarkably simple -- a Vite plugin, a CLI command, and you are ready to build.

In the next post, we will organize all these pieces into a modular feature architecture that scales from small projects to large team codebases.

## References

- [Tailwind CSS v4 Documentation](https://tailwindcss.com/docs)
- [Tailwind CSS v4 Announcement](https://tailwindcss.com/blog/tailwindcss-v4)
- [shadcn/ui Documentation](https://ui.shadcn.com/docs)
- [shadcn/ui Vite Guide](https://ui.shadcn.com/docs/installation/vite)
- [shadcn/ui Tailwind v4 Guide](https://ui.shadcn.com/docs/tailwind-v4)
- [shadcn/ui Theming](https://ui.shadcn.com/docs/theming)
- [shadcn/ui Theme Generator](https://ui.shadcn.com/themes)
- [shadcn/ui Dark Mode](https://ui.shadcn.com/docs/dark-mode)
- [Radix UI](https://www.radix-ui.com)
- [Lucide Icons](https://lucide.dev)
