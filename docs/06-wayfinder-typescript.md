# 06 — Wayfinder & TypeScript

Wayfinder is Laravel's answer to "how do I call backend routes from TypeScript without hardcoding URLs?" It reads your `routes/*.php` and your controllers, then emits typed `.ts` files that mirror them. When you import `store` from `@/routes/password/confirm`, you're using a machine-generated function that knows the URL, HTTP method, and (optionally) query params.

This chapter explains what gets generated, how to use it, and where the `@/` alias comes from.

## Why bother?

Before Wayfinder (or Ziggy), you'd write:

```tsx
// FRAGILE — string typo goes undetected until runtime
<Link href="/password/confirm">Confirm</Link>
```

Rename the route to `/user/confirm-password` on the server and this silently breaks. With Wayfinder:

```tsx
import { store } from '@/routes/password/confirm';
<Link href={store()} />
```

- Autocomplete lists every route.
- Rename the server route → run the dev server → the generated file updates → TypeScript errors point at every consumer.
- No handwritten mapping to maintain.

## What Wayfinder generates

Two folders, both under `resources/js/`:

- **`routes/`** — one file per route (or nested folder per route group). Contains typed callers for each URL.
- **`actions/`** — mirrors your controller class structure (`App/Http/Controllers/Settings/ProfileController.ts`). Same shape as routes but keyed by controller method.

Plus a small runtime helper at `resources/js/wayfinder/index.ts` that provides `queryParams()` and the shared TypeScript types.

The Vite plugin `@laravel/vite-plugin-wayfinder` (installed as a devDependency, wired up in `vite.config.ts:27-29`) regenerates these files whenever `routes/*.php` or your controllers change. `formVariants: true` enables the `.form()` helper you'll see below.

## Anatomy of a generated route file

Here's the file that `pages/auth/confirm-password.tsx` imports:

```ts
// resources/js/routes/password/confirm/index.ts  (excerpt)
import { queryParams, type RouteQueryOptions, type RouteDefinition, type RouteFormDefinition } from './../../../wayfinder'

/**
 * @see \Laravel\Fortify\Http\Controllers\ConfirmablePasswordController::store
 * @see vendor/laravel/fortify/src/Http/Controllers/ConfirmablePasswordController.php:51
 * @route '/user/confirm-password'
 */
export const store = (options?: RouteQueryOptions): RouteDefinition<'post'> => ({
    url: store.url(options),
    method: 'post',
})

store.definition = {
    methods: ["post"],
    url: '/user/confirm-password',
} satisfies RouteDefinition<["post"]>

store.url = (options?: RouteQueryOptions) => {
    return store.definition.url + queryParams(options)
}

store.post = (options?: RouteQueryOptions): RouteDefinition<'post'> => ({
    url: store.url(options),
    method: 'post',
})

const storeForm = (options?: RouteQueryOptions): RouteFormDefinition<'post'> => ({
    action: store.url(options),
    method: 'post',
})

store.form = storeForm
```

What's on `store` after all this:

- `store()` — returns `{ url: '/user/confirm-password', method: 'post' }`. Feed this to `<Link>` or `router.post(...)`.
- `store.url()` — just the URL string. Feed this to a raw `<a href>` or anywhere a string is expected.
- `store.post()` — same as `store()`, explicit for clarity.
- `store.form()` — returns `{ action: '/user/confirm-password', method: 'post' }`. Feed this to the `<Form>` component (which expects `action` + `method`, not `url` + `method`).
- `store.definition` — introspection: `{ methods: ['post'], url: '/user/confirm-password' }`.

All of them accept an optional `RouteQueryOptions` argument for query strings:

```ts
store({ query: { redirect: '/dashboard' } })
// → { url: '/user/confirm-password?redirect=%2Fdashboard', method: 'post' }
```

## Anatomy: `resources/js/wayfinder/index.ts`

The runtime helper is small. The parts you may care about:

```ts
export type RouteDefinition<TMethod extends Method | Method[]> = {
    url: string;
} & (TMethod extends Method[] ? { methods: TMethod } : { method: TMethod });

export type RouteFormDefinition<TMethod extends Method> = {
    action: string;
    method: TMethod;
};

export type RouteQueryOptions = {
    query?: QueryParams;
    mergeQuery?: QueryParams;
};

export const queryParams = (options?: RouteQueryOptions) => {
    // Builds a "?a=1&b=2" string from the object; supports arrays and nested objects.
    // With `mergeQuery`, merges into the current URL's existing query.
};
```

- `RouteDefinition` — narrowed by TypeScript: if you called `store.post(...)`, the resulting object's `method` is *literally* `'post'`, not just `string`. This lets `<Form>` (which is generic on method) type-check the payload.
- `RouteQueryOptions` — pass either `query` (replaces existing) or `mergeQuery` (keeps existing params and only overrides overlapping keys).

You'll rarely import these helpers directly; they exist because the generated code needs them.

## How the generated code is consumed

### In a `<Form>`

```tsx
// resources/js/pages/auth/confirm-password.tsx
import { store } from '@/routes/password/confirm';

<Form {...store.form()} resetOnSuccess={['password']}>
    …
</Form>
```

`store.form()` returns `{ action: '/user/confirm-password', method: 'post' }`. Spreading it into `<Form>` sets both attributes.

### In a `<Link>`

```tsx
// resources/js/pages/welcome.tsx
import { dashboard, login } from '@/routes';

<Link href={dashboard()}>Dashboard</Link>
<Link href={login()}>Log in</Link>
```

`dashboard()` returns `{ url: '/dashboard', method: 'get' }`. `<Link>` accepts either a string URL or that route object.

### In a settings nav

```tsx
// resources/js/layouts/settings/layout.tsx
import { edit as editAppearance } from '@/routes/appearance';
import { edit } from '@/routes/profile';
import { edit as editSecurity } from '@/routes/security';

const sidebarNavItems: NavItem[] = [
    { title: 'Profile',    href: edit(),           icon: null },
    { title: 'Security',   href: editSecurity(),   icon: null },
    { title: 'Appearance', href: editAppearance(), icon: null },
];
```

Renaming imports (`edit as editSecurity`) is standard when the same name (`edit`) appears in multiple files.

### Programmatically

```ts
import { router } from '@inertiajs/react';
import { destroy } from '@/routes/profile';

router.delete(destroy().url);   // or just: router.delete('/settings/profile');
```

## Actions vs routes — what's the difference?

- **`resources/js/routes/**`** — keyed by URL patterns (mirrors `routes/*.php`). Best when you're thinking "I want to navigate to `/settings/profile`."
- **`resources/js/actions/**`** — keyed by controller method (mirrors your PHP class tree). Best when you're thinking "I want to POST to `ProfileController@update`."

Both point at the same URLs. Use whichever names read more naturally at the call site. The pages in this repo mostly use `routes/`.

Example action:

```ts
import ProfileController from '@/actions/App/Http/Controllers/Settings/ProfileController';

<Form {...ProfileController.update.form()}>…</Form>
```

## Comparison to Ziggy

Ziggy is the older Laravel package that ships a JS helper `route('profile.edit')` — a string-based lookup. This project does not use Ziggy (verify: no `tightenco/ziggy` in `composer.json`, no `ziggy` in `package.json`). Wayfinder is strictly better here because you get real TypeScript autocompletion and typos are compile-time errors.

## The `@/` alias

Every import in the frontend uses `@/…`. That comes from `tsconfig.json:110-112`:

```json
"paths": {
    "@/*": ["./resources/js/*"]
}
```

- TypeScript uses this to resolve types.
- Vite picks up the same `paths` automatically (via `@vitejs/plugin-react` and the module resolver), so imports work at runtime too.

Without the alias, `resources/js/pages/settings/profile.tsx` would need to import a component like this:

```ts
import Heading from '../../../components/heading';  // 😬
```

With the alias:

```ts
import Heading from '@/components/heading';         // 🙂
```

## Regeneration

Normally automatic. The Vite plugin watches the PHP side and re-emits when routes change. If you ever need to force it, restart `npm run dev` (or run `composer run dev` which spawns the whole dev environment via `php artisan dev`).

If you see TypeScript errors after a route rename, it's usually because the old generated file cached in your editor. Restart the TS server (VS Code: `Cmd+Shift+P` → "Restart TS server").

## Don't edit generated files

Every file under `resources/js/routes/**`, `resources/js/actions/**`, and `resources/js/wayfinder/**` is regenerated. Edits will be blown away. `vite.config.ts:49-52` excludes them from the linter for that reason.

If a generated file is missing a helper you want (e.g. a specific query-param shape), the fix is to change the Laravel route/controller — not to edit the TS.

## What's next

- Every external package explained: [07-packages.md](07-packages.md).
- Or jump to hooks & utilities: [08-hooks-and-utilities.md](08-hooks-and-utilities.md).
