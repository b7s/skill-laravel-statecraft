# Cross-Context Comments Reference

## Overview

Cross-context relationships live in code comments, not separate files. AI agents read code directly — a comment next to the listener registration is found instantly. A separate `CONTEXT_MAP.md` is a file the agent may never open.

## Why Comments in Code

AI agents parse code file by file. When an agent reads `EventServiceProvider`, it sees every cross-context dependency in one place. A comment on that line tells it the relationship: who publishes, who subscribes, why.

Without comments, the agent must trace event classes → listener classes → job classes to reconstruct the map. With comments, the map is readable in one pass.

## Comment Format

One line. Bracket pair. Pattern name. Done.

```
// [Upstream → Downstream] Pattern
```

That's it. No paragraphs. No descriptions of what happens — the listener code already shows that.

## Where Comments Go

### EventServiceProvider — Customer/Supplier

```php
protected $listen = [
    // [Billing → Fulfillment] Customer/Supplier
    \App\Events\Billing\InvoicePaid::class => [
        \App\Listeners\Fulfillment\OnInvoicePaidStartShipment::class,
    ],

    // [Billing → Compliance] Customer/Supplier
    \App\Events\Billing\InvoiceCreated::class => [
        \App\Listeners\Compliance\OnInvoiceCreatedStartReview::class,
    ],

    // [Catalog → Billing] Customer/Supplier
    \App\Events\Catalog\InventoryReserved::class => [
        \App\Listeners\Billing\OnInventoryReservedProcessPayment::class,
    ],
];
```

### Models — Shared Primitives

```php
// app/Models/Invoice.php
class Invoice extends Model
{
    // shared: Fulfillment, Compliance
    protected $fillable = ['order_id', 'amount_cents'];
}
```

### ACL Translators — Anti-Corruption Layer

```php
// [Legacy ERP → Fulfillment] ACL
final class ErpShipmentTranslator
{
    public function translate(ErpShipmentWebhookData $data): ShipmentDispatched
    {
        return new ShipmentDispatched(
            shipmentId: $data->external_id,
            orderId: $data->reference_number,
            carrier: $data->carrier_name ?? 'unknown',
        );
    }
}
```

## Summary

| Pattern | Where | Comment |
|---|---|---|
| Customer/Supplier | `EventServiceProvider` | `// [Billing → Fulfillment] Customer/Supplier` |
| Shared Primitives | Model `$fillable` | `// shared: Fulfillment, Compliance` |
| ACL | Translator class | `// [Legacy ERP → Fulfillment] ACL` |

## Rules

1. Every cross-context dependency has a comment.
2. Comments use the `[Upstream → Downstream] Pattern` format.
3. No long descriptions — the code explains the rest.
4. No separate documentation file.
