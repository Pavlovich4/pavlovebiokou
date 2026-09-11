# pavlovebiokou — Architecture & Onboarding Docs

These docs explain how every piece of this app fits together, written for someone brand-new to Inertia and React. Read them in order the first time — each one builds on the previous.

## Reading order

| # | File | What you'll learn |
|---|------|-------------------|
| 1 | [01-overview.md](01-overview.md) | The big picture. What Laravel, Inertia, React, Vite, Tailwind, Fortify, Wayfinder each do and how they fit. Traces one request end-to-end. |
| 2 | [02-backend-architecture.md](02-backend-architecture.md) | The Laravel side: `bootstrap/app.php`, routes, controllers, middleware, Fortify service provider, shared props. |
| 3 | [03-frontend-architecture.md](03-frontend-architecture.md) | The frontend map: `resources/js/` folders, `app.tsx`, `vite.config.ts`, `tsconfig.json`, entry point. |
| 4 | [04-inertia-explained.md](04-inertia-explained.md) | Inertia primitives: `<Head>`, `<Link>`, `<Form>`, `usePage`, `router`, flash data, progress bar. |
| 5 | [05-layout-system.md](05-layout-system.md) | **The layout system.** How the resolver picks a layout by page name, how `Page.layout = { title, description }` flows to the DOM, how to add or customize layouts. |
| 6 | [06-wayfinder-typescript.md](06-wayfinder-typescript.md) | Wayfinder + TypeScript: what the generated `routes/` and `actions/` folders are, how to use `store.form()`, how the `@/` alias works. |
| 7 | [07-packages.md](07-packages.md) | Every external package (PHP + JS) with its role and files where it's used. |
| 8 | [08-hooks-and-utilities.md](08-hooks-and-utilities.md) | Line-by-line explanation of every custom hook and utility (`use-appearance`, `use-flash-toast`, `cn`, etc.). |
| 9 | [09-confirm-password-line-by-line.md](09-confirm-password-line-by-line.md) | Worked example: `pages/auth/confirm-password.tsx` explained line by line. |
| 10 | [10-adding-features.md](10-adding-features.md) | Practical recipes: add a page, add a form, add a layout, add a shared prop, show a toast. |
| 11 | [11-dev-workflow.md](11-dev-workflow.md) | Day-to-day commands (`composer dev`, `npm run dev`, tests, lint, format) and common gotchas. |

## The request lifecycle at a glance

Understanding this diagram unlocks the rest of the docs. This is what happens when a browser asks for a page.

```
                            ┌───────────────────────────────────────────────┐
                            │ Browser                                       │
                            │  1. Types URL: /dashboard                     │
                            └───────────────────┬───────────────────────────┘
                                                │  HTTP GET /dashboard
                                                ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│ LARAVEL (server)                                                              │
│                                                                               │
│  2. routes/web.php  →  matches Route::inertia('dashboard', 'dashboard')       │
│                                                                               │
│  3. Middleware stack (bootstrap/app.php):                                     │
│       • auth, verified                                                        │
│       • HandleAppearance (light/dark cookie)                                  │
│       • HandleInertiaRequests → attaches shared props: auth.user, name, …    │
│                                                                               │
│  4. Response: Inertia::render('dashboard', props)                             │
│       First visit  → full HTML page with a <div id="app" data-page="…"/>      │
│       Later visits → JSON payload {component:'dashboard', props, url, …}      │
└───────────────────┬───────────────────────────────────────────────────────────┘
                    │  HTML or JSON
                    ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│ INERTIA CLIENT  (resources/js/app.tsx)                                        │
│                                                                               │
│  5. createInertiaApp reads the payload's component name: 'dashboard'          │
│                                                                               │
│  6. Runs the layout resolver:                                                 │
│       name === 'welcome'              → no layout                             │
│       name starts with 'auth/'        → AuthLayout                            │
│       name starts with 'settings/'    → [AppLayout, SettingsLayout]           │
│       else                            → AppLayout                             │
│                                                                               │
│  7. Loads pages/dashboard.tsx dynamically                                     │
│                                                                               │
│  8. If Dashboard.layout === { breadcrumbs: […] }, spreads that object as     │
│     PROPS onto the resolved layout, so the tree becomes:                      │
│                                                                               │
│         <AppLayout breadcrumbs={[…]}>                                         │
│           <Dashboard />                                                       │
│         </AppLayout>                                                          │
│                                                                               │
│  9. React renders the tree into #app. Subsequent visits swap the page        │
│     component without a full page reload.                                     │
└───────────────────────────────────────────────────────────────────────────────┘
```

Keep this diagram in mind — every doc below zooms into one of these boxes.

## Where to look first

- **"How does the login form actually submit?"** → [04-inertia-explained.md](04-inertia-explained.md) + [06-wayfinder-typescript.md](06-wayfinder-typescript.md).
- **"Why does `Login.layout = { title, description }` work?"** → [05-layout-system.md](05-layout-system.md).
- **"What's `cn()`?"** → [08-hooks-and-utilities.md](08-hooks-and-utilities.md).
- **"I want to add a new page."** → [10-adding-features.md](10-adding-features.md).

## Conventions used in these docs

- File paths are absolute from the repo root (`resources/js/app.tsx`, not `./app.tsx`).
- Line numbers refer to the state of the file at the time this doc was written. If a file changes, the ideas are still right but the numbers may drift.
- Code blocks that show real code from the repo start with `// path/to/file` on line 1 so you can find it.
