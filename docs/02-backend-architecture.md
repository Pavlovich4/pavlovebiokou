# 02 — Backend Architecture (Laravel side)

This chapter walks every server-side file that participates in serving an Inertia page. If you know Laravel already, skim the first section and focus on `HandleInertiaRequests` and `FortifyServiceProvider`.

## Boot sequence: `bootstrap/app.php`

```php
// bootstrap/app.php
return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware): void {
        $middleware->encryptCookies(except: ['appearance', 'sidebar_state']);

        $middleware->web(append: [
            HandleAppearance::class,
            HandleInertiaRequests::class,
            AddLinkHeadersForPreloadedAssets::class,
        ]);
    })
    ->withExceptions(function (Exceptions $exceptions): void {
        $exceptions->shouldRenderJsonWhen(
            fn (Request $request) => $request->is('api/*') || $request->expectsJson(),
        );
    })->create();
```

Line by line:

- `withRouting(...)` tells Laravel which route files to load. Only `routes/web.php` is listed here — the settings routes are pulled in via a `require` at the end of `web.php`. There's no `api.php` because this app has no REST API.
- `encryptCookies(except: [...])` — `appearance` and `sidebar_state` cookies are readable by JavaScript, so they can't be encrypted. Everything else is encrypted by default.
- `->web(append: [...])` appends three custom middleware to the `web` group *after* Laravel's defaults:
  - `HandleAppearance` — reads the `appearance` cookie (light/dark/system) so the server can render the right theme.
  - `HandleInertiaRequests` — the important one, covered next.
  - `AddLinkHeadersForPreloadedAssets` — sends `Link: rel=preload` HTTP headers so the browser starts fetching Vite chunks earlier.
- `shouldRenderJsonWhen(...)` — API-style requests get JSON error responses; the rest get Inertia error pages.

## Route files

### `routes/web.php`

```php
// routes/web.php
use Illuminate\Support\Facades\Route;

Route::inertia('/', 'welcome')->name('home');

Route::middleware(['auth', 'verified'])->group(function () {
    Route::inertia('dashboard', 'dashboard')->name('dashboard');
});

require __DIR__.'/settings.php';
```

- `Route::inertia($url, $pageName)` is a shortcut for `Route::get($url, fn() => Inertia::render($pageName))`. Use it when a page needs no props from a controller.
- The `auth, verified` middleware group requires a logged-in user with a verified email.
- Settings routes are pulled in with `require` so they can use the same URL space without polluting `web.php`.

### `routes/settings.php`

```php
// routes/settings.php
Route::middleware(['auth'])->group(function () {
    Route::redirect('settings', '/settings/profile');

    Route::get('settings/profile', [ProfileController::class, 'edit'])->name('profile.edit');
    Route::patch('settings/profile', [ProfileController::class, 'update'])->name('profile.update');
});

Route::middleware(['auth', 'verified'])->group(function () {
    Route::delete('settings/profile', [ProfileController::class, 'destroy'])->name('profile.destroy');

    Route::get('settings/security', [SecurityController::class, 'edit'])
        ->middleware(RequirePassword::class)
        ->name('security.edit');

    Route::put('settings/password', [SecurityController::class, 'update'])
        ->middleware('throttle:6,1')
        ->name('user-password.update');

    Route::inertia('settings/appearance', 'settings/appearance')->name('appearance.edit');
});
```

New middleware to know:

- `RequirePassword` — before showing the security page, Laravel first bounces the user to the "confirm your password" page (`auth/confirm-password`). That's the page you're studying in [09-confirm-password-line-by-line.md](09-confirm-password-line-by-line.md).
- `throttle:6,1` — max 6 password updates per minute per user.

### Where do login / register / password reset routes come from?

**Fortify auto-registers them.** You won't find `Route::post('login', …)` anywhere in this repo. When the `laravel/fortify` package is installed and enabled (see `config/fortify.php:104` `'middleware' => ['web']`), it declares `/login`, `/logout`, `/forgot-password`, `/reset-password`, `/user/confirm-password`, `/user/two-factor-*`, `/email/verification-notification`, and so on. Confirm this any time by running `php artisan route:list --except-vendor=false`.

## HandleInertiaRequests — the middleware that ships props to the client

```php
// app/Http/Middleware/HandleInertiaRequests.php
class HandleInertiaRequests extends Middleware
{
    protected $rootView = 'app';

    public function version(Request $request): ?string
    {
        return parent::version($request);
    }

    public function share(Request $request): array
    {
        return [
            ...parent::share($request),
            'name' => config('app.name'),
            'auth' => [
                'user' => $request->user(),
            ],
            'sidebarOpen' => ! $request->hasCookie('sidebar_state') || $request->cookie('sidebar_state') === 'true',
        ];
    }
}
```

What each piece does:

- **`protected $rootView = 'app';`** — the Blade template that wraps the initial HTML shell. Look in `resources/views/app.blade.php`; it contains the `<div id="app" data-page="…"></div>` where Inertia mounts the React tree, plus `@vite(...)` and `@inertiaHead`.
- **`version()`** — the "asset version" string. If it changes between two requests, Inertia forces a full page reload so users don't run stale JS. `parent::version()` uses the Vite manifest hash, which is exactly right.
- **`share()`** — the props that show up on *every* page. From the frontend you'd read these with `usePage().props`:

  ```tsx
  import { usePage } from '@inertiajs/react';
  const { auth, name, sidebarOpen } = usePage().props;
  ```

  See `resources/js/pages/welcome.tsx:5` for a real example (`const { auth } = usePage().props;`).

- **`sidebarOpen`** — the trick here: if the cookie is missing OR set to `'true'`, the sidebar starts open. That's why it isn't encrypted (JS also writes it when the user toggles the sidebar).

## FortifyServiceProvider — routing Fortify through Inertia

Fortify normally renders Blade views for its login/register/reset pages. This app rewires each view to render an Inertia page instead.

```php
// app/Providers/FortifyServiceProvider.php  (relevant parts)
public function boot(): void
{
    $this->configureActions();
    $this->configureViews();
    $this->configureRateLimiting();
}

private function configureActions(): void
{
    Fortify::resetUserPasswordsUsing(ResetUserPassword::class);
}

private function configureViews(): void
{
    Fortify::loginView(fn (Request $request) => Inertia::render('auth/login', [
        'canResetPassword' => Features::enabled(Features::resetPasswords()),
        'status' => $request->session()->get('status'),
    ]));

    Fortify::resetPasswordView(fn (Request $request) => Inertia::render('auth/reset-password', [
        'email' => $request->email,
        'token' => $request->route('token'),
        'passwordRules' => Password::defaults()->toPasswordRulesString(),
    ]));

    Fortify::requestPasswordResetLinkView(fn (Request $request) => Inertia::render('auth/forgot-password', [
        'status' => $request->session()->get('status'),
    ]));

    Fortify::verifyEmailView(fn (Request $request) => Inertia::render('auth/verify-email', [
        'status' => $request->session()->get('status'),
    ]));

    Fortify::twoFactorChallengeView(fn () => Inertia::render('auth/two-factor-challenge'));

    Fortify::confirmPasswordView(fn () => Inertia::render('auth/confirm-password'));
}
```

Key ideas:

- `Fortify::xxxView(fn () => Inertia::render(...))` says "when Fortify wants to show this screen, hand off to my Inertia page instead". The page name (e.g. `'auth/login'`) matches a file at `resources/js/pages/auth/login.tsx`.
- Props like `canResetPassword`, `status`, `email`, `token` are all handed to the React component as props (see the `type Props` block at the top of `pages/auth/login.tsx:13-16`).
- **`Fortify::resetUserPasswordsUsing(ResetUserPassword::class)`** — swaps in a custom action class. `ResetUserPassword` lives at `app/Actions/Fortify/ResetUserPassword.php` and it just validates the new password and calls `$user->forceFill([...])->save()`. This is how you customize any Fortify behavior — Fortify defines a contract, you provide a class that implements it.

Rate limiter definitions (`login`, `two-factor`) live in this same file. They're referenced from `config/fortify.php:117-120`.

## `config/fortify.php` — enabled features

```php
// config/fortify.php  (excerpt)
'features' => [
    Features::resetPasswords(),
    Features::emailVerification(),
    Features::twoFactorAuthentication([
        'confirm' => true,
        'confirmPassword' => true,
    ]),
],
```

- `resetPasswords()` — enables the `/forgot-password` and `/reset-password` routes and email sending.
- `emailVerification()` — enables `/email/verify` and the `verified` middleware.
- `twoFactorAuthentication(['confirm' => true, 'confirmPassword' => true])` — enables TOTP-based 2FA. `confirm: true` means users must scan the QR code AND enter a valid TOTP before 2FA is actually enabled. `confirmPassword: true` means changing 2FA settings requires re-entering the password (which is what triggers the "confirm password" flow via `RequirePassword` middleware).

Not enabled: `Features::registration()`. That's why there's no register page in `pages/auth/`.

## Controller pattern: `ProfileController`

```php
// app/Http/Controllers/Settings/ProfileController.php
class ProfileController extends Controller
{
    public function edit(Request $request): Response
    {
        return Inertia::render('settings/profile', [
            'mustVerifyEmail' => $request->user() instanceof MustVerifyEmail,
            'status' => $request->session()->get('status'),
        ]);
    }

    public function update(ProfileUpdateRequest $request): RedirectResponse
    {
        $request->user()->fill($request->validated());

        if ($request->user()->isDirty('email')) {
            $request->user()->email_verified_at = null;
        }

        $request->user()->save();

        Inertia::flash('toast', ['type' => 'success', 'message' => __('Profile updated.')]);

        return to_route('profile.edit');
    }

    public function destroy(ProfileDeleteRequest $request): RedirectResponse
    {
        $user = $request->user();
        Auth::logout();
        $user->delete();
        $request->session()->invalidate();
        $request->session()->regenerateToken();
        return redirect('/');
    }
}
```

Two patterns worth memorizing:

1. **GET returns an Inertia page.** Return type is `Inertia\Response`. The first argument to `Inertia::render` is the page name, second is a props array. Props become React props on the client.

2. **POST/PATCH/DELETE returns a redirect.** Return type is `Illuminate\Http\RedirectResponse`. Inertia sees the redirect, follows it, and mounts the redirected page. On the client this looks like a normal navigation.

3. **Validation goes through a FormRequest.** `ProfileUpdateRequest` (in `app/Http/Requests/Settings/`) defines the `rules()`; if they fail, Laravel automatically redirects back with `$errors` in the session. Inertia surfaces those as `errors` in the page props — that's how the `<InputError message={errors.email} />` in the profile form receives its message.

4. **Flash a toast**: `Inertia::flash('toast', [...])` puts the payload in one-time session storage. The frontend picks it up via the `use-flash-toast` hook and calls Sonner. See [08-hooks-and-utilities.md](08-hooks-and-utilities.md).

## `User` model & 2FA

`app/Models/User.php` uses the `TwoFactorAuthenticatable` trait from Fortify. That trait adds `two_factor_secret`, `two_factor_recovery_codes`, and helper methods (`enableTwoFactorAuthentication()`, `hasEnabledTwoFactorAuthentication()`, etc.). You interact with them from `SecurityController` and the frontend `manage-two-factor.tsx` component.

## Custom Fortify actions

`app/Actions/Fortify/ResetUserPassword.php` shows the pattern for overriding any Fortify behavior:

1. Implement a Fortify contract (`ResetsUserPasswords`, `CreatesNewUsers`, `UpdatesUserPasswords`, …).
2. Register the class in `FortifyServiceProvider::configureActions()` via a `Fortify::xxxUsing(...)` call.
3. Fortify uses your class instead of its default.

The `App\Concerns\PasswordValidationRules` and `App\Concerns\ProfileValidationRules` traits centralize validation rules that multiple actions/requests share.

## Recap: what happens when a form is submitted

Take login as an example:

1. React `<Form>` POSTs to `/login` (URL supplied by Wayfinder, see `resources/js/routes/index.ts:88` `logout` — login uses the same pattern).
2. Fortify's `AuthenticatedSessionController@login` handles the POST (registered by Fortify itself).
3. On success it redirects to `config('fortify.home')` which is `/dashboard`.
4. On failure it redirects back with validation errors.
5. Inertia follows the redirect and remounts either the dashboard or the login page.

You never wrote a controller for login — Fortify wrote it. You only wrote the view (via `Fortify::loginView(...)`) and the React page.

## What's next

- Move to the client with [03-frontend-architecture.md](03-frontend-architecture.md).
- Or jump to Inertia primitives: [04-inertia-explained.md](04-inertia-explained.md).
