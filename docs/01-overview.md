# 01 — The Big Picture

Before diving into any code, get the mental model of what each layer is *for*. This app is a Laravel backend that renders React components on the client via Inertia. Nothing is a "single-page app in the traditional sense" and there is no REST API for the frontend to consume.

## The stack, in one sentence each

- **Laravel** — the server framework. Handles routing, database, authentication, sessions, validation, and returning responses.
- **Inertia.js** — the glue between Laravel and React. Instead of controllers returning JSON, they return a *page name* + *props*, and Inertia arranges for the right React component to be mounted in the browser with those props.
- **React** — the UI library. Pages and components are React functions that return JSX.
- **Vite** — the dev server and bundler. In development it hot-reloads changed files; in production it builds optimized JS/CSS bundles.
- **TypeScript** — a typed superset of JavaScript. Gives you autocompletion and catches typos before you ship them.
- **TailwindCSS v4** — utility-class styling. `className="flex gap-4"` instead of writing custom CSS.
- **shadcn/ui** — copy-pasted, customizable UI components (Button, Input, Dialog…). Built on top of Radix primitives. Lives in `resources/js/components/ui/`.
- **Radix UI** — headless (no-styling) accessibility primitives. Shadcn wraps them with Tailwind classes.
- **Fortify** — Laravel's headless authentication backend. Registers login, register, password reset, email verification, and 2FA routes. It doesn't ship views — this app tells it to render Inertia pages instead.
- **Wayfinder** — generates TypeScript functions from your Laravel routes and controllers so you can call them from the frontend with full type safety, no hardcoded URLs.
- **Sonner** — the toast notification library used for "profile updated" style pop-ups.

## Why Inertia (and not a REST or GraphQL API)?

In a classic SPA you have two teams of work: build a JSON API on the backend, and build a React app that fetches from it. That means duplicated validation, duplicated auth logic, custom URL/link handling on both sides.

Inertia removes the API layer. Your controller returns something like:

```php
return Inertia::render('dashboard', ['stats' => $stats]);
```

The first request returns a plain HTML shell with the page name + props embedded as JSON. Every follow-up navigation (`<Link>` click, form submit) is an AJAX request that returns *just* the JSON — Inertia swaps out the mounted page component without a full page reload. You get an SPA feel without maintaining an API.

## End-to-end trace: `GET /dashboard`

Follow this once and every other doc will make sense.

1. **User types `/dashboard` in the browser.**

2. **`bootstrap/app.php`** boots Laravel and registers `HandleInertiaRequests` middleware in the `web` group (see `bootstrap/app.php:20-24`).

3. **`routes/web.php` line 8** matches the URL:

   ```php
   Route::middleware(['auth', 'verified'])->group(function () {
       Route::inertia('dashboard', 'dashboard')->name('dashboard');
   });
   ```

   `Route::inertia('dashboard', 'dashboard')` is a shortcut that renders the Inertia page named `'dashboard'` with no controller class needed. The middleware chain requires the user to be logged in and email-verified.

4. **`HandleInertiaRequests::share()`** (`app/Http/Middleware/HandleInertiaRequests.php:36-46`) adds shared props that *every* Inertia response includes:

   ```php
   return [
       ...parent::share($request),
       'name' => config('app.name'),
       'auth' => ['user' => $request->user()],
       'sidebarOpen' => ! $request->hasCookie('sidebar_state') || $request->cookie('sidebar_state') === 'true',
   ];
   ```

5. **Response leaves the server.** If this is the very first visit, the response is a full HTML document with a `<div id="app" data-page='{...JSON...}'></div>` root. If the user was already in the SPA (clicked a `<Link>`), the response is a small JSON blob:

   ```json
   {
     "component": "dashboard",
     "props": {"name": "…", "auth": {"user": {…}}, "sidebarOpen": true},
     "url": "/dashboard",
     "version": "…"
   }
   ```

6. **Inertia client (`resources/js/app.tsx`)** reads the payload. It calls the `layout` resolver with `'dashboard'`. The resolver returns `AppLayout` (see `app.tsx:13-24`).

7. **Wayfinder-generated import** for `dashboard()` lives in `resources/js/routes/index.ts:225` — that's how the `Link` in the welcome page (`resources/js/pages/welcome.tsx:15`) can point at the dashboard with type safety.

8. **The page component `resources/js/pages/dashboard.tsx`** is loaded on demand. It has `Dashboard.layout = { breadcrumbs: [{ title: 'Dashboard', href: dashboard() }] }` at the bottom. Because the static `.layout` property is an *object* (not a component), Inertia spreads it as props onto the resolved layout:

   ```tsx
   <AppLayout breadcrumbs={[{ title: 'Dashboard', href: … }]}>
     <Dashboard />
   </AppLayout>
   ```

   Full details of this mechanism are in [05-layout-system.md](05-layout-system.md).

9. **`AppLayout`** wraps in an `AppSidebarLayout` which renders the sidebar, the breadcrumb header, and finally `{children}` — where `<Dashboard />` gets mounted.

10. **React renders**, the user sees the page. Any subsequent navigation is intercepted by Inertia's `<Link>` component and repeats steps 3–9 without a full page reload.

## What lives where

| Concern | Backend file(s) | Frontend file(s) |
|---------|-----------------|------------------|
| Routes | `routes/web.php`, `routes/settings.php`, Fortify auto-registers auth routes | `resources/js/routes/**` (generated) |
| Pages (screens) | Controllers returning `Inertia::render('page/name', $props)` | `resources/js/pages/**` |
| Layouts | — | `resources/js/layouts/**` |
| Reusable UI | — | `resources/js/components/**` |
| Shared props | `HandleInertiaRequests::share()` | Read via `usePage().props` |
| Forms | Controllers + FormRequests | `<Form {...store.form()}>` |
| Auth (login, 2FA, …) | Fortify + `FortifyServiceProvider` | Inertia pages under `pages/auth/**` |
| Styling | — | Tailwind classes + `components/ui/**` (shadcn) |

## What's next

- If you want to understand the **server side** first: [02-backend-architecture.md](02-backend-architecture.md).
- If you'd rather see the **client side** first: [03-frontend-architecture.md](03-frontend-architecture.md).
- The star chapter answering *"why does `Login.layout = { title, description }` work?"* is [05-layout-system.md](05-layout-system.md).
