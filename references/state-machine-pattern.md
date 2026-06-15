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
use App\Data\Billing\InvoicePaidPayload;
use App\Data\Billing\InvoiceCancelledPayload;
use App\Exceptions\InvalidTransitionException;

class Invoice extends Model
{
    protected $fillable = ['order_id', 'amount_cents', 'currency', 'status', 'payment_id'];

    protected function casts(): array
    {
        return ['status' => InvoiceStatus::class];
    }

    public function markPaid(string $paymentId): InvoicePaidPayload
    {
        if ($this->status !== InvoiceStatus::Pending) {
            throw new InvalidTransitionException(
                sprintf('Cannot pay invoice from status "%s"', $this->status->value)
            );
        }

        $this->status = InvoiceStatus::Paid;
        $this->payment_id = $paymentId;
        $this->save();

        return InvoicePaidPayload::fromEvent($this, $paymentId);
    }

    public function cancel(): InvoiceCancelledPayload
    {
        if ($this->status === InvoiceStatus::Paid) {
            throw new InvalidTransitionException('Cannot cancel a paid invoice.');
        }

        $this->status = InvoiceStatus::Cancelled;
        $this->save();

        return InvoiceCancelledPayload::fromEvent($this);
    }
}
```

**Forbidden patterns:**
```php
// BAD: Status checked in controller
if ($order->status === 'pending') { $order->status = 'approved'; }

// BAD: Status changed directly
$order->status = InvoiceStatus::Paid;

// BAD: No Data DTO returned
$invoice->markPaid('pay_123'); // returns void

// BAD: Side effect inside transition method
$invoice->markPaid('pay_123'); // sends email inside

// BAD: Model holds foreign context's column
$invoice->warehouse_id = 'wh_01'; // That's Fulfillment's concern
```

**Rules:**
- Transition methods are the **only** place that mutates the status column.
- Transition methods call `$this->save()` before returning the event. The action never calls `->save()` on the model.
- No facades (`Mail::`, `Event::`, `Cache::`, `DB::`) inside transition methods. `save()` is allowed — it is model persistence, not a facade.
- No cross-context model references. Reference by ID only.
- Data DTOs live in `app/Data/{Context}/`.
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
### 4. Data DTOs Carry Event Information

Every domain transition emits a Data DTO that carries the information downstream listeners need. Instead of extending a shared base class, each DTO is a plain `readonly class` with its own constructor. A `fromEvent()` factory method centralises construction so that the model transition method needs only one line.

**Rules:**
- Data DTOs live in `app/Data/{Context}/`. They are **readonly data carriers** — no behaviour, no side effects.
- One DTO per transition. `InvoicePaidPayload` is correct. `InvoiceStatusChangedData` is wrong.
- Include only data needed by listeners/workflow steps — not the entire model state.
- Never include infrastructure objects (Request, Response, Eloquent models).
- Use a `fromEvent()` static factory to construct the DTO from the domain model.
- Listeners access typed properties directly (`$data->paymentId`).
- No base class, no abstract methods, no interface required — any `event()` dispatchable object works.

```php
<?php

declare(strict_types=1);

namespace App\Data\Billing;

use App\Models\Invoice;
use DateTimeImmutable;

final readonly class InvoicePaidPayload
{
    public function __construct(
        public string $invoiceId,
        public string $orderId,
        public string $paymentId,
        public DateTimeImmutable $paidAt,
    ) {}

    public static function fromEvent(Invoice $invoice, string $paymentId): self
    {
        return new self(
            invoiceId: (string) $invoice->id,
            orderId: $invoice->order_id,
            paymentId: $paymentId,
            paidAt: new DateTimeImmutable(),
        );
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Data\Billing;

use App\Models\Invoice;
use DateTimeImmutable;

final readonly class InvoiceCancelledPayload
{
    public function __construct(
        public string $invoiceId,
        public string $orderId,
        public DateTimeImmutable $cancelledAt,
    ) {}

    public static function fromEvent(Invoice $invoice): self
    {
        return new self(
            invoiceId: (string) $invoice->id,
            orderId: $invoice->order_id,
            cancelledAt: new DateTimeImmutable(),
        );
    }
}
```

**What this unlocks:**
- The model transition returns the DTO, which the action dispaches. No base class, no interface, no ceremony.
- The audit log is recorded separately via `Audit::record()` — the DTO is not coupled to persistence.
- Adding a new transition means creating a new Data DTO file and updating the model — no additional wiring.
### 5. Action Class Bridges Model and Infrastructure

The action calls the model's transition method (which persists internally), records the audit log separately from the Data DTO, and dispatches the Data DTO unconditionally via `DB::afterCommit()`.

```php
<?php

declare(strict_types=1);

namespace App\Actions\Billing;

use App\Models\Invoice;
use Illuminate\Support\Facades\DB;

final class MarkInvoicePaid
{
    public function __invoke(Invoice $invoice, string $paymentId): Invoice
    {
        return DB::transaction(function () use ($invoice, $paymentId): Invoice {
            $data = $invoice->markPaid($paymentId);

            Audit::record('invoice.paid', $invoice, [
                'payment_id' => $paymentId,
            ]);

            DB::afterCommit(static fn () => event($data));

            return $invoice;
        });
    }
}
```

The action body is four lines: transition, audit, dispatch, return. Audit and event dispatch are fully decoupled — the Data DTO only carries data for downstream listeners, not metadata for persistence.

**Service – Call action that dispatches the Data DTO:**

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

**Actions always dispatch the Data DTO:**

```php
namespace App\Services\Billing;

final class InvoiceBatchService
{
    public function __construct(
        private readonly MarkInvoicePaid $markPaid,
    ) {}

    public function processBatch(Invoice $invoice, string $paymentId): Invoice
    {
        return $this->markPaid($invoice, $paymentId);
    }
}
```

See `action-service-pattern.md` for the full action/service rules.

### 6. Testing

Test transition methods in isolation. They do not touch the database — they validate state, mutate properties, and return Data DTOs.

```php
<?php
declare(strict_types=1);

use App\Models\Invoice;
use App\Enums\Billing\InvoiceStatus;
use App\Exceptions\InvalidTransitionException;

it('can be paid from pending', function () {
    $invoice = Invoice::factory()->create(['status' => InvoiceStatus::Pending->value]);

    $data = $invoice->markPaid('pay_123');

    expect($invoice->status)->toBe(InvoiceStatus::Paid)
        ->and($data->paymentId)->toBe('pay_123');
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
| `StatusChangedPayload` DTO | Forces listeners to re-implement transition logic | Named Data DTOs (`InvoicePaidPayload`) |
| Model references another context's data | Hard coupling; cannot evolve independently | Reference by ID via Shared Primitives |

## Directory Structure

```
app/Models/
└── Invoice.php

app/Enums/Billing/
└── InvoiceStatus.php

app/Data/Billing/
├── InvoicePaidPayload.php
└── InvoiceCancelledPayload.php

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
