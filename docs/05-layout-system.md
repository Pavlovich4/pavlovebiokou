# 05 — The Layout System

This is the chapter that answers your specific question about `ConfirmPassword.layout = { title, description }`. By the end you'll understand exactly how those two strings end up as an `<h1>` and a `<p>` in the DOM, and you'll be able to add, swap, or customize any layout with confidence.

## The two mechanisms Inertia gives you

Inertia v3 supports two ways to attach a layout to a page:

1. **Per-page persistent layout** — set on the page component itself: `Page.layout = MyLayout`. Historically this was the default.
2. **Global resolver** — set once in `createInertiaApp({ layout: (name) => Component })`. Runs for every page.

**This project uses the global resolver.** Look at `resources/js/app.tsx:13-24`:

```tsx
layout: (name) => {
    switch (true) {
        case name === 'welcome':
            return null;
        case name.startsWith('auth/'):
            return AuthLayout;
        case name.startsWith('settings/'):
            return [AppLayout, SettingsLayout];
        default:
            return AppLayout;
    }
},
```

The resolver is called with the **page name** (the string you passed to `Inertia::render(...)` on the server). It returns:

- `null` — no layout; the page renders standalone (used by `welcome`).
- A single component — that component wraps the page.
- An array of components — nested layouts, outer to inner.

## But then how does `Page.layout = { title, description }` work?

Both mechanisms coexist. Inertia's rule is:

> When the resolved layout is a *component*, and the page component has a static `.layout` property that is an **object**, that object is spread as **props** onto the layout component.

Put another way, this…

```tsx
// resources/js/pages/auth/confirm-password.tsx  (bottom)
ConfirmPassword.layout = {
    title: 'Confirm password',
    description: 'This is a secure area of the application. Please confirm your password before continuing.',
};
```

…combined with `layout: (name) => name.startsWith('auth/') ? AuthLayout : …` from the resolver, produces effectively this tree at render time:

```tsx
<AuthLayout
    title="Confirm password"
    description="This is a secure area of the application. Please confirm your password before continuing."
>
    <ConfirmPassword />
</AuthLayout>
```

Notice the object keys become props on the layout, and the page component is passed as `children`.

That's the whole trick. The rest of this doc traces how those props travel through the layout files.

## Tracing the props: from static object to DOM

### Step 1 — the outer layout is a shim

```tsx
// resources/js/layouts/auth-layout.tsx
import AuthLayoutTemplate from '@/layouts/auth/auth-simple-layout';

export default function AuthLayout({
    title = '',
    description = '',
    children,
}: {
    title?: string;
    description?: string;
    children: React.ReactNode;
}) {
    return (
        <AuthLayoutTemplate title={title} description={description}>
            {children}
        </AuthLayoutTemplate>
    );
}
```

- Accepts `title`, `description`, and `children` as props. Defaults them to empty strings.
- Delegates to `AuthLayoutTemplate` (which is imported from `./auth/auth-simple-layout`).

Why a two-file split? So the *shape* of the layout API (props: title, description, children) is stable, and you can swap the *implementation* (simple vs card vs split) by changing one import line. See the "Change the auth style" section below.

### Step 2 — the concrete template renders the DOM

```tsx
// resources/js/layouts/auth/auth-simple-layout.tsx
import { Link } from '@inertiajs/react';
import AppLogoIcon from '@/components/app-logo-icon';
import { home } from '@/routes';
import type { AuthLayoutProps } from '@/types';

export default function AuthSimpleLayout({
    children,
    title,
    description,
}: AuthLayoutProps) {
    return (
        <div className="bg-background flex min-h-svh flex-col items-center justify-center gap-6 p-6 md:p-10">
            <div className="w-full max-w-sm">
                <div className="flex flex-col gap-8">
                    <div className="flex flex-col items-center gap-4">
                        <Link href={home()} className="…">
                            <div className="…"><AppLogoIcon className="…" /></div>
                            <span className="sr-only">{title}</span>
                        </Link>

                        <div className="space-y-2 text-center">
                            <h1 className="text-xl font-medium">{title}</h1>
                            <p className="text-muted-foreground text-center text-sm">
                                {description}
                            </p>
                        </div>
                    </div>
                    {children}
                </div>
            </div>
        </div>
    );
}
```

Notice lines 27 and 28: `<h1>{title}</h1>` and `<p>…{description}…</p>`. That's where your `Page.layout = { title, description }` finally lands in the DOM.

So the full chain for `pages/auth/confirm-password.tsx` is:

```
ConfirmPassword.layout = { title: 'Confirm password', description: '…' }
      │
      ▼
Inertia resolver sees page name 'auth/confirm-password' → returns AuthLayout
      │
      ▼
Inertia mounts: <AuthLayout title="Confirm password" description="…">
                    <ConfirmPassword />
                </AuthLayout>
      │
      ▼
AuthLayout forwards to <AuthSimpleLayout title="…" description="…">
      │
      ▼
DOM:  <h1>Confirm password</h1>
      <p>This is a secure area of the application…</p>
      <div>…the ConfirmPassword page form…</div>
```

Same story for `Login.layout` on `pages/auth/login.tsx:103-106`, and for `Dashboard.layout = { breadcrumbs: [...] }` on `pages/dashboard.tsx:29-36` (with `AppLayout` receiving the breadcrumbs and forwarding them to `AppSidebarLayout` → `AppSidebarHeader`).

## Nested layouts (settings pages)

The resolver returns an **array** for settings pages:

```tsx
case name.startsWith('settings/'):
    return [AppLayout, SettingsLayout];
```

That means the render tree is:

```tsx
<AppLayout>
    <SettingsLayout>
        <SettingsProfile />
    </SettingsLayout>
</AppLayout>
```

Outer wraps inner. `AppLayout` provides the sidebar + main content shell; `SettingsLayout` fills the content region with its own sub-navigation (Profile / Security / Appearance) and passes the actual page component as its `children`.

Look at `resources/js/layouts/settings/layout.tsx` — it renders the settings sub-nav (using Wayfinder routes `edit`, `editSecurity`, `editAppearance`) and highlights the current route with `useCurrentUrl`.

If a page under `settings/*` sets `Page.layout = { breadcrumbs: […] }`, those breadcrumbs are passed to *all* layouts in the array. Since `SettingsLayout`'s props type is `PropsWithChildren` only, it ignores `breadcrumbs`; `AppLayout` accepts them and uses them.

## The full menu of layouts in this repo

| File | Purpose |
|------|---------|
| `layouts/app-layout.tsx` | Thin shim → forwards to `AppSidebarLayout`. |
| `layouts/app/app-sidebar-layout.tsx` | Sidebar + main content + breadcrumb header. Used by everything under `/` and `/settings/*` except auth. |
| `layouts/app/app-header-layout.tsx` | Alternative shell with a top header instead of a sidebar. Not currently wired up by the resolver. |
| `layouts/auth-layout.tsx` | Thin shim → forwards to `AuthSimpleLayout`. |
| `layouts/auth/auth-simple-layout.tsx` | Centered logo + title/description + children. Used by all `pages/auth/*`. |
| `layouts/auth/auth-card-layout.tsx` | Card-style variant. Not currently wired up. |
| `layouts/auth/auth-split-layout.tsx` | Two-column (image + form) variant. Not currently wired up. |
| `layouts/settings/layout.tsx` | Sub-nav for /settings/*. |

## Recipes

### Change the auth pages from "simple" to "card" style

One-line change in `resources/js/layouts/auth-layout.tsx`:

```diff
- import AuthLayoutTemplate from '@/layouts/auth/auth-simple-layout';
+ import AuthLayoutTemplate from '@/layouts/auth/auth-card-layout';
```

That's it. All `pages/auth/*` now use the card layout, and any `Page.layout = { title, description }` still works because the card layout accepts the same prop shape (`AuthLayoutProps` from `resources/js/types/ui.ts:16-21`).

### Add a brand-new layout (e.g. for a marketing area)

1. Create `resources/js/layouts/marketing-layout.tsx`:

   ```tsx
   import type { ReactNode } from 'react';
   export default function MarketingLayout({
       children,
       heroTitle = '',
   }: {
       children: ReactNode;
       heroTitle?: string;
   }) {
       return (
           <div>
               <header className="bg-black text-white p-8">{heroTitle}</header>
               <main>{children}</main>
           </div>
       );
   }
   ```

2. Wire it up in `resources/js/app.tsx` — add a case to the resolver:

   ```tsx
   case name.startsWith('marketing/'):
       return MarketingLayout;
   ```

3. On the server, render pages under that prefix:

   ```php
   Route::inertia('pricing', 'marketing/pricing');
   ```

4. Set per-page hero title from the page component:

   ```tsx
   Pricing.layout = { heroTitle: 'Pick a plan' };
   ```

### Opt a single page out of the layout

Either name the page in a way the resolver returns `null` for (like `welcome`), or override `Page.layout = null` at the bottom of the file. The global resolver runs first, but if the static `.layout` is explicitly `null` (or a component), the page's own value wins.

### Give a page a component-based layout (advanced)

If you want a page to have a totally custom layout that isn't in the resolver, you can assign a component:

```tsx
Foo.layout = (page: ReactNode) => <SpecialLayout>{page}</SpecialLayout>;
```

Rarely needed given the global resolver, but it's the escape hatch.

## Why is the mechanism designed this way?

- **Global resolver** = zero boilerplate at the page level. New auth page? Drop it in `pages/auth/`; it's already wrapped.
- **`Page.layout = { … }` object override** = per-page customization without duplicating the layout wrapper. The page just declares "here's my title", not "here's my whole shell".
- **The shim layer** (`auth-layout.tsx` re-exports `auth-simple-layout.tsx`) = one place to change styles for all auth pages.

Combined, you can add a new auth page in one file and it automatically gets the right shell, title bar, dark-mode support, logo, and description — you only write the form.

## Common pitfalls

- **`Page.layout` must be assigned to the same identifier you `export default`.** If you rename the function but forget the assignment, the object is orphaned. In this repo the convention is: the function name matches the assignment (`function ConfirmPassword …` + `ConfirmPassword.layout = …`).
- **The keys in your object must match the props the layout accepts.** `AuthLayout` expects `title` and `description`. If you write `Page.layout = { titel: 'Login' }` (typo), TypeScript won't catch it because `.layout` is loosely typed. The rendered layout will just show empty strings.
- **Nested layouts share the same props.** If two layouts in the array both accept `breadcrumbs`, both get them. Usually harmless.

## What's next

- Type-safe URLs and forms: [06-wayfinder-typescript.md](06-wayfinder-typescript.md).
- The confirm-password page, line by line: [09-confirm-password-line-by-line.md](09-confirm-password-line-by-line.md).
