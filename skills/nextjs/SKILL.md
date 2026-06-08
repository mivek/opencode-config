---
name: nextjs
description: Conventions for Next.js projects (App Router + TypeScript). Load when working on Next.js codebases — pages, components, server actions, API routes, layouts.
compatibility: opencode
metadata:
  stack: nextjs
  routing: app-router
---

# Next.js (App Router) conventions

## Detect routing first

Before writing any code, check:
- `app/` directory exists → **App Router** (Next.js 13+)
- `pages/` directory exists → **Pages Router** (legacy)
- Both → hybrid project, default to App Router for new code unless told otherwise

This skill assumes **App Router**. If user is on Pages Router, ask before applying these conventions.

## Project structure expected

```
app/
  layout.tsx              # root layout (required)
  page.tsx                # home page
  <route>/
    page.tsx              # route component (Server Component by default)
    layout.tsx            # nested layout (optional)
    loading.tsx           # loading UI (optional)
    error.tsx             # error boundary (optional)
    not-found.tsx         # 404 (optional)
  api/
    <route>/
      route.ts            # API route handlers (GET, POST, etc.)
components/
  ui/                     # primitive components
  <feature>/              # feature-grouped components
lib/
  <utility>.ts            # shared utilities
```

## Server vs Client components

- **Default to Server Components.** They're cheaper, render on server, and don't ship JS to client.
- Add `'use client'` directive **only when** you need:
  - `useState`, `useEffect`, or other hooks
  - Event handlers (`onClick`, `onChange`, etc.)
  - Browser APIs (`window`, `localStorage`)
  - Third-party libraries that require client environment

Mistake to avoid : marking the whole page tree `'use client'` "just in case". Always start server, mark client only where needed.

## Data fetching

- In Server Components : `async/await` with native `fetch()` directly. Next.js handles caching automatically.
- In Client Components : use SWR, TanStack Query, or React 19's `use()`.
- **Never** fetch data in Client Components if it can be done in a Server Component parent.

```tsx
// Good: server component fetches, passes to client
export default async function Page() {
  const data = await fetch('https://api.example.com/data').then(r => r.json())
  return <ClientChart data={data} />
}
```

## Server Actions

For mutations, prefer Server Actions over API routes :

```tsx
// app/items/actions.ts
'use server'
export async function createItem(formData: FormData) {
  // ... server-side logic
  revalidatePath('/items')
}

// Used in a client component:
<form action={createItem}>...</form>
```

API routes (`route.ts`) are still useful for webhooks or external consumers.

## TypeScript conventions

- `strict: true` in `tsconfig.json` is the default — never relax it
- Use the `Props` type for component props : `export default function MyComp(props: Props) { ... }`
- For dynamic routes, type params : `{ params: Promise<{ id: string }> }` (Next.js 15+ async params)
- Don't use `any` — use `unknown` and narrow

## Styling

Detect what's used in the project first :
- **Tailwind** (`tailwind.config.*` exists) — use utility classes
- **CSS Modules** (`*.module.css`) — import as `styles`
- **shadcn/ui** (`components/ui/` with cva patterns) — extend existing components

Match the existing convention. Don't introduce a new styling system.

## Standard commands

| Task | Command |
|---|---|
| Dev server | `npm run dev` (or `pnpm dev`) |
| Production build | `npm run build` |
| Type-check | `npm run typecheck` or `tsc --noEmit` |
| Lint | `npm run lint` |
| Run tests | depends on setup — check `package.json` scripts |

## Common pitfalls

- **Hydration mismatch** : usually caused by `Date.now()`, `Math.random()`, or `window.*` accessed during SSR. Use `useEffect` or check `typeof window !== 'undefined'`.
- **Cookies/headers in Server Components** : import from `next/headers`, must be in async component
- **revalidatePath not working** : check the path matches exactly, including dynamic segments
- **'Module not found' for `@/` imports** : verify `tsconfig.json` has `paths: { "@/*": ["./*"] }`
- **Server Action 'use server' missing** : every server action file needs the directive at top OR the function needs inline `'use server'`

## When generating new code

1. Determine if the new code is Server or Client first
2. Place it in the appropriate folder per the convention above
3. Use the existing project's TypeScript config (don't add new compiler options)
4. Follow existing component patterns (look at a neighbor before writing)
5. If adding a new dependency, mention it explicitly — don't silently add to package.json
