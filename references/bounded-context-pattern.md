# Bounded Context Pattern Reference

## Overview

A Bounded Context is an explicit boundary within which a particular domain model and its Ubiquitous Language apply consistently. Outside that boundary, the same word may mean something entirely different — and that is correct, not a bug to fix.

## Why Bounded Contexts Exist

Without bounded contexts, a single `Order` model accumulates every column that any part of the system might ever need: `billing_address`, `shipping_tracking_code`, `compliance_review_notes`, `tax_rate`, `fulfillment_warehouse_id`. This is the "model with 40 nullable columns."

The problem is not laziness. The problem is that **different contexts genuinely need different models of the same business concept.** Billing's "Order" has an amount and a tax rate. Fulfillment's "Order" has a warehouse and a tracking code. These are not missing columns on a single table — they are different models that happen to share an identity.

## The God Entity Anti-Pattern

```php
// BAD: One model serving 4 bounded contexts
class Order extends Model
{
    protected $fillable = [
        'status',           // Which status? Billing? Fulfillment?
        'billing_address',  // Billing cares
        'tracking_code',    // Fulfillment cares
        'compliance_notes', // Compliance cares
        'tax_rate',         // Billing cares
        'warehouse_id',     // Fulfillment cares
        'risk_score',       // Compliance cares
        'payment_id',       // Billing cares
        'carrier_code',     // Fulfillment cares
    ];
}
```

**What goes wrong:**
- Half the columns are null at any given time.
- Status means different things to different contexts.
- Any change to any column risks breaking invariants in a context you didn't think about.
- Tests are bloated: you must set up 40 columns to test one transition.

## The Bounded Context Solution

Split the god model into context-specific Eloquent models, each with only the columns it needs:

```php
// app/Models/Invoice.php — Billing Context
class Invoice extends Model
{
    protected $fillable = ['order_id', 'amount_cents', 'currency', 'status', 'payment_id'];

    public function markPaid(string $paymentId): InvoicePaid { /* ... */ }
}

// app/Models/Shipment.php — Fulfillment Context
class Shipment extends Model
{
    protected $fillable = ['order_id', 'warehouse_id', 'tracking_code', 'status'];

    public function dispatch(string $trackingCode): ShipmentDispatched { /* ... */ }
}

// app/Models/ComplianceReview.php — Compliance Context
class ComplianceReview extends Model
{
    protected $fillable = ['order_id', 'risk_score', 'reviewer_name', 'status'];

    public function approve(string $reviewerName): ComplianceApproved { /* ... */ }
}
```

**What goes right:**
- Each model has only the columns its context needs.
- Each model has its own status enum with transitions that make sense in that context.
- Tests are small and focused: test Billing transitions without knowing Fulfillment exists.
- Changes to Fulfillment's model cannot break Billing's invariants.

## Design Rules

### 1. One Model Per Context Per Business Concept

If "Order" means different things to Billing and Fulfillment, there are two models: `Invoice` and `Shipment`. They share an `order_id` string column (via Shared Primitives pattern) but nothing else.

### 2. Contexts Own Their Ubiquitous Language

Within a context, a term has exactly one meaning. "Status" in Billing means `InvoiceStatus`. "Status" in Fulfillment means `ShipmentStatus`. Do not create a single `OrderStatus` enum across contexts.

### 3. No Direct Cross-Context Table Access

Billing's actions must never query Fulfillment's tables directly. If Billing needs data from Fulfillment, it uses one of the 4 integration patterns.

### 4. Models Reference Other Contexts by ID Only

```php
// GOOD: Reference by ID column
class Invoice extends Model
{
    protected $fillable = ['order_id', 'amount_cents', 'currency', 'status'];
}

// BAD: BelongsTo relationship to another context's model
class Invoice extends Model
{
    public function shipment(): BelongsTo  // This belongs to Fulfillment!
    {
        return $this->belongsTo(Shipment::class);
    }
}
```

### 5. Each Context Has Its Own Directory Grouping

Context-specific files (enums, events, actions, listeners, workflows) are grouped in subfolders by context name. Models live flat in `app/Models/` as Laravel expects.

```
app/Models/
├── Invoice.php
├── Shipment.php
└── ComplianceReview.php

app/Enums/
├── Billing/
│   └── InvoiceStatus.php
├── Fulfillment/
│   └── ShipmentStatus.php
└── Compliance/
    └── ComplianceStatus.php

app/Events/
├── Billing/
│   ├── InvoicePaid.php
│   └── InvoiceCancelled.php
├── Fulfillment/
│   └── ShipmentDispatched.php
└── Shared/
    └── (shared event contracts, if any)

app/Exceptions/
└── InvalidTransitionException.php

app/Actions/
├── Billing/
│   ├── MarkInvoicePaid.php
│   └── CreateInvoice.php
├── Fulfillment/
│   └── StartShipmentWorkflow.php
└── Compliance/
    └── StartComplianceReview.php

app/Listeners/
├── Fulfillment/
│   └── OnInvoicePaidStartShipment.php
└── Compliance/
    └── OnInvoiceCreatedStartReview.php
```

**Note:** No `app/ValueObjects/` or `app/Domain/` directories. Use Laravel primitives (string, int) with validation. Create custom casts only when validation/transformation logic is complex and shared.

### 6. Contexts Are Not Microservices

A context is a domain boundary, not a deployment boundary. All contexts may live in the same Laravel application, the same database, the same repository. Splitting into separate services is a deployment decision made later.

### 7. Merge Contexts When They Share the Same Language

If two modules genuinely use the same terms, the same invariants, and the same lifecycle, they are one context. Do not split for the sake of splitting.

## Testing

Each context's models are tested in isolation. No test should need another context's database tables.

```php
use App\Models\Invoice;
use App\Enums\Billing\InvoiceStatus;
use App\Exceptions\InvalidTransitionException;

it('marks invoice as paid', function () {
    $invoice = Invoice::factory()->create(['status' => InvoiceStatus::Pending->value]);

    $event = $invoice->markPaid('pay_123');

    expect($invoice->status)->toBe(InvoiceStatus::Paid);
    expect($event->paymentId)->toBe('pay_123');
});

it('cannot pay a paid invoice', function () {
    $invoice = Invoice::factory()->create(['status' => InvoiceStatus::Paid->value]);

    $invoice->markPaid('pay_456');
})->throws(InvalidTransitionException::class);
```

**Coverage goals:**
- Every context's models tested in isolation.
- Every cross-context listener tested with a fake event.
- No test depends on another context's database tables.

## Common Pitfalls

| Pitfall | Why It Hurts | Solution |
|---|---|---|
| God model with nullable columns | Half-null objects, implicit coupling | Split into context-specific models |
| Single status enum across contexts | Status means different things; transitions conflict | One enum per context per model |
| Direct cross-context table access | Hard coupling; changes in one context break the other | Use integration patterns |
| Sharing models across contexts | Two contexts depend on the same class; cannot evolve independently | Share IDs via Shared Primitives; own your own model |
| Confusing contexts with microservices | Premature distribution; distributed monolith | Domain first, deployment second |
| Over-splitting | Too many tiny contexts; integration overhead exceeds benefit | Merge contexts that share the same language |
| Missing cross-context comments | Hidden dependencies; no one knows what depends on what | Add `[Upstream → Downstream] Pattern` comments in EventServiceProvider |
