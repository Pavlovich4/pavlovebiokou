# 04 — Inertia Primitives (for a React novice)

Once you understand these seven things — `<Head>`, `<Link>`, `<Form>`, `usePage`, `router`, flash data, and the progress bar — you know 90% of what Inertia gives you. All imports come from `@inertiajs/react`.

## `<Head>` — set the page title (and other head tags)

```tsx
import { Head } from '@inertiajs/react';

export default function Dashboard() {
    return (
        <>
            <Head title="Dashboard" />
            <div>…the page…</div>
        </>
    );
}
```

- Inertia injects the `title` into the document's `<title>` tag. Because `app.tsx:12` wraps titles with `${title} - ${appName}`, this renders `<title>Dashboard - Laravel</title>`.
- `<Head>` doesn't render any DOM itself — it just manages `<head>` tags via a helper mechanism. React returns `<> … </>` (a fragment) because you need to return one element from a component.
- Real example: `resources/js/pages/dashboard.tsx:8`.

## `<Link>` — SPA-style navigation

```tsx
import { Link } from '@inertiajs/react';
import { dashboard } from '@/routes';

<Link href={dashboard()} className="…">
    Go to dashboard
</Link>
```

- Looks like an anchor, feels like an anchor, but Inertia intercepts the click. Instead of a full page reload, it fires an AJAX request, gets back the JSON payload for the next page, and swaps out the mounted page component.
- `dashboard()` is a Wayfinder-generated route function (see [06-wayfinder-typescript.md](06-wayfinder-typescript.md)). It returns `{ url: '/dashboard', method: 'get' }`. `<Link>` accepts either a string URL or that object.
- Real example: `resources/js/pages/welcome.tsx:14-19`.

Useful props on `<Link>`:
- `method="post"` — for POST/PUT/DELETE links (e.g. a "Logout" button that's really a `<Link method="post" href={logout()}>` in disguise).
- `as="button"` — render as a `<button>` instead of `<a>` (useful when it's really an action, not navigation).
- `preserveScroll` / `preserveState` — advanced options for keeping scroll position or state across the visit.

## `<Form>` — the v3 form component

The modern way to submit a form in Inertia. Replaces the older `useForm` hook.

```tsx
import { Form } from '@inertiajs/react';
import { store } from '@/routes/password/confirm';

<Form {...store.form()} resetOnSuccess={['password']}>
    {({ processing, errors }) => (
        <div>
            <input name="password" />
            <p>{errors.password}</p>
            <button disabled={processing}>Confirm</button>
        </div>
    )}
</Form>
```

Line by line:

- `store.form()` returns `{ action: '/user/confirm-password', method: 'post' }`. Spreading it into `<Form>` sets both.
- `resetOnSuccess={['password']}` — after a successful submit, empty out the `password` field (so it doesn't linger in the DOM).
- `{({ processing, errors }) => (…)}` is the **render-prop pattern**. `<Form>` calls its children as a function, passing:
  - `processing: boolean` — `true` while the request is in-flight. Use to disable the submit button and show a spinner.
  - `errors: Record<string, string>` — validation errors from the server, keyed by field name. Empty when the form is valid or hasn't been submitted yet.
- `<Form>` handles:
  - Reading the values from named form inputs (`<input name="password" />` becomes payload `{ password: '…' }`).
  - Attaching the CSRF token automatically.
  - Sending the request to `action`.
  - Populating `errors` from the response if validation fails.
  - Following the redirect on success.

Real examples:
- `resources/js/pages/auth/confirm-password.tsx:14` (studied in depth in [09-confirm-password-line-by-line.md](09-confirm-password-line-by-line.md)).
- `resources/js/pages/auth/login.tsx:23-27` — larger form with multiple fields.

### The older `useForm` hook

Not used heavily here, but you may see it in older tutorials:

```tsx
import { useForm } from '@inertiajs/react';

const { data, setData, post, processing, errors } = useForm({
    email: '', password: '', remember: false,
});

<form onSubmit={(e) => { e.preventDefault(); post('/login'); }}>
    <input value={data.email} onChange={e => setData('email', e.target.value)} />
</form>
```

The `<Form>` component is nicer for standard cases. Use `useForm` when you need imperative control (e.g. multi-step wizards).

## `usePage()` — read shared props on the client

```tsx
import { usePage } from '@inertiajs/react';

export default function Nav() {
    const { auth, name } = usePage().props;

    return (
        <div>
            <span>{name}</span>
            {auth.user ? <span>Hi {auth.user.name}</span> : <a href="/login">Log in</a>}
        </div>
    );
}
```

- Returns the current page object: `{ component, props, url, version }`.
- The shape of `props` is whatever the current page controller returned *plus* whatever `HandleInertiaRequests::share()` added.
- The `auth.user` and `name` you see here come from the shared props defined in `app/Http/Middleware/HandleInertiaRequests.php:38-44`.
- Real example: `resources/js/pages/welcome.tsx:5`.

Typing tip: cast `usePage<Props>()` where `Props` is your custom prop type if you want full autocompletion.

## `router` — imperative navigation

For when you need to trigger a visit from code (not from a `<Link>` or `<Form>`).

```tsx
import { router } from '@inertiajs/react';

router.visit('/dashboard');
router.post('/logout');
router.delete(`/users/${id}`);
router.reload({ only: ['stats'] });        // re-request current page, keep only 'stats' prop
router.cancelAll();                        // abort any in-flight request
```

The `router` is also how Inertia exposes low-level events like the `flash` event that `use-flash-toast` listens to:

```ts
router.on('flash', (event) => { … });
```

## Flash data → toast pipeline

Server side (controller):

```php
Inertia::flash('toast', ['type' => 'success', 'message' => 'Profile updated.']);
return to_route('profile.edit');
```

Client side (`resources/js/hooks/use-flash-toast.ts`):

```ts
import { router } from '@inertiajs/react';
import { useEffect } from 'react';
import { toast } from 'sonner';

export function useFlashToast(): void {
    useEffect(() => {
        return router.on('flash', (event) => {
            const flash = (event as CustomEvent).detail?.flash;
            const data = flash?.toast;
            if (!data) return;
            toast[data.type](data.message);
        });
    }, []);
}
```

Trace:

1. Controller calls `Inertia::flash('toast', {...})` — Inertia adds the payload to a `flash` field on the response.
2. Inertia client fires a `'flash'` event on `router` with `event.detail.flash`.
3. `use-flash-toast` subscribes with `router.on('flash', cb)`, and calls Sonner's `toast.success(...)` (or `.error`, `.warning`, `.info`).
4. `<Toaster />` (added in `app.tsx:30`) renders the actual toast UI.

The `useEffect(() => router.on(...), [])` pattern: `router.on` returns an unsubscribe function; React runs that on unmount. Empty dependency array means "run once on mount".

The hook is called once at the top of the app (e.g. in a layout) and works everywhere from then on.

## Progress bar

`app.tsx:34-36`:

```tsx
progress: { color: '#4B5563' },
```

Every time Inertia is loading a new page, a thin colored bar shows at the top of the browser. You can also disable it (`progress: false`) or customize the delay, color, and include-spinner options.

## Deferred / optional / merge props (v3 features)

Not used in this app yet, but you'll hit them eventually.

- **`Inertia::optional(fn () => …)`** — the prop is not evaluated on the initial visit, only if the client asks for it explicitly via `router.reload({ only: ['expensiveThing'] })`. Replaces the old `Inertia::lazy()`.
- **`Inertia::defer(fn () => …)`** — the initial page ships without this prop, then Inertia does a follow-up request in the background to fetch it. Great for below-the-fold data. Use with an animated skeleton in the UI.
- **`Inertia::merge(fn () => …)`** — new response merges with the existing prop instead of replacing it. Useful for infinite scroll (appending pages).

## Recap: request → response → UI

For a form submit:

1. `<Form>` (or `router.post`) sends `POST /some/url` with the field values and CSRF token.
2. Server middleware runs (`HandleInertiaRequests` re-attaches shared props).
3. Controller either:
   - Redirects on success → Inertia follows the redirect, fetches JSON for the target, remounts.
   - Redirects back with validation errors → the current page remounts, `errors` populates in your render-prop.
4. `<Form>` sets `processing` back to `false`.

For a `<Link>` click:

1. Inertia intercepts, sends `GET` for the target.
2. Response contains `{ component, props, url }`.
3. Inertia loads (or re-uses) the page module, resolves the layout, renders.

## What's next

- Understand the layout resolver and `Page.layout`: [05-layout-system.md](05-layout-system.md).
- Or dive into typed routes: [06-wayfinder-typescript.md](06-wayfinder-typescript.md).
