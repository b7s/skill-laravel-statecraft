# Action and Services Pattern Reference

## Overview

The Action Pattern represents **one atomic operation**: save an invoice, generate a PDF, send an email, delete a user. Each action is a single class that does **exactly one thing**.

Actions live in `App\Actions\{Context}\` as a flat folder — one file per action. They receive validated data, perform ONE operation, wrap mutations in DB transactions, and return results.

**Actions use `__invoke()` magic method** — makes them invokable objects: `$action($param)` instead of `$action->execute($param)` or `$action->handle($param)`.

**Actions work alongside Services:**
- **Actions** = Atomic operations (one thing only)
- **Services** = Orchestration (calling multiple actions) OR domain logic (calculations, validation)
- **Actions NEVER call other actions** — only services orchestrate

## Actions vs Services

| Dimension | Action | Service |
|---|---|---|
| **Purpose** | Perform ONE atomic operation | Orchestrate multiple actions OR encapsulate domain logic |
| **Location** | `app/Actions/{Context}/` | `app/Services/{Context}/` |
| **Mutations** | YES — wraps in `DB::transaction()` | Orchestrators: call actions; Logic services: NO mutations |
| **Returns** | Model, Collection, DTO, string, or void | Orchestrators: Model/DTO; Logic services: calculated value |
| **Examples** | `CreateInvoice`, `MarkInvoicePaid`, `GenerateInvoicePdf` | Orchestrators: `InvoiceService`, `OrderService`; Logic: `TaxCalculator`, `PricingEngine` |
| **Composability** | NEVER calls other actions | Orchestrators call actions; Logic services call other logic services |
| **Testability** | Test with real DB (RefreshDatabase) | Orchestrators: test with real actions; Logic: pure assertions |

**Key principle:**
- **1 operation = Action**
- **2+ operations = Service orchestrating Actions**
- **Pure logic = Service (no actions)**

## Core Rules

### 1. One Action = One Atomic Operation

Each action does exactly one thing. No composition, no orchestration, no calling other actions.

```php
<?php
declare(strict_types=1);

namespace App\Actions\Billing;

use App\Models\Invoice;
use App\Enums\Billing\InvoiceStatus;

final class CreateInvoice
{
    public function __invoke(array $data): Invoice
    {
        return DB::transaction(function () use ($data): Invoice {
            return Invoice::query()->create([
                'order_id' => $data['order_id'],
                'amount_cents' => $data['amount_cents'],
                'tax_amount_cents' => $data['tax_amount_cents'],
                'total_cents' => $data['total_cents'],
                'status' => InvoiceStatus::default(),
            ]);
        });
    }
}

final class MarkInvoicePaid
{
    public function __invoke(Invoice $invoice, string $paymentId, bool $dispatchEvent = true): Invoice
    {
        return DB::transaction(function () use ($invoice, $paymentId, $dispatchEvent): Invoice {
            $event = $invoice->markPaid($paymentId);
            $invoice->save();

            if ($dispatchEvent) {
                event($event);
            }

            return $invoice;
        });
    }
}

final class GenerateInvoicePdf
{
    public function __invoke(Invoice $invoice): string
    {
        $pdf = Pdf::loadView('invoices.pdf', ['invoice' => $invoice]);
        $path = "invoices/{$invoice->id}.pdf";
        Storage::put($path, $pdf->output());
        return $path;
    }
}
```

**BAD — Action calling other actions:**

```php
final class CreateAndProcessInvoice
{
    public function __construct(
        private readonly CreateInvoice $createInvoice,
        private readonly GenerateInvoicePdf $generatePdf,
        private readonly SendInvoiceEmail $sendEmail,
    ) {}

    public function __invoke(array $data): Invoice
    {
        $invoice = $this->createInvoice($data);
        $pdfPath = $this->generatePdf($invoice);
        $this->sendEmail($invoice, $pdfPath);

        return $invoice;
    }
}
```

**GOOD — Service orchestrating actions:**

```php
namespace App\Services\Billing;

use App\Actions\Billing\CreateInvoice;
use App\Actions\Billing\GenerateInvoicePdf;
use App\Actions\Billing\SendInvoiceEmail;
use App\Models\Invoice;

final class InvoiceService
{
    public function __construct(
        private readonly CreateInvoice $createInvoice,
        private readonly GenerateInvoicePdf $generatePdf,
        private readonly SendInvoiceEmail $sendEmail,
    ) {}

    public function createAndProcess(array $data): Invoice
    {
        $invoice = $this->createInvoice($data);
        $pdfPath = $this->generatePdf($invoice);
        $this->sendEmail($invoice, $pdfPath);

        return $invoice;
    }
}
```

### 2. Services Orchestrate, Actions Execute

**Two types of services:**

1. **Orchestrator Services** — Call multiple actions to complete a workflow
2. **Logic Services** — Pure calculations, no mutations, no action calls

**Orchestrator Service — combining logic services + actions:**

```php
namespace App\Services\Billing;

use App\Actions\Billing\CreateInvoice;
use App\Actions\Billing\MarkInvoicePaid;
use App\Actions\Billing\GenerateInvoicePdf;
use App\Models\Invoice;

final class InvoiceService
{
    public function __construct(
        private readonly TaxCalculator $taxCalculator,
        private readonly PaymentGateway $paymentGateway,
        private readonly CreateInvoice $createInvoice,
        private readonly MarkInvoicePaid $markPaid,
        private readonly GenerateInvoicePdf $generatePdf,
    ) {}

    public function createAndCharge(array $data): Invoice
    {
        $taxAmount = $this->taxCalculator->calculate($data['amount_cents'], $data['country']);

        $invoice = $this->createInvoice([
            'order_id' => $data['order_id'],
            'amount_cents' => $data['amount_cents'],
            'tax_amount_cents' => $taxAmount,
            'total_cents' => $data['amount_cents'] + $taxAmount,
        ]);

        $payment = $this->paymentGateway->charge($invoice->customer_id, $invoice->total_cents);

        $invoice = $this->markPaid($invoice, $payment->id);

        $this->generatePdf($invoice);

        return $invoice;
    }
}
```

**Logic Service (no actions, no mutations):**

```php
namespace App\Services\Billing;

final class TaxCalculator
{
    public function calculate(int $amountCents, string $countryCode): int
    {
        $rate = $this->getRateForCountry($countryCode);
        return (int) round($amountCents * $rate);
    }

    private function getRateForCountry(string $countryCode): float
    {
        return match ($countryCode) {
            'BR' => 0.18,
            'US' => 0.07,
            'DE' => 0.19,
            default => 0.0,
        };
    }
}
```

**Logic Service with External API:**

```php
namespace App\Services\Billing;

final class PaymentGateway
{
    public function __construct(
        private readonly string $apiKey,
    ) {}

    public function charge(string $customerId, int $amountCents): PaymentResult
    {
        $response = Http::withToken($this->apiKey)
            ->post('https://api.stripe.com/v1/charges', [
                'customer' => $customerId,
                'amount' => $amountCents,
                'currency' => 'usd',
            ]);

        if ($response->failed()) {
            return PaymentResult::failed($response->json('error.message'));
        }

        return PaymentResult::success(
            transactionId: $response->json('id'),
            amountCents: $amountCents,
        );
    }
}
```

### 3. No HTTP Concerns Inside Actions

Actions receive validated data as an array or DTO. They never receive `Request`, `FormRequest`, or any HTTP object.

```php
// BAD
final class UpdateUser
{
    public function __invoke(UpdateUserRequest $request): User { /* ... */ }
}

// GOOD
final class UpdateUser
{
    public function __invoke(array $data, User $user): User { /* ... */ }
}
```

HTTP-only concerns (logout, session invalidate, redirect) stay in the controller:

```php
final class UserController extends Controller
{
    public function destroy(User $user, DeleteUser $action): RedirectResponse
    {
        $action($user);

        Auth::logout();
        session()->invalidate();

        return redirect('/');
    }
}
```

### 4. Form Requests Validate Before the Action

```php
final class UpdateUserRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'name' => ['required', 'string', 'max:255'],
            'email' => ['required', 'string', 'email', 'lowercase', 'max:255', Rule::unique('users')->ignore($this->user->id)],
        ];
    }
}

final class UserController extends Controller
{
    public function update(UpdateUserRequest $request, User $user, UpdateUser $action): RedirectResponse
    {
        ($action)($request->validated(), $user);

        return redirect()->back();
    }
}
```

### 5. Every Mutating Action Wraps in a DB Transaction

Actions that mutate state wrap their operation in `DB::transaction()`. No nested action calls.

```php
final class MarkInvoicePaid
{
    public function __invoke(Invoice $invoice, string $paymentId, bool $dispatchEvent = true): Invoice
    {
        return DB::transaction(function () use ($invoice, $paymentId, $dispatchEvent): Invoice {
            $event = $invoice->markPaid($paymentId);
            $invoice->save();

            if ($dispatchEvent) {
                event($event);
            }

            return $invoice;
        });
    }
}
```

If you need multiple mutations, use a Service:

```php
final class InvoiceService
{
    public function __construct(
        private readonly MarkInvoicePaid $markPaid,
        private readonly CreateActivity $createActivity,
    ) {}

    public function markPaidAndLog(Invoice $invoice, string $paymentId): Invoice
    {
        $invoice = $this->markPaid($invoice, $paymentId);
        $this->createActivity([
            'invoice_id' => $invoice->id,
            'type' => 'payment_received',
        ]);

        return $invoice;
    }
}
```

### 6. Controllers Call Services for Workflows

For simple operations, controllers can call actions directly. For workflows, controllers call services.

**Simple operation — Controller calls Action:**

```php
final class InvoiceController extends Controller
{
    public function destroy(Invoice $invoice, DeleteInvoice $action): RedirectResponse
    {
        $action($invoice);
        return redirect()->route('invoices.index');
    }
}
```

**Complex workflow — Controller calls Service:**

```php
final class InvoiceController extends Controller
{
    public function store(CreateInvoiceRequest $request, InvoiceService $service): RedirectResponse
    {
        $invoice = $service->createAndProcess($request->validated());
        return redirect()->route('invoices.show', $invoice);
    }
}
```

**Wrong — Controller orchestrating actions:**

```php
final class InvoiceController extends Controller
{
    public function store(
        CreateInvoiceRequest $request,
        CreateInvoice $createInvoice,
        GenerateInvoicePdf $generatePdf,
        SendInvoiceEmail $sendEmail,
    ): RedirectResponse {
        $invoice = $createInvoice($request->validated());
        $pdfPath = $generatePdf($invoice);
        $sendEmail($invoice, $pdfPath);

        return redirect()->route('invoices.show', $invoice);
    }
}
```

### 7. Actions Return Results

```php
// Write — returns the created/updated model
final class SaveInvoice
{
    public function __invoke(array $data): Invoice { /* ... */ }
}

// Read — returns collection or paginator
final class ListInvoices
{
    public function __invoke(array $filters, int $perPage = 15): LengthAwarePaginator
    {
        return Invoice::query()
            ->when($filters['status'] ?? null, static fn ($q, $status) => $q->where('status', $status))
            ->when($filters['from'] ?? null, static fn ($q, $from) => $q->where('created_at', '>=', $from))
            ->paginate($perPage);
    }
}

// Delete — returns void
final class DeleteInvoice
{
    public function __invoke(Invoice $invoice): void
    {
        DB::transaction(static fn () => $invoice->delete());
    }
}

// Generate file — returns path
final class GenerateInvoicePdf
{
    public function __invoke(Invoice $invoice): string
    {
        $pdf = Pdf::loadView('invoices.pdf', ['invoice' => $invoice]);
        $path = "invoices/{$invoice->id}.pdf";
        Storage::put($path, $pdf->output());
        return $path;
    }
}
```

## When to Use Actions vs Services

| Scenario | Use | Example |
|---|---|---|
| **Single database operation** | Action | `CreateInvoice`, `DeleteUser` |
| **Single file operation** | Action | `GenerateInvoicePdf`, `ExportCsv` |
| **Single email/SMS** | Action | `SendInvoiceEmail`, `SendSmsNotification` |
| **Single external API call** | Action | `ChargePaymentCard`, `SendSlackMessage` |
| **Mark/Update status** | Action | `MarkInvoicePaid`, `ApproveOrder` |
| **Multiple operations (workflow)** | Service (orchestrator) | `InvoiceService`, `OrderCheckoutService` |
| **Complex calculation** | Service (logic) | `TaxCalculator`, `ShippingCostCalculator` |
| **Business rule validation** | Service (logic) | `OrderEligibilityChecker` |
| **Data transformation** | Service (logic) | `PriceFormatter`, `AddressNormalizer` |

## Sync vs Async: Services Dispatch Jobs

Services decide sync vs async. Critical/fast operations call actions directly (sync). Slow/fallible operations dispatch jobs (async). Jobs use actions for the actual work.

### Decision Matrix

| Operation | Pattern | Why |
|---|---|---|
| Fast calculation (< 100ms) | Service → Action (sync) | No async overhead needed |
| Database query | Service → Action (sync) | Fast, needs immediate result |
| External API call | Service → Job → Action (async) | Slow, can fail, needs retry |
| Email/SMS | Service → Job → Action (async) | Slow, non-critical, can be delayed |
| PDF generation | Service → Job → Action (async) | CPU-intensive, can be queued |
| File upload/processing | Service → Job → Action (async) | I/O-intensive, can fail |
| Payment processing | Service → Action (sync) | Critical, needs immediate feedback |
| Webhook call | Service → Job → Action (async) | Can fail, needs retry logic |

### Service Orchestrating Sync + Async

```php
final class OrderCheckoutService
{
    public function __construct(
        private readonly CreateOrder $createOrder,
        private readonly TaxCalculator $taxCalculator,
        private readonly InventoryChecker $inventoryChecker,
    ) {}

    public function checkout(array $data): Order
    {
        $taxAmount = $this->taxCalculator->calculate($data['amount_cents'], $data['country']);
        $hasStock = $this->inventoryChecker->check($data['items']);

        if (!$hasStock) {
            throw new InsufficientStockException();
        }

        $order = $this->createOrder([
            'customer_id' => $data['customer_id'],
            'amount_cents' => $data['amount_cents'],
            'tax_amount_cents' => $taxAmount,
            'status' => OrderStatus::Pending,
        ]);

        SendOrderConfirmationEmail::dispatch($order->id);
        NotifyWarehouse::dispatch($order->id);
        UpdateAnalytics::dispatch($order->id);

        return $order;
    }
}
```

### Job Uses Action

```php
final class GenerateInvoicePdfJob implements ShouldQueue
{
    use Queueable;

    public int $tries = 3;

    public function backoff(): array
    {
        return [60, 300, 900];
    }

    public function __construct(
        public readonly int $invoiceId,
    ) {}

    public function handle(GenerateInvoicePdf $generatePdf): void
    {
        $invoice = Invoice::findOrFail($this->invoiceId);

        $pdfPath = $generatePdf($invoice);

        $invoice->pdf_path = $pdfPath;
        $invoice->save();
    }
}
```

### Async Example: Service dispatches job for external API

```php
final class InvoicePaymentService
{
    public function __construct(
        private readonly MarkInvoicePaid $markPaid,
    ) {}

    public function processPayment(Invoice $invoice, string $paymentId): Invoice
    {
        $invoice = $this->markPaid($invoice, $paymentId);

        NotifyAccountingSystem::dispatch($invoice->id);

        return $invoice;
    }
}

final class NotifyAccountingSystem implements ShouldQueue
{
    use Queueable;

    public int $tries = 3;

    public function backoff(): array
    {
        return [60, 300, 900];
    }

    public function __construct(
        public readonly int $invoiceId,
    ) {}

    public function handle(AccountingApiService $api): void
    {
        $invoice = Invoice::findOrFail($this->invoiceId);

        $api->sendInvoice($invoice);
    }
}
```

### File Operations

```php
final class StoreImportFile
{
    public function __invoke(UploadedFile $file): ImportBatch
    {
        return DB::transaction(function () use ($file): ImportBatch {
            $path = $file->store('imports');

            return ImportBatch::query()->create([
                'filename' => $file->getClientOriginalName(),
                'file_path' => $path,
                'status' => ImportStatus::Pending,
            ]);
        });
    }
}

final class CustomerImportService
{
    public function __construct(
        private readonly StoreImportFile $storeImportFile,
    ) {}

    public function import(UploadedFile $file): ImportBatch
    {
        $batch = $this->storeImportFile($file);

        ProcessCustomerImport::dispatch($batch->id, $batch->file_path);

        return $batch;
    }
}

final class ProcessCustomerImport implements ShouldQueue
{
    use Queueable;

    public function __construct(
        public readonly int $batchId,
        public readonly string $filePath,
    ) {}

    public function handle(CustomerImportProcessor $processor): void
    {
        $batch = ImportBatch::findOrFail($this->batchId);

        $result = $processor->process($this->filePath);

        $batch->status = ImportStatus::Completed;
        $batch->imported_count = $result->successCount;
        $batch->failed_count = $result->failureCount;
        $batch->save();
    }
}
```

## Directory Structure

```
app/Actions/
├── Billing/
│   ├── CreateInvoice.php
│   ├── SaveInvoice.php
│   ├── MarkInvoicePaid.php
│   ├── CancelInvoice.php
│   ├── ListInvoices.php
│   ├── GetInvoice.php
│   └── DeleteInvoice.php
├── Fulfillment/
│   ├── CreateShipment.php
│   ├── DispatchShipment.php
│   └── ListShipments.php
├── Compliance/
│   ├── StartComplianceReview.php
│   ├── ApproveCompliance.php
│   └── RejectCompliance.php
└── User/
    ├── RegisterUser.php
    ├── UpdateUser.php
    └── DeleteUser.php
```

Flat folder per context. No subfolders beyond the context.

## Naming Convention

| Operation | Name Pattern | Example |
|---|---|---|
| Create | `Create{Entity}` | `CreateInvoice` |
| Save (create or update) | `Save{Entity}` | `SaveInvoice` |
| Update | `Update{Entity}` | `UpdateUser` |
| Delete | `Delete{Entity}` | `DeleteInvoice` |
| List (paginated) | `List{Entities}` | `ListInvoices` |
| Get (single) | `Get{Entity}` | `GetInvoice` |
| Transition | `{Verb}{Entity}` | `MarkInvoicePaid`, `DispatchShipment` |
| Notify | `Send{What}Notification` | `SendWelcomeEmail` |
| Workflow start | `Start{Workflow}` | `StartShipmentWorkflow` |
| ACL translate | `Translate{Source}` | `TranslateErpShipment` |

## Testing

**Test the real action against a real database.** Assert on outcomes — database state, model state, response shape — not on method calls. Mocks are the last choice, reserved for external boundaries you do not control.

**Database Safety:** Ensure `php artisan test` is used (it switches to the test database automatically), or that `DB_CONNECTION` in `.env.testing` points to a database dedicated for testing. Never run tests against development, staging, or production databases.

```php
it('marks invoice as paid', function () {
    $invoice = Invoice::factory()->create(['status' => InvoiceStatus::Pending->value]);

    $action = new MarkInvoicePaid();
    $result = $action($invoice, 'pay_123');

    expect($result->status)->toBe(InvoiceStatus::Paid);
    expect($result->payment_id)->toBe('pay_123');
    assertDatabaseHas('invoices', ['id' => $invoice->id, 'status' => InvoiceStatus::Paid->value]);
});

it('rolls back on failure', function () {
    $invoice = Invoice::factory()->create(['status' => InvoiceStatus::Pending->value]);

    try {
        DB::transaction(function () use ($invoice) {
            $action = new MarkInvoicePaid();
            $action($invoice, 'pay_123');
            throw new \RuntimeException('Simulated failure');
        });
    } catch (\RuntimeException) {
        // expected
    }

    expect($invoice->fresh()->status)->toBe(InvoiceStatus::Pending);
});
```

**Coverage goals:**
- Every action tested for happy path
- Mutating actions tested for rollback on failure
- Read actions tested with filter combinations
- Transition actions tested for invalid state guards
- Orchestrator services tested with real actions (not mocked)

## Common Pitfalls

| Pitfall | Why It Hurts | Solution |
|---|---|---|
| Action receives Request object | Not reusable from console/jobs/other services | Accept array or DTO |
| Action validates input | Duplicates Form Request validation | Let Form Request validate |
| Action calls other actions | Violates Single Responsibility; should be service | Extract to Service |
| Action dispatches jobs | Two responsibilities (execute + schedule) | Service dispatches jobs; actions dispatch domain events only |
| No DB transaction | Partial writes on failure | Wrap in `DB::transaction()` |
| Querying another context's model directly | Hard coupling across bounded contexts | Use integration patterns |
| HTTP concerns in action | Not reusable from non-HTTP layers | Keep logout, session, redirect in controller |
| Subfolders inside Actions/ | Unnecessary complexity | Flat folder; IDEs handle 100+ files |
| Controller orchestrating actions | Business logic in HTTP layer | Extract to Service |
