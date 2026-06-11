---
name: laravel-statecraft
description: >
  Laravel/PHP backend skill that enforces the Bounded Context + State Machine +
  Job Orchestration pattern for bulletproof business logic. Uses standard Laravel directory
  conventions (app/Models, app/Actions, app/Enums, app/Events). Bounded Contexts own their
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

**Quality is non-negotiable:** Every feature requires Pest tests, PHPStan level 6, Laravel Pint, and b7s/catraca quality checks.

## Core Directives

1. **Bounded Contexts Are the First Boundary** — Split before you share.
2. **Eloquent Models Own Transitions** — No status checks in controllers or services.
3. **Events Are Facts** — Transitions return domain events. Actions dispatch them by default; the model does NOT dispatch them.
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

## Why Bounded Contexts?

A single `Order` model with 40 columns is an **architectural lie**. Each context gets its own model, enum, transitions, events. Connected through explicit integration patterns.

See `references/bounded-context-pattern.md` for the full breakdown.

## Request Flow

**HTTP Request** → **Route** → **FormRequest** (validation) → **Controller**

From the controller, two paths diverge:

**Simple path:** Controller → Action → `Model.transition()` inside `DB::transaction()` → `AuditLog::create()` → `DB::afterCommit(fn() => event(...))` → HTTP Response

**Complex path:** Controller → Service → multiple Actions → each Action runs `Model.transition()` inside `DB::transaction()` → `AuditLog::create()` → `DB::afterCommit(fn() => event(...))` → Service dispatches Jobs → HTTP Response

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
    public function rules(): array
    {
        return [
            'payment_id' => ['required', 'string'],
        ];
    }

    public function payload(): PayInvoicePayload
    {
        return new PayInvoicePayload(
            paymentId: $this->string('payment_id')->toString(),
        );
    }
}

// Action
final class MarkInvoicePaid
{
    public function __invoke(Invoice $invoice, string $paymentId, bool $dispatchEvent = true): Invoice
    {
        return DB::transaction(function () use ($invoice, $paymentId, $dispatchEvent): Invoice {
            $event = $invoice->markPaid($paymentId);
            $invoice->save();

            AuditLog::query()->create([
                'event_type' => 'invoice.paid',
                'auditable_type' => Invoice::class,
                'auditable_id' => $invoice->id,
                'actor_type' => Auth::user() ? User::class : 'system',
                'actor_id' => Auth::id() ?? 'scheduler',
                'context' => ['payment_id' => $paymentId, 'amount_cents' => $invoice->amount_cents],
                'occurred_at' => now(),
            ]);

            if ($dispatchEvent) {
                DB::afterCommit(static fn () => event($event));
            }

            return $invoice;
        });
    }
}

// Model
class Invoice extends Model
{
    public function markPaid(string $paymentId): InvoicePaid
    {
        if ($this->status !== InvoiceStatus::Pending) {
            throw new InvalidTransitionException('Cannot pay from this status');
        }

        $this->status = InvoiceStatus::Paid;
        $this->payment_id = $paymentId;

        return new InvoicePaid($this->id, $this->order_id, $paymentId, now());
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

    public function createAndProcess(CreateInvoicePayload $payload): Invoice
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
├── Models/                          # Eloquent models with transition methods
├── Enums/{Context}/                 # Status enums (one per context)
├── Events/{Context}/                # Domain events
├── Exceptions/                      # Typed exceptions
├── Actions/{Context}/               # One action per file, flat folder
├── Listeners/{Context}/             # Event listeners
├── Jobs/{Context}/                  # Queued jobs for async operations
├── Services/{Context}/              # Orchestrator + logic services
├── Infrastructure/{Context}/ACL/    # Anti-Corruption Layer (rarely needed)
└── Http/
    ├── Controllers/
    └── Requests/                    # Form Requests
```

No custom `app/Domain/` or `app/ValueObjects/` folders.

## The Core Patterns

### Pattern One: State Machine

Models contain transition methods that validate state, change it, return a domain event.

```php
class Invoice extends Model
{
    public function markPaid(string $paymentId): InvoicePaid { /* ... */ }
    public function cancel(): InvoiceCancelled { /* ... */ }
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

1. A bounded context defines its own model + enum + events.
2. The model's transition method validates state, changes it, returns a domain event.
3. The action calls the transition, persists with `save()`, and dispatches the event by default.
4. A listener starts a job chain in response.

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
```

### Testing Requirements

| Type | Coverage |
|---|---|
| Model transitions | Every valid + invalid transition |
| Actions | Happy path + rollback + assert database state |
| Listeners | Job dispatch with `Bus::fake()` |
| Jobs | Execution + failure handling |

**Database Safety:** Every test that touches the database must run on a dedicated test database. Use `php artisan test` which automatically switches to the test database, or ensure `DB_CONNECTION` in your `.env.testing` points to a separate database. Never run tests against a development, staging, or production database.

See `references/quality-gates.md` for complete testing patterns.

## Quality Gates

| # | Gate | Rule |
|---|---|---|
| 1 | Context Isolation | Each context owns models, enums, events |
| 2 | Integration Correctness | Customer/Supplier or shared primitives |
| 3 | Status Type Safety | Enum with `default()` method |
| 4 | Transition Ownership | Status checks in model only |
| 5 | Side Effect Purity | No emails/API calls in model methods |
| 6 | Event Explicitness | Transitions return events; actions dispatch by default |
| 7 | Action Consistency | One action per file, DB transactions, no HTTP |
| 8 | Test Coverage | Every action, transition, listener tested |
| 9 | Database Safety | Tests run on a dedicated test database only |
| 10 | PHPStan | Level 6 compliance |
| 11 | Code Style | Laravel Pint formatted |
| 12 | Quality Metrics | b7s/catraca passes |
| 13 | Automated | Run after every change |
| 14 | Error Consistency | RFC 9457 Problem+JSON for all API errors |
| 15 | Audit Trail | State-changing actions write append-only audit records |
| 16 | Transaction Safety | Events inside transactions use `DB::afterCommit()` |
| 17 | Request Tracing | X-Request-ID middleware on all API routes |
| 18 | API Versioning | Versioned from day one (`/v1/` prefix) |

## Stop Conditions

- **Fits** when: 5+ statuses, non-trivial transitions, async steps, multiple contexts.
- **Overkill** when: 2-3 statuses, no async — plain conditionals are fine.
- **Escalate** when: two contexts share more than value objects.

## References

- `references/bounded-context-pattern.md` — God entities, context isolation
- `references/cross-context-comments.md` — Inline comment conventions
- `references/integration-patterns.md` — Customer/Supplier, ACL, shared primitives
- `references/action-service-pattern.md` — Actions + Services, sync vs async, payload pattern, invokable controllers
- `references/state-machine-pattern.md` — Model transitions, enums, events
- `references/job-orchestration-pattern.md` — Chains, batches, retry logic, afterCommit
- `references/audit-log-pattern.md` — Append-only audit records, actor tracking, JSONB context
- `references/api-patterns.md` — Problem+JSON error responses, idempotency keys, route versioning, Sunset headers
- `references/php-rules.md` — PHP/Laravel coding standards, request tracing
- `references/quality-gates.md` — Testing, PHPStan, Pint, Catraca
