# React & Next.js Architecture Guide

> A comprehensive guide for building scalable, maintainable React/Next.js 16 applications with shadcn/ui, Tailwind CSS v4, Atomic Design principles, Supabase integration, and React Hook Form + Zod validation.

**Last Updated**: January 2025  
**Next.js Version**: 16+  
**Tailwind CSS Version**: 4+  
**React Version**: 19+

---

## Table of Contents

1. [Project Structure](#1-project-structure)
2. [Component Architecture (Atomic Design)](#2-component-architecture-atomic-design)
3. [shadcn/ui Integration](#3-shadcnui-integration)
4. [Layout Components](#4-layout-components)
5. [Styling with CVA & Tailwind v4](#5-styling-with-cva--tailwind-v4)
6. [Forms with React Hook Form & Zod](#6-forms-with-react-hook-form--zod)
7. [Custom Hooks](#7-custom-hooks)
8. [Utilities & Helpers](#8-utilities--helpers)
9. [Supabase Integration](#9-supabase-integration)
10. [Naming Conventions](#10-naming-conventions)
11. [Code Extraction Rules](#11-code-extraction-rules)
12. [TypeScript Patterns](#12-typescript-patterns)
13. [Best Practices Summary](#13-best-practices-summary)

---

## 1. Project Structure

### Recommended Folder Structure (Next.js 16 App Router)

```
src/
├── app/                          # Next.js App Router
│   ├── (auth)/                   # Route group: authentication pages
│   │   ├── login/
│   │   │   ├── page.tsx
│   │   │   └── actions.ts        # Server Actions for login
│   │   ├── register/
│   │   │   ├── page.tsx
│   │   │   └── actions.ts
│   │   └── layout.tsx            # Auth layout (redirects if logged in)
│   ├── (dashboard)/              # Route group: protected pages
│   │   ├── dashboard/
│   │   │   └── page.tsx
│   │   ├── settings/
│   │   │   └── page.tsx
│   │   └── layout.tsx            # Protected layout (auth guard here!)
│   ├── auth/
│   │   └── confirm/
│   │       └── route.ts          # Email confirmation handler
│   ├── api/                      # API routes
│   │   └── webhooks/
│   ├── layout.tsx                # Root layout
│   ├── page.tsx                  # Home page
│   └── globals.css               # Tailwind v4 CSS-first config
│
├── components/
│   ├── ui/                       # shadcn/ui components (atoms)
│   │   ├── button.tsx
│   │   ├── card.tsx
│   │   ├── input.tsx
│   │   ├── badge.tsx
│   │   ├── avatar.tsx
│   │   ├── dialog.tsx
│   │   ├── form.tsx              # React Hook Form wrapper components
│   │   └── ...
│   │
│   ├── layout/                   # Layout primitives
│   │   ├── container.tsx
│   │   ├── stack.tsx
│   │   ├── grid.tsx
│   │   ├── flex.tsx
│   │   ├── center.tsx
│   │   └── index.ts
│   │
│   ├── composed/                 # Molecules (combinations of ui/ atoms)
│   │   ├── stat-card.tsx
│   │   ├── user-info.tsx
│   │   ├── form-field.tsx
│   │   ├── search-input.tsx
│   │   └── nav-link.tsx
│   │
│   ├── blocks/                   # Organisms (complex UI sections)
│   │   ├── dashboard-header.tsx
│   │   ├── stats-grid.tsx
│   │   ├── data-table.tsx
│   │   ├── sidebar-nav.tsx
│   │   └── user-menu.tsx
│   │
│   ├── layouts/                  # Templates (page layouts)
│   │   ├── dashboard-layout.tsx
│   │   ├── auth-layout.tsx
│   │   ├── marketing-layout.tsx
│   │   └── settings-layout.tsx
│   │
│   ├── forms/                    # Form components (use RHF + Zod)
│   │   ├── login-form.tsx
│   │   ├── register-form.tsx
│   │   ├── profile-form.tsx
│   │   └── settings-form.tsx
│   │
│   └── providers/                # Context providers
│       ├── theme-provider.tsx
│       ├── query-provider.tsx
│       └── index.tsx
│
├── hooks/                        # Global reusable hooks
│   ├── use-debounce.ts
│   ├── use-local-storage.ts
│   ├── use-media-query.ts
│   ├── use-mounted.ts
│   └── use-click-outside.ts
│
├── lib/                          # Core utilities
│   ├── utils.ts                  # cn() and general helpers
│   ├── supabase/
│   │   ├── client.ts             # Browser client
│   │   ├── server.ts             # Server client
│   │   └── proxy.ts              # Proxy helper (session refresh)
│   └── validations/              # Zod schemas (shared client/server)
│       ├── auth.ts
│       ├── profile.ts
│       └── index.ts
│
├── services/                     # API/data fetching layer
│   ├── auth.ts
│   ├── users.ts
│   ├── projects.ts
│   └── api-client.ts
│
├── types/                        # Global TypeScript types
│   ├── index.ts
│   ├── database.ts               # Supabase generated types
│   └── api.ts
│
├── constants/                    # App constants
│   ├── routes.ts
│   ├── config.ts
│   └── messages.ts
│
└── proxy.ts                      # Next.js 16 Proxy (replaces middleware.ts)
```

### Key Changes in Next.js 16

| Change | Old (Next.js 15) | New (Next.js 16) |
|--------|------------------|------------------|
| Request interceptor | `middleware.ts` | `proxy.ts` |
| Function export | `middleware()` | `proxy()` |
| Runtime | Edge Runtime | Node.js Runtime |
| Auth location | Middleware | Layouts (RSC) |
| Config flag | `skipMiddlewareUrlNormalize` | `skipProxyUrlNormalize` |

**Important**: In Next.js 16, authentication should happen in **Layouts** using React Server Components, NOT in the proxy. The proxy is for routing (rewrites, redirects, headers) only.

---

## 2. Component Architecture (Atomic Design)

### The Five Levels

| Level | Description | Location | Examples |
|-------|-------------|----------|----------|
| **Atoms** | Smallest UI elements, no business logic | `components/ui/` | Button, Input, Badge, Avatar |
| **Molecules** | Combinations of atoms, single purpose | `components/composed/` | StatCard, UserInfo, SearchInput |
| **Organisms** | Complex UI sections, may have local state | `components/blocks/` | DashboardHeader, DataTable, Sidebar |
| **Templates** | Page layouts with slots for content | `components/layouts/` | DashboardLayout, AuthLayout |
| **Pages** | Actual pages with real data | `app/*/page.tsx` | Dashboard, Settings, Profile |

### Visual Hierarchy Example

```
Page (app/dashboard/page.tsx)
└── Template (DashboardLayout)
    ├── Organism (DashboardHeader)
    │   ├── Atom (Text) — title
    │   └── Molecule (UserInfo)
    │       ├── Atom (Avatar)
    │       └── Atom (Text)
    │
    └── Organism (StatsGrid)
        └── Molecule (StatCard) × 4
            ├── Atom (Card)
            ├── Atom (Text)
            ├── Atom (Badge)
            └── Atom (Icon)
```

### Component Design Rules

```typescript
// ✅ GOOD: Atom - simple, reusable, no business logic
export function Badge({ variant, children }: BadgeProps) {
  return <span className={badgeVariants({ variant })}>{children}</span>;
}

// ✅ GOOD: Molecule - combines atoms, single purpose
export function StatCard({ label, value, icon: Icon }: StatCardProps) {
  return (
    <Card>
      <CardContent>
        <Text variant="label">{label}</Text>
        <Text variant="h2">{value}</Text>
        <Icon className="h-5 w-5" />
      </CardContent>
    </Card>
  );
}

// ✅ GOOD: Organism - complex section, may have state
export function StatsGrid({ stats }: StatsGridProps) {
  return (
    <Grid cols={4} gap="md">
      {stats.map((stat) => (
        <StatCard key={stat.label} {...stat} />
      ))}
    </Grid>
  );
}

// ❌ BAD: Mixing concerns, too many responsibilities
export function DashboardWithEverything() {
  const [data, setData] = useState();
  // fetching, state management, UI all mixed together
}
```

---

## 3. shadcn/ui Integration

### What is shadcn/ui?

- **NOT** a component library you install as a dependency
- A collection of **copy-paste components** you own
- Built on **Radix UI** (accessibility) + **Tailwind CSS** (styling)
- Uses **CVA** (class-variance-authority) for variants

### Installation

```bash
# Initialize shadcn/ui (supports Tailwind v4)
npx shadcn@latest init

# Add components as needed
npx shadcn@latest add button card input badge avatar dialog form
```

### shadcn/ui as Atoms

shadcn/ui components live in `components/ui/` and serve as your **atoms**:

```
components/ui/
├── button.tsx      # Already uses CVA
├── card.tsx        # Compound component
├── input.tsx       # Form atom
├── badge.tsx       # Display atom
├── avatar.tsx      # User display atom
├── dialog.tsx      # Radix-powered molecule
├── form.tsx        # React Hook Form integration
└── dropdown-menu.tsx
```

---

## 4. Layout Components

### Why Layout Components?

- **Reduce repetition**: No more `flex flex-col gap-4` everywhere
- **Enforce consistency**: Spacing follows design system
- **Improve readability**: `<Stack gap="md">` vs `<div className="flex flex-col gap-4">`
- **Type safety**: Props constrained to valid values

### The Layout Primitives

#### Container

```typescript
// components/layout/container.tsx
import { cn } from "@/lib/utils";
import { cva, type VariantProps } from "class-variance-authority";

const containerVariants = cva("mx-auto w-full px-4 sm:px-6 lg:px-8", {
  variants: {
    size: {
      sm: "max-w-screen-sm",
      md: "max-w-screen-md",
      lg: "max-w-screen-lg",
      xl: "max-w-screen-xl",
      "2xl": "max-w-screen-2xl",
      full: "max-w-full",
      prose: "max-w-prose",
    },
  },
  defaultVariants: {
    size: "xl",
  },
});

interface ContainerProps
  extends React.HTMLAttributes<HTMLDivElement>,
    VariantProps<typeof containerVariants> {
  as?: React.ElementType;
}

export function Container({
  as: Component = "div",
  size,
  className,
  ...props
}: ContainerProps) {
  return (
    <Component
      className={cn(containerVariants({ size }), className)}
      {...props}
    />
  );
}
```

#### Stack (VStack / HStack)

```typescript
// components/layout/stack.tsx
import { cn } from "@/lib/utils";
import { cva, type VariantProps } from "class-variance-authority";

const stackVariants = cva("flex", {
  variants: {
    direction: {
      row: "flex-row",
      column: "flex-col",
    },
    align: {
      start: "items-start",
      center: "items-center",
      end: "items-end",
      stretch: "items-stretch",
    },
    justify: {
      start: "justify-start",
      center: "justify-center",
      end: "justify-end",
      between: "justify-between",
    },
    gap: {
      none: "gap-0",
      xs: "gap-1",
      sm: "gap-2",
      md: "gap-4",
      lg: "gap-6",
      xl: "gap-8",
    },
  },
  defaultVariants: {
    direction: "column",
    gap: "md",
  },
});

interface StackProps
  extends React.HTMLAttributes<HTMLDivElement>,
    VariantProps<typeof stackVariants> {
  as?: React.ElementType;
}

export function Stack({ as: Component = "div", direction, align, justify, gap, className, ...props }: StackProps) {
  return <Component className={cn(stackVariants({ direction, align, justify, gap }), className)} {...props} />;
}

export function VStack(props: Omit<StackProps, "direction">) {
  return <Stack direction="column" {...props} />;
}

export function HStack(props: Omit<StackProps, "direction">) {
  return <Stack direction="row" {...props} />;
}
```

#### Grid

```typescript
// components/layout/grid.tsx
import { cn } from "@/lib/utils";
import { cva, type VariantProps } from "class-variance-authority";

const gridVariants = cva("grid", {
  variants: {
    cols: {
      1: "grid-cols-1",
      2: "grid-cols-2",
      3: "grid-cols-3",
      4: "grid-cols-4",
      6: "grid-cols-6",
      12: "grid-cols-12",
    },
    gap: {
      none: "gap-0",
      sm: "gap-2",
      md: "gap-4",
      lg: "gap-6",
    },
  },
  defaultVariants: {
    cols: 1,
    gap: "md",
  },
});

interface GridProps extends React.HTMLAttributes<HTMLDivElement>, VariantProps<typeof gridVariants> {
  as?: React.ElementType;
}

export function Grid({ as: Component = "div", cols, gap, className, ...props }: GridProps) {
  return <Component className={cn(gridVariants({ cols, gap }), className)} {...props} />;
}
```

---

## 5. Styling with CVA & Tailwind v4

### Tailwind CSS v4: CSS-First Configuration

Tailwind v4 moves configuration from JavaScript to CSS using the `@theme` directive.

#### globals.css (Tailwind v4)

```css
/* app/globals.css */

/* Single import replaces @tailwind directives */
@import "tailwindcss";

/* Theme configuration (replaces tailwind.config.js) */
@theme {
  /* Colors */
  --color-background: hsl(0 0% 100%);
  --color-foreground: hsl(222.2 84% 4.9%);
  --color-primary: hsl(222.2 47.4% 11.2%);
  --color-primary-foreground: hsl(210 40% 98%);
  --color-secondary: hsl(210 40% 96.1%);
  --color-muted: hsl(210 40% 96.1%);
  --color-destructive: hsl(0 84.2% 60.2%);
  --color-border: hsl(214.3 31.8% 91.4%);
  
  /* Typography */
  --font-sans: "Inter", ui-sans-serif, system-ui, sans-serif;
  --font-mono: "JetBrains Mono", ui-monospace, monospace;
  
  /* Border radius */
  --radius-sm: 0.25rem;
  --radius-md: 0.375rem;
  --radius-lg: 0.5rem;
}

/* Base layer for global styles */
@layer base {
  * { @apply border-border; }
  body { @apply bg-background text-foreground; }
}
```

### Key Tailwind v4 Changes

| v3 (Old) | v4 (New) |
|----------|----------|
| `tailwind.config.js` | `@theme` in CSS |
| `@tailwind base/components/utilities` | `@import "tailwindcss"` |
| `content: [...]` array | Automatic detection |
| `bg-opacity-50` | `bg-black/50` |

### The `cn()` Utility

```typescript
// lib/utils.ts
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

---

## 6. Forms with React Hook Form & Zod

### Installation

```bash
npm install react-hook-form zod @hookform/resolvers
```

### Zod Schema (Shared Client/Server)

```typescript
// lib/validations/auth.ts
import { z } from "zod";

export const loginSchema = z.object({
  email: z.string().min(1, "Email is required").email("Invalid email"),
  password: z.string().min(1, "Password is required").min(8, "Min 8 characters"),
});

export type LoginInput = z.infer<typeof loginSchema>;

export const registerSchema = loginSchema.extend({
  name: z.string().min(2, "Name must be at least 2 characters"),
  confirmPassword: z.string(),
}).refine((data) => data.password === data.confirmPassword, {
  message: "Passwords don't match",
  path: ["confirmPassword"],
});

export type RegisterInput = z.infer<typeof registerSchema>;
```

### Form Component with React Hook Form

```typescript
// components/forms/login-form.tsx
"use client";

import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { useActionState, startTransition } from "react";
import { loginSchema, type LoginInput } from "@/lib/validations/auth";
import { loginAction } from "@/app/(auth)/login/actions";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";

export function LoginForm() {
  const [state, formAction, isPending] = useActionState(loginAction, {
    success: false,
    message: "",
  });

  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<LoginInput>({
    resolver: zodResolver(loginSchema),
  });

  const onSubmit = (data: LoginInput) => {
    const formData = new FormData();
    formData.append("email", data.email);
    formData.append("password", data.password);
    startTransition(() => formAction(formData));
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
      {state.message && !state.success && (
        <div className="text-sm text-destructive">{state.message}</div>
      )}

      <div className="space-y-2">
        <Label htmlFor="email">Email</Label>
        <Input id="email" type="email" {...register("email")} />
        {errors.email && <p className="text-sm text-destructive">{errors.email.message}</p>}
      </div>

      <div className="space-y-2">
        <Label htmlFor="password">Password</Label>
        <Input id="password" type="password" {...register("password")} />
        {errors.password && <p className="text-sm text-destructive">{errors.password.message}</p>}
      </div>

      <Button type="submit" disabled={isPending} className="w-full">
        {isPending ? "Signing in..." : "Sign in"}
      </Button>
    </form>
  );
}
```

### Server Action with Zod Validation

```typescript
// app/(auth)/login/actions.ts
"use server";

import { redirect } from "next/navigation";
import { revalidatePath } from "next/cache";
import { createClient } from "@/lib/supabase/server";
import { loginSchema } from "@/lib/validations/auth";

export type ActionState = {
  success: boolean;
  message: string;
  fields?: Record<string, string>;
};

export async function loginAction(prevState: ActionState, formData: FormData): Promise<ActionState> {
  const rawData = {
    email: formData.get("email") as string,
    password: formData.get("password") as string,
  };

  // Validate with same Zod schema as client
  const validatedFields = loginSchema.safeParse(rawData);

  if (!validatedFields.success) {
    return { success: false, message: "Invalid form data", fields: rawData };
  }

  const supabase = await createClient();
  const { error } = await supabase.auth.signInWithPassword(validatedFields.data);

  if (error) {
    return { success: false, message: error.message, fields: rawData };
  }

  revalidatePath("/", "layout");
  redirect("/dashboard");
}
```

---

## 7. Custom Hooks

```typescript
// hooks/use-debounce.ts
import { useEffect, useState } from "react";

export function useDebounce<T>(value: T, delay: number = 500): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);
  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);
  return debouncedValue;
}

// hooks/use-mounted.ts
import { useEffect, useState } from "react";

export function useMounted(): boolean {
  const [mounted, setMounted] = useState(false);
  useEffect(() => setMounted(true), []);
  return mounted;
}

// hooks/use-media-query.ts
import { useEffect, useState } from "react";

export function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState(false);
  useEffect(() => {
    const media = window.matchMedia(query);
    if (media.matches !== matches) setMatches(media.matches);
    const listener = () => setMatches(media.matches);
    media.addEventListener("change", listener);
    return () => media.removeEventListener("change", listener);
  }, [matches, query]);
  return matches;
}

export const useIsMobile = () => useMediaQuery("(max-width: 768px)");
```

---

## 8. Utilities & Helpers

```typescript
// lib/utils.ts
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}

export function formatDate(date: Date | string | number): string {
  return new Intl.DateTimeFormat("en-US", {
    month: "long", day: "numeric", year: "numeric",
  }).format(new Date(date));
}

export function formatCurrency(amount: number, currency = "USD"): string {
  return new Intl.NumberFormat("en-US", { style: "currency", currency }).format(amount);
}

export function truncate(str: string, length: number): string {
  return str.length <= length ? str : `${str.slice(0, length)}...`;
}

export function getInitials(name: string): string {
  return name.split(" ").map((n) => n[0]).join("").toUpperCase().slice(0, 2);
}

export function sleep(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}
```

---

## 9. Supabase Integration

### Client (Browser)

```typescript
// lib/supabase/client.ts
import { createBrowserClient } from "@supabase/ssr";
import type { Database } from "@/types/database";

export function createClient() {
  return createBrowserClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}
```

### Server (Server Components, Actions)

```typescript
// lib/supabase/server.ts
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";
import type { Database } from "@/types/database";

export async function createClient() {
  const cookieStore = await cookies();

  return createServerClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() { return cookieStore.getAll(); },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options)
            );
          } catch { /* Server Component - ignore */ }
        },
      },
    }
  );
}
```

### Proxy Helper

```typescript
// lib/supabase/proxy.ts
import { createServerClient } from "@supabase/ssr";
import { NextResponse, type NextRequest } from "next/server";

export async function updateSession(request: NextRequest) {
  let supabaseResponse = NextResponse.next({ request });

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() { return request.cookies.getAll(); },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value }) => request.cookies.set(name, value));
          supabaseResponse = NextResponse.next({ request });
          cookiesToSet.forEach(({ name, value, options }) =>
            supabaseResponse.cookies.set(name, value, options)
          );
        },
      },
    }
  );

  // Use getUser() - validates JWT (not getSession which doesn't)
  const { data: { user } } = await supabase.auth.getUser();
  return { supabaseResponse, user };
}
```

### Proxy (Next.js 16)

```typescript
// proxy.ts (root or src/)
import { type NextRequest } from "next/server";
import { updateSession } from "@/lib/supabase/proxy";

// Next.js 16: proxy.ts replaces middleware.ts
// Use for routing only - auth goes in Layouts!
export async function proxy(request: NextRequest) {
  return (await updateSession(request)).supabaseResponse;
}

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)"],
};
```

### Auth in Layouts (Best Practice)

```typescript
// app/(dashboard)/layout.tsx
import { redirect } from "next/navigation";
import { createClient } from "@/lib/supabase/server";

export default async function DashboardLayout({ children }: { children: React.ReactNode }) {
  const supabase = await createClient();
  const { data: { user }, error } = await supabase.auth.getUser();

  if (error || !user) {
    redirect("/login");
  }

  return <div className="min-h-screen">{children}</div>;
}
```

---

## 10. Naming Conventions

| Type | Convention | Example |
|------|------------|---------|
| Files | kebab-case | `stat-card.tsx` |
| Components | PascalCase | `StatCard` |
| Hooks | camelCase + `use` | `useDebounce` |
| Server Actions | camelCase + `Action` | `loginAction` |
| Zod schemas | camelCase + `Schema` | `loginSchema` |
| Inferred types | PascalCase + `Input` | `LoginInput` |
| Constants | SCREAMING_SNAKE | `API_URL` |

---

## 11. Code Extraction Rules

| Used in... | Action |
|------------|--------|
| 1 place | Keep in same file |
| 2 places | OK to duplicate |
| 3+ places | Extract to shared location |

---

## 12. TypeScript Patterns

```typescript
// Extend HTML props with CVA variants
interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {}

// Infer types from Zod schemas
export type LoginInput = z.infer<typeof loginSchema>;

// Discriminated union for action state
type ActionState =
  | { success: true; data: unknown }
  | { success: false; message: string };
```

---

## 13. Best Practices Summary

### Next.js 16
- Use `proxy.ts` (not `middleware.ts`)
- Put auth in Layouts, not proxy
- Use `async` for params/searchParams

### Forms
- Use React Hook Form + Zod
- Share schemas between client/server
- Use Server Actions for mutations

### Supabase
- Use `@supabase/ssr` package
- Use `getUser()` to validate (not `getSession()`)
- Refresh tokens in proxy, protect in layouts

### Tailwind v4
- Configure with `@theme` in CSS
- Use `@import "tailwindcss"` (no directives)
- Automatic content detection

### Components
- Follow Atomic Design hierarchy
- Use CVA for variants
- Use Layout components for structure

---

*This guide is designed for Claude Code and development teams building React/Next.js applications.*
