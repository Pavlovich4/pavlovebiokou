# 08 — Custom Hooks & Utilities

The `resources/js/hooks/` and `resources/js/lib/` folders contain small pieces of code you'll import often. This chapter reads each one line by line and explains the React ideas along the way.

## Quick React vocabulary

Before diving in:

- **Hook** — a function whose name starts with `use…` and is called from a React component or another hook. React runs it every time the component re-renders. Hooks let a component "remember" values across renders (`useState`), run side effects (`useEffect`), etc.
- **Effect** — code that runs *after* React commits to the DOM. Used for subscriptions, timers, listeners. Runs with `useEffect(fn, deps)`; the returned function is the cleanup, run on unmount or before the next effect run.
- **State** — a value the component owns. When it changes, React re-renders. Set with `useState()`.
- **Ref** — a mutable box that persists across renders without triggering a re-render. From `useRef()`.

## `lib/utils.ts` — the tiniest but most-used file

```ts
// resources/js/lib/utils.ts
import type { InertiaLinkProps } from '@inertiajs/react';
import { clsx } from 'clsx';
import type { ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) {
    return twMerge(clsx(inputs));
}

export function toUrl(url: NonNullable<InertiaLinkProps['href']>): string {
    return typeof url === 'string' ? url : url.url;
}
```

### `cn(...classes)`

Merges className strings safely.

- `clsx('a', condition && 'b', { 'c': isActive })` → `'a b c'` when the condition is truthy — filters falsy values, expands objects.
- `twMerge('bg-red-500 bg-blue-500 p-4')` → `'bg-blue-500 p-4'` — knows Tailwind semantics; a later `bg-*` wins.
- `cn(...)` combines both so you get conditional class strings AND Tailwind conflict resolution.

Real usage: `resources/js/components/input-error.tsx:12` — `className={cn('text-sm text-red-600 …', className)}`. That lets the caller override or add classes without fighting the defaults.

### `toUrl(hrefOrRoute)`

Wayfinder route callers return `{ url: '/foo', method: 'get' }`. Sometimes you need just the URL string (e.g. as a React `key`). `toUrl(...)` accepts either a string or the route object and returns the string.

Example: `resources/js/layouts/settings/layout.tsx:49` — `key={`${toUrl(item.href)}-${index}`}`.

## `hooks/use-appearance.tsx` — light/dark/system theme

This is the most involved custom hook, and a good introduction to React's `useSyncExternalStore`. Full source:

```tsx
// resources/js/hooks/use-appearance.tsx
import { useSyncExternalStore } from 'react';

export type ResolvedAppearance = 'light' | 'dark';
export type Appearance = ResolvedAppearance | 'system';

const listeners = new Set<() => void>();
let currentAppearance: Appearance = 'system';

const prefersDark = (): boolean => {
    if (typeof window === 'undefined') return false;
    return window.matchMedia('(prefers-color-scheme: dark)').matches;
};

const setCookie = (name: string, value: string, days = 365): void => {
    if (typeof document === 'undefined') return;
    const maxAge = days * 24 * 60 * 60;
    document.cookie = `${name}=${value};path=/;max-age=${maxAge};SameSite=Lax`;
};

const getStoredAppearance = (): Appearance => {
    if (typeof window === 'undefined') return 'system';
    return (localStorage.getItem('appearance') as Appearance) || 'system';
};

const isDarkMode = (appearance: Appearance): boolean =>
    appearance === 'dark' || (appearance === 'system' && prefersDark());

const applyTheme = (appearance: Appearance): void => {
    if (typeof document === 'undefined') return;
    const isDark = isDarkMode(appearance);
    document.documentElement.classList.toggle('dark', isDark);
    document.documentElement.style.colorScheme = isDark ? 'dark' : 'light';
};

const subscribe = (callback: () => void) => {
    listeners.add(callback);
    return () => listeners.delete(callback);
};

const notify = (): void => listeners.forEach((listener) => listener());
const mediaQuery = (): MediaQueryList | null =>
    typeof window === 'undefined' ? null : window.matchMedia('(prefers-color-scheme: dark)');
const handleSystemThemeChange = (): void => applyTheme(currentAppearance);

export function initializeTheme(): void {
    if (typeof window === 'undefined') return;
    if (!localStorage.getItem('appearance')) {
        localStorage.setItem('appearance', 'system');
        setCookie('appearance', 'system');
    }
    currentAppearance = getStoredAppearance();
    applyTheme(currentAppearance);
    mediaQuery()?.addEventListener('change', handleSystemThemeChange);
}

export function useAppearance(): UseAppearanceReturn {
    const appearance: Appearance = useSyncExternalStore(
        subscribe,
        () => currentAppearance,
        () => 'system',
    );

    const resolvedAppearance: ResolvedAppearance = isDarkMode(appearance) ? 'dark' : 'light';

    const updateAppearance = (mode: Appearance): void => {
        currentAppearance = mode;
        localStorage.setItem('appearance', mode);
        setCookie('appearance', mode);
        applyTheme(mode);
        notify();
    };

    return { appearance, resolvedAppearance, updateAppearance } as const;
}
```

### The mental model

Three concerns:

1. **Store** — a *module-level* `currentAppearance` variable. Not React state. Persists across the whole app.
2. **Persistence** — write to `localStorage` (survives page reloads) AND a cookie (readable by the server for SSR). That's why `bootstrap/app.php:18` excepts `appearance` from encryption.
3. **Subscription** — anywhere `useAppearance()` is called, that component re-renders when the theme changes.

### `useSyncExternalStore` — what is it?

Added in React 18. Subscribes to an external data source (any store outside React) in a concurrent-mode-safe way. Signature:

```ts
useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot?)
```

- `subscribe(callback)` — you tell the store to call `callback` whenever the data changes. Return an unsubscribe.
- `getSnapshot()` — return the current value.
- `getServerSnapshot()` — return the value on the server (used during SSR to avoid hydration mismatches).

Whenever `subscribe`'s callback is invoked, React knows the value may have changed and calls `getSnapshot()` again. If the value is `!==` the previous, it re-renders any component using this hook.

### Trace: user clicks "Dark" in appearance settings

1. `updateAppearance('dark')` is called.
2. It sets the module var `currentAppearance = 'dark'`.
3. Writes localStorage and the cookie.
4. `applyTheme('dark')` toggles the `dark` class on `<html>`. Tailwind's `dark:` variants take effect immediately.
5. `notify()` calls every subscriber (which are the `useSyncExternalStore` callbacks registered by each `useAppearance()` consumer).
6. Every component using `useAppearance` re-renders, `getSnapshot()` returns `'dark'`, `resolvedAppearance` becomes `'dark'`.

### Why the SSR guards?

`typeof window === 'undefined'` and `typeof document === 'undefined'` — these run at module evaluation time. If the app is server-rendered (Inertia SSR at `127.0.0.1:13714`), there is no `window`. The guards make sure the code doesn't crash.

### `initializeTheme()` called from `app.tsx:40`

Runs *outside* React so the `dark` class is on `<html>` before the first render — no flash of the wrong theme.

## `hooks/use-flash-toast.ts` — server-side flash → Sonner toast

```ts
// resources/js/hooks/use-flash-toast.ts
import { router } from '@inertiajs/react';
import { useEffect } from 'react';
import { toast } from 'sonner';
import type { FlashToast } from '@/types/ui';

export function useFlashToast(): void {
    useEffect(() => {
        return router.on('flash', (event) => {
            const flash = (event as CustomEvent).detail?.flash;
            const data = flash?.toast as FlashToast | undefined;

            if (!data) {
                return;
            }

            toast[data.type](data.message);
        });
    }, []);
}
```

Line by line:

- `router.on('flash', cb)` — Inertia's client fires a `'flash'` event whenever the server included flash data in a response.
- The event is a `CustomEvent`; `event.detail.flash` is the flash payload object.
- We check for a `toast` key specifically (`flash.toast`) so this hook only reacts to toasts.
- `toast[data.type]` — Sonner exposes `toast.success`, `toast.error`, `toast.warning`, `toast.info`. Indexing by the string type lets one call handle all four.
- `router.on` returns an unsubscribe function; returning it from `useEffect` means React runs it on unmount.
- Empty deps `[]` — subscribe once, ever.

Wire-up on the backend: `Inertia::flash('toast', ['type' => 'success', 'message' => 'Profile updated.'])` (see `ProfileController::update` at `app/Http/Controllers/Settings/ProfileController.php:41`).

Where the hook is called: usually in a layout so it lives for the whole session. Check `resources/js/layouts/` for a `useFlashToast()` invocation.

## Other hooks (short summaries)

- **`hooks/use-current-url.ts`** — returns helpers like `isCurrentOrParentUrl(href)` used by the settings nav (`layout.tsx:32, 54`) to highlight the active link. Reads the current URL from `usePage().url`.

- **`hooks/use-mobile.tsx`** — subscribes to a `matchMedia('(max-width: …)')` query and returns a boolean. Used by shadcn's sidebar to decide whether to render as a drawer or a fixed sidebar.

- **`hooks/use-mobile-navigation.ts`** — small state helper for mobile drawer open/close.

- **`hooks/use-initials.tsx`** — takes `"Jane Doe"`, returns `"JD"`. Used in avatar fallbacks (see `components/user-info.tsx`).

- **`hooks/use-clipboard.ts`** — copy-to-clipboard helper with a "copied!" flag. Used by 2FA recovery codes UI.

- **`hooks/use-two-factor-auth.ts`** — encapsulates the 2FA setup flow: fetching the QR code, secret key, and recovery codes; enabling/disabling; confirming with a TOTP.

Read each file when you touch related UI; they're small.

## Convention: how new hooks go here

1. File name: `use-something.ts` or `.tsx`.
2. Export the hook function named `useSomething`.
3. Import from `@/hooks/use-something` at call sites.
4. If the hook wraps a store (like `use-appearance`), put the store in the same file at module scope.
5. If it plugs into an Inertia router event, follow the `useEffect(() => router.on(...), [])` pattern.

## What's next

- The worked example: [09-confirm-password-line-by-line.md](09-confirm-password-line-by-line.md).
- Or practical how-tos: [10-adding-features.md](10-adding-features.md).
