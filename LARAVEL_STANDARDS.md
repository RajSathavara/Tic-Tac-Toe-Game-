# Laravel Project Standards

Copy this file into every new Laravel project. Read it before scaffolding features. Target: **Laravel 11 / 12** apps (Blade + optional API).

---

## 1. How to use

1. Place `LARAVEL_STANDARDS.md` at the project root (or link it from `.cursor/rules`).
2. Follow the architecture rules for every new feature.
3. Add packages only when the checklist below says you need them — do not install the full library list by default.
4. Keep `.env.example` as the documented source of truth for environment keys.

---

## 2. Architecture — where code lives

| Layer | Path | Rule |
|-------|------|------|
| Controllers | `app/Http/Controllers` | Thin: Form Request → Service → view / JSON / Resource |
| Services | `app/Services` | Business logic only — no HTTP, no `request()`, no views |
| Models | `app/Models` | Eloquent, relations, scopes, casts — no fat workflows |
| Form Requests | `app/Http/Requests` | Validation + `authorize()` |
| API Resources | `app/Http/Resources` | Shape JSON responses |
| Policies | `app/Policies` | Authorization per model/action |
| Providers | `app/Providers` | Bindings, gates, observers, bootstrapping |
| Migrations | `database/migrations` | One concern per migration; reversible when possible |
| Seeders | `database/seeders` | Demo / baseline data |
| Factories | `database/factories` | Test and seeder data builders |
| Views | `resources/views` | Blade grouped by feature (`orders/index.blade.php`) |
| Frontend source | `resources/css`, `resources/js` | Vite entrypoints; never edit compiled output by hand |
| Built assets | `public/build` | Vite output only — do not commit hand edits |
| Static public | `public/` | Favicon, static images only — no business logic |
| Routes | `routes/web.php`, `routes/api.php` | Web UI vs API; keep route files thin |
| Config | `config/` | App settings; secrets stay in `.env` |

### Request flow

```
Route → Middleware → Form Request → Controller → Service → Model / External API
                                              ↓
                                    View / Resource / JSON
```

### Routes and middleware

- Register web routes in `routes/web.php`, API routes in `routes/api.php`.
- Prefer route groups with `middleware(['auth', 'verified'])` (or custom aliases).
- Name routes (`->name('orders.store')`) and use `route()` helpers — avoid hard-coded URLs.
- Put reusable HTTP logic in middleware; keep controllers free of cross-cutting concerns.

### Config vs `.env`

- Read settings via `config('app.name')`, never `env()` outside config files.
- Secrets and environment-specific values belong in `.env`; document every key in `.env.example`.

---

## 3. Suggested project skeleton

```
app/
  Http/
    Controllers/
    Middleware/
    Requests/
    Resources/
  Models/
  Policies/
  Providers/
  Services/
    OrderService.php
database/
  factories/
  migrations/
  seeders/
resources/
  css/
    app.css
  js/
    app.js
  views/
    layouts/
    orders/
public/
  build/          # Vite output (generated)
  images/         # Static images only
routes/
  web.php
  api.php
.env.example
LARAVEL_STANDARDS.md
```

Create `app/Services` (and `Requests` / `Resources` folders) early so features stay consistent.

---

## 4. Coding standards (daily rules)

### Naming

| Thing | Convention | Example |
|-------|------------|---------|
| Classes | PascalCase | `OrderService` |
| Methods / variables | camelCase | `createOrder()` |
| DB tables | snake_case, plural | `order_items` |
| DB columns | snake_case | `user_id`, `created_at` |
| Route names | dot.snake | `orders.store` |
| Config keys | snake_case | `mail.from` |

### Controllers

- Inject services via the constructor or method injection.
- No raw queries, no long business flows, no validation rules inline when a Form Request exists.
- Return views for Blade, Resources/JSON for AJAX/API.

```php
public function store(StoreOrderRequest $request, OrderService $orders)
{
    $order = $orders->create($request->validated());

    if ($request->wantsJson()) {
        return new OrderResource($order);
    }

    return redirect()->route('orders.show', $order);
}
```

### Services

- One service per domain: `OrderService`, `InvoiceService`, `UserService`.
- Accept arrays/DTOs of already-validated data; return models or value objects.
- No `request()`, session, or redirects inside services.

### Models

- Prefer `$fillable` (explicit) over empty `$guarded = []` in production patterns.
- Define `$casts`, relations, and local scopes on the model.
- Keep multi-step workflows in services, not model methods that orchestrate half the app.

### Form Requests

- Every write endpoint gets a Form Request.
- Put `authorize()` and `rules()` here; controllers stay thin.

### API Resources

- Use Resources for all JSON payloads so field names stay stable.
- Nest related data deliberately; avoid dumping entire models.

### Migrations and seeders

- Name clearly: `create_orders_table`, `add_status_to_orders_table`.
- Add foreign keys and indexes for columns you filter/join on.
- Prefer `down()` that reverses the change when safe.
- Seeders should be safe to re-run where practical (updateOrCreate / checks).
- Factories power tests and local seed data — keep them realistic.

### `.env` and secrets

- Never commit `.env`.
- Keep `.env.example` complete (keys without real secrets).
- Typical keys to document: `APP_*`, `DB_*`, `MAIL_*`, `QUEUE_CONNECTION`, `CACHE_STORE`, `FILESYSTEM_DISK`, third-party API keys.

### Security (baseline)

- CSRF on all web form/AJAX mutating requests.
- Authorize with Policies/Gates — do not rely on hiding UI alone.
- Mass-assignment only through validated + fillable fields.
- Store uploads outside the web root when possible; validate mime/size.

---

## 5. AJAX and server-side patterns

### Prefer Laravel routes + CSRF

- Blade forms: `@csrf`.
- AJAX (fetch/axios): send `X-CSRF-TOKEN` from `<meta name="csrf-token">` or cookie `XSRF-TOKEN`.
- Use `Accept: application/json` so validation failures return JSON.

### Consistent JSON shape

Prefer a stable envelope for custom AJAX (or use API Resources directly):

```json
{
  "success": true,
  "message": "Order created.",
  "data": { }
}
```

### Validation errors

- Return **422** with field errors (`errors` object). Laravel does this by default for JSON requests.
- Do not invent a second error format per page.

### Example fetch pattern

```js
const token = document.querySelector('meta[name="csrf-token"]').content;

const res = await fetch('/orders', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Accept': 'application/json',
    'X-CSRF-TOKEN': token,
  },
  body: JSON.stringify(payload),
});

if (res.status === 422) {
  const { errors } = await res.json();
  // map errors to fields
  return;
}

const json = await res.json();
```

### Server-side first

- Default to full page forms + redirects for simple CRUD.
- Add AJAX only when UX needs it (inline updates, live search, modals).
- Reuse the same Form Request + Service for both Blade and AJAX entry points.

---

## 6. Assets and `public/`

| Location | Purpose |
|----------|---------|
| `resources/css`, `resources/js` | Source files edited by developers |
| Vite (`vite.config.js`) | Builds to `public/build` |
| `public/images`, `public/favicon.ico` | Truly static files |
| `public/build` | Generated — do not hand-edit |
| `storage/app/public` + `php artisan storage:link` | User uploads |

Rules:

- Use `@vite(['resources/css/app.css', 'resources/js/app.js'])` in Blade layouts.
- Do not drop ad-hoc CSS/JS into `public/` for app logic.
- Keep third-party static vendor files out of git when a package/CDN/Vite import can own them.

---

## 7. Library recommendation guide

Install only what the feature needs. Each row: **when**, **benefit**, **skip if**.

### Auth / starters

| Package | When to use | Main benefit | Skip if |
|---------|-------------|--------------|---------|
| **Laravel Breeze** | New app needs login/register quickly (Blade, Livewire, React, or Vue) | Minimal, official auth scaffolding | You need teams/2FA out of the box |
| **Laravel Jetstream** | Teams, profile photos, 2FA, richer starter kit | Batteries-included auth UX | Simple auth is enough — use Breeze |
| **Laravel Sanctum** | SPA cookie auth or mobile/API tokens | First-party API auth without OAuth complexity | Pure server-rendered Blade with session only |

### Spatie (quality of life)

| Package | When to use | Main benefit | Skip if |
|---------|-------------|--------------|---------|
| **spatie/laravel-permission** | Roles & permissions (admin panels, multi-role users) | Battle-tested roles/permissions API | Single “admin vs user” flag is enough |
| **spatie/laravel-medialibrary** | Models with many files/images and conversions | Attachments + image conversions on models | One simple file column is enough |
| **spatie/laravel-activitylog** | Audit trail (“who changed what”) | Automatic model activity logging | No compliance/audit need |
| **spatie/laravel-backup** | Scheduled DB/file backups to disk/S3 | One-command, notifiable backups | Hosting already handles backups |
| **spatie/laravel-query-builder** | Public APIs with filter/sort/include | Safe, expressive query strings | Few fixed list endpoints |

### PDF / Excel / images

| Package | When to use | Main benefit | Skip if |
|---------|-------------|--------------|---------|
| **barryvdh/laravel-dompdf** | Invoices, simple reports from Blade HTML | Familiar Blade → PDF | Complex print layouts (consider Snappy/Browsershot) |
| **maatwebsite/excel** | CSV/XLSX import-export | Queued imports, exports, concerns | Tiny one-off CSV you can stream yourself |
| **intervention/image** | Resize/crop/watermark uploads | Simple image API | Medialibrary conversions already cover needs |

### Dev / ops / social

| Package | When to use | Main benefit | Skip if |
|---------|-------------|--------------|---------|
| **Laravel Telescope** | Local/staging deep debugging | Requests, jobs, exceptions, dumps | Production without access control |
| **barryvdh/laravel-debugbar** | Local request timing/queries | Instant toolbar feedback | Production — never enable there |
| **Laravel Horizon** | Redis queues at scale | Dashboard, balancing, metrics | Sync/`database` queue is enough |
| **Laravel Socialite** | Login with Google/GitHub/etc. | OAuth providers with little boilerplate | No social login required |

### Quick chooser

- **Admin CRUD + login** → Breeze (+ Spatie Permission if roles grow).
- **API for mobile/SPA** → Sanctum (+ Query Builder if lists are rich).
- **Invoices PDF** → DomPDF.
- **Bulk Excel** → Laravel Excel.
- **Uploads with thumbs** → Medialibrary or Intervention Image.
- **Queues in production** → Redis + Horizon.

---

## 8. New project checklist

Use this on day one of every Laravel app:

- [ ] Laravel installed; `.env` created from `.env.example`
- [ ] `.env.example` lists every required key (no real secrets)
- [ ] Folders ready: `app/Services`, `app/Http/Requests`, `app/Http/Resources`
- [ ] Auth approach chosen: Breeze / Jetstream / Sanctum / none
- [ ] Vite assets wired (`resources/css`, `resources/js`); layout uses `@vite`
- [ ] `public/` limited to static files + build output
- [ ] Database migrations use clear names, FKs, and indexes
- [ ] Seeders/factories set up for local demo data
- [ ] Default JSON/AJAX error handling agreed (422 + field errors)
- [ ] Policies/Gates planned for any non-public write actions
- [ ] Extra libraries added **only** when a feature needs them (Permission, DomPDF, Excel, etc.)
- [ ] This `LARAVEL_STANDARDS.md` kept at the repo root

---

## 9. Feature workflow (every ticket)

1. Migration (+ factory/seeder if needed).
2. Model with fillable, casts, relations.
3. Form Request + Policy.
4. Service method(s) for the business flow.
5. Thin controller + routes (named).
6. Blade view and/or API Resource.
7. AJAX only if the UX requires it — same Request/Service underneath.

Ship small, keep layers honest, add packages deliberately.
