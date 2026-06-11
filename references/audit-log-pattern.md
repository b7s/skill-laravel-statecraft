# Audit Log Pattern Reference

## Overview

An audit log is an **append-only, immutable record of meaningful domain events**. It is not a log file, it is not a `created_at`/`updated_at` timestamp, and it is not a soft-deleted row. It is a durable trace of what changed, who changed it, and when.

Build the audit log **before** the rest of the system — literally. Every state-changing action writes an audit record as part of its transaction.

## Design Rules

### 1. Append-Only, Enforced

- **Never update** an audit record.
- **Never delete** an audit record.
- The schema enforces this. No `updated_at`, no soft delete, no `DELETE` operations in code.

### 2. Schema

```php
Schema::create('audit_logs', function (Blueprint $table) {
    $table->id();
    $table->string('event_type');           // e.g., 'invoice.paid', 'key.revoked'
    $table->morphs('auditable');            // Polymorphic relation to the entity
    $table->morphs('actor');                 // actor_type + actor_id
    $table->jsonb('context');                // Event-specific data as JSONB
    $table->timestamp('occurred_at');
    $table->timestamps();

    $table->index(['auditable_type', 'auditable_id']);
    $table->index(['actor_type', 'actor_id']);
    $table->index('event_type');
});
```

### 3. capture actor_type + actor_id

An action can be taken by a **human**, by an **API key**, or by the **system itself**. These are three different stories in a security review:

```php
// Human actor
$auditLog->actor_type = 'App\Models\User';
$auditLog->actor_id = 42;

// API key actor
$auditLog->actor_type = 'App\Models\ApiKey';
$auditLog->actor_id = 'key_abc123';

// System actor
$auditLog->actor_type = 'system';
$auditLog->actor_id = 'scheduler';
```

### 4. Store Event-Specific Context as JSONB

Forcing every event type into a fixed set of nullable columns is brittle. JSONB allows flexible, queryable context per event type:

```php
// Invoice paid
[
    'invoice_id' => $invoice->id,
    'order_id' => $invoice->order_id,
    'payment_id' => $paymentId,
    'amount_cents' => $invoice->amount_cents,
]

// Key revoked
[
    'key_id' => $key->id,
    'scopes' => $key->scopes,
    'revoked_reason' => 'user_request',
]
```

## Transaction Sequence

Inside every state-changing action, follow this exact sequence:

```php
final class MarkInvoicePaid
{
    public function __invoke(Invoice $invoice, string $paymentId, bool $dispatchEvent = true): Invoice
    {
        return DB::transaction(function () use ($invoice, $paymentId, $dispatchEvent): Invoice {
            // 1. Perform the operation
            $event = $invoice->markPaid($paymentId);
            $invoice->save();

            // 2. Write the audit record (before side effects)
            AuditLog::query()->create([
                'event_type' => 'invoice.paid',
                'auditable_type' => Invoice::class,
                'auditable_id' => $invoice->id,
                'actor_type' => Auth::user() ? User::class : 'system',
                'actor_id' => Auth::id() ?? 'scheduler',
                'context' => [
                    'payment_id' => $paymentId,
                    'amount_cents' => $invoice->amount_cents,
                ],
                'occurred_at' => now(),
            ]);

            // 3. Dispatch domain event after commit — see job-orchestration-pattern.md
            if ($dispatchEvent) {
                DB::afterCommit(fn () => event($event));
            }

            return $invoice;
        });
    }
}
```

**Why this order matters:** The audit record is written **before** anything that could fail independently. It is the most durable thing in the whole system. If a downstream job (email, webhook) fails, the audit log still accurately records that the state transition happened. The event is deferred via `DB::afterCommit()` so it only fires after the transaction commits — listeners and their jobs never see uncommitted data.

## Model

```php
<?php
declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class AuditLog extends Model
{
    protected $fillable = [
        'event_type',
        'auditable_type',
        'auditable_id',
        'actor_type',
        'actor_id',
        'context',
        'occurred_at',
    ];

    protected function casts(): array
    {
        return [
            'context' => 'array',
            'occurred_at' => 'datetime',
        ];
    }

    public function auditable(): MorphTo
    {
        return $this->morphTo();
    }

    public function actor(): MorphTo
    {
        return $this->morphTo();
    }
}
```

## Testing

```php
it('creates an audit log when invoice is paid', function () {
    $invoice = Invoice::factory()->create(['status' => InvoiceStatus::Pending]);

    $action = new MarkInvoicePaid();
    $action($invoice, 'pay_123');

    assertDatabaseHas('audit_logs', [
        'event_type' => 'invoice.paid',
        'auditable_type' => Invoice::class,
        'auditable_id' => $invoice->id,
    ]);

    $log = AuditLog::query()->latest()->first();
    expect($log->context)->toHaveKey('payment_id', 'pay_123')
        ->and($log->context)->toHaveKey('amount_cents', $invoice->amount_cents);
});
```

## Common Pitfalls

| Pitfall | Why It Hurts | Solution |
|---|---|---|
| Updating audit records | Breaks immutability guarantee | Schema blocks `updated_at`; code never updates |
| Missing `actor_type` | Cannot distinguish human vs API vs system | Always store both `actor_type` and `actor_id` |
| Soft-deleting audit records | Creates gaps in audit trail | No soft delete; append-only |
| Writing audit **after** side effects | Side effect fails, audit misses the event | Write audit **before** dispatching side effects via `DB::afterCommit()` |
| Flat nullable columns for context | Schema explosion, hard to query | Use JSONB for event-specific context |
| No `occurred_at` | Relies on `created_at` which may differ in batch imports | Always store explicit `occurred_at` |
