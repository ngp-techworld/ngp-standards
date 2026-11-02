---
name: ngp-laravel
description: NGP TechWorld standard for Laravel/PHP backends (JSON REST APIs with Sanctum, PostgreSQL, Pest). Use when creating, editing or reviewing any NGP Laravel project - controllers, actions, services, models, migrations, routes, policies, tests, config.
---

# NGP Laravel Standard

> DRAFT v0.1. Items marked ⏳ wait for the approved stack list.
> Also follow `ngp-general`.

## 1. Approved stack

| Area | Choice | Notes |
|---|---|---|
| Runtime | PHP 8.4+ ⏳ | `composer.json` `"php"` must equal the production version's floor (e.g. `^8.4`). CI tests that exact version. |
| Framework | Laravel 13 | NGP website backend (Laravel 12) upgrades when next touched. |
| Auth | Sanctum | Cookie (SPA) for our web apps on the same site; tokens for mobile apps. See §7. |
| Database | PostgreSQL 16+ | **Everywhere**: local, tests, CI, production. No SQLite, no MySQL. |
| Cache / queue | `database` driver for MVP, Redis + Horizon once there are real users ⏳ | |
| Realtime | Laravel Reverb | Only when polling is not enough. |
| Roles | `spatie/laravel-permission` ⏳ | Once a product has more than "admin + user". |
| API docs | `dedoc/scramble` (OpenAPI) | Frontends generate their types from it. |
| Tests | Pest 4+ | Not raw PHPUnit classes. |
| Format | Laravel Pint, committed `pint.json` | |
| Static analysis | Larastan, level 6 to start, raise over time | |
| Admin UI | Our React dashboard. No Filament in new projects. | |

Do not add a package for something Laravel already does (HTTP client, validation, queues,
notifications, scheduling, file storage).

## 2. Folder structure

Standard Laravel structure, grouped by **domain** inside each folder. One way for every project:

```
app/
├── Actions/<Domain>/            one class = one use case (CreateBooking, ApproveSalon)
├── Services/<Domain>/           domain logic reused by several actions (PricingService, OtpService)
├── Models/                      Eloquent models only (+ Models/Concerns, Models/Scopes)
├── Enums/                       backed enums (OrderStatus, UserRole)
├── Http/
│   ├── Controllers/Api/V1/      Public/, Customer/, Admin/ ... thin controllers
│   ├── Requests/<Domain>/       FormRequests (validation + authorize)
│   ├── Resources/<Domain>/      JsonResources (response shape)
│   └── Middleware/
├── Policies/                    one policy per model that needs authorization
├── Events/, Listeners/, Jobs/, Notifications/, Mail/
├── Exceptions/                  domain exceptions (InvalidOrderTransition)
└── Support/                     small framework-free helpers (Phone, Money)
routes/
├── api.php                      requires routes/api/v1/*.php
└── api/v1/{public,customer,admin}.php
database/{migrations,factories,seeders}
tests/{Feature/<Domain>,Unit/<Domain>}
lang/{ar,en}/
```

Why not per-module folders (`app/Modules/*`, as in Dawri)? It works well, but a new developer
already knows the standard Laravel layout from the docs. Grouping by domain *inside* it gives the
same separation without a custom structure. ⏳ If the team prefers modules for large products,
we adopt the Dawri layout as the single rule instead; we don't allow both.

## 3. The request flow (who does what)

```
Route → FormRequest (validate + authorize) → Controller → Action → Model / Service
                                                   ↓
                                             JsonResource → JSON
```

| Layer | Does | Never does |
|---|---|---|
| **Controller** | Receives the validated request, calls **one** action, returns a Resource. ≤ ~15 lines per method. | Queries, business rules, `if` chains, `try/catch` for flow. |
| **FormRequest** | Validation rules, `authorize()` through a Policy, `prepareForValidation()` normalizing (phone, trimming). | Writing to the DB. |
| **Action** | One use case, public `handle()` (or `__invoke`) with typed args. Wraps multi-write work in `DB::transaction`. Fires events. | Reading `request()`; returning HTTP responses. |
| **Service** | Logic shared by several actions (pricing, OTP, availability). Stateless; injected. | HTTP concerns. |
| **Model** | Relationships, casts, scopes, small accessors. | Business workflows, calling external APIs. |
| **Resource** | Exact response shape. Hides internal columns. | Queries (eager-load in the action/controller). |

Example:

```php
// app/Http/Controllers/Api/V1/Customer/BookingController.php
final class BookingController extends Controller
{
    public function store(StoreBookingRequest $request, CreateBooking $createBooking): BookingResource
    {
        $booking = $createBooking->handle($request->user(), $request->validated());

        return BookingResource::make($booking->load('salon', 'service'));
    }
}
```

```php
// app/Actions/Booking/CreateBooking.php
final class CreateBooking
{
    public function __construct(private readonly AvailabilityService $availability) {}

    public function handle(User $customer, array $data): Booking
    {
        return DB::transaction(function () use ($customer, $data) {
            $this->availability->assertSlotIsFree($data['salon_id'], $data['starts_at']);

            $booking = $customer->bookings()->create([...$data, 'status' => BookingStatus::Pending]);

            BookingCreated::dispatch($booking);

            return $booking;
        });
    }
}
```

## 4. Code style

- `declare(strict_types=1);` at the top of every PHP file in `app/`.
- Classes `final` by default. Constructor property promotion with `readonly`.
- Type every parameter, return and property. No `mixed` unless unavoidable.
- Backed enums instead of string constants; cast them on the model (`'status' => OrderStatus::class`).
- State changes go through one method that checks allowed transitions
  (`$order->transitionTo(OrderStatus::Accepted)`), throwing a domain exception otherwise
  (pattern from faz3a's order state machine).
- No `env()` outside `config/`. Product settings live in `config/<product>.php`.
- No facades inside Actions/Services when constructor injection is simple; facades are fine in
  controllers, routes, and tests.
- Early returns, no `else` after `return`.
- Naming: Actions are verb + noun (`CancelOrder`), Services noun + `Service`, Events past tense
  (`OrderCancelled`), Jobs verb (`SendOrderReminder`), Requests `<Verb><Noun>Request`.

## 5. API conventions

- All routes under `/api/v1`. Breaking changes → `/api/v2`, never silently.
- REST, plural nouns, kebab-case: `GET /api/v1/salons/{salon}/time-slots`.
- JSON keys `snake_case`.
- Always return through a `JsonResource`. Lists are paginated (`?page=`, `per_page` max 100) and use
  Laravel's `data` / `links` / `meta` shape.
- **One error shape** for every error, set up in `bootstrap/app.php`:

  ```json
  { "message": "رسالة واضحة للمستخدم", "code": "slot_unavailable", "errors": { "field": ["..."] } }
  ```

  `message` is translated (Arabic by default), `code` is a stable machine string the frontend can
  switch on, `errors` appears only for validation (422).
- Status codes: 200, 201 (created), 204 (no body), 401, 403, 404, 409 (conflict / invalid state),
  422 (validation), 429 (rate limit). Never 200 with `"success": false`.
- Rate-limit auth, OTP and public forms (`throttle:` middleware with named limiters).

## 6. Database

- Migrations are **never edited** after they ran on any shared environment; add a new one.
- Table names plural snake_case; foreign keys `<model>_id` with `->constrained()->cascadeOnDelete()`
  or `->restrictOnDelete()` chosen on purpose.
- Every column used in `where`/`orderBy` on large tables gets an index.
- Timestamps in UTC (`timestamptz` on Postgres). Money per `ngp-general` §8.
- Translatable content: `jsonb` columns with `spatie/laravel-translatable` (as in Schildwerk).
- Avoid N+1: eager-load (`with`), and enable `Model::preventLazyLoading(! app()->isProduction())`
  in `AppServiceProvider`.
- Multi-tenant data (per restaurant, per salon): a global scope + a `BelongsToTenant` trait,
  covered by a test that proves tenant A cannot read tenant B (digital-menu pattern).
- Seeders: `DatabaseSeeder` creates only what local dev needs. Demo data in separate seeders.
  Never seed a fixed admin password for production; use an artisan command
  (`php artisan <product>:create-admin`).

## 7. Auth & authorization

- Web apps on our own domain: Sanctum **SPA cookie** auth (`statefulApi()`, CSRF cookie).
- Mobile apps and third parties: Sanctum **tokens** with abilities and an expiry.
- Every protected action is checked by a **Policy** (`authorize()` in the FormRequest or
  `Gate::authorize` in the action). No role `if`s scattered in controllers.
- Roles as an enum (`UserRole`), permissions via spatie when needed.

## 8. Background work

- Anything slow or external (mail, push, image processing, Telegram) runs in a queued Job or
  queued Notification.
- Jobs are idempotent (safe to run twice) and set `$tries` and `backoff`.
- Scheduled commands are named `<product>:<verb>-<noun>` (`dawri:expire-bookings`), defined in
  `routes/console.php`, and each has a test.
- Production runs a queue worker (systemd/Supervisor, or Horizon with Redis) and `schedule:run`
  every minute.

## 9. Testing (Pest)

- Feature test for every endpoint: happy path, validation failure, unauthorized (401/403).
- Unit tests for Services, state machines, price calculations.
- Tests run on **PostgreSQL** (a `<product>_test` database), never SQLite, so they match production.
- Use factories with states (`Order::factory()->paid()->create()`), not hand-built arrays.
- Fake the outside world: `Mail::fake()`, `Queue::fake()`, `Http::fake()`, `Notification::fake()`.
- Name tests as sentences: `it('rejects a booking in a slot that is already taken')`.

## 10. Config & environment

- `.env.example` complete and production-safe (no real secrets). `config/database.php` default
  connection is `pgsql`.
- `composer.json` `name` is `ngptechworld/<product>-backend`, not `laravel/laravel`. Remove
  skeleton scripts that don't apply (e.g. `npm run build` in an API with no assets).
- Delete unused skeleton front-end assets (`resources/js`, Vite, Tailwind) in pure API projects,
  unless Blade emails need them.
- Laravel Boost may be installed; its generated `AGENTS.md`/`CLAUDE.md` must also point to these
  NGP skills.

## 11. Checklist before opening a PR

- [ ] `vendor/bin/pint` and `vendor/bin/phpstan analyse` pass
- [ ] `php artisan test` passes on PostgreSQL
- [ ] New endpoints have FormRequest, Policy check, Resource, and feature tests
- [ ] New env vars are in `.env.example`; new translations exist in `lang/ar`
- [ ] No `dd()`, `dump()`, `ray()` or commented-out code left
