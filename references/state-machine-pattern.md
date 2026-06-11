# State Machine Pattern Reference

## Overview

The state machine layer owns the truth about which transitions are legal for an Eloquent model **within its Bounded Context**. Transition methods live directly on the Eloquent model in `app/Models/`. They validate state, mutate the status, and return a domain event.

See `bounded-context-pattern.md` for why god entities are split into context-specific models.

## When to Use

Use a state machine when:
- An entity has 5 or more possible statuses.
- Transitions form a non-trivial graph (not just a linear sequence).
- Different transitions require different side effects.
- Invalid transitions must be prevented at the domain level.

Do NOT use a state machine when:
- There are only 2-3 statuses and simple linear progression.
- No business rules depend on the current status.

## Design Rules

### 1. Status Must Be an Enum

Never use strings for status. Each context has its own status enum in `app/Enums/{Context}/`.

```php
// app/Enums/Billing/InvoiceStatus.php
enum InvoiceStatus: string
{
    case Draft = 'draft';
    case Pending = 'pending';
    case Paid = 'paid';
    case Cancelled = 'cancelled';

    public static function default(): self { return self::Draft; }
}

// app/Enums/Fulfillment/ShipmentStatus.php
enum ShipmentStatus: string
{
    case Pending = 'pending';
    case Dispatched = 'dispatched';
    case Delivered = 'delivered';

    public static function default(): self { return self::Pending; }
}
```

For the full enum standard (including `default()`, `label()`, `options()`, `values()`), see `references/php-rules.md`.

### 2. Eloquent Model Owns All Transitions

Every status change is a named method on the model. The method validates, changes state, and returns a domain event.

```php
<?php
declare(strict_types=1);

namespace App\Models;

use App\Enums\Billing\InvoiceStatus;
use App\Events\Billing\InvoicePaid;
use App\Events\Billing\InvoiceCancelled;
use App\Exceptions\InvalidTransitionException;

class Invoice extends Model
{
    protected $fillable = ['order_id', 'amount_cents', 'currency', 'status', 'payment_id'];

    protected function casts(): array
    {
        return ['status' => InvoiceStatus::class];
    }

    public function markPaid(string $paymentId): InvoicePaid
    {
        if ($this->status !== InvoiceStatus::Pending) {
            throw new InvalidTransitionException(
                sprintf('Cannot pay invoice from status "%s"', $this->status->value)
            );
        }

        $this->status = InvoiceStatus::Paid;
        $this->payment_id = $paymentId;

        return new InvoicePaid(
            invoiceId: $this->id,
            orderId: $this->order_id,
            paymentId: $paymentId,
            paidAt: new \DateTimeImmutable(),
        );
    }

    public function cancel(): InvoiceCancelled
    {
        if ($this->status === InvoiceStatus::Paid) {
            throw new InvalidTransitionException('Cannot cancel a paid invoice.');
        }

        $this->status = InvoiceStatus::Cancelled;

        return new InvoiceCancelled(
            invoiceId: $this->id,
            orderId: $this->order_id,
            cancelledAt: new \DateTimeImmutable(),
        );
    }
}
```

**Forbidden patterns:**
```php
// BAD: Status checked in controller
if ($order->status === 'pending') { $order->status = 'approved'; }

// BAD: Status changed directly
$order->status = InvoiceStatus::Paid;

// BAD: No event returned
$invoice->markPaid('pay_123'); // returns void

// BAD: Side effect inside transition method
$invoice->markPaid('pay_123'); // sends email inside

// BAD: Model holds foreign context's column
$invoice->warehouse_id = 'wh_01'; // That's Fulfillment's concern
```

**Rules:**
- Transition methods are the **only** place that mutates the status column.
- No facades (`Mail::`, `Event::`, `Cache::`, `DB::`) inside transition methods.
- No cross-context model references. Reference by ID only.
- Domain events live in `app/Events/{Context}/`.
- Exceptions live in `app/Exceptions/`.

### 3. Use Typed Exceptions

```php
<?php
declare(strict_types=1);

namespace App\Exceptions;

final class InvalidTransitionException extends \DomainException {}
```

**Exception hierarchy:**
```
\DomainException
├── InvalidTransitionException
├── InvalidStateException
└── BusinessRuleViolationException
```

### 4. Domain Events Are Value Objects

Events live in `app/Events/{Context}/`.

```php
<?php
declare(strict_types=1);

namespace App\Events\Billing;

final readonly class InvoicePaid
{
    public function __construct(
        public int $invoiceId,
        public string $orderId,
        public string $paymentId,
        public \DateTimeImmutable $paidAt,
    ) {}
}
```

**Rules:**
- Use `readonly class` or `readonly` properties.
- Include the entity ID and a timestamp at minimum.
- Include only data needed by listeners/workflow steps — not the entire model state.
- Never include infrastructure objects (Request, Response, Eloquent models).
- One event per transition. `InvoicePaid` is correct. `InvoiceStatusChanged` is wrong.

### 5. Action Class Bridges Model and Infrastructure

The action calls the model's transition method, persists with `save()` inside a DB transaction, and dispatches the event by default using `DB::afterCommit()`. Pass `$dispatchEvent = false` to suppress the event when needed (e.g., batch processing, re-imports).

```php
<?php
declare(strict_types=1);

namespace App\Actions\Billing;

use App\Models\Invoice;

final class MarkInvoicePaid
{
    public function __invoke(Invoice $invoice, string $paymentId, bool $dispatchEvent = true): Invoice
    {
        return DB::transaction(function () use ($invoice, $paymentId, $dispatchEvent): Invoice {
            $event = $invoice->markPaid($paymentId);
            $invoice->save();

            if ($dispatchEvent) {
                DB::afterCommit(fn () => event($event));
            }

            return $invoice;
        });
    }
}
```

**Service – Call action that's dispatch the event:**

```php
namespace App\Services\Billing;

final class InvoicePaymentService
{
    public function __construct(
        private readonly MarkInvoicePaid $markPaid,
    ) {}

    public function processPayment(Invoice $invoice, string $paymentId): Invoice
    {
        return $this->markPaid($invoice, $paymentId);
    }
}
```

**Suppressing the event when needed (e.g., batch processing):**

```php
namespace App\Services\Billing;

final class InvoiceBatchService
{
    public function __construct(
        private readonly MarkInvoicePaid $markPaid,
    ) {}

    public function processBatch(Invoice $invoice, string $paymentId): Invoice
    {
        return $this->markPaid($invoice, $paymentId, dispatchEvent: false);
    }
}
```

See `action-service-pattern.md` for the full action/service rules.

### 6. Testing

Test transition methods in isolation. They do not touch the database — they validate state, mutate properties, and return events.

```php
<?php
declare(strict_types=1);

use App\Models\Invoice;
use App\Enums\Billing\InvoiceStatus;
use App\Exceptions\InvalidTransitionException;

it('can be paid from pending', function () {
    $invoice = Invoice::factory()->create(['status' => InvoiceStatus::Pending->value]);

    $event = $invoice->markPaid('pay_123');

    expect($invoice->status)->toBe(InvoiceStatus::Paid)
        ->and($event->paymentId)->toBe('pay_123');
});

it('cannot be paid from draft', function () {
    $invoice = Invoice::factory()->create(['status' => InvoiceStatus::Draft->value]);

    $invoice->markPaid('pay_123');
})->throws(InvalidTransitionException::class);

it('cannot be paid twice', function () {
    $invoice = Invoice::factory()->create(['status' => InvoiceStatus::Pending->value]);
    $invoice->markPaid('pay_123');

    $invoice->markPaid('pay_456');
})->throws(InvalidTransitionException::class);
```

**Test the action against a real database** — the action wraps the transition, persists, and dispatches the event. Assert on real outcomes, not mocks.

**Database Safety:** Always run tests on a dedicated test database. Use `php artisan test` (switches automatically) or configure `.env.testing` with a separate `DB_DATABASE`. Never run tests against development or production databases.

```php
it('marks invoice as paid through the action', function () {
    $invoice = Invoice::factory()->create(['status' => InvoiceStatus::Pending->value]);

    $action = new MarkInvoicePaid();
    $result = $action($invoice, 'pay_123');

    expect($result->status)->toBe(InvoiceStatus::Paid)
        ->and($result->payment_id)->toBe('pay_123');

    assertDatabaseHas('invoices', [
        'id' => $invoice->id,
        'status' => InvoiceStatus::Paid->value,
        'payment_id' => 'pay_123',
    ]);
});
```

**Coverage goals:**
- Every valid transition is tested once.
- Every invalid transition is tested once.
- **No test requires another context's database tables.**

## Common Pitfalls

| Pitfall | Why It Hurts | Solution |
|---|---|---|
| God model with nullable columns | Half-null objects; implicit coupling | Split into context-specific models |
| Single status enum across contexts | Statuses conflict; combinatorial explosion | One enum per context per model |
| String status columns | Typos pass silently; no IDE support | Enum |
| Status checks in controllers | Business rules leak to HTTP layer | Move to model transition method |
| Side effects in transition method | Untestable; hidden dependencies | Extract to action or workflow step |
| Generic exceptions | Callers cannot distinguish failures | Typed exceptions |
| `StatusChanged` events | Forces listeners to re-implement transition logic | Named transition events (`InvoicePaid`) |
| Model references another context's data | Hard coupling; cannot evolve independently | Reference by ID via Shared Primitives |

## Directory Structure

```
app/Models/
└── Invoice.php

app/Enums/Billing/
└── InvoiceStatus.php

app/Events/Billing/
├── InvoicePaid.php
└── InvoiceCancelled.php

app/Exceptions/
└── InvalidTransitionException.php

app/Actions/Billing/
├── CreateInvoice.php
├── MarkInvoicePaid.php
├── CancelInvoice.php
├── ListInvoices.php
└── GetInvoice.php

app/Services/Billing/
├── TaxCalculator.php
└── InvoiceService.php

...
```
