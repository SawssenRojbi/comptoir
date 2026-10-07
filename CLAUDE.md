# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Comptoir is a small B2B e-commerce app (professional supplies). Monorepo with two independent apps:

- `api/` — Laravel 12 REST API (PHP 8.3), PostgreSQL 16, Sanctum, Pest tests.
- `web/` — Next.js 16 App Router front (TypeScript, React 19), Vitest + Playwright.

The project language is French: UI strings, validation messages, comments, docs and route names (`/catalogue`, `/panier`, `/commande`, `/inscription`) are in French. Keep new code consistent with that.

Setup steps live in `README.md`; `docs/architecture.md` has diagrams (routes, data model, `GET /api/orders` sequence) and a table of known weak points with file:line references — read it before touching orders, stock, invoicing or security.

## Commands

Database: `docker compose up -d` (creates role `comptoir`/`comptoir` and DBs `comptoir`, `comptoir_test`, `comptoir_e2e`; override port with `COMPTOIR_DB_PORT`).

API (run from `api/`):

```bash
php artisan serve                          # http://127.0.0.1:8000
php artisan test                           # all Pest tests (Feature tests hit comptoir_test with RefreshDatabase)
vendor/bin/pest --testsuite=Unit           # unit tests only, no DB needed
vendor/bin/pest tests/Feature/OrderTest.php            # single file
vendor/bin/pest --filter="nom du test"                 # single test by name
./vendor/bin/pint --test                   # format check (CI); drop --test to fix
php artisan invoices:export {order}        # writes storage/app/invoices/facture-{id}.pdf
```

`php artisan migrate:fresh --seed` wipes the dev DB — destructive, let the user run it. Demo accounts use password `password` (see README).

Web (run from `web/`):

```bash
npm run dev               # http://localhost:3000
npm run test              # Vitest (lib/*.test.ts, node environment)
npx vitest run lib/cart.test.ts         # single file
npm run lint              # ESLint
npm run typecheck         # tsc --noEmit
npm run format:check      # Prettier
npx playwright test       # e2e in web/e2e; starts/reuses API on :8000 and front on :3000
```

CI (`.github/workflows/lint.yml`) only runs Pint, ESLint and `tsc` — tests are not run in CI, so run them locally.

## Architecture

**API flow:** `routes/api.php` → thin controller in `app/Http/Controllers/Api/` receiving a validated `FormRequest` → model or service → `JsonResource` (responses wrapped in `{"data": ...}`; errors use Laravel's standard 404/401/422 JSON). Logic spanning multiple tables goes in `app/Services/` — e.g. `OrderService::place()` resolves/creates the guest customer, locks products, checks stock, writes lines, decrements stock and computes the total inside one transaction.

**Money:** amounts are integer cents everywhere in the DB and API (`price_cents`, `total_cents`, `unit_price_cents` frozen at order time). The exception is invoicing: `LegacyInvoiceCalculator` works in float euros (VAT by country/category, pro discount, promo codes, shipping) and `InvoicePdfExporter` reads its `$lastResult` property rather than the return value — don't remove it. Order `total_cents` and the invoice total are computed independently and do not match (no VAT/shipping/discount in the order total).

**Auth:** only `GET /api/me` is behind `auth:sanctum`. `GET/POST /api/orders` are public; `POST /api/orders` takes either `customer_id` or a `guest` object (guest customers have a null password and are reused by email).

**Front:** `web/lib/api.ts` is the API client (`getProducts`, `getProduct`, `createOrder`, `registerCustomer`, returning `ApiResult<T>` for POSTs). Server components use `API_URL` (falls back to `NEXT_PUBLIC_API_URL`); browser code only sees `NEXT_PUBLIC_API_URL`. Shared API types are in `web/types/api.ts`. The cart lives entirely in `localStorage` (key `comptoir.panier`, `lib/cart.ts`); the API recomputes prices on order. Format prices with `lib/money.ts` (`formatPrice`, French format "12,50 €"). Note `app/catalogue/page.tsx` bypasses `lib/api.ts` with a direct `fetch` — prefer the client for new code.

## Conventions

- `.editorconfig`: 4-space indent by default, 2 spaces for `web/` TS/JS/JSON/CSS and YAML; LF.
- PHP formatting: Pint `laravel` preset. Front formatting: Prettier (`web/.prettierrc.json`).
- `@/` alias maps to `web/` root (both tsconfig and Vitest).
