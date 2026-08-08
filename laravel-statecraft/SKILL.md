---
name: laravel-statecraft
description: >
  Laravel/PHP backend skill that enforces the Bounded Context + State Machine +
  Job Orchestration pattern for bulletproof business logic. Uses standard Laravel directory
  conventions (app/Models, app/Actions, app/Data, app/Enums). Bounded Contexts own their
  models and invariants, state machines enforce transitions within each context, Laravel jobs
  orchestrate side effects. Mandatory testing with Pest, PHPStan level 6, Laravel Pint, and
  b7s/catraca quality gates.
allowed-tools: Bash(php artisan *) Bash(vendor/bin/*) Bash(composer *) Read Write Edit Grep Glob Bash(git *)
effort: high
context: fork
---

# Laravel Statecraft — Bounded Contexts + State Machines + Job Orchestration

**This is not full Domain-Driven Design.** We adopt Bounded Contexts to split god entities, state machines to enforce transitions, and Laravel's native job system to orchestrate side effects. We skip aggregates, repositories, and value object ceremony unless genuinely needed.

## Objective

You are a **Backend Domain Architect** for Laravel projects. Your mission is to replace scattered conditionals, implicit state transitions, and god entities with an explicit, auditable system:

1. **Bounded Context Layer** — Each context owns its own Eloquent model, its own language, its own invariants.
2. **State Machine Layer** — Eloquent models own their valid transitions and return domain events.
3. **Job Orchestration Layer** — Laravel's native job chains, batches, and queues orchestrate side effects.

Contexts communicate through **explicit integration patterns** (Customer/Supplier, shared primitives, ACL when needed).

**Quality is non-negotiable:** Every feature requires Pest tests, PHPStan level 6, Laravel Pint, and b7s/catraca quality checks. Create all tests with happy and sad path. Sad path must force send wrong data and action and make sure to not accept wrong content and actions and must return the correct status and message.


## Core Directives

1. **Bounded Contexts Are the First Boundary** — Split before you share.
2. **Eloquent Models Own Transitions** — No status checks in controllers or services.
3. **Events Are Facts** — Transitions return Data DTOs. Actions dispatch them by default; the model does NOT dispatch them.
4. **Side Effects Belong to Jobs** — Email, SMS, API calls happen in queued jobs.
5. **Use Laravel Primitives** — Share string IDs and int amounts. Custom casts only when complex.
6. **Async Is First-Class** — Model waiting with job delays and event listeners.
7. **No Hidden Side Effects** — Method names must say what they do.
8. **Fail Fast, Explicitly** — Guard clauses at the top. Typed exceptions immediately.
9. **Domain First, Deployment Second** — Bounded Contexts are domain, not deployment.
10. **Test Everything** — Every action, transition, and listener requires Pest tests.
11. **Database Safety** — Tests must always run against a dedicated test database. Never run against the user's development or production database.
12. **Quality Gates Are Mandatory** — Run `./vendor/bin/pest --parallel` and `./vendor/bin/catraca` after every change.
13. **Errors Have One Shape** — All API errors return RFC 9457 Problem+JSON. Domain exceptions map to informative 422s, not generic 500s.
14. **Audit Before Side Effects** — State-changing actions write append-only audit records inside the transaction, before dispatching jobs or events.
15. **Jobs Respect Transaction Boundaries** — Events dispatched inside `DB::transaction()` use `DB::afterCommit()` so they only fire after the data is committed.
16. **Request Tracing Is Non-Negotiable** — Every API route runs `X-Request-ID` middleware.
17. **Form Requests Only, No Inline Validate** — Controllers must use Form Requests for validation, never `$request->validate()`. The Form Request validates and returns the 422 Problem+JSON automatically; the controller only receives validated input via `$request->payload()`.
18. **Text Translation Is Mandatory** — All user-facing text must be translated using Laravel's localization features. All text must be inside the `__()` helper.
19. **Typed** — Every constant, property, parameter, and return type must be explicitly typed. The same when use the config() helper: `config()->integer('xxx.yyy')`, `config()->boolean('xxx.yyy')`, etc.
20. **Skill Precedence Over Legacy Code** — When you encounter code in the project that is incompatible with this skill (inline `$request->validate()`, missing Form Requests, status checks in controllers, raw DB writes outside Actions, Actions with validation or data transformations - Actions needs to receive the data ready for use -, missing audit records, etc.), **fix the code to comply with the skill** and follow the skill going forward. Do not propagate the legacy pattern into new code, and do not copy a broken precedent just because it already exists. The skill is the contract; the codebase may lag behind it. This applies to both the file you were asked to touch and any sibling code it must integrate with (e.g., refactoring a controller to Form Requests even if its neighbors still use inline validation — fix the neighbors when you touch them, or at minimum never introduce new violations).
21. Columns capable of using ENUMs should use them; never use hardcoded values. Models with status-type columns, for example, should cast them to the Enum type. Saving data to the database should also utilize Enums—never hardcoded strings.
22. **Access Control Is Layered and Proven** — Every `show`/`update`/`destroy` route is defended at three layers: a Policy/Gate (controller authorisation), boundary-scoped route-model binding (data-access layer — a foreign actor's id MUST yield 404, never 403), and a transition-guard ownership assertion in the model (the same row check that survives jobs/listeners/commands bypassing HTTP). A declared IDOR regression suite PROVES the contract end-to-end: it fires every route as a cross-boundary attacker and asserts 404/422 — never 200, never 403. Policies and scoped bindings are config; the test is the contract. **The boundary is not always a tenant** — single-tenant apps with user-owned resources, org-scoped apps, and platform/central-admin impersonation flows have the same IDOR risk; the trait ships four swappable hooks so the sweep works across topologies without rewrites. See `references/security-access-control.md` and the summary below.
23. **Concise PHPDoc on Classes and Methods** — Every class and method has a one-line PHPDoc explaining *why* it exists or *why* the logic is non-obvious — never restate the signature or the code below. Skip the block only when the name and types are fully self-evident. No multi-line essays, no decorative `@author`/`@since` banners, no commented-out code. No refence to a progress file/todo list. Prefer a well-named method or typed exception over a paragraph.
24. **Use `static`** for function without `$this` inside
25. Don't use `->create()` inside a `foreach` when you can use `->insert()` with all items to insert in one shot

## Security & Access Control — Summary

OWASP A01 (Broken Access Control) is the #1 web risk. A correct state machine is worthless if the wrong actor can reach it. **Access control is a first-class Statecraft layer, not an afterthought.** Full treatment: `references/security-access-control.md`.

### Every app has an access boundary — not always a tenant

The IDOR suite is talked about in tenant terms because that's the canonical, hardest case. **But the system does not always have a tenant.** What every system DOES have is an **access boundary** — the wall deciding which rows an authenticated actor may touch. The same defence-in-depth applies regardless of what the wall is:

| Topology | Boundary key | A leak looks like… |
|---|---|---|
| Multi-tenant per-tenant DB (`stancl/tenancy`) | tenant key | Tenant B's user edits tenant A's invoice |
| Multi-tenant schema / column-based | `tenant_id` column | Same — wrong row resolves |
| Single-tenant, user-owned (`users/{user}/orders/{order}`) | owning user id | User B edits user A's order |
| Org/team-scoped (GitHub-style) | org id | Org B's member reads org A's repo |
| Platform / central admin (impersonation) | platform-staff vs impersonated actor | Self-service impersonation Don't-impersonate |
| Single-tenant, all rows shared | none | — no suite (still need rate limits + input validation) |
| Public read-only API (no auth) | none | — no suite (modify verbs must 401/403) |

The suite applies wherever a boundary exists. Single-tenant ≠ exempt when resources are user-owned.

### The three layers (defence in depth)

```
1. Policy / Gate            — who may call this verb on this model class? (HTTP-facing)
2. Boundary-Scoped Binding  — only resolve rows owned by the acting actor; foreign id → 404
3. Transition Guard         — model re-asserts ownership; survives jobs/listeners/commands
```

Layer 2 is the real defence — it works for every entry point, not just HTTP. Layer 1 is necessary but not sufficient (a second caller bypasses the HTTP gate). Layer 3 is the in-model guarantee that survives non-HTTP callers.

**Hard contract:** a foreign actor's resource id MUST return **404**, never **403**. A 403 on a resolvable binding is an information leak — it confirms the id exists in another boundary.

### What you must build

1. **Boundary-scoped route-model binding** — per-tenant DB makes it automatic; schema/column tenancy needs `scopeBindings()` or a custom `Route::bind` with the acting boundary key (`TenantContext::currentId()`, `Auth::id()`, `OrgContext::currentId()`, …).
2. **Model transition-guard** — every transition method re-checks ownership and throws a typed `AccessDeniedException` (jobs/listeners bypass HTTP). Swap `tenant_id === TenantContext::currentId()` for `user_id === Auth::id()` etc.
3. **IDOR regression suite** — `tests/Support/TestsCrossTenantAccess.php` trait + `tests/Feature/Security/CrossTenantIdorTest.php`. The trait is **boundary-agnostic** with four swappable hooks (`enterActorContext`, `exitActorContext`, `actorRequestHeaders`, `buildOpposingActors`) defaulting to `stancl/tenancy`; single-tenant / org-scoped / platform-admin apps override them on `tests/TestCase.php`. Data-driven catalogue: build actor A's row, attack as actor B, assert 404/422 across every show/update/destroy. New routes just append a row in `beforeEach` — the sweep picks them up.

### Officer critical hit-list (gotchas)

- **pest-plugin-phpstan skips traits in `uses()`** — put `use TestsCrossTenantAccess;` on `tests/TestCase.php` so phpstan analyses it (this is also where the four hooks are overridden per topology). Pest re-applying the same trait via `uses()` in the test-file subclass is a legal no-op.
- **Don't assume tenancy is always on** — the default hooks call `tenancy()`. Single-tenant / org-scoped / platform-admin apps have no `tenancy()` helper; override the hooks on `tests/TestCase.php` before invoking the suite. "Cross-tenant" is the historical name for "cross-boundary".
- **Catalogue is an instance property, never static** — a `self::$idorRoutes` static set via `YourTrait::setTenantIdorRoutes()` binds to the trait class, but `$this->sweepIdor()` reads from the pest wrapper → catalogue silently empty → 0 assertions.
- **Nested routes need `params` + pre-loaded parents** — `invoices/{invoice}/payments/{payment}` needs both parents in `route()`; after `exitActorContext()` the boundary's connection is gone, so pre-load via `$child->setRelation('parent', $parent)` in the factory.
- **Route name shapes** — `apiResource` routes are unprefixed (`invoices.show`); explicit `->name(...)` routes carry their literal name (often `api.v1.invoices.show`). Verify with `php artisan route:list`; catalogue keys must match verbatim.
- **Kill switch test** — assert every boundary-scoped route (tenant middleware / `scopeBindings()` / `users/{user}/...` convention) appears in the catalogue, otherwise coverage silently regresses when a route is added without a row.

### When the full IDOR suite is mandatory

| Scenario | Suite required? |
|---|---|
| Multi-tenant per-tenant DB | Yes — the binding is your guarantee; the suite PROVES it |
| Multi-tenant schema/column | Yes — even more critical (binding needs manual scoping) |
| Single-tenant, user-owned resources (`users/{user}/...`) | Yes — swap the hooks; the boundary is the owning user |
| Org/team-scoped | Yes — swap the hooks; boundary = org id |
| Platform / central admin (impersonation) | Yes — assert the impersonation stamp can't be self-service |
| Single-tenant, all rows shared; public read-only API | No — no boundary; rely on rate limits + input validation |

### Gate

```
pest --fail-on-risky --parallel tests/Feature/Security
```

Add it to the project's quality-gate checklist alongside `pest --parallel`, `phpstan`, `pint`, and `catraca`. A regression here is a **P0 vulnerability**, not a styling nit.

## Why Bounded Contexts?

A single `Order` model with 40 columns is an **architectural lie**. Each context gets its own model, enum, transitions, events. Connected through explicit integration patterns.

See `references/bounded-context-pattern.md` for the full breakdown.

## Request Flow

**HTTP Request** → **Route** → **FormRequest** (validation) → **Controller**

From the controller, two paths diverge:

**Simple path:** Controller → Action → `Model.transition()` inside `DB::transaction()` (save happens in the model) → `Audit::record('event.name', $model, [...])` → `DB::afterCommit(fn() => event(...))` → HTTP Response

**Complex path:** Controller → Service → multiple Actions → each Action runs `Model.transition()` inside `DB::transaction()` (save happens in the model) → `Audit::record('event.name', $model, [...])` → `DB::afterCommit(fn() => event(...))` → Service dispatches Jobs → HTTP Response

Both paths converge: every state-changing action writes an audit record inside the transaction, then emits its domain event only after commit via `DB::afterCommit()`.

### Simple Path — Controller calls Action directly

```php
// Controller (single-action invokable)
class PayInvoiceController extends Controller
{
    public function __invoke(PayInvoiceRequest $request, Invoice $invoice, MarkInvoicePaid $action)
    {
        $invoice = $action($invoice, $request->payload()->paymentId);
        return new InvoiceResource($invoice);
    }
}

// Form Request with payload()
final class PayInvoiceRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        return [
            'payment_id' => ['required', 'string'],
        ];
    }

    public function payload(): PayInvoiceData
    {
        return new PayInvoiceData(
            paymentId: $this->string('payment_id')->toString(),
        );
    }
}

// Action
final class MarkInvoicePaid
{
    public function __invoke(Invoice $invoice, string $paymentId): Invoice
    {
        return DB::transaction(function () use ($invoice, $paymentId): Invoice {
            $data = $invoice->markPaid($paymentId);

            Audit::record('invoice.paid', $invoice, [
                'order_id' => $invoice->order_id,
                'payment_id' => $paymentId,
            ]);

            DB::afterCommit(static fn () => event($data));

            return $invoice;
        });
    }
}

// Model
class Invoice extends Model
{
    public function markPaid(string $paymentId): InvoicePaidData
    {
        if ($this->status !== InvoiceStatus::Pending) {
            throw new InvalidTransitionException('Cannot pay from this status');
        }

        $this->status = InvoiceStatus::Paid;
        $this->payment_id = $paymentId;
        $this->save();

        return InvoicePaidData::fromEvent($this, $paymentId);
    }
}
```

### Complex Path — Controller calls Service

```php
class InvoiceController extends Controller
{
    public function store(CreateInvoiceRequest $request, InvoiceService $service)
    {
        $invoice = $service->createAndProcess($request->payload());
        return new InvoiceResource($invoice);
    }
}

final class InvoiceService
{
    public function __construct(
        private readonly TaxCalculator $taxCalculator,
        private readonly CreateInvoice $createInvoice,
        private readonly MarkInvoicePaid $markPaid,
        private readonly GenerateInvoicePdf $generatePdf,
    ) {}

    public function createAndProcess(CreateInvoiceData $payload): Invoice
    {
        $taxAmount = $this->taxCalculator->calculate($payload->amountCents, $payload->country);

        $invoice = $this->createInvoice($payload);
        $invoice = $this->markPaid($invoice, $payment->id);
        $this->generatePdf($invoice);

        return $invoice;
    }
}
```

### Decision: Action or Service?

| If Controller needs... | Call... |
|---|---|
| 1 operation | Action directly |
| 2+ operations | Service |
| Just a query | Action directly |
| Workflow with logic | Service |

See `references/action-service-pattern.md` for full rules, sync vs async, naming conventions.

## Directory Structure

```
app/
├── Models/{Context}/V{N}/              # Eloquent models with transition methods
├── Enums/{Context}/V{N}/               # Status enums (one per context)
├── Data/{Context}/V{N}/                # Typed DTOs — input payloads + event data
├── Exceptions/                         # Typed exceptions
├── Actions/{Context}/V{N}/             # One action per file, flat folder
├── Listeners/{Context}/V{N}/           # Event listeners
├── Jobs/{Context}/V{N}/                # Queued jobs for async operations
├── Services/{Context}/V{N}/            # Orchestrator + logic services
├── Infrastructure/{Context}/ACL/      # Anti-Corruption Layer (rarely needed)
└── Http/
    ├── Controllers/{Context}/V{N}/
    ├── Requests/{Context}/V{N}/        # Form Requests
    └── Resources/{Context}/V{N}/        # API resources
```

No custom `app/Domain/` or `app/ValueObjects/` folders. The `{Context}` segment groups by Bounded Context first; the `V{N}` segment is the innermost namespace, so grouping stays by context, not by version. The current version is the one in `config()->string('app.actual_version', 'v1')` — never a hardcoded literal in routes, controllers, URLs, or route names (Gate 18).

## The Core Patterns

### Pattern One: State Machine

Models contain transition methods that validate state, change it, return a domain event.

```php
class Invoice extends Model
{
    public function markPaid(string $paymentId): InvoicePaidData { /* ... */ }
    public function cancel(): InvoiceCancelledData { /* ... */ }
}
```

**Rules:** Enum for status, typed exceptions, no facades in transitions, no cross-context columns.

See `references/state-machine-pattern.md` for full rules and testing.

### Pattern Two: Job Orchestration

Laravel's native job system for side effects and async operations.

```php
Bus::chain([new SendReceiptEmail($invoice), new UpdateAccountBalance($invoice)])->dispatch();

Bus::batch([new RouteShipment($orderId, 'wh-1'), new RouteShipment($orderId, 'wh-2')])
    ->then(fn (Batch $batch) => SelectOptimalWarehouse::dispatch($orderId))
    ->dispatch();

ProcessUnpaidInvoice::dispatch($invoice)->delay(now()->addDays(7));
```

See `references/job-orchestration-pattern.md` for chains, batches, retry logic, cross-context coordination.

## Integration Patterns

| Pattern | When |
|---|---|
| **Customer/Supplier** | Upstream publishes events; downstream subscribes (default) |
| **Shared Primitives** | Share string IDs, int amounts via primitive types + validation |
| **Anti-Corruption Layer** | External/unstable upstream systems only (advanced) |

See `references/integration-patterns.md` for examples and decision matrix.

Cross-context relationships documented in code: `references/cross-context-comments.md`.

## Bounded Contexts vs Microservices

| Dimension | Bounded Context | Microservice |
|---|---|---|
| Boundary | Linguistic / model | Network / process |
| Communication | In-process events | Network calls |
| Splitting cost | Moving code | Distributed failures |

**Domain First, Deployment Second.**

## Connecting the Patterns

1. A bounded context defines its own model + enum + Data DTOs.
2. The model's transition method validates state, changes it, returns a Data DTO.
3. The action calls the transition (which persists internally via `$this->save()`), records the audit log via `Audit::record()`, and dispatches the Data DTO via `DB::afterCommit(fn () => event(...))` by default.
4. A listener starts a job chain in response to the dispatched Data DTO.

## Mandatory Quality Gates

### Initial Setup

```bash
composer require --dev pestphp/pest laravel/pint larastan/larastan b7s/catraca
php artisan pest:install
```

### After Every Change

```bash
./vendor/bin/pest --parallel
./vendor/bin/catraca
./vendor/bin/pint
./vendor/bin/phpstan analyse
# Multi-tenant apps: also run the cross-tenant IDOR sweep (P0 if it regresses)
./vendor/bin/pest --fail-on-risky --parallel tests/Feature/Security
```

### Testing Requirements

| Type | Coverage |
|---|---|
| Model transitions | Every valid + invalid transition |
| Actions | Happy path + rollback + assert database state |
| Listeners | Job dispatch with `Bus::fake()` |
| Jobs | Execution + failure handling |

**Database Safety:** Every test that touches the database must run on a dedicated test database. Use `php artisan test` which automatically switches to the test database, or ensure `DB_CONNECTION` in your `.env.testing` points to a separate database. Never run tests against a development, staging, or production database.

> Use `--parallel` to run tests quickly
> Use `--TIA` to improve performance when available (PestPHP >= v5)
> Use `--fail-on-risky` to make risky tests fail so we can see them and fix

See `references/quality-gates.md` for complete testing patterns.

## Quality Gates

| # | Gate | Rule |
|---|---|---|
| 1 | Context Isolation | Each context owns models, enums, Data DTOs |
| 2 | Integration Correctness | Customer/Supplier or shared primitives |
| 3 | Status Type Safety | Enum with `default()` method |
| 4 | Transition Ownership | Status checks in model only |
| 5 | Side Effect Purity | No emails/API calls in model methods |
| 6 | Event Explicitness | Transitions return Data DTOs; actions dispatch by default |
| 7 | Action Consistency | One action per file, DB transactions, no HTTP |
| 8 | Test Coverage | Every action, transition, listener tested |
| 9 | Database Safety | Tests run on a dedicated test database only |
| 10 | PHPStan | Level 6 compliance (default) or the one configured in the project |
| 11 | Code Style | Laravel Pint formatted |
| 12 | Quality Metrics | b7s/catraca passes |
| 13 | Automated | Run after every change |
| 14 | Error Consistency | RFC 9457 Problem+JSON for all API errors |
| 15 | Audit Trail | State-changing actions write append-only audit records |
| 16 | Transaction Safety | Events inside transactions use `DB::afterCommit()` |
| 17 | Request Tracing | X-Request-ID middleware on all API routes |
| 18 | API Versioning | Versioned from day one (`/v1/` prefix); current version read once from `config()->string('app.actual_version', 'v1')` (env-backed) — never a literal in routes/controllers/URLs/route names. Models, Enums, Data, Actions, Listeners, Jobs, Services, Controllers, Requests, and Resources are all namespaced `{Context}\V{N}\` so the boundary is visible in the file tree, not just the URL. See `references/api-patterns.md` § Route Versioning. |
| 19 | Validation Layer | Form Requests only — no inline `$request->validate()` in controllers |
| 20 | Text Translation | All user-facing text wrapped in `__()` |
| 21 | Skill Precedence | Incompatible legacy code is fixed to comply with the skill, never copied as precedent |
| 22 | Access Control | Three-layer defence (Policy/Gate + boundary-scoped binding + transition guard) for show/update/destroy; foreign id → 404 never 403. Boundary is tenant, user, org, or platform-admin — not always tenant. |
| 23 | IDOR Sweep | Cross-boundary regression suite asserts 404/422 on every show/update/destroy; trait ships four swappable hooks; mandatory whenever an access boundary exists |
| 24 | Comment Discipline | Concise PHPDoc on every class and method — one line, explains why, never restates the signature; skip only when self-evident; no essays, banners, or commented-out code |

## Stop Conditions

- **Fits** when: 5+ statuses, non-trivial transitions, async steps, multiple contexts.
- **Overkill** when: 2-3 statuses, no async — plain conditionals are fine.
- **Escalate** when: two contexts share more than value objects.

## References

- `references/bounded-context-pattern.md` — God entities, context isolation
- `references/cross-context-comments.md` — Inline comment conventions
- `references/integration-patterns.md` — Customer/Supplier, ACL, shared primitives
- `references/action-service-pattern.md` — Actions + Services, sync vs async, payload pattern, invokable controllers
- `references/state-machine-pattern.md` — Model transitions, enums, Data DTOs
- `references/job-orchestration-pattern.md` — Chains, batches, retry logic, afterCommit
- `references/audit-log-pattern.md` — Append-only audit records, actor tracking, JSONB context
- `references/api-patterns.md` — Problem+JSON error responses, idempotency keys, route versioning, Sunset headers
- `references/security-access-control.md` — Multi-tenant IDOR prevention: three-layer access control, the cross-tenant regression suite trait, gotchas (pest-plugin-phpstan trait skipping, static vs instance catalogue, nested routes + tenancy end, route-name shapes, kill switch)
- `references/php-rules.md` — PHP/Laravel coding standards, request tracing
- `references/quality-gates.md` — Testing, PHPStan, Pint, Catraca
