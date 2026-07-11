# Job Orchestration Pattern Reference

## Overview

Laravel's native job system provides everything needed to orchestrate side effects, async operations, and multi-step processes. No custom workflow infrastructure required.

**Core tools:**
- **Job Chains** — Sequential operations
- **Job Batches** — Parallel operations with completion callbacks
- **Delayed Jobs** — Timeouts and scheduled operations
- **Event Listeners + Jobs** — Cross-context coordination

## When to Use

| Situation | Pattern |
|---|---|
| Multiple sequential steps | Job Chain |
| Parallel operations | Job Batch |
| Need retry logic with backoff | Job with `tries` + `backoff()` |
| Wait for external events | Delayed Job with state check |
| Operations span bounded contexts | Event Listener → Job |

**Do NOT use jobs** when the operation is synchronous, fast (< 100ms), and needs no retry.

## Job Chains — Sequential Operations

Use chains when steps must execute in order, and each step depends on the previous.

```php
<?php

use App\Jobs\Billing\SendReceiptEmail;
use App\Jobs\Billing\UpdateAccountBalance;
use App\Jobs\Billing\NotifyAccountingTeam;
use Illuminate\Support\Facades\Bus;

Bus::chain([
    new SendReceiptEmail($invoice),
    new UpdateAccountBalance($invoice),
    new NotifyAccountingTeam($invoice),
])->onQueue('orders')->dispatch();
```

**Execution:** Jobs run one after another. If any job fails, the chain stops.

### Chain with Failure Handling

```php
Bus::chain([
    new ChargePayment($invoice),
    new SendReceipt($invoice),
])
->catch(function (\Throwable $e) use ($invoice) {
    Log::error('Payment chain failed', [
        'invoice_id' => $invoice->id,
        'error' => $e->getMessage(),
    ]);

    RefundPayment::dispatch($invoice);
})
->dispatch();
```

## Job Batches — Parallel Operations

Use batches when steps can execute in parallel and you need to know when all complete.

```php
<?php

use App\Jobs\Fulfillment\RouteShipment;
use Illuminate\Support\Facades\Bus;
use Illuminate\Bus\Batch;

Bus::batch([
    new RouteShipment($orderId, 'warehouse-1'),
    new RouteShipment($orderId, 'warehouse-2'),
    new RouteShipment($orderId, 'warehouse-3'),
])
->then(function (Batch $batch) {
    SelectOptimalWarehouse::dispatch($batch->options['orderId']);
})
->catch(function (Batch $batch, \Throwable $e) {
    Log::error('Batch routing failed', ['batch_id' => $batch->id]);
})
->finally(function (Batch $batch) {
    CleanupTempData::dispatch($batch->id);
})
->options(['orderId' => $orderId])
->dispatch();
```

## Delayed Jobs — Timeouts and Scheduling

Use delayed jobs when you need to wait before executing, or implement timeouts.

```php
ProcessUnpaidInvoice::dispatch($invoice)->delay(now()->addDays(7));
SendReminderEmail::dispatch($user)->delay(now()->addHour());
```

### Job with State Check (Cancellation Pattern)

Laravel doesn't provide built-in job cancellation. Instead, check model state at the start of the job:

```php
<?php

namespace App\Jobs\Billing;

use App\Models\Invoice;
use App\Enums\Billing\InvoiceStatus;
use App\Actions\Billing\CancelInvoice;

class ProcessUnpaidInvoice implements ShouldQueue
{
    public function __construct(
    public readonly int $invoiceId,
) {}

    public function handle(CancelInvoice $cancelInvoice): void
    {
        $invoice = Invoice::query()->find($this->invoiceId);

        if ($invoice === null || $invoice->status !== InvoiceStatus::Pending) {
            return;
        }

        $cancelInvoice($invoice);

        SendCancellationEmail::dispatch($invoice);
    }
}
```

**Register the delayed job in a listener:**

```php
namespace App\Listeners\Billing;

use App\Data\Billing\InvoiceCreatedData;
use App\Jobs\Billing\ProcessUnpaidInvoice;

class OnInvoiceCreatedScheduleTimeout
{
    public function handle(InvoiceCreatedData $data): void
    {
        ProcessUnpaidInvoice::dispatch($data->invoiceId)
            ->delay(now()->addDays(7));
    }
}
```

## Retry Logic and Failure Handling

### Retry with Exponential Backoff

```php
<?php

namespace App\Jobs\Billing;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class NotifyExternalApi implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public int $maxExceptions = 3;
    public int $timeout = 30;

    public function backoff(): array
    {
        return [60, 300, 900];
    }

    public function handle(): void
    {
        Http::timeout(30)->post('https://external-api.example/webhook', [
            'data' => $this->data,
        ]);
    }

    public function failed(\Throwable $exception): void
    {
        Log::error('External API notification failed permanently', [
            'job' => self::class,
            'error' => $exception->getMessage(),
        ]);
    }
}
```

### Custom Retry Logic

```php
public function handle(): void
{
    try {
        $this->processPayment();
    } catch (RateLimitException $e) {
        $this->release(60);
    } catch (PaymentDeclinedException $e) {
        $this->fail($e);
    }
}

public function retryUntil(): \DateTime
{
    return now()->addHours(24);
}
```

## Dispatching Events Inside Transactions

Events dispatched inside a `DB::transaction()` may trigger listeners (and their downstream jobs) **before the transaction commits**. If the transaction later rolls back, those side effects fire against uncommitted (or rolled-back) data. Use `DB::afterCommit()` to defer event dispatch until the transaction actually commits.

### Rule: Actions Use `DB::afterCommit()` for Their Own Domain Event

Actions emit exactly one domain event — their own. They never dispatch jobs or events from other contexts. The `DB::afterCommit()` callback ensures the event only fires after the data is genuinely committed.

```php
final class MarkInvoicePaid
{
    public function __invoke(Invoice $invoice, string $paymentId): Invoice
    {
        return DB::transaction(function () use ($invoice, $paymentId): Invoice {
            $data = $invoice->markPaid($paymentId);

            DB::afterCommit(static fn () => event($data));

            return $invoice;
        });
    }
}
```

### Listeners Dispatch Jobs Normally

Listeners run **after** the action's transaction commits (because the event was deferred via `DB::afterCommit()`). Listeners can dispatch jobs directly — no `afterCommit()` needed:

```php
final class OnInvoicePaidSendReceipt
{
    public function handle(InvoicePaidData $data): void
    {
        SendReceiptEmail::dispatch($data->invoiceId);
    }
}
```

### Services Orchestrating Multiple Actions

When a service calls multiple actions inside a transaction, each action's event is deferred by its own `DB::afterCommit()`. All events fire together after the outer transaction commits:

```php
final class InvoiceService
{
    public function createAndCharge(CreateInvoiceData $payload): Invoice
    {
        return DB::transaction(function () use ($payload): Invoice {
            $invoice = $this->createInvoice($payload);
            $invoice = $this->markPaid($invoice, $payload->paymentId);

            return $invoice;
        });
        // After this transaction commits, both InvoiceCreatedData and InvoicePaidData DTOs fire
    }
}
```

### `DB::afterCommit()` vs `->afterCommit()` on Job Dispatch

| Mechanism | Scope | When to Use |
|---|---|---|
| `DB::afterCommit(static fn () => ...)` | **Any code** inside a transaction — events, jobs, logs | Actions emitting domain events inside `DB::transaction()` |
| `SomeJob::dispatch()->afterCommit()` | **Queued jobs only** | Jobs dispatched inside a transaction that read the written data |

**Prefer `DB::afterCommit()` in actions** because actions emit domain events, not dispatch jobs directly. Jobs are dispatched by listeners or services.

### When to Use `DB::afterCommit()`

| Scenario | Use `DB::afterCommit()`? | Reason |
|---|---|---|
| Action emits domain event inside transaction | **Yes** | Event must only fire after data is committed |
| Service calls actions inside a transaction | **Yes** (each action handles its own) | All events fire after outer commit |
| Event dispatched outside any transaction | No | Nothing to wait for |
| Listener dispatches a job | No | Listener already runs post-commit |

### Testing `DB::afterCommit()` Events

```php
it('dispatches Data DTO after transaction commits', function () {
    Event::fake([InvoicePaidData::class]);
    $invoice = Invoice::factory()->create(['status' => InvoiceStatus::Pending]);

    $action = new MarkInvoicePaid();
    $result = $action($invoice, 'pay_123');

    Event::assertDispatched(InvoicePaidData::class);
});
```

---

## Cross-Context Coordination

Use event listeners to trigger jobs in other contexts. See `integration-patterns.md` for the full pattern.

### Event Listener → Job Chain

```php
<?php

namespace App\Listeners\Fulfillment;

use App\Data\Billing\InvoicePaidData;
use App\Jobs\Fulfillment\RouteShipment;
use App\Jobs\Fulfillment\NotifyWarehouse;
use App\Jobs\Fulfillment\TrackShipment;
use Illuminate\Support\Facades\Bus;

class OnInvoicePaidStartShipment
{
    public function handle(InvoicePaidData $data): void
    {
        Bus::chain([
            new RouteShipment($data->orderId),
            new NotifyWarehouse($data->orderId),
            new TrackShipment($data->orderId),
        ])
        ->onQueue('fulfillment')
        ->dispatch();
    }
}
```

**Registration in EventServiceProvider:**
```php
protected $listen = [
    \App\Data\Billing\InvoicePaidData::class => [
        \App\Listeners\Fulfillment\OnInvoicePaidStartShipment::class,
    ],
];
```

### Event Listener → Job Batch

```php
<?php

namespace App\Listeners\Compliance;

use App\Data\Billing\InvoiceCreatedData;
use App\Jobs\Compliance\CheckCustomerRisk;
use App\Jobs\Compliance\CheckOrderRisk;
use App\Jobs\Compliance\CheckPaymentRisk;
use Illuminate\Support\Facades\Bus;

class OnInvoiceCreatedStartComplianceReview
{
    public function handle(InvoiceCreatedData $data): void
    {
        Bus::batch([
            new CheckCustomerRisk($data->customerId),
            new CheckOrderRisk($data->orderId),
            new CheckPaymentRisk($data->paymentMethod),
        ])
        ->then(function (Batch $batch) use ($data) {
            FinalizeComplianceReview::dispatch($data->invoiceId);
        })
        ->name("compliance-review-{$data->invoiceId}")
        ->dispatch();
    }
}
```

## Bounded Context Rules

### A Job Belongs to Exactly One Context

```php
// GOOD — each job lives in its own context
app/Jobs/Billing/SendReceiptEmail.php
app/Jobs/Fulfillment/NotifyWarehouse.php
app/Jobs/Compliance/RunRiskCheck.php

// BAD — shared job folder
app/Jobs/SendEmail.php // Which context owns this?
```

### Cross-Context Coordination via Events, Not Direct Job Calls

```php
// BAD — Billing job directly dispatches Fulfillment job
namespace App\Jobs\Billing;

class ProcessPayment
{
    public function handle(): void
    {
        \App\Jobs\Fulfillment\StartShipment::dispatch($orderId); // WRONG
    }
}

// GOOD — Billing action dispatches event, Fulfillment listener starts job
namespace App\Services\Billing;

class InvoicePaymentService
{
    public function __construct(
        private readonly MarkInvoicePaid $markPaid,
    ) {}

    public function processPayment(Invoice $invoice, string $paymentId): Invoice
    {
        return $this->markPaid($invoice, $paymentId);
    }
}

// The MarkInvoicePaid action dispatches InvoicePaidData by default.
// No manual event() call needed in the service.
```

## Testing

### Testing Job Dispatch

```php
<?php

use App\Jobs\Billing\SendReceiptEmail;
use Illuminate\Support\Facades\Bus;

it('dispatches receipt email job', function () {
    Bus::fake();

    $service = new InvoicePaymentService(new MarkInvoicePaid());
    $service->processPayment($invoice, 'pay_123');

    Bus::assertDispatched(SendReceiptEmail::class);
});
```

### Testing Job Chains

```php
<?php

use App\Listeners\Fulfillment\OnInvoicePaidStartShipment;
use App\Data\Billing\InvoicePaidData;
use App\Jobs\Fulfillment\RouteShipment;
use App\Jobs\Fulfillment\NotifyWarehouse;
use Illuminate\Support\Facades\Bus;

it('dispatches shipment job chain', function () {
    Bus::fake();
    $data = new InvoicePaidData(
        invoiceId: '1',
        orderId: 'ord_123',
        paymentId: 'pay_123',
        paidAt: now(),
    );

    $listener = new OnInvoicePaidStartShipment();
    $listener->handle($data);

    Bus::assertChained([
        RouteShipment::class,
        NotifyWarehouse::class,
    ]);
});
```

### Testing Job Batches

```php
<?php

use App\Listeners\Compliance\OnInvoiceCreatedStartComplianceReview;
use App\Data\Billing\InvoiceCreatedData;
use App\Jobs\Compliance\CheckCustomerRisk;
use Illuminate\Support\Facades\Bus;

it('dispatches compliance check batch', function () {
    Bus::fake();
    $data = new InvoiceCreatedData(
        invoiceId: '1',
        customerId: 'cust_123',
        orderId: 'ord_123',
        paymentMethod: 'card',
    );

    $listener = new OnInvoiceCreatedStartComplianceReview();
    $listener->handle($data);

    Bus::assertBatched(function ($batch) {
        return $batch->jobs->contains(fn ($job) => $job instanceof CheckCustomerRisk);
    });
});
```

### Testing Delayed Jobs

```php
<?php

use App\Listeners\Billing\OnInvoiceCreatedScheduleTimeout;
use App\Data\Billing\InvoiceCreatedData;
use App\Jobs\Billing\ProcessUnpaidInvoice;
use Illuminate\Support\Facades\Bus;

it('schedules timeout job for 7 days', function () {
    Bus::fake();
    $data = new InvoiceCreatedData(
        invoiceId: '1',
        customerId: 'cust_123',
        orderId: 'ord_123',
        paymentMethod: 'card',
    );

    $listener = new OnInvoiceCreatedScheduleTimeout();
    $listener->handle($data);

    Bus::assertDispatched(ProcessUnpaidInvoice::class, function ($job) {
        return $job->delay->greaterThan(now()->addDays(6));
    });
});
```

## Common Pitfalls

| Pitfall | Why It Hurts | Solution |
|---|---|---|
| Synchronous operations in jobs | Job delays but doesn't parallelize | Only queue I/O-bound work |
| Missing retry logic | Transient failures become permanent | Add `tries`, `backoff()`, `retryUntil()` |
| No timeout | Jobs hang forever | Set `public int $timeout = 300;` |
| Cross-context job calls | Hard coupling | Use events + listeners |
| No state checks in delayed jobs | Process stale data | Check model state at job start |
| Forgetting `DB::afterCommit()` | Event fires before transaction commits | Use `DB::afterCommit(static fn () => event(...))` in actions |
| Missing `failed()` method | No cleanup on permanent failure | Implement `failed(\Throwable $e)` |

## Summary

**Laravel provides everything needed for job orchestration:**
- **Chains** for sequential steps
- **Batches** for parallel steps
- **Delays** for timeouts and scheduling
- **Events + Listeners** for cross-context coordination
- **Retry logic** for resilience
- **State checks** to handle stale data

**No custom workflow infrastructure required. Use Laravel's native tools.**
