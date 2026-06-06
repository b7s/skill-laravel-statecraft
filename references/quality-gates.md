# Quality Gates Reference

## Overview

Quality gates are mandatory checkpoints that ensure code meets standards before being considered complete. Every feature, refactoring, or bug fix must pass all gates.

## Required Tools

| Tool | Purpose | Level/Preset |
|---|---|---|
| **Pest** | Testing framework | — |
| **Laravel Pint** | Code style formatter | Laravel preset |
| **PHPStan (Larastan)** | Static analysis | Level 6 |
| **Catraca** | Code quality metrics (DRY, SRP, complexity) | Default rules |

## Initial Setup

```bash
composer require --dev pestphp/pest laravel/pint nunomaduro/larastan b7s/catraca
php artisan pest:install

cat > phpstan.neon <<'EOF'
includes:
- vendor/larastan/larastan/extension.neon

parameters:
  level: 6
  paths:
  - app
  excludePaths:
  - app/Console/Kernel.php
  checkMissingIterableValueType: false
EOF
```

## Mandatory Quality Gate Workflow

**After every feature, refactoring, or bug fix, run these commands in order:**

### Step 1: Fix Code Style

```bash
./vendor/bin/pint
```

Auto-fixes indentation, import ordering, trailing commas, blank lines before returns, PSR-12 compliance. **Always run first** — fixes formatting issues before other tools analyze.

### Step 2: Run Tests

```bash
./vendor/bin/pest --parallel
```

Verifies all tests pass, no regressions introduced, new features are covered. **If tests fail:** Fix the code or update the tests. Never skip failing tests.

### Step 3: Static Analysis

```bash
./vendor/bin/phpstan analyse
```

Catches type errors, undefined methods/properties, invalid return types, null pointer risks, dead code. **Must pass at level 6.**

### Step 4: Code Quality Metrics

```bash
./vendor/bin/catraca
```

Detects code duplication (DRY violations), Single Responsibility violations, cyclomatic complexity issues, large classes/methods, god objects. **Exit code 0 = pass, 1 = fail.**

## Testing Requirements

### Coverage by Type

| Component | Test Type | What to Test |
|---|---|---|
| **Model Transitions** | Unit | Every valid transition + every invalid guard |
| **Actions** | Unit | Happy path + DB transaction rollback on failure |
| **Event Listeners** | Unit | Job dispatch verification with `Bus::fake()` |
| **Jobs** | Unit | Execution + failure handling + retry logic |
| **Controllers** | Feature | HTTP request → action → response |

### Model Transition Tests

```php
<?php

use App\Models\Invoice;
use App\Enums\Billing\InvoiceStatus;
use App\Exceptions\InvalidTransitionException;

it('can be paid from pending status', function () {
    $invoice = Invoice::factory()->create(['status' => InvoiceStatus::Pending]);

    $event = $invoice->markPaid('pay_123');

    expect($invoice->status)->toBe(InvoiceStatus::Paid)
        ->and($invoice->payment_id)->toBe('pay_123');
        ->and($event->paymentId)->toBe('pay_123');
});

it('cannot be paid from draft status', function () {
    $invoice = Invoice::factory()->create(['status' => InvoiceStatus::Draft]);

    $invoice->markPaid('pay_123');
})->throws(InvalidTransitionException::class);

it('cannot be paid twice', function () {
    $invoice = Invoice::factory()->create(['status' => InvoiceStatus::Pending]);
    $invoice->markPaid('pay_123');
    $invoice->save();
    $invoice->refresh();

    $invoice->markPaid('pay_456');
})->throws(InvalidTransitionException::class);
```

### Action Tests

```php
<?php

use App\Actions\Billing\MarkInvoicePaid;
use App\Models\Invoice;
use App\Enums\Billing\InvoiceStatus;

it('marks invoice as paid', function () {
    $invoice = Invoice::factory()->create(['status' => InvoiceStatus::Pending]);

    $action = new MarkInvoicePaid();
    $result = $action($invoice, 'pay_123');

    expect($result->status)->toBe(InvoiceStatus::Paid)
        ->and($result->payment_id)->toBe('pay_123');
});

it('rolls back transaction on failure', function () {
    $invoice = Invoice::factory()->create(['status' => InvoiceStatus::Pending]);

    try {
        DB::transaction(function () use ($invoice) {
            $invoice->markPaid('pay_123');
            $invoice->save();
            throw new \RuntimeException('Simulated failure');
        });
    } catch (\RuntimeException) {
        // expected
    }

    expect($invoice->fresh()->status)->toBe(InvoiceStatus::Pending);
});
```

### Event Listener Tests

```php
<?php

use App\Listeners\Fulfillment\OnInvoicePaidStartShipment;
use App\Events\Billing\InvoicePaid;
use App\Jobs\Fulfillment\RouteShipment;
use App\Jobs\Fulfillment\NotifyWarehouse;
use Illuminate\Support\Facades\Bus;

it('dispatches shipment job chain', function () {
    Bus::fake();
    $event = new InvoicePaid(
        invoiceId: 1,
        orderId: 'ord_123',
        paymentId: 'pay_123',
        paidAt: now()
    );

    $listener = new OnInvoicePaidStartShipment();
    $listener->handle($event);

    Bus::assertChained([
        RouteShipment::class,
        NotifyWarehouse::class,
    ]);
});
```

### Job Tests

```php
<?php

use App\Jobs\Billing\SendReceiptEmail;
use App\Models\Invoice;
use Illuminate\Support\Facades\Mail;

it('sends receipt email', function () {
    Mail::fake();
    $invoice = Invoice::factory()->create(['status' => InvoiceStatus::Paid]);

    $job = new SendReceiptEmail($invoice);
    $job->handle();

    Mail::assertSent(ReceiptMail::class, function ($mail) use ($invoice) {
        return $mail->hasTo($invoice->customer_email)
            && $mail->invoice->id === $invoice->id;
    });
});

it('retries on failure', function () {
    $job = new SendReceiptEmail(Invoice::factory()->create());

    expect($job->tries)->toBe(3)
        ->and($job->backoff())->toBe([60, 300, 900]);
});
```

## PHPStan Level 6 Requirements

### What PHPStan Checks at Level 6

- All type declarations are honored
- Return types match actual returns
- Property types match assignments
- Method calls exist on the target class
- Array access on valid types only
- No undefined variables
- No impossible type checks

### Common Fixes

| Problem | Bad | Good |
|---|---|---|
| Property type mismatch | `$this->amount = $stringVal;` | `$this->amount = (int) $stringVal;` |
| Missing return type | `public function calculate($a, $b)` | `public function calculate(int $a, int $b): int` |
| Nullable return | `public function find(int $id): Model` | `public function find(int $id): ?Model` |
| Generic array | `/** @return array */` | `/** @return array<int, Item> */` |

## Catraca Quality Metrics

| Metric | Threshold             | Why |
|---|-----------------------|---|
| **Code duplication** | < 2% duplicated lines | DRY violations |
| **Class length** | < 1000 lines          | SRP violations |
| **Method length** | < 50 lines            | Too much responsibility |
| **Cyclomatic complexity** | < 10 per method       | Hard to test/understand |
| **Parameter count** | < 5 per method        | Refactor to DTO |

## Summary

**Non-negotiable rules:**
1. Run `./vendor/bin/pint` → `./vendor/bin/pest --parallel` → `./vendor/bin/phpstan analyse` → `./vendor/bin/catraca` after every change
2. All commands must exit with code 0 (success)
3. Fix all issues before considering work complete
4. No commits with failing quality gates
5. Every feature requires tests covering happy path + error cases
