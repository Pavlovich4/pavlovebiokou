# 09 — `confirm-password.tsx` Line by Line

You asked to understand this specific file end-to-end. Now that the pieces are covered (Inertia, Wayfinder, the layout system, hooks), let's walk it. File under study: `resources/js/pages/auth/confirm-password.tsx`.

Full source, then a line-by-line commentary.

```tsx
// resources/js/pages/auth/confirm-password.tsx
import { Form, Head } from '@inertiajs/react';                     // 1
import InputError from '@/components/input-error';                 // 2
import PasswordInput from '@/components/password-input';           // 3
import { Button } from '@/components/ui/button';                   // 4
import { Label } from '@/components/ui/label';                     // 5
import { Spinner } from '@/components/ui/spinner';                 // 6
import { store } from '@/routes/password/confirm';                 // 7

export default function ConfirmPassword() {                        // 9
    return (                                                        // 10
        <>                                                          // 11
            <Head title="Confirm password" />                       // 12

            <Form {...store.form()} resetOnSuccess={['password']}>  // 14
                {({ processing, errors }) => (                      // 15
                    <div className="space-y-6">                     // 16
                        <div className="grid gap-2">                // 17
                            <Label htmlFor="password">Password</Label> // 18
                            <PasswordInput                          // 19
                                id="password"
                                name="password"
                                placeholder="Password"
                                autoComplete="current-password"
                                autoFocus
                            />

                            <InputError message={errors.password} /> // 27
                        </div>

                        <div className="flex items-center">          // 30
                            <Button
                                className="w-full"
                                disabled={processing}
                                data-test="confirm-password-button"
                            >
                                {processing && <Spinner />}          // 36
                                Confirm password
                            </Button>
                        </div>
                    </div>
                )}
            </Form>
        </>
    );
}

ConfirmPassword.layout = {                                          // 47
    title: 'Confirm password',
    description:
        'This is a secure area of the application. Please confirm your password before continuing.',
};
```

## Line-by-line commentary

### Line 1 — `import { Form, Head } from '@inertiajs/react';`
- `Form` is Inertia's v3 form component. It wraps the browser XHR, sets the CSRF token, packages named inputs into a payload, and exposes `processing` + `errors` via a render prop.
- `Head` manages document `<head>` tags. Here it sets `<title>`.

### Lines 2–6 — local component imports
- `InputError` (`components/input-error.tsx`) — renders a `<p>` with red text if `message` is truthy, else nothing.
- `PasswordInput` (`components/password-input.tsx`) — a text input with a show/hide toggle (eye icon).
- `Button`, `Label`, `Spinner` — shadcn/ui primitives from `components/ui/`.

### Line 7 — `import { store } from '@/routes/password/confirm';`
- `store` is a **Wayfinder-generated** function representing the POST route `/user/confirm-password`. Called as `store()` it returns `{ url, method }`. Called as `store.form()` it returns `{ action: '/user/confirm-password', method: 'post' }`, which is exactly what `<Form>` needs.
- The generated file is `resources/js/routes/password/confirm/index.ts`. See [06-wayfinder-typescript.md](06-wayfinder-typescript.md) for the anatomy.
- Which Fortify controller handles the POST? The comment in the generated file tells you: `Laravel\Fortify\Http\Controllers\ConfirmablePasswordController::store`.

### Line 9 — `export default function ConfirmPassword() {`
- Named export required by Inertia's naming convention (Inertia does `page.default`).
- The function returns JSX that becomes the mounted DOM.

### Line 11 — `<>`
- A **React fragment**. Every component must return exactly one root element. `<>…</>` groups multiple siblings without adding a wrapper `<div>` to the DOM.

### Line 12 — `<Head title="Confirm password" />`
- Because `app.tsx:12` wraps titles as `${title} - ${appName}`, the resulting `<title>` tag is `"Confirm password - Laravel"` (or whatever `VITE_APP_NAME` is set to).

### Line 14 — `<Form {...store.form()} resetOnSuccess={['password']}>`
- `store.form()` → `{ action: '/user/confirm-password', method: 'post' }`. Spreading it sets both attributes on the form element.
- `resetOnSuccess={['password']}` — after a successful submit, clear the `password` field. Prevents the plaintext password from lingering in the DOM.

### Line 15 — `{({ processing, errors }) => (`
- **Render prop pattern.** Instead of children being static JSX, `<Form>`'s children are a function that returns JSX. `<Form>` calls this function with a state object, so the children can read submission state without any extra hooks.
- `processing` — `true` while the request is in flight.
- `errors` — an object keyed by field name. Empty when there are no validation errors. Populated by Inertia when the server responds with a 422.

### Lines 16–17 — layout classes
- `space-y-6` — Tailwind: vertical spacing of `1.5rem` between children.
- `grid gap-2` — Tailwind: CSS grid with `0.5rem` gap between children.

### Line 18 — `<Label htmlFor="password">Password</Label>`
- shadcn's `Label` wraps Radix's label primitive. Clicking the label focuses the input with matching `id`.

### Lines 19–26 — the password input
- `id="password"` — used by the Label's `htmlFor`.
- `name="password"` — the field name Inertia sends to the server. Must match the validation rule name on the backend (`ConfirmablePasswordController@store` expects `'password'`).
- `autoComplete="current-password"` — hints to the browser's password manager.
- `autoFocus` — focuses the input on mount.
- Because `<PasswordInput>` spreads `...props` onto the underlying `<input>` (see `components/password-input.tsx:11`), all these attributes reach the DOM element.

### Line 27 — `<InputError message={errors.password} />`
- Reads `errors.password` from the render-prop state. If the server responded with `{ errors: { password: 'The password is incorrect.' } }`, the `<p>` renders that message. Otherwise nothing is rendered.

### Lines 30–39 — the submit button
- `<Button>` is a shadcn primitive. By default it renders a `<button type="submit">`.
- `disabled={processing}` — prevents double-submission and gives visual feedback.
- `data-test="confirm-password-button"` — a hook for automated tests (e.g. Playwright / Pest browser tests). No effect on runtime; testing code uses `page.locator('[data-test="confirm-password-button"]')` to find and click it reliably even if the visible label changes.
- Line 36 — `{processing && <Spinner />}` — a JSX conditional. When `processing` is truthy, renders `<Spinner />`; when falsy, renders nothing (React ignores `false`).

### Lines 47–51 — the layout override

```tsx
ConfirmPassword.layout = {
    title: 'Confirm password',
    description:
        'This is a secure area of the application. Please confirm your password before continuing.',
};
```

Full mechanism is in [05-layout-system.md](05-layout-system.md); here's the short version for this specific page.

1. `app.tsx:17-18` — the layout resolver matches page names starting with `auth/` and returns `AuthLayout`. Since this file is `pages/auth/confirm-password.tsx`, its page name is `'auth/confirm-password'`. Match.
2. Because `ConfirmPassword.layout` is an *object* (not a component, not a function, not `null`), Inertia spreads it as props onto `AuthLayout`.
3. `AuthLayout` (`resources/js/layouts/auth-layout.tsx`) accepts `{ title, description, children }` and forwards them to `AuthSimpleLayout`.
4. `AuthSimpleLayout` (`resources/js/layouts/auth/auth-simple-layout.tsx:27-28`) renders `<h1>{title}</h1>` and `<p>{description}</p>` above the `{children}` slot.

Final DOM shape (paraphrased):

```html
<div class="…centered card…">
    <a href="/"><logo /></a>
    <h1>Confirm password</h1>
    <p>This is a secure area of the application. Please confirm your password before continuing.</p>

    <!-- children: the ConfirmPassword form -->
    <form action="/user/confirm-password" method="post">
        <label for="password">Password</label>
        <div class="relative">
            <input id="password" name="password" type="password" …>
            <button type="button" aria-label="Show password">…</button>
        </div>
        <button type="submit" data-test="confirm-password-button">Confirm password</button>
    </form>
</div>
```

## What happens on submit

1. User types their password and clicks Confirm.
2. `<Form>` intercepts the submit event, packages `{ password: '…' }`, adds the CSRF token, and POSTs to `/user/confirm-password`.
3. `processing` becomes `true` — button disables, spinner appears.
4. Fortify's `ConfirmablePasswordController::store` (in vendor, wired up automatically by the `laravel/fortify` package) hashes and checks the password.
5. **On success**: Fortify stores a "password confirmed" timestamp in the session and redirects back to the original URL (e.g. `/settings/security`). Inertia follows the redirect. Because the `RequirePassword` middleware sees the fresh timestamp, it lets the user through this time. The user lands on `/settings/security`.
6. **On failure**: Fortify redirects back to `/user/confirm-password` with a 422 and `session()->flash('errors', ['password' => 'The password is incorrect.'])`. Inertia repopulates the page, `errors.password` is now `'The password is incorrect.'`, and `<InputError>` renders it in red.

## Compare: `login.tsx`

Same shape, more fields. See `resources/js/pages/auth/login.tsx:103-106`:

```tsx
Login.layout = {
    title: 'Log in to your account',
    description: 'Enter your email and password below to log in',
};
```

Exactly the same mechanism, exactly the same layout. That's the payoff of the shared `AuthLayout`: every auth page describes itself with two strings and gets a consistent shell.

## What's next

- Build muscle memory with [10-adding-features.md](10-adding-features.md).
- Or wrap up with day-to-day commands: [11-dev-workflow.md](11-dev-workflow.md).
