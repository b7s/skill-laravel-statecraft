# PHP/Laravel Rules and Best Practices

## Table of Contents
1. [Design Principles](#design-principles)
2. [Type Safety](#type-safety)
3. [Code Quality Standards](#code-quality-standards)
4. [Performance Optimization](#performance-optimization)
5. [Error Handling](#error-handling)
6. [Laravel-Specific Rules](#laravel-specific-rules)
7. [FilamentPHP Rules](#filamentphp-rules)
8. [Quality Gate Checklist](#quality-gate-checklist)

---

## Design Principles

### Single Responsibility Principle (SRP)
Every class should have one reason to change:

- **Controllers** — Request/response format changes
- **Actions** — One specific operation changes
- **Services** — Calculation/validation logic changes  
- **Formatters** — Output format changes
- **Models** — Entity structure changes
- **Enums** — New states are added

**Example:**
```php
// Bad — mixed concerns
class UserController
{
    public function store(Request $request): Response
    {
        $data = $request->validate([...]); // Validation
        $user = new User;                   // Business logic
        $user->save();
        Mail::to($user)->send(...);         // Side effect
        return response()->json([...]);     // Formatting
    }
}

// Good — delegated concerns
class UserController
{
    public function __construct(
        private readonly CreateUser $createUser,
    ) {}

    public function store(CreateUserRequest $request): UserResource
    {
        $user = $this->createUser->execute($request->validated());
        return new UserResource($user);
    }
}
```

### Thin Layers Pattern
When any layer grows beyond its limit, extract a new class:

| Layer | Max Lines | Responsibility |
|---|---|---|
| Controllers / Commands | 1000 | Accept input, delegate, return output |
| Actions | 250 | One specific operation with DB transaction |
| Services | 250 | Calculation, validation, external integration |
| Formatters | 100 | Pure output rendering |

### DRY (Don't Repeat Yourself)
Identify duplication via `b7s/catraca`. Common duplications:
- Query conditions
- Validation rules
- Error message formatting
- Array transformations

**Extract to:** Services, Traits, Helper methods, or Actions

### Separation of Concerns

**Formatting is NOT business logic** — Pure functions, no I/O

**Validation is NOT business logic** — Extract to FormRequests, DTOs, or Validators

**Side effects must be explicit:**
```php
// Good — explicit
public function createAndNotify(UserData $data): User;

// Bad — hidden side effect
public function create(UserData $data): User; // also sends email?
```

### Dependency Injection

**Prefer constructor injection:**
```php
class ReportGenerator
{
    public function __construct(
        private readonly PdfRenderer $pdf,
        private readonly StorageInterface $storage,
    ) {}
}
```

**Avoid Service Locator Pattern** — No "bag of services"

### Result Objects Pattern
Always return structured results, never echo output:

```php
readonly class PaymentResult
{
    public function __construct(
        public bool $success,
        public string $transactionId,
        public ?string $errorMessage = null,
    ) {}

    public function isSuccess(): bool { return $this->success; }
    
    public function toArray(): array
    {
        return [
            'success' => $this->success,
            'transaction_id' => $this->transactionId,
            'error' => $this->errorMessage,
        ];
    }
}
```

---

## Type Safety

### Strict Typing (Mandatory)
```php
<?php
declare(strict_types=1);
```

### Explicit Types Everywhere
Every constant, property, parameter, and return type must be explicitly typed.

**Typed constants (PHP 8.3+):**
```php
const int MAX_RETRIES = 3;
const string API_VERSION = 'v2';
```

**Typed properties:**
```php
class UserImporter
{
    private LoggerInterface $logger;
    private int $timeout = 30;
    private ?string $apiKey = null;
}
```

**Typed parameters and returns:**
```php
public function process(array $input, ImportOptions $options): ImportResult
{
    // ...
}
```

**PHPDoc for ambiguous local variables:**
```php
/** @var array<int, User> $users */
$users = $repository->findActive();
```

**PHPDoc for Model Properties:**
```php
/**
 * @property int $id
 * @property string $name
 * @property string $email
 * @property \Carbon\Carbon $created_at
 * @property \Carbon\Carbon $updated_at
 */
class User extends Model
{
    protected $fillable = ['name', 'email'];
}
```

**For relationships:**
```php
use Illuminate\Database\Eloquent\Relations\BelongsTo;

/**
 * @return BelongsTo<Company, $this>
 */
public function company(): BelongsTo
{
    return $this->belongsTo(Company::class);
}
```

### Use Enums, Never Magic Strings

**Bad — stringly typed:**
```php
class Payment
{
    public function __construct(public string $status) {}
}
```

**Good — type-safe:**
```php
enum PaymentStatus: string
{
    case Pending = 'pending';
    case Completed = 'completed';
    case Failed = 'failed';
    
    public function label(): string
    {
        return match ($this) {
            self::Pending => 'Awaiting payment',
            self::Completed => 'Payment received',
            self::Failed => 'Payment failed',
        };
    }
    
    public static function default(): self
    {
        return self::Pending;
    }
    
    public static function options(): array
    {
        return array_reduce(
            self::cases(),
            static fn (array $carry, self $status): array => 
                $carry + [$status->value => $status->label()],
            [],
        );
    }
    
    public static function values(): array
    {
        return array_column(self::cases(), 'value');
    }
}
```

**Enum rules:**
- Always use backed enums (`string` or `int`)
- Always add `default()` method
- Add `label()`, `options()`, `values()` for UI/forms
- Add domain methods (`isFinal()`, `isPending()`)
- Extract shared methods to trait when 3+ enums exist

**When to use Enums:**
- Status codes
- Type discriminators
- Feature flags
- Fixed configuration options

**When NOT to use:**
- Free-text user input
- Runtime-configurable values

---

## Code Quality Standards

### Constructor Property Promotion
```php
readonly class ImportService
{
    public function __construct(
        private CsvParser $parser,
        private LoggerInterface $logger,
    ) {}
}
```

### Readonly Classes
When **all** constructor properties are `readonly`, make the **entire class readonly**:

```php
readonly class UserData
{
    public function __construct(
        private string $name,
        private string $email,
        private DateTimeImmutable $createdAt,
    ) {}
}
```

### No Empty Constructors
Never allow empty `__construct()` unless private (singletons/factories).

### Control Structures
Always use curly braces:
```php
// Good
if ($value === null) {
    return;
}

// Bad
if ($value === null)
    return;
```

### Static Closures
```php
$filtered = array_filter(
    $items,
    static fn (Item $item): bool => $item->isActive(),
);
```

### Import Classes, Not FQCN
```php
// Good
use App\Models\User;
use App\Services\TaxCalculator;

class ImportService
{
    public function process(): User { /* ... */ }
}

// Bad
class ImportService
{
    public function process(): \App\Models\User { /* ... */ }
}
```

### Consistent Naming

| Pattern | Example |
|---|---|
| Actions | `CreateInvoice`, `MarkInvoicePaid` |
| Services | `TaxCalculator`, `PricingEngine` |
| Controllers | `InvoiceController` |
| Jobs | `SendReceiptEmail`, `ProcessImport` |
| Events | `InvoicePaid`, `UserCreated` |
| Exceptions | `InvalidTransitionException` |
| Interfaces | `PaymentProcessorInterface` |
| Enums | `OrderStatus`, `PaymentStatus` |

### Service Extraction Rules

**Extract when:**
1. Method doesn't use `$this`
2. Same logic in 2+ locations
3. Class exceeds 250 lines
4. Method has 3+ levels of indentation
5. Mixed abstraction levels

**Service categories:**
- **Calculators** — `TaxCalculator`, `PricingEngine`
- **Validators** — `SchemaValidator`, `FileValidator`
- **Formatters** — `JsonFormatter`, `PdfRenderer`
- **Gateways** — `StripePaymentGateway`, `SendGridEmailService`

---

## Performance Optimization

### Import Native Functions
```php
use function array_filter;
use function array_map;
use function count;
use function str_contains;
use function trim;
```

**Why:** Bypasses namespace resolution overhead

### Prefer Static Methods
When method doesn't use `$this`:
```php
public static function normalize(string $input): string
{
    return strtolower(trim($input));
}
```

### Use Strict Comparison
```php
if ($value === null) { /* ... */ }
if ($status === Status::Active) { /* ... */ }
```

### Modern String Functions
```php
// Good
if (str_contains($haystack, $needle)) {}
if (str_starts_with($path, '/')) {}
if (str_ends_with($file, '.php')) {}

// Avoid
if (strpos($haystack, $needle) !== false) {}
```

### Avoid `count()` in Loops
```php
// Best
foreach ($items as $item) { /* ... */ }

// Good
$count = count($items);
for ($i = 0; $i < $count; $i++) { /* ... */ }

// Bad
for ($i = 0; $i < count($items); $i++) { /* ... */ }
```

### Use Generators for Large Datasets
```php
public function readLines(string $path): \Generator
{
    $handle = fopen($path, 'r');
    while ($line = fgets($handle)) {
        yield trim($line);
    }
    fclose($handle);
}
```

### Boolean Condition Ordering
Order by cost (cheapest first):

| Cost | Examples |
|---|---|
| **0** | `$x`, `true`, `isset($x)`, `empty($y)`, `is_array($z)` |
| **1** | `$obj->prop`, `$arr['key']`, `$a === $b` |
| **2** | `(int) $x`, `fn() => true` |
| **3** | `$this->method()`, `new Entity()` |

**Rules:**
1. Guards first (`isset`, `empty`, `is_*`)
2. Property/array access after guards
3. Never reorder side effects

```php
// Good
if (isset($array['key']) && is_array($array['key'])) {}

// Bad
if (is_array($array['key']) && isset($array['key'])) {}
```

---

## Error Handling

### Fail Fast with Guard Clauses
Validate preconditions first, keep happy path at top level:

```php
public function processOrder(OrderData $data): OrderResult
{
    if ($data->items === []) {
        return new OrderResult(success: false, error: 'No items');
    }

    $customer = $this->customerRepo->find($data->customerId);
    if ($customer === null) {
        return new OrderResult(success: false, error: 'Customer not found');
    }

    if (!$customer->isActive()) {
        return new OrderResult(success: false, error: 'Customer inactive');
    }

    // Happy path at top level
    $total = $this->calculator->calculate($data->items);
    $payment = $this->payment->charge($customer, $total);
    
    if (!$payment->isSuccess()) {
        return new OrderResult(success: false, error: 'Payment failed');
    }

    return new OrderResult(success: true, orderId: $payment->transactionId);
}
```

**Benefits:** Flat indentation, readable, early exit, easier debugging

### When to Throw vs Return

- **Throw:** Programmer errors, invariant violations
- **Return:** Expected business failures
- **Skip:** Optional operations

### Typed Exceptions
```php
class ValidationException extends \RuntimeException {}
class NotFoundException extends \RuntimeException {}
class InvalidTransitionException extends \DomainException {}
```

### Validate Directory Creation
```php
// Good
if (!is_dir($dir) && !mkdir($dir, 0755, true) && !is_dir($dir)) {
    throw new \RuntimeException(sprintf('Directory "%s" was not created', $dir));
}
```

### Graceful Degradation
```php
$tool = $resolver->resolve('optional-tool');
if ($tool === null) {
    return new TaskResult(skipped: true, message: 'Missing optional-tool');
}
```

---

## Laravel-Specific Rules

### Commands
- Place in `app/Console/Commands/`
- Use `php artisan make:command`
- Pass `--no-interaction` for non-interactive execution
- Max 60 lines (thin orchestrator)

### Controllers
- Keep thin, delegate to Actions
- Max 1000 lines
- No business logic

### Models
- Use `casts()` method over `$casts` property (Laravel 12+)
- Always create factories and seeders
- Always use `Model::query()->where()` not `Model::where()`
- Add PHPDoc for all properties (see Type Safety section)

### Database
- Avoid `DB::`; prefer `Model::query()`
- Prevent N+1 with eager loading
- When modifying columns, include ALL previous attributes
- Laravel 12+ native limit: `$query->latest()->limit(10)`

### Relationships
- Always use proper Eloquent methods
- Always add return type hints
- Prefer relationship methods over raw queries

### Routing
- Prefer named routes
- Use `route()` function, never hardcode URLs

### Queue
- Use `ShouldQueue` for time-consuming operations
- Add retry logic with `$tries` and `backoff()`

### Defer Function (Use with Great Care)

**When to use:** Non-critical operations after HTTP response
- Logging, analytics
- Cache warming
- Cleanup operations

**When NOT to use:**
- Database transactions (use regular code)
- Critical operations (use jobs)
- Operations needing retry (use jobs)

**Limitations:**
- Only HTTP/console contexts (not jobs/queues)
- Same process (can still slow worker)
- No retry mechanism

**Pattern:**
```php
public function store(CreateUserRequest $request): UserResource
{
    $user = $this->createUser->execute($request->validated());
    
    // Place at END to signal "after response"
    defer(fn () => Log::info('User created', ['id' => $user->id]));
    
    return new UserResource($user);
}

// Bad — use job instead
defer(fn () => $this->sendEmail($user)); // Needs retry!
```

### Code Style
```bash
vendor/bin/pint --dirty --format agent
```

### Tests (Pest Required)
```bash
php artisan make:test --pest {name}
php artisan test --compact
php artisan test --compact --filter=testName
```

**Rules:**
- Test happy paths, failure paths, edge cases
- Use datasets to reduce duplication
- Browser tests in `tests/Browser/`
- Never delete tests without approval
- Always use test database

---

## FilamentPHP Rules

### Commands
Always use Filament Artisan commands: `php artisan --help`

### Patterns
- Use static `make()` methods
- `Get $get` to read form field values
- `Set $set` inside `->afterStateUpdated()` on `->live()` fields
- Prefer `->live(onBlur: true)` on text inputs
- `Repeater` for `HasMany` with `->relationship()`
- **Never** use `->dehydrated(false)` on fields that need saving
- **Never** assume public file visibility — use `->visibility('public')`

### Correct Property Types
- `$navigationIcon`: `protected static string | BackedEnum | null`
- `$navigationGroup`: `protected static string | UnitEnum | null`
- `$view`: `protected string` (not static) on `Page`/`Widget`

### Correct Namespaces
- Form fields: `Filament\Forms\Components\`
- Infolist entries: `Filament\Infolists\Components\`
- Layout: `Filament\Schemas\Components\`
- Schema utilities: `Filament\Schemas\Components\Utilities\`
- Table columns: `Filament\Tables\Columns\`
- Table filters: `Filament\Tables\Filters\`
- Actions: `Filament\Actions\` (never sub-namespaces)
- Icons: `Filament\Support\Icons\Heroicon` enum

---

## Quality Gate Checklist

**After every feature/change, run in order:**

```bash
./vendor/bin/pint                    # 1. Fix code style
./vendor/bin/pest --parallel         # 2. Run tests
./vendor/bin/phpstan analyse         # 3. Static analysis (level 6)
./vendor/bin/catraca                 # 4. Quality metrics
```

**All must pass before work is complete.**

---

## General Rules

- Never modify unrelated files
- Prefer PHPDoc blocks over inline comments
- Every change requires tests
- Run minimum tests needed for speed
- Always add return types
- Omit parentheses around `new`: `new Product()->getPrice()` not `(new Product())->getPrice()`
