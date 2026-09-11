# 07 — External Packages Reference

Every third-party dependency in this repo, with a one-line purpose and pointers to where you'll see it used. Two tables: PHP (composer) and JS (npm).

## PHP dependencies (`composer.json`)

### Production (`require`)

| Package | Version | What it does | Where you'll see it |
|---------|---------|--------------|---------------------|
| `php` | `^8.3` | The language runtime. | Everywhere. |
| `inertiajs/inertia-laravel` | `^3.0` | Server-side Inertia adapter. Provides `Inertia::render()`, `Inertia::flash()`, `Inertia::optional()`, and the `HandleInertiaRequests` base middleware. | `app/Http/Middleware/HandleInertiaRequests.php`, every controller that calls `Inertia::render(...)`, `FortifyServiceProvider`. |
| `laravel/framework` | `^13.17` | Core Laravel. Routing, Eloquent, Auth, Validation, Sessions, Blade. | Everywhere on the backend. |
| `laravel/fortify` | `^1.37.2` | Headless authentication. Registers login, logout, password reset, email verification, and 2FA routes and controllers. No views. | `config/fortify.php`, `app/Providers/FortifyServiceProvider.php`, `app/Actions/Fortify/ResetUserPassword.php`, `app/Models/User.php` (uses the `TwoFactorAuthenticatable` trait). |
| `laravel/wayfinder` | `^0.1.14` | Reads routes/controllers and emits typed TS files. Backend half of the Wayfinder toolchain. | Invisible at runtime; produces `resources/js/routes/**` and `resources/js/actions/**` via the Vite plugin. |
| `laravel/chisel` | `^0.1.0` | Scaffolding / generator helpers. | Artisan CLI. |
| `laravel/tinker` | `^3.0` | Interactive REPL for the app (`php artisan tinker`). | Local debugging. |

### Development (`require-dev`)

| Package | Version | What it does | Where you'll see it |
|---------|---------|--------------|---------------------|
| `pestphp/pest` | `^5.1` | Modern PHP testing framework (Pest syntax on top of PHPUnit). | `tests/`, run with `php artisan test`. |
| `pestphp/pest-plugin-laravel` | `^5.0` | Laravel-specific Pest helpers (`get()`, `post()`, `actingAs()`, etc.). | Test files. |
| `laravel/pint` | `^1.27` | Opinionated PHP code formatter (Laravel's fork of PHP-CS-Fixer). | Run `vendor/bin/pint --dirty --format agent` before committing. |
| `larastan/larastan` | `^3.9` | PHPStan preset for Laravel. Static analysis. | `phpstan.neon`; run `composer run types:check`. |
| `laravel/boost` | `^2.2` | MCP server for LLM tooling (docs search, DB query, browser logs). | `.mcp.json`. See `boost.json`. Not part of the running app. |
| `laravel/pail` | `^1.2.5` | Real-time tailing of Laravel logs (`php artisan pail`). | Local debugging. |
| `laravel/pao` | `^1.0.6` | (Laravel package tooling.) | CLI. |
| `laravel/sail` | `^1.53` | Docker-based local dev environment. | Optional; not used if you run natively. |
| `nunomaduro/collision` | `^8.9.3` | Prettier error pages / CLI errors. | Development. |
| `mockery/mockery` | `^1.6` | Test doubles / mocking library. | Test files. |
| `fakerphp/faker` | `^1.24` | Test data generator. | Model factories under `database/factories/`. |

## JS dependencies (`package.json`)

### Runtime dependencies

| Package | Version | What it does | Where you'll see it |
|---------|---------|--------------|---------------------|
| `react` | `^19.2.0` | UI library. | Every `.tsx` file. |
| `react-dom` | `^19.2.0` | React renderer for the browser. | Consumed internally by Inertia. |
| `@inertiajs/react` | `^3.0.0` | Inertia React adapter. Exports `createInertiaApp`, `<Head>`, `<Link>`, `<Form>`, `usePage`, `useForm`, `router`. | `resources/js/app.tsx`, every page, most components. |
| `@inertiajs/vite` | `^3.0.0` | Inertia Vite plugin (SSR bundling, dev helpers). | `vite.config.ts:2, 21`. |
| `vite` | `^8.0.0` | Dev server + bundler. | Run via `npm run dev`, `npm run build`. |
| `laravel-vite-plugin` | `^3.0.0` | Bridges Vite with Laravel's `@vite(...)` Blade directive; supports hot reload on PHP file changes; bundles Bunny Fonts. | `vite.config.ts:12-20`. |
| `@vitejs/plugin-react` | `^6.1.1` | JSX/TSX compilation + Fast Refresh for React. Also exports `reactCompilerPreset`. | `vite.config.ts:5, 22`. |
| `@rolldown/plugin-babel` | (dev-listed but used at runtime) | Babel adapter for the new Rolldown-based Vite; runs the React Compiler preset. | `vite.config.ts:23-25`. |
| `babel-plugin-react-compiler` | (dev-listed) | The React 19 compiler that auto-memoizes components. | Wired through `reactCompilerPreset`. |
| `@laravel/vite-plugin-wayfinder` | (dev) | Generates typed TS routes/actions. | `vite.config.ts:27-29`. |
| `tailwindcss` | `^4.0.0` | Utility-first CSS framework. | `className="…"` everywhere. |
| `@tailwindcss/vite` | `^4.1.11` | Vite plugin for Tailwind v4 (no config file — read from `resources/css/app.css`). | `vite.config.ts:26`. |
| `tailwind-merge` | `^3.0.1` | Merges conflicting Tailwind classes intelligently (`bg-red-500 bg-blue-500` → `bg-blue-500`). | Used inside `cn()` — `resources/js/lib/utils.ts:8`. |
| `clsx` | `^2.1.1` | Tiny helper to build className strings from conditionals. | Also inside `cn()`. |
| `class-variance-authority` | `^0.7.1` | Type-safe component variants. shadcn's `<Button variant="ghost">` uses `cva()` under the hood. | `resources/js/components/ui/button.tsx` and other UI files. |
| `lucide-react` | `^0.475.0` | Icon component set. | `import { Eye, EyeOff } from 'lucide-react'` in `password-input.tsx:1`, and throughout `components/`. |
| `sonner` | `^2.0.0` | Toast notifications. `<Toaster />` renders the container, `toast.success(...)` shows a toast. | `resources/js/app.tsx:30`, `resources/js/hooks/use-flash-toast.ts`. |
| `input-otp` | `^1.4.2` | One-time-code input (used in 2FA challenge). | `components/ui/input-otp.tsx`, `pages/auth/two-factor-challenge.tsx`. |
| `tw-animate-css` | `^1.4.0` | Tailwind-friendly `animate.css` classes. | Utility class references (`animate-*`) in components. |
| `concurrently` | `^10.0.3` | Run multiple CLI commands in parallel (used by `composer run dev`). | Dev tooling. |
| `typescript` | `^5.7.2` | TS compiler + type checker. | Everywhere via `.ts`/`.tsx`. |
| `@types/react`, `@types/react-dom` | `^19.2.0` | Type definitions for React. | Silent. |

### Radix UI primitives (`@radix-ui/react-*`)

All headless (no styling), accessibility-first primitives. shadcn wraps each with Tailwind classes in `resources/js/components/ui/`. You almost never import Radix directly — you import the shadcn wrapper.

| Package | Used by |
|---------|---------|
| `@radix-ui/react-avatar` | `components/ui/avatar.tsx` |
| `@radix-ui/react-checkbox` | `components/ui/checkbox.tsx` |
| `@radix-ui/react-collapsible` | `components/ui/collapsible.tsx` |
| `@radix-ui/react-dialog` | `components/ui/dialog.tsx`, `sheet.tsx` (mobile menu) |
| `@radix-ui/react-dropdown-menu` | `components/ui/dropdown-menu.tsx` |
| `@radix-ui/react-label` | `components/ui/label.tsx` |
| `@radix-ui/react-navigation-menu` | `components/ui/navigation-menu.tsx` |
| `@radix-ui/react-select` | `components/ui/select.tsx` |
| `@radix-ui/react-separator` | `components/ui/separator.tsx` |
| `@radix-ui/react-slot` | Used by `<Button asChild>` (any component that composes into its child) |
| `@radix-ui/react-toggle`, `-toggle-group` | `components/ui/toggle.tsx`, `toggle-group.tsx` |
| `@radix-ui/react-tooltip` | `components/ui/tooltip.tsx`; `<TooltipProvider>` in `app.tsx:28` |

### Dev tooling

| Package | Purpose |
|---------|---------|
| `vite-plus` | Enhanced Vite config helpers (`defineConfig`, `lazyPlugins`, `lint`, `fmt`, `check`). Used in `vite.config.ts`. Provides the `vp` binary invoked by npm scripts. |
| `@types/node` | Type definitions for Node globals. |

### Optional dependencies

Platform-specific binaries (linux/win64) for Rollup, Tailwind Oxide, and LightningCSS. Installed automatically only on the matching OS.

## The `.mcp.json` and `boost.json` files

These configure the Laravel Boost MCP server so LLM tools (like this one) can query the DB schema, search Laravel docs, and read browser logs during development. Not part of the running app.

## What's next

- Explore custom hooks and utilities: [08-hooks-and-utilities.md](08-hooks-and-utilities.md).
- Or work through the confirm-password page: [09-confirm-password-line-by-line.md](09-confirm-password-line-by-line.md).
