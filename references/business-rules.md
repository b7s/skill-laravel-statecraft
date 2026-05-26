# Business Rules Coding Standards

These standards are derived from project AGENTS.md and apply specifically to business logic implemented with the state machine + workflow engine pattern.

## Strict Typing

Every PHP file MUST begin with `declare(strict_types=1);`.

```php
<?php
declare(strict_types=1);

namespace App\Orders\Domain;
```

Every parameter, property, constant, and return type MUST have an explicit type declaration.

```php
// Bad — no types
private $status;

// Good — explicit types
private OrderStatus $status;
```

```php
// Bad — untyped constants
const MAX_RETRIES = 3;

// Good — typed constants (PHP 8.3+)
const int MAX_RETRIES = 3;
```

## Enums Over Strings

Replace every status, type discriminator, and fixed-option value with a typed Enum.

```php
// Bad — stringly typed
class Payment
{
    public function __construct(public string $status) {}
}

// Good — enum
enum PaymentStatus: string
{
    case Pending = 'pending';
    case Completed = 'completed';
    case Failed = 'failed';
}

class Payment
{
    public function __construct(public PaymentStatus $status) {}
}
```

Enum methods keep logic close to data:

```php
enum OrderStatus: string
{
    case Draft = 'draft';
    case Submitted = 'submitted';
    case Approved = 'approved';
    case Rejected = 'rejected';
    case Fulfilled = 'fulfilled';
    case Cancelled = 'cancelled';

    public function label(): string
    {
        return match ($this) {
            self::Draft => 'Draft',
            self::Submitted => 'Submitted',
            self::Approved => 'Approved',
            self::Rejected => 'Rejected',
            self::Fulfilled => 'Fulfilled',
            self::Cancelled => 'Cancelled',
        };
    }

    public function isFinal(): bool
    {
        return in_array($this, [self::Fulfilled, self::Rejected, self::Cancelled], true);
    }

    public static function options(): array
    {
        return array_reduce(
            self::cases(),
            static fn (array $carry, self $status): array => $carry + [$status->value => $status->label()],
            [],
        );
    }
}
```

## Readonly Classes and Properties

When all constructor-promoted properties are immutable, make the **entire class readonly**.

```php
// Bad — redundant readonly on every parameter
class UserData
{
    public function __construct(
        private readonly string $name,
        private readonly string $email,
    ) {}
}

// Good — class-level readonly
readonly class UserData
{
    public function __construct(
        private string $name,
        private string $email,
    ) {}
}
```

**When to use `readonly class`:**
- All properties initialized via constructor promotion.
- No property needs to be mutable after construction.
- The class has no dynamic properties.

**When NOT to use `readonly class`:**
- Any property needs to be mutable (e.g., counters, state tracking).
- The class extends a non-readonly parent (PHP limitation).

Domain events, DTOs, and context objects should almost always be `readonly`.

## Constructor Property Promotion

Use PHP 8+ constructor property promotion to eliminate boilerplate.

```php
// Bad — unnecessary repetition
class ImportService
{
    private CsvParser $parser;
    private LoggerInterface $logger;

    public function __construct(CsvParser $parser, LoggerInterface $logger)
    {
        $this->parser = $parser;
        $this->logger = $logger;
    }
}

// Good — concise
readonly class ImportService
{
    public function __construct(
        private CsvParser $parser,
        private LoggerInterface $logger,
    ) {}
}
```

## Empty Constructors

Do not allow empty `__construct()` methods with zero parameters unless the constructor is private (e.g., for singletons or static factories).

```php
// Bad — pointless
class Calculator
{
    public function __construct() {}

    public static function add(int $a, int $b): int
    {
        return $a + $b;
    }
}

// Good — no constructor needed
class Calculator
{
    public static function add(int $a, int $b): int
    {
        return $a + $b;
    }
}
```

## Guard Clauses and Fail Fast

Every method should validate its preconditions first, then proceed with the happy path at the top level of indentation.

```php
// Bad — deeply nested, hides the happy path
public function processOrder(OrderData $data): OrderResult
{
    if ($data->items !== []) {
        $customer = $this->customerRepo->find($data->customerId);
        if ($customer !== null) {
            if ($customer->isActive()) {
                // happy path buried here
            }
            return new OrderResult(success: false, error: 'Customer inactive');
        }
        return new OrderResult(success: false, error: 'Customer not found');
    }
    return new OrderResult(success: false, error: 'No items');
}

// Good — guard clauses, flat happy path
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

    // happy path at top level
}
```

**When to throw vs return:**
- **Throw** for programmer errors and invariant violations (invalid arguments, impossible states).
- **Return** for expected business failures (customer not found, payment declined).
- **Skip** for optional operations that can be safely bypassed.

## Result Objects

Services should return structured result objects, not echo output or raw arrays.

```php
<?php
declare(strict_types=1);

namespace App\Orders\Application;

readonly class PaymentResult
{
    public function __construct(
        public bool $success,
        public string $transactionId,
        public ?string $errorMessage = null,
    ) {}

    public function isSuccess(): bool
    {
        return $this->success;
    }

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

**Benefits:**
- Multiple output formats (JSON, HTML, CLI) from the same result.
- Testable assertions on result data.
- Composable — results can be merged, filtered, transformed.
- Type-safe — the class enforces the structure.

## Separation of Concerns

### Formatting Is Not Business Logic

Formatters are pure functions that take a result object and return a string.

```php
class JsonFormatter
{
    public function format(ResultInterface $result): string
    {
        return json_encode($result->toArray(), JSON_THROW_ON_ERROR);
    }
}
```

### Validation Is Not Business Logic

Extract validation rules into dedicated classes:
- Form Requests (Laravel) — `CreateUserRequest`
- DTOs with validation — `UserData::fromArray($input)`
- Standalone validators — `EmailValidator::assert($email)`

### Side Effects Should Be Explicit

If a method sends emails, writes files, or makes API calls, make it obvious:

```php
// Good — name reveals the side effect
public function createAndNotify(UserData $data): User;

// Bad — hidden side effect
public function create(UserData $data): User; // also sends email?
```

### Directory Creation Must Be Validated

Never create a directory without checking the result. `mkdir()` can fail silently due to permissions, race conditions, or disk issues. Always validate and throw explicitly:

```php
// Bad — ignores mkdir failure
if (! is_dir($libDir)) {
    mkdir($libDir, 0755, true);
}

// Good — validates the result and fails fast
if (!is_dir($libDir) && !mkdir($libDir, 0755, true) && !is_dir($libDir)) {
    throw new \RuntimeException(sprintf('Directory "%s" was not created', $libDir));
}
```

## Dependency Injection

### Prefer Constructor Injection

Services should receive dependencies via constructor, not instantiate them inside methods.

```php
class ReportGenerator
{
    public function __construct(
        private readonly PdfRenderer $pdf,
        private readonly StorageInterface $storage,
    ) {}
}
```

### Avoid Service Locator Pattern

Do not pass around a "bag of services" or a container.

```php
// Bad — unclear dependencies
class InvoiceService
{
    public function __construct(private Container $container) {}

    public function generate(): void
    {
        $pdf = $this->container->get(PdfRenderer::class); // hidden dependency
    }
}

// Good — explicit dependencies
class InvoiceService
{
    public function __construct(private PdfRenderer $pdf) {}
}
```

## Control Structures

Always use curly braces for control structures, even for single-line bodies.

```php
// Good
if ($value === null) {
    return;
}

// Bad — error-prone
if ($value === null)
    return;
```

## Lambdas and Closures

Lambdas not using `$this` should be `static`.

```php
$filtered = array_filter(
    $items,
    static fn (Item $item): bool => $item->isActive(),
);
```

## Import Classes and Namespaces

Always import classes with `use` statements. Never use fully qualified class names inline.

```php
// Bad
class UserImporter
{
    public function import(): \App\Models\User
    {
        $validator = new \App\Validators\EmailValidator();
    }
}

// Good
use App\Models\User;
use App\Validators\EmailValidator;

class UserImporter
{
    public function import(): User
    {
        $validator = new EmailValidator();
    }
}
```

## Consistent Naming

| Pattern | Example |
|---|---|
| Commands | `*Command` suffix |
| Controllers | `*Controller` suffix |
| Services | `*Service` suffix |
| Repositories | `*Repository` suffix |
| Formatters | `*Formatter` suffix |
| Interfaces | `*Contract` suffix (or `*Interface`) |
| Traits | Descriptive noun, no suffix |
| Enums | TitleCase keys: `Active`, `Pending` |
| Value Objects | Noun describing the concept: `OrderId`, `Money` |
| Domain Events | Past-tense noun phrase: `OrderSubmitted`, `PaymentReceived` |

## Explicit Types for Local Variables

PHP does not support typed local variables. When a variable's type cannot be inferred by static analysis, annotate it with PHPDoc:

```php
/** @var array<int, User> $users */
$users = $repository->findActive();

/** @var ?string $cachedValue */
$cachedValue = $cache->get('key');
```

## Modern String Functions

Prefer PHP 8+ string functions:

```php
// Good
if (str_contains($haystack, $needle)) { }
if (str_starts_with($path, '/')) { }
if (str_ends_with($file, '.php')) { }

// Avoid
if (strpos($haystack, $needle) !== false) { }
```

## Boolean Condition Ordering

In `&&` and `||` expressions, cheaper conditions should come first so PHP can short-circuit.

```php
// Good — guard first
if (isset($array['key']) && is_array($array['key'])) { }

// Bad — expensive check before guard
if (is_array($array['key']) && isset($array['key'])) { }
```

**Cost model:**
| Cost | Expression types |
|---|---|
| 0 | Variables, literals, guards (`isset`, `empty`, `is_array`) |
| 1 | Property/array access, comparisons, cheap functions (`count`, `strlen`) |
| 2 | Default (casts, closures, non-guard func calls) |
| 3 | Method calls, static calls, `new`, assignments |

## PHPDoc for Model Properties

Every model/entity class MUST declare all its properties in a class-level PHPDoc block with correct types.

```php
<?php
declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Carbon\Carbon;

/**
 * @property int $id
 * @property string $name
 * @property string $email
 * @property Carbon $created_at
 * @property Carbon $updated_at
 */
class User extends Model
{
    protected $fillable = ['name', 'email'];
}
```

**Rules:**
- Include `@property` for every database column, including timestamps.
- Use nullable types when a column can be null: `@property ?string $deleted_at`.
- Use union types for columns with multiple possible types: `@property int|string $status`.
- Use full FQCN for non-scalar types: `@property \Carbon\Carbon $created_at`.
- For relationships, document the return type on the method itself, not as a property.

## Thin Layers Pattern

Every layer in your application should be thin:

| Layer | Responsibility | Max Size |
|---|---|---|
| Controllers / Commands | Accept input, delegate, return output | ≤ 100 lines |
| Application Services | Orchestrate business operations | ≤ 250 lines |
| Domain Entities | Enforce invariants, emit events | ≤ 200 lines |
| Repositories / Query Builders | Abstract data access | ≤ 300 lines |
| Value Objects / DTOs | Encapsulate data with validation | ≤ 100 lines |
| Workflow Steps | Perform one discrete action | ≤ 100 lines |

When any layer grows beyond its limit, extract a new class.

## Service Extraction Signals

Extract a service when you encounter any of these smells:
1. A method does not use `$this` (can be static / standalone).
2. The same logic exists in 2+ locations.
3. A class exceeds 250 lines.
4. A method has more than 3 levels of indentation.
5. Mixed levels of abstraction in one method.

## Interface Segregation

Services that are swappable should implement an interface:

```php
interface PaymentProcessorContract
{
    public function charge(Customer $customer, Money $amount): PaymentResult;
    public function refund(string $transactionId): RefundResult;
}
```

This enables:
- Polymorphic usage (`$gateway->charge(...)`).
- Easy testing with mocks.
- Future extensibility (e.g., switching from Stripe to PayPal).

## Typed Exceptions

Use specific exception types rather than generic `\Exception`:

```php
class ValidationException extends \RuntimeException {}
class NotFoundException extends \RuntimeException {}
class PaymentFailedException extends \RuntimeException {}
class InvalidTransitionException extends \DomainException {}
```

This allows callers to catch exactly what they expect:

```php
try {
    $order->approve('admin');
} catch (InvalidTransitionException $e) {
    // handle business rule violation
} catch (NotFoundException $e) {
    // handle missing entity
}
```

## Match Expressions Over If/Else Chains

When comparing an enum or string against multiple fixed values, prefer `match`:

```php
$label = match ($status) {
    PaymentStatus::Pending => 'Awaiting payment',
    PaymentStatus::Completed => 'Payment received',
    PaymentStatus::Failed => 'Payment failed',
};
```

`match` is exhaustive for enums in PHP 8.1+ — adding a new case without updating the `match` is a compile-time error.

## Performance: Import Native Functions

Always import native PHP functions with `use function` to bypass namespace resolution overhead:

```php
<?php
declare(strict_types=1);

namespace App\Service;

use function array_filter;
use function array_map;
use function count;
use function sprintf;
use function str_contains;

class ImportService
{
    public function process(array $lines): array
    {
        return array_filter(
            array_map(static fn (string $line): string => trim($line), $lines),
            static fn (string $line): bool => str_contains($line, '@'),
        );
    }
}
```

## Performance: Static Methods and Closures

When a method does not use `$this`, declare it `static`:

```php
public static function normalize(string $input): string
{
    return strtolower(trim($input));
}
```

Non-static closures capture `$this` by default, creating a reference cycle and preventing garbage collection.

## Config via Typed Helpers

Use Laravel's typed repository helpers to read config with correct types:

```php
// Wrong — returns mixed, may be string "0.38" depending on cache state
$threshold = config('assistant.similarity_threshold');

// Right — always returns float
$threshold = config()->float('assistant.similarity_threshold');
```

Available helpers: `config()->float()`, `config()->integer()`, `config()->bool()`, `config()->string()`.
