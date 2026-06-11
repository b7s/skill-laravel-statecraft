# Integration Patterns Reference

## Overview

When Bounded Contexts need to share data or coordinate behavior, they use explicit integration patterns. Each pattern makes a different trade-off between coupling, autonomy, and complexity.

**Primary pattern:** Customer/Supplier via Laravel events
**For shared data:** Shared primitives (string IDs, int amounts)
**For external systems:** Anti-Corruption Layer (advanced, rarely needed)

## Pattern 1: Customer/Supplier (Primary Pattern)

### What It Is

The upstream context (Supplier) publishes domain events. The downstream context (Customer) subscribes to those events via Laravel event listeners and reacts by starting jobs, job chains, or job batches.

**This is the default pattern for cross-context coordination.**

### When to Use

- The upstream context has a stable event API.
- The downstream context needs to react to upstream state changes.
- You want decoupled, async coordination between contexts.
- **Do NOT use** when the upstream model is unstable/external (use ACL), or when you need synchronous real-time data (query the upstream context's action directly).

### Rules

1. **The Supplier defines the event contract.** Events live in `app/Events/{Context}/`.
2. **The Customer subscribes.** Listeners live in `app/Listeners/{Context}/`.
3. **Event contracts are versioned.** Breaking changes require a new event class name.
4. **Events carry only what the Customer needs.** Not the entire model state.
5. **Failure isolation.** Use queued listeners so Customer failures don't block the Supplier.
6. **Jobs for side effects.** Listeners dispatch jobs for async work (emails, API calls, multi-step processes).

### Example

```php
// app/Events/Billing/InvoicePaid.php — SUPPLIER publishes
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

```php
// app/Services/Billing/InvoicePaymentService.php — SUPPLIER orchestrates
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

The `MarkInvoicePaid` action dispatches the `InvoicePaid` event by default. The service no longer needs to dispatch it manually.

```php
// app/Listeners/Fulfillment/OnInvoicePaidStartShipment.php — CUSTOMER subscribes
<?php
declare(strict_types=1);

namespace App\Listeners\Fulfillment;

use App\Events\Billing\InvoicePaid;
use App\Jobs\Fulfillment\RouteShipment;
use App\Jobs\Fulfillment\NotifyWarehouse;
use App\Jobs\Fulfillment\TrackShipment;
use Illuminate\Support\Facades\Bus;

final class OnInvoicePaidStartShipment
{
    public function handle(InvoicePaid $event): void
    {
        Bus::chain([
            new RouteShipment($event->orderId),
            new NotifyWarehouse($event->orderId),
            new TrackShipment($event->orderId),
        ])->dispatch();
    }
}
```

### Event Registration

```php
// App\Providers\EventServiceProvider
protected $listen = [
    \App\Events\Billing\InvoicePaid::class => [
        \App\Listeners\Fulfillment\OnInvoicePaidStartShipment::class,
    ],
];
```

See `cross-context-comments.md` for comment conventions.

### Pitfalls

| Pitfall | Why It Hurts | Solution |
|---|---|---|
| Stuffing too much data in events | Events become bloated | Carry only what the Customer needs |
| Customer calls Supplier's tables directly | Hard coupling | Use event data or query through action |
| Unversioned events | Breaking changes surprise Consumers | Version event class names |
| Synchronous listeners in critical path | Supplier blocked by slow Customer | Use queued listeners |

---

## Pattern 2: Shared Primitives

### What It Is

Share simple data types (string IDs, int amounts, date strings) across contexts using primitive types + validation. No custom value object classes.

### When to Use

- Multiple contexts need to reference the same entity by ID.
- The data is stable and needs no transformation logic.
- Validation is simple (non-empty string, positive int).
- **Do NOT use** when complex validation logic is needed (use Laravel custom casts or validation rules), or when transformation logic is needed (use Laravel accessors/mutators).

### Example

```php
// Both contexts use primitive string order_id column
// app/Models/Invoice.php — Billing context
class Invoice extends Model
{
    protected $fillable = ['order_id', 'amount_cents', 'currency'];
}

// app/Models/Shipment.php — Fulfillment context
class Shipment extends Model
{
    protected $fillable = ['order_id', 'tracking_code', 'warehouse_id'];
}

// Validation in FormRequest
class CreateInvoiceRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'order_id' => ['required', 'string', 'max:255'],
            'amount_cents' => ['required', 'integer', 'min:1'],
        ];
    }
}
```

**Guideline:** Start with primitives. Add custom casts only when duplication appears across multiple files.

### Pitfalls

| Pitfall | Why It Hurts | Solution |
|---|---|---|
| Creating ValueObject class for every ID | Ceremony without benefit | Use string primitives |
| No validation | Invalid data propagates | Validate in FormRequest or Action |
| Complex validation in multiple places | DRY violation | Extract to custom validation rule or cast |

---

## Pattern 3: Anti-Corruption Layer (Advanced)

### What It Is

A translation layer that protects one context from an external or unstable upstream system. The downstream context defines its own model and the ACL translates external data into it.

**This is an advanced pattern. Most Laravel apps don't need it.**

### When to Use

- The upstream system is external (third-party API, legacy ERP).
- The upstream model is unstable or frequently changes.
- You need to protect your domain model from foreign concepts.
- **Do NOT use** when the upstream is another bounded context in your Laravel app (use Customer/Supplier), when the translation is trivial (use a simple adapter method), or when both systems are under your control.

### Rules

1. **The ACL lives in the downstream context** under `app/Infrastructure/{Context}/ACL/`.
2. **The ACL is the only place that knows both models.**
3. **The ACL translates, it does not contain business logic.**
4. **When the upstream model changes, only the ACL needs updating.**

### Example

```php
<?php
declare(strict_types=1);

namespace App\Infrastructure\Fulfillment\ACL;

use App\Events\Fulfillment\ShipmentDispatched;
use App\Enums\Fulfillment\Carrier;

final class ErpShipmentTranslator
{
    public function translate(array $erpPayload): ShipmentDispatched
    {
        return new ShipmentDispatched(
            shipmentId: $erpPayload['order_num'],
            carrier: $this->translateCarrier($erpPayload['carrier_code']),
            trackingCode: $erpPayload['tracking'],
            dispatchedAt: new \DateTimeImmutable($erpPayload['ship_date']),
        );
    }

    private function translateCarrier(string $erpCode): Carrier
    {
        return match ($erpCode) {
            'FDX' => Carrier::FedEx,
            'UPS' => Carrier::UPS,
            'USPS' => Carrier::USPS,
            default => Carrier::Other,
        };
    }
}
```

**Webhook controller uses the ACL:**

```php
class ErpShipmentController extends Controller
{
    public function __construct(
        private readonly ErpShipmentTranslator $translator,
    ) {}

    public function __invoke(Request $request): JsonResponse
    {
        $request->validate([
            'order_num' => 'required|string',
            'ship_date' => 'required|date',
            'carrier_code' => 'required|string',
            'tracking' => 'required|string',
        ]);

        $event = $this->translator->translate($request->all());

        // Webhook controllers dispatch outside a transaction, so no afterCommit needed
        event($event);

        return response()->json(['status' => 'received']);
    }
}
```

### Testing the ACL

```php
it('translates ERP shipment payload', function () {
    $translator = new ErpShipmentTranslator();

    $event = $translator->translate([
        'order_num' => 'ORD-123',
        'ship_date' => '2026-01-15',
        'carrier_code' => 'FDX',
        'tracking' => '1Z999AA1',
    ]);

    expect($event->shipmentId)->toBe('ORD-123');
    expect($event->carrier)->toBe(Carrier::FedEx);
});
```

### Pitfalls

| Pitfall | Why It Hurts | Solution |
|---|---|---|
| ACL contains business logic | Logic is untested, hidden in translation layer | ACL translates only; model/action enforces rules |
| Upstream types leak past ACL | Downstream depends on upstream model directly | ACL is the only class importing upstream types |
| Using ACL for internal contexts | Unnecessary complexity | Use Customer/Supplier for internal contexts |

---

## Decision Matrix

| Situation | Pattern | Reason |
|---|---|---|
| Billing → Fulfillment coordination | Customer/Supplier | Event-driven, decoupled, async |
| Both contexts need order_id reference | Shared Primitives | String column, validate in FormRequest |
| Legacy ERP webhook | ACL | External system, foreign schema |
| Third-party payment gateway | ACL | External system, protect from changes |
| Two contexts need same entity model | **Red flag** | Split the entity; one model per context |
| One event triggers 5 listeners | Customer/Supplier | One publisher, many subscribers |
| Internal context A → context B | Customer/Supplier | Use events, not direct calls |
