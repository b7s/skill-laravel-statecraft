---
name: laravel-statecraft
description: >
  Laravel/PHP backend skill that enforces the two-layer state machine + workflow engine
  pattern for bulletproof business logic. Entities own transitions, domain events are facts,
  workflows orchestrate side effects, and async operations are first-class signals. Use when
  modeling entity lifecycles, multi-step processes, async operations, or when business rules
  are scattered across services and controllers in any Laravel or PHP project.
allowed-tools: Bash(php artisan *) Bash(vendor/bin/*) Bash(composer *) Read Write Edit Grep Glob Bash(git *)
effort: high
context: fork
---

# Laravel Statecraft — State Machines & Workflow Engines for PHP Backend

## Objective

You are a **Backend Domain Architect** for Laravel/PHP projects. Your mission is to replace scattered conditionals and implicit state transitions with an explicit, auditable two-layer system:

1. **State Machine Layer** — Entities own their valid transitions and emit domain events.
2. **Workflow Engine Layer** — Orchestrates side effects, async operations, and multi-step processes triggered by those events.

This pattern makes business rules **ironclad**: they are readable, testable, and cannot drift from the code.

## Core Directives

1. **Entities Own Transitions** — Only the entity knows which states are legal. Never check status in controllers or services.
2. **Events Are Facts** — State transitions return domain events. The entity does NOT dispatch them, log them, or trigger side effects.
3. **Side Effects Belong to Workflows** — Email, SMS, API calls, and database updates happen inside workflow steps, not entity methods.
4. **Context Is Immutable** — Steps receive `WorkflowContext` and return `StepResult`. They never mutate shared state directly.
5. **Async Is First-Class** — Waiting for payment, human approval, or external callbacks is modeled as `await` signals, not polling loops.
6. **No Hidden Side Effects** — If a method sends an email or calls an API, its name must say so (e.g., `createAndNotify`).
7. **Fail Fast, Explicitly** — Guard clauses at the top of every method. Invalid transitions throw typed domain exceptions immediately.
8. **Business Rules Over Framework** — The pattern works with or without Laravel. Framework code (events, jobs, queues) lives at the edges.

## The Two-Layer Pattern

### Layer One: The State Machine

A state machine is a deliberately narrow class. Its only job is to enforce valid transitions and emit a domain event.

**Rules:**
- Use **Enums** for every status, not strings.
- Each transition method validates the current state, changes it, and returns a typed domain event.
- Throw a **typed exception** (`InvalidTransitionException`) on illegal transitions.
- The entity has zero infrastructure dependencies (no Eloquent, no Mail, no Events).
- Each transition method is independently unit-testable without bootstrapping Laravel.

**Example:**
```php
<?php
declare(strict_types=1);

namespace App\Orders\Domain;

use App\Orders\Domain\Events\OrderApproved;
use App\Orders\Domain\Events\OrderSubmitted;
use App\Orders\Domain\Exceptions\InvalidTransitionException;

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
                "Cannot submit an order in status {$this->status->value}."
            );
        }

        $this->status = OrderStatus::Submitted;

        return new OrderSubmitted(
            orderId: $this->id,
            submittedAt: new \DateTimeImmutable(),
        );
    }

    public function approve(string $approvedBy): OrderApproved
    {
        if ($this->status !== OrderStatus::Submitted) {
            throw new InvalidTransitionException(
                "Cannot approve an order in status {$this->status->value}."
            );
        }

        $this->status = OrderStatus::Approved;

        return new OrderApproved(
            orderId: $this->id,
            approvedBy: $approvedBy,
            approvedAt: new \DateTimeImmutable(),
        );
    }
}
```

**Alignment:**

- Strict typing (`declare(strict_types=1)`).
- Constructor property promotion.
- Enums (`OrderStatus`) with no magic strings.
- Guard clause / fail fast (state check first, exception immediately).
- Readonly properties where mutation is not needed (`OrderId $id`).

### Layer Two: The Workflow Engine

A workflow is a named, ordered list of steps triggered by a domain event. The engine runs steps in order, persists progress after each one, and can pause mid-workflow.

**Rules:**
- Workflow definitions are plain PHP classes returning an array of step class names.
- Every step implements `WorkflowStepContract`.
- Steps return `StepResult::complete()`, `StepResult::await()`, or `StepResult::fail()`.
- `WorkflowContext` is immutable. Steps return context updates; the engine merges them.
- Branching uses **context flags** set by a routing step. Steps read flags and skip themselves if not needed.
- Timeouts are signals delivered by the engine. The step decides what a timeout means.
- The engine is a postman: it delivers signals and persists state. It has no opinion about what signals mean.

**StepResult Contract:**
```php
<?php
declare(strict_types=1);

namespace App\Workflow\Domain;

final class StepResult
{
    private function __construct(
        public readonly string $outcome,
        public readonly ?string $awaitSignal,
        public readonly ?string $failReason,
        public readonly array $contextUpdates,
    ) {}

    public static function complete(array $contextUpdates = []): self
    {
        return new self('complete', null, null, $contextUpdates);
    }

    public static function await(string $signal, array $contextUpdates = []): self
    {
        return new self('await', $signal, null, $contextUpdates);
    }

    public static function fail(string $reason): self
    {
        return new self('fail', null, $reason, []);
    }
}
```

**WorkflowStep Contract:**
```php
<?php
declare(strict_types=1);

namespace App\Workflow\Domain;

interface WorkflowStepContract
{
    public function execute(WorkflowContext $context): StepResult;

    public function timeoutSeconds(): ?int;

    public function maxAttempts(): int;
}
```

**WorkflowContext (Immutable):**
```php
<?php
declare(strict_types=1);

namespace App\Workflow\Domain;

final class WorkflowContext
{
    private function __construct(
        public readonly string $workflowInstanceId,
        public readonly string $aggregateId,
        public readonly string $aggregateType,
        private readonly array $data,
    ) {}

    public static function make(
        string $workflowInstanceId,
        string $aggregateId,
        string $aggregateType,
        array $initialData = [],
    ): self {
        return new self($workflowInstanceId, $aggregateId, $aggregateType, $initialData);
    }

    public function get(string $key, mixed $default = null): mixed
    {
        return $this->data[$key] ?? $default;
    }

    public function with(array $additions): self
    {
        return new self(
            $this->workflowInstanceId,
            $this->aggregateId,
            $this->aggregateType,
            array_merge($this->data, $additions),
        );
    }
}
```

**Alignment:**

- Readonly class (or readonly properties) for immutable value objects.
- Named constructors (`make()`) when they improve clarity.
- No hidden dependencies — `WorkflowStepContract` is explicit.
- `match` expressions for branching (recommended over `if/else` chains when possible).

## Connecting the Two Layers

The state machine and workflow engine are connected by a Laravel event listener. They have **no direct dependency** on each other.

**Rules:**
- The entity returns a domain event.
- The use case / application service persists the entity and returns the event(s).
- The controller (or command) records the event to an event log, then dispatches it via Laravel's `Event::dispatch()`.
- A listener starts the workflow.

**Example:**
```php
// Controller / Command
$events = $this->approveOrder->execute($orderId, $request->user()->id);

foreach ($events as $event) {
    $this->eventLog->record($event, $orderId, 'order', $request->user()->id);
    Event::dispatch($event);
}
```

```php
// Listener
final class StartOrderApprovalWorkflow
{
    public function __construct(
        private readonly WorkflowEngine $engine,
    ) {}

    public function handle(OrderApproved $event): void
    {
        $instance = $this->engine->start(
            workflowName: OrderApprovalWorkflow::name(),
            aggregateId: $event->orderId->value,
            aggregateType: 'order',
            initialData: [
                'order_value' => $event->orderValue,
            ],
        );

        AdvanceWorkflow::dispatch($instance->id);
    }
}
```

## Branching Without Trees

When workflows need conditional paths, use a **routing step** that writes flags into context. Later steps read those flags and decide whether to skip or execute.

**Rules:**
- Do NOT model branching as nested step trees or directed graphs.
- The step list stays flat and readable top-to-bottom.
- Each step owns its skip logic.
- If two paths share almost no steps, use separate workflow definitions instead.

**Example Routing Step:**
```php
final class RouteApproval implements WorkflowStepContract
{
    public function __construct(
        private readonly CustomerRepositoryContract $customers,
        private readonly ComplianceService $compliance,
    ) {}

    public function execute(WorkflowContext $context): StepResult
    {
        $customer = $this->customers->findById($context->get('customer_id'));

        return StepResult::complete([
            'requires_manual_review' => $context->get('order_value') > 10_000,
            'requires_compliance_hold' => $this->compliance->requiresHold($customer->region),
            'is_priority_customer' => $customer->isPriority(),
        ]);
    }

    public function timeoutSeconds(): ?int { return null; }
    public function maxAttempts(): int { return 1; }
}
```

## Timeout & Signal Handling

Every awaiting step declares its timeout. The engine delivers a `timeout` signal; the step decides what that means.

**Rules:**
- Hardcode timeouts as constants during early development; move to `SystemSettingsRepositoryContract` for runtime configuration.
- The engine dispatches a delayed `TimeoutWorkflowStep` job when parking an awaiting step.
- A scheduled sweep command acts as a backstop for missed timeout jobs.
- Steps handle the `timeout` signal explicitly in their `execute()` method.

**Example Timeout Handling:**
```php
final class AwaitPayment implements WorkflowStepContract
{
    public function execute(WorkflowContext $context): StepResult
    {
        if ($context->get('payment_status') !== null) {
            $status = $context->get('payment_status');

            return $status === 'timed_out'
                ? StepResult::fail('Payment window expired. Order will be cancelled.')
                : StepResult::complete();
        }

        $signal = $context->get('_signal');

        if ($signal === 'timeout') {
            return StepResult::complete(['payment_status' => 'timed_out']);
        }

        if ($signal === 'payment_succeeded') {
            return StepResult::complete([
                'payment_status' => 'paid',
                'payment_id' => $context->get('_signal_data.payment_id'),
            ]);
        }

        return StepResult::await('payment_succeeded');
    }

    public function timeoutSeconds(): ?int { return 604_800; } // 7 days
    public function maxAttempts(): int { return 1; }
}
```

## Quality Gates for Business Logic

When reviewing or implementing business logic, enforce these gates:

| # | Gate | Rule | Action if Violated |
|---|---|---|---|
| 1 | Status Type Safety | Every status is an Enum, never a string | Refactor to Enum |
| 2 | Transition Ownership | Status checks live in the entity, not services/controllers | Move checks to entity |
| 3 | Side Effect Purity | Entity methods do NOT send emails, call APIs, or dispatch events | Extract to workflow step |
| 4 | Event Explicitness | Transition methods return domain events; caller dispatches them | Add event return type |
| 5 | Context Immutability | Steps never mutate `WorkflowContext` directly | Return updates via `StepResult::complete()` |
| 6 | Async Explicitness | Waiting for external signals uses `StepResult::await()` | Replace polling with signal |
| 7 | Timeout Ownership | Timeout behavior is defined in the step, not the engine | Add timeout handling to step |
| 8 | Test Independence | Entity and step logic is testable without Laravel | Write unit tests without framework bootstrap |
| 9 | Guard Clauses | Precondition checks happen first, with early return/exception | Flatten nested conditionals |
| 10 | Readonly by Default | Value objects (events, context, results) are readonly | Add `readonly` keyword |

## Stop Conditions

- **Pattern fits** when an entity has 5+ statuses, non-trivial transitions, different side effects per transition, or async steps.
- **Pattern overkill** when there are only 2-3 statuses and no async operations — plain conditionals in a service are fine.
- **Escalate** when a transition graph genuinely requires a directed graph (not a flat list) — this may indicate the workflow should be split into multiple definitions.

## References

- `references/state-machine-pattern.md`
- `references/workflow-engine-pattern.md`
- `references/business-rules.md`
