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
3. **Events Are Facts** — Transitions return domain events. The model does NOT dispatch them.
4. **Side Effects Belong to Jobs** — Email, SMS, API calls happen in queued jobs.
5. **Use Laravel Primitives** — Share string IDs and int amounts. Custom casts only when complex.
6. **Async Is First-Class** — Model waiting with job delays and event listeners.
7. **No Hidden Side Effects** — Method names must say what they do.
8. **Fail Fast, Explicitly** — Guard clauses at the top. Typed exceptions immediately.
9. **Domain First, Deployment Second** — Bounded Contexts are domain, not deployment.
10. **Test Everything** — Every action, transition, and listener requires Pest tests.
11. **Quality Gates Are Mandatory** — Run `./vendor/bin/pest --parallel` and `./vendor/bin/catraca` after every change.

## Why Bounded Contexts?

A single `Order` model with 40 columns is an **architectural lie**. Each context gets its own model, enum, transitions, events. Connected through explicit integration patterns.

See `references/bounded-context-pattern.md` for the full breakdown.

## Request Flow

```
HTTP Request → Route → FormRequest (validation) → Controller
                                                         ↓
                                              ┌──────────┴──────────┐
                                              │                      │
                                         [SIMPLE]               [COMPLEX]
                                              │                      │
                                              ↓                      ↓
                                         Action              Service → Actions
                                              │                      │
                                              ↓                      ↓
                                         Model.transition()    Model.transition()
                                         DB::transaction()     DB::transaction()
                                              │                      │
                                              ↓                      ↓
                                         Returns result         Dispatches event
                                              │                      │
                                              └──────────┬───────────┘
                                                         ↓
                                                   HTTP Response
```

### Simple Path — Controller calls Action directly

```php
class InvoiceController extends Controller
{
    public function pay(PayInvoiceRequest $request, Invoice $invoice, MarkInvoicePaid $action)
    {
        $invoice = $action($invoice, $request->validated('payment_id'));
        return new InvoiceResource($invoice);
    }
}

final class MarkInvoicePaid
{
    public function __invoke(Invoice $invoice, string $paymentId): Invoice
    {
        return DB::transaction(function () use ($invoice, $paymentId): Invoice {
            $invoice->markPaid($paymentId);
            $invoice->save();
            return $invoice;
        });
    }
}

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
        $invoice = $service->createAndProcess($request->validated());
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

    public function createAndProcess(array $data): Invoice
    {
        $taxAmount = $this->taxCalculator->calculate($data['amount_cents'], $data['country']);

        $invoice = $this->createInvoice([...]);
        $invoice = $this->markPaid($invoice, $payment->id);
        $this->generatePdf($invoice);

        event(new InvoicePaid($invoice->id, $invoice->order_id, $payment->id, now()));

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
3. The action calls the transition, persists with `save()`. The service dispatches the event.
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
| Actions | Happy path + rollback |
| Listeners | Job dispatch with `Bus::fake()` |
| Jobs | Execution + failure handling |

See `references/quality-gates.md` for complete testing patterns.

## Quality Gates

| # | Gate | Rule |
|---|---|---|
| 1 | Context Isolation | Each context owns models, enums, events |
| 2 | Integration Correctness | Customer/Supplier or shared primitives |
| 3 | Status Type Safety | Enum with `default()` method |
| 4 | Transition Ownership | Status checks in model only |
| 5 | Side Effect Purity | No emails/API calls in model methods |
| 6 | Event Explicitness | Transitions return events; services dispatch |
| 7 | Action Consistency | One action per file, DB transactions, no HTTP |
| 8 | Test Coverage | Every action, transition, listener tested |
| 9 | PHPStan | Level 6 compliance |
| 10 | Code Style | Laravel Pint formatted |
| 11 | Quality Metrics | b7s/catraca passes |
| 12 | Automated | Run after every change |

## Stop Conditions

- **Fits** when: 5+ statuses, non-trivial transitions, async steps, multiple contexts.
- **Overkill** when: 2-3 statuses, no async — plain conditionals are fine.
- **Escalate** when: two contexts share more than value objects.

## References

- `references/bounded-context-pattern.md` — God entities, context isolation
- `references/cross-context-comments.md` — Inline comment conventions
- `references/integration-patterns.md` — Customer/Supplier, ACL, shared primitives
- `references/action-service-pattern.md` — Actions + Services, sync vs async
- `references/state-machine-pattern.md` — Model transitions, enums, events
- `references/job-orchestration-pattern.md` — Chains, batches, retry logic
- `references/php-rules.md` — PHP/Laravel coding standards
- `references/quality-gates.md` — Testing, PHPStan, Pint, Catraca
