# State Machine Pattern Reference

## Overview

The state machine layer owns the truth about which transitions are legal for an entity. It is deliberately narrow: no database access, no email sending, no event dispatching. It knows states, validates transitions, and emits domain events.

## When to Use

Use a state machine when:
- An entity has 5 or more possible statuses.
- Transitions form a non-trivial graph (not just a linear sequence).
- Different transitions require different side effects.
- Invalid transitions must be prevented at the domain level, not caught by database constraints.
- You need an audit trail of who changed what and when.

Do NOT use a state machine when:
- There are only 2-3 statuses and simple linear progression.
- No business rules depend on the current status.
- The "state" is purely presentational (e.g., a computed CSS class).

## Design Rules

### 1. Status Must Be an Enum

Never use strings, integers, or booleans for status. Always use a typed Enum.

```php
<?php
declare(strict_types=1);

namespace App\Orders\Domain;

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
}
```

**Why:**
- Type safety prevents invalid states.
- `match` expressions enforce exhaustive handling.
- IDE autocompletion eliminates typos.
- Refactoring renames updates all references automatically.

### 2. Entity Owns All Transitions

Every status change is a named method on the entity. The method validates, changes state, and returns a domain event.

```php
final class Order
{
    private OrderStatus $status;

    public function __construct(
        private readonly OrderId $id,
    ) {
        $this->status = OrderStatus::Draft;
    }

    public function submit(): OrderSubmitted
    {
        if ($this->status !== OrderStatus::Draft) {
            throw new InvalidTransitionException(
                sprintf('Cannot submit order from status "%s"', $this->status->value)
            );
        }

        $this->status = OrderStatus::Submitted;

        return new OrderSubmitted(
            orderId: $this->id,
            submittedAt: new \DateTimeImmutable(),
        );
    }
}
```

**Forbidden patterns:**
```php
// BAD: Status checked in controller
if ($order->status === 'pending') { $order->status = 'approved'; }

// BAD: Status changed directly
$order->status = OrderStatus::Approved;

// BAD: No event returned
$order->approve(); // returns void

// BAD: Side effect inside entity
$order->approve(); // sends email inside
```

### 3. Use Typed Exceptions

Every invalid transition throws a specific, typed exception. Never throw generic `\Exception` or `\RuntimeException`.

```php
<?php
declare(strict_types=1);

namespace App\Orders\Domain\Exceptions;

final class InvalidTransitionException extends \DomainException
{
    public function __construct(string $message)
    {
        parent::__construct($message);
    }
}
```

**Exception hierarchy suggestion:**
```
\DomainException
  ├── InvalidTransitionException
  ├── InvalidStateException
  └── BusinessRuleViolationException
```

### 4. Domain Events Are Value Objects

Events are immutable value objects with named constructor promotion.

```php
<?php
declare(strict_types=1);

namespace App\Orders\Domain\Events;

final readonly class OrderSubmitted
{
    public function __construct(
        public OrderId $orderId,
        public \DateTimeImmutable $submittedAt,
    ) {}
}
```

**Rules for events:**
- Use `readonly class` or `readonly` properties.
- Include the entity ID and a timestamp at minimum.
- Include only data needed by listeners/workflow steps.
- Never include infrastructure objects (Request, Response, Eloquent models).

### 5. Entity Has No Infrastructure Dependencies

The entity class must not:
- Extend Eloquent `Model`.
- Use `Mail::`, `Event::`, `Cache::`, or `DB::` facades.
- Accept repository interfaces in its constructor.
- Call external services.

**Allowed dependencies:**
- Other value objects (OrderId, Money, EmailAddress).
- PHP built-in types and standard library classes (`DateTimeImmutable`, `Stringable`).

### 6. Application Service Bridges Domain and Infrastructure

The use case / application service receives the event, persists the entity, and returns the event for the caller to dispatch.

```php
<?php
declare(strict_types=1);

namespace App\Orders\Application;

use App\Orders\Domain\Events\OrderApproved;
use App\Orders\Domain\OrderRepositoryContract;

final readonly class ApproveOrder
{
    public function __construct(
        private OrderRepositoryContract $repository,
    ) {}

    public function execute(string $orderId, string $approvedBy): OrderApproved
    {
        $order = $this->repository->findById(OrderId::fromString($orderId));
        $event = $order->approve($approvedBy);

        $this->repository->save($order);

        return $event;
    }
}
```

**Rules:**
- The service is `readonly` if it holds no mutable state.
- Return the event, not the entity.
- The service does NOT dispatch the event. The controller/command does.

### 7. Testing Strategy

State machines are tested as plain PHP classes. No Laravel bootstrapping required.

```php
<?php
declare(strict_types=1);

use App\Orders\Domain\Order;
use App\Orders\Domain\OrderId;
use App\Orders\Domain\OrderStatus;
use App\Orders\Domain\Exceptions\InvalidTransitionException;

it('starts as draft', function () {
    $order = new Order(OrderId::generate());

    expect($order->status)->toBe(OrderStatus::Draft);
});

it('can be submitted from draft', function () {
    $order = new Order(OrderId::generate());

    $event = $order->submit();

    expect($order->status)->toBe(OrderStatus::Submitted);
    expect($event->orderId)->toBe($order->id);
});

it('cannot be approved from draft', function () {
    $order = new Order(OrderId::generate());

    $order->approve('admin');
})->throws(InvalidTransitionException::class);

it('cannot be submitted twice', function () {
    $order = new Order(OrderId::generate());
    $order->submit();

    $order->submit();
})->throws(InvalidTransitionException::class);
```

**Coverage goals:**
- Every valid transition tested once.
- Every invalid transition tested once.
- Every exception message asserted where it carries meaningful data.

## Common Pitfalls

| Pitfall | Why It Hurts | Solution |
|---|---|---|
| String status columns | Typos pass silently; no IDE support | Enum |
| Status checks in controllers | Business rules leak to HTTP layer | Move to entity |
| `save()` returns void | No trace of what happened | Return domain event |
| Side effects in entity | Untestable; hidden dependencies | Extract to workflow step |
| Generic exceptions | Callers cannot distinguish failures | Typed exceptions |
| Mutable events | Listeners may accidentally modify state | `readonly class` |

## Directory Structure

```
src/Orders/
  Domain/
    Order.php                    # Entity + state machine
    OrderStatus.php               # Enum
    OrderId.php                   # Value object
    Events/
      OrderSubmitted.php
      OrderApproved.php
      OrderRejected.php
    Exceptions/
      InvalidTransitionException.php
  Application/
    ApproveOrder.php              # Use case
    SubmitOrder.php
    Contracts/
      OrderRepositoryContract.php
```
