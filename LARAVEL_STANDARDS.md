# Laravel Project Standards

Copy this file into every new Laravel project. Read it before scaffolding features. Target: **Laravel 11 / 12** apps (Blade + optional API).

---

## 1. How to use

1. Place `LARAVEL_STANDARDS.md` at the project root (or link it from `.cursor/rules`).
2. Follow the architecture rules for every new feature.
3. Add packages only when the checklist below says you need them — do not install the full library list by default.
4. Keep `.env.example` as the documented source of truth for environment keys.
5. Prefer **Pest** (or PHPUnit) tests for every meaningful write path; factories power those tests.

---

## 2. Architecture — where code lives

| Layer | Path | Rule |
|-------|------|------|
| Controllers | `app/Http/Controllers` | Thin: Form Request → Service → view / JSON / Resource |
| Services | `app/Services` | Business logic only — no HTTP, no `request()`, no views |
| Models | `app/Models` | Eloquent, relations, scopes, casts — no fat workflows |
| Enums | `app/Enums` | Status/type values as backed enums (not magic strings) |
| Form Requests | `app/Http/Requests` | Validation + `authorize()` |
| API Resources | `app/Http/Resources` | Shape JSON responses |
| Policies | `app/Policies` | Authorization per model/action |
| Jobs | `app/Jobs` | Slow / async work (email, PDF, imports) |
| Events / Listeners | `app/Events`, `app/Listeners` | Decouple side effects from the main flow |
| Observers | `app/Observers` | Model lifecycle hooks (prefer Events when logic grows) |
| Notifications | `app/Notifications` | Mail / database / Slack user messages |
| Mail | `app/Mail` | Mailable classes for emails |
| Actions (optional) | `app/Actions` | Single-purpose use cases when a full Service feels heavy |
| Console | `app/Console/Commands` | Artisan commands + scheduled tasks |
| Providers | `app/Providers` | Bindings, gates, observers, bootstrapping |
| Migrations | `database/migrations` | One concern per migration; reversible when possible |
| Seeders | `database/seeders` | Demo / baseline data |
| Factories | `database/factories` | Realistic builders for tests + seeders |
| Tests | `tests/Feature`, `tests/Unit` | Feature for HTTP/flows; Unit for pure logic |
| Views | `resources/views` | Blade grouped by feature (`orders/index.blade.php`) |
| Frontend source | `resources/css`, `resources/js` | Vite entrypoints; never edit compiled output by hand |
| Built assets | `public/build` | Vite output only — do not commit hand edits |
| Static public | `public/` | Favicon, static images only — no business logic |
| Routes | `routes/web.php`, `routes/api.php` | Web UI vs API; keep route files thin |
| Config | `config/` | App settings; secrets stay in `.env` |

### Request flow

```
Route → Middleware → Form Request → Controller → Service → Model / Job / Event
                                              ↓
                                    View / Resource / JSON
```

### Routes and middleware

- Register web routes in `routes/web.php`, API routes in `routes/api.php`.
- Prefer route groups with `middleware(['auth', 'verified'])` (or custom aliases).
- Name routes (`->name('orders.store')`) and use `route()` helpers — avoid hard-coded URLs.
- Put reusable HTTP logic in middleware; keep controllers free of cross-cutting concerns.
- Prefer `Route::resource` / `apiResource` for CRUD; add custom routes next to them with clear names.

### Config vs `.env`

- Read settings via `config('app.name')`, never `env()` outside config files.
- Secrets and environment-specific values belong in `.env`; document every key in `.env.example`.

---

## 3. Suggested project skeleton

```
app/
  Actions/                 # optional single-purpose classes
  Console/Commands/
  Enums/
  Events/
  Http/
    Controllers/
    Middleware/
    Requests/
    Resources/
  Jobs/
  Listeners/
  Mail/
  Models/
  Notifications/
  Observers/
  Policies/
  Providers/
  Services/
    OrderService.php
database/
  factories/
    OrderFactory.php
    UserFactory.php
  migrations/
  seeders/
    DatabaseSeeder.php
resources/
  css/
    app.css
  js/
    app.js
  views/
    layouts/
    orders/
    emails/
public/
  build/          # Vite output (generated)
  images/         # Static images only
routes/
  web.php
  api.php
  console.php
tests/
  Feature/
    OrderTest.php
  Unit/
    OrderServiceTest.php
  Pest.php          # if using Pest
.env.example
LARAVEL_STANDARDS.md
phpunit.xml
```

Create `app/Services`, `Requests`, `Resources`, and matching **factories + tests** early so features stay consistent.

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
| Enums | PascalCase + backed string/int | `OrderStatus::Paid` |
| Test methods | descriptive sentences (Pest) or `test_` / `it_` | `it('creates an order')` |

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
- Dispatch jobs/events from services when side effects should be async or decoupled.

### Models

- Prefer `$fillable` (explicit) over empty `$guarded = []` in production patterns.
- Define `$casts` (including enums, dates, bools), relations, and local scopes on the model.
- Use `SoftDeletes` when records must be recoverable; index `deleted_at` if you filter often.
- Keep multi-step workflows in services, not model methods that orchestrate half the app.
- Always eager-load relations you will use (`with()`, `load()`) — avoid N+1 queries.
- Prefer query scopes for repeated filters (`scopeActive`, `scopeForUser`).

### Enums

- Use PHP backed enums for status/type columns (`app/Enums/OrderStatus.php`).
- Cast on the model: `'status' => OrderStatus::class`.
- Never scatter magic strings like `'paid'` across controllers and Blade.

### Form Requests

- Every write endpoint gets a Form Request.
- Put `authorize()` and `rules()` here; controllers stay thin.
- Use Rule objects / `Rule::enum()` for enums and unique constraints.

### API Resources

- Use Resources for all JSON payloads so field names stay stable.
- Nest related data deliberately; avoid dumping entire models.
- Prefer `whenLoaded()` so you do not leak unloaded relations.

### Migrations and seeders

- Name clearly: `create_orders_table`, `add_status_to_orders_table`.
- Add foreign keys and indexes for columns you filter/join on.
- Prefer `down()` that reverses the change when safe.
- Use meaningful column types (`foreignId`, `uuid`, `json`, `decimal`) — not everything as `string`.
- Seeders should be safe to re-run where practical (`updateOrCreate` / checks).
- `DatabaseSeeder` should call domain seeders; keep each seeder focused.

### `.env` and secrets

- Never commit `.env`.
- Keep `.env.example` complete (keys without real secrets).
- Typical keys to document: `APP_*`, `DB_*`, `MAIL_*`, `QUEUE_CONNECTION`, `CACHE_STORE`, `FILESYSTEM_DISK`, `SESSION_DRIVER`, third-party API keys.
- Local defaults: `QUEUE_CONNECTION=sync` is fine; use `database`/`redis` when testing real queues.

### Security (baseline)

- CSRF on all web form/AJAX mutating requests.
- Authorize with Policies/Gates — do not rely on hiding UI alone.
- Mass-assignment only through validated + fillable fields.
- Store uploads outside the web root when possible; validate mime/size.
- Rate-limit login and public APIs (`throttle` middleware).
- Never log passwords, tokens, or full card data.

### Code style and quality

- Run **Laravel Pint** before committing (`vendor/bin/pint`).
- Prefer explicit return types and constructor property promotion where the team already uses them.
- No dead imports; no commented-out blocks left in PRs.
- Keep methods short; extract private methods or Actions when a service method grows.

---

## 5. Factories (required for every model you test)

Every important Eloquent model should have a factory. Factories feed **seeders** and **tests** — do not build test rows by hand with `Model::create([...])` everywhere.

### Rules

- One factory per model: `database/factories/OrderFactory.php`.
- Defaults must be **valid** and realistic (Faker ok; avoid nonsense that breaks validation).
- Use states for variants: `->pending()`, `->paid()`, `->trashed()`.
- Use relationships: `Order::factory()->for($user)->has(OrderItem::factory()->count(3))`.
- Never put business side effects (emails, jobs) inside `definition()` — keep factories dumb data.
- After adding a required DB column, update the factory in the same PR.

### Example

```php
// database/factories/OrderFactory.php
public function definition(): array
{
    return [
        'user_id' => User::factory(),
        'status' => OrderStatus::Pending,
        'total' => fake()->randomFloat(2, 10, 500),
        'notes' => fake()->optional()->sentence(),
    ];
}

public function paid(): static
{
    return $this->state(fn () => ['status' => OrderStatus::Paid]);
}
```

### Seeder usage

```php
User::factory()
    ->count(10)
    ->has(Order::factory()->count(3)->paid())
    ->create();
```

### Test usage

```php
$order = Order::factory()->paid()->create();
$user = User::factory()->create();
$order = Order::factory()->for($user)->create();
```

---

## 6. Testing (Feature + Unit)

Ship tests with features — not “later”. Prefer **Pest** on new apps; PHPUnit is fine if the repo already uses it.

### Layout

| Type | Path | What it covers |
|------|------|----------------|
| Feature | `tests/Feature` | HTTP routes, auth, full create/update flows, policies |
| Unit | `tests/Unit` | Pure service/helpers/enums with no HTTP |
| Browser (optional) | Laravel Dusk | Only when JS-heavy UI must be verified |

### Rules

- Use `RefreshDatabase` (or equivalent) on Feature tests that touch the DB.
- Build data with **factories**, not raw SQL or brittle fixtures.
- Assert status codes, redirects, DB state, and JSON structure — not only “no exception”.
- Cover happy path + at least one validation failure + one unauthorized case for write endpoints.
- Fake mail, notifications, and queues: `Mail::fake()`, `Notification::fake()`, `Queue::fake()`, `Event::fake()` when side effects are not the focus.
- Do not hit real external APIs in tests — fake HTTP (`Http::fake()`) or inject stubs.
- Name tests by behavior: `it('creates an order for the authenticated user')`.
- Run locally before push: `php artisan test` (or `./vendor/bin/pest`).

### Feature test example (Pest)

```php
use App\Models\User;
use App\Models\Order;
use function Pest\Laravel\actingAs;
use function Pest\Laravel\post;
use function Pest\Laravel\assertDatabaseHas;

it('creates an order for the authenticated user', function () {
    $user = User::factory()->create();

    actingAs($user)
        ->post(route('orders.store'), [
            'notes' => 'Rush order',
        ])
        ->assertRedirect();

    assertDatabaseHas('orders', [
        'user_id' => $user->id,
        'notes' => 'Rush order',
    ]);
});

it('rejects guests from creating orders', function () {
    post(route('orders.store'), [])->assertRedirect(route('login'));
});

it('validates required fields', function () {
    $user = User::factory()->create();

    actingAs($user)
        ->postJson(route('orders.store'), [])
        ->assertStatus(422)
        ->assertJsonValidationErrors(['notes']); // adjust to your rules
});
```

### Unit test example

```php
it('marks an order as paid', function () {
    $order = Order::factory()->create();
    $service = app(OrderService::class);

    $service->markPaid($order);

    expect($order->fresh()->status)->toBe(OrderStatus::Paid);
});
```

### What must have tests (minimum)

- Auth-protected create/update/delete for each main resource
- Policy denial (wrong user / missing permission)
- Critical service calculations (totals, status transitions)
- Jobs that mutate data (assert DB after `Bus::dispatchSync` or process fakes carefully)

### CI tip

- Run Pint + `php artisan test` in CI on every PR.
- Keep the suite fast: avoid `sleep`, real SMTP, and huge seeders inside tests.

---

## 7. AJAX and server-side patterns

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
- Write a Feature test with `postJson` / `getJson` for AJAX endpoints.

---

## 8. Assets and `public/`

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

## 9. Jobs, queues, events, notifications

### Jobs

- Put slow work in `app/Jobs` (emails, PDFs, imports, image processing).
- Keep jobs idempotent when possible (safe if retried).
- Use `$tries`, `$backoff`, and `failed()` for real queue drivers.
- Dispatch from services: `GenerateInvoicePdf::dispatch($order)`.

### Events & listeners

- Fire domain events after important state changes (`OrderPaid`).
- Listeners handle side effects (notify user, sync CRM) so the service stays readable.
- Prefer events over fat observers when multiple subscribers exist.

### Observers

- Use for simple model hooks (clear cache on update).
- Move heavy logic to listeners/jobs — observers hide too much if overloaded.

### Notifications & mail

- User-facing messages → `Notification` or `Mailable`.
- Queue them (`implements ShouldQueue`) in production.
- Always `Mail::fake()` / `Notification::fake()` in tests unless you assert delivery.

### Scheduling

- Define schedules in `routes/console.php` (Laravel 11+) or the Console kernel.
- Log/monitor scheduled commands; keep them thin wrappers around services.

---

## 10. Eloquent, caching, and performance

- Eager load: `Order::with(['user', 'items'])->paginate()`.
- Paginate list endpoints — never `->get()` unbounded tables in controllers.
- Use `chunk()` / `lazy()` for bulk processing in commands/jobs.
- Cache expensive read-only queries with tags or clear keys on write (`Cache::remember`).
- Prefer DB indexes over clever PHP filtering.
- Select only needed columns when payloads are large; Resources still shape the output.
- Log slow paths with context (`Log::info('order.created', ['order_id' => $order->id])`) — never dump secrets.

---

## 11. Library recommendation guide

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

### Testing / DX / UI

| Package | When to use | Main benefit | Skip if |
|---------|-------------|--------------|---------|
| **Pest** | New apps or team wants expressive tests | Cleaner syntax on top of PHPUnit | Repo already standardized on PHPUnit only |
| **Laravel Pint** | Every project (shipped with Laravel) | Zero-config code style | — always use it |
| **Livewire** | Interactive UI without a full SPA | Server-driven components | You already committed to Inertia/React/Vue |
| **Inertia.js** | Modern SPA feel with Laravel routes/controllers | Shared backend auth + JS pages | Simple Blade CRUD is enough |

### Dev / ops / social

| Package | When to use | Main benefit | Skip if |
|---------|-------------|--------------|---------|
| **Laravel Telescope** | Local/staging deep debugging | Requests, jobs, exceptions, dumps | Production without access control |
| **barryvdh/laravel-debugbar** | Local request timing/queries | Instant toolbar feedback | Production — never enable there |
| **Laravel Horizon** | Redis queues at scale | Dashboard, balancing, metrics | Sync/`database` queue is enough |
| **Laravel Socialite** | Login with Google/GitHub/etc. | OAuth providers with little boilerplate | No social login required |
| **Laravel Dusk** | Critical browser flows (checkout, wizards) | Real browser automation | Feature tests already cover the flow |

### Quick chooser

- **Admin CRUD + login** → Breeze (+ Spatie Permission if roles grow).
- **API for mobile/SPA** → Sanctum (+ Query Builder if lists are rich).
- **Invoices PDF** → DomPDF.
- **Bulk Excel** → Laravel Excel.
- **Uploads with thumbs** → Medialibrary or Intervention Image.
- **Queues in production** → Redis + Horizon.
- **Tests** → Pest + factories + Feature tests on day one.
- **Interactive Blade** → Livewire; **JS pages** → Inertia.

---

## 12. New project checklist

Use this on day one of every Laravel app:

- [ ] Laravel installed; `.env` created from `.env.example`
- [ ] `.env.example` lists every required key (no real secrets)
- [ ] Folders ready: `app/Services`, `app/Http/Requests`, `app/Http/Resources`, `app/Enums`
- [ ] Auth approach chosen: Breeze / Jetstream / Sanctum / none
- [ ] Vite assets wired (`resources/css`, `resources/js`); layout uses `@vite`
- [ ] `public/` limited to static files + build output
- [ ] Database migrations use clear names, FKs, and indexes
- [ ] **Factory per main model**; seeders use factories
- [ ] **Pest or PHPUnit** configured; sample Feature test passes
- [ ] Pint runs clean (`vendor/bin/pint --test`)
- [ ] Default JSON/AJAX error handling agreed (422 + field errors)
- [ ] Policies/Gates planned for any non-public write actions
- [ ] Queue/mail drivers chosen for local vs production
- [ ] Extra libraries added **only** when a feature needs them (Permission, DomPDF, Excel, etc.)
- [ ] This `LARAVEL_STANDARDS.md` kept at the repo root

---

## 13. Feature workflow (every ticket)

1. Migration (+ indexes/FKs).
2. Model with fillable, casts, relations, enum casts.
3. **Factory** (+ states) and seeder update if needed.
4. Form Request + Policy.
5. Service method(s) for the business flow (jobs/events if async).
6. Thin controller + named routes.
7. Blade view and/or API Resource.
8. AJAX only if the UX requires it — same Request/Service underneath.
9. **Feature tests** (happy path, validation, unauthorized) + Unit tests for tricky logic.
10. Run Pint + `php artisan test` before opening the PR.

Ship small, keep layers honest, add packages deliberately, and never merge untested write paths.
