# Workflow Engine Pattern Reference

## Overview

The workflow engine orchestrates side effects, async operations, and multi-step processes that follow a state transition. It is deliberately separate from the state machine: the entity announces "what happened," and the engine decides "what to do about it."

## Core Concepts

- **Workflow Definition** — A named, ordered list of step classes. Lives entirely in code.
- **Workflow Instance** — The running execution for a specific entity. Lives in the database.
- **Workflow Step** — A discrete PHP class that performs one action and returns a `StepResult`.
- **WorkflowContext** — Immutable shared state passed from step to step.
- **Signal** — An external input (webhook, user action, timeout) that resumes a paused workflow.

## Design Rules

### 1. Workflow Definition Is a Plain PHP Class

```php
<?php
declare(strict_types=1);

namespace App\Workflows\OrderApproval;

use App\Workflow\Domain\WorkflowDefinitionContract;

final class OrderApprovalWorkflow implements WorkflowDefinitionContract
{
    public static function name(): string
    {
        return 'order_approval';
    }

    public function steps(): array
    {
        return [
            ValidateOrderItems::class,
            RouteApproval::class,              // sets context flags
            ReserveInventory::class,
            AwaitComplianceClearance::class,   // skips if not required
            CreateInvoice::class,
            AwaitPayment::class,
            AwaitManualApproval::class,        // skips if not required
            NotifyFulfillmentTeam::class,      // uses priority flag
        ];
    }
}
```

**Rules:**
- The step list is flat. Read it top-to-bottom to understand the entire flow.
- Steps that conditionally skip themselves read context flags and return `complete()` immediately.
- If two paths share almost no steps, create separate workflow definitions.
- Changing a workflow definition is a code change with a readable diff.

### 2. Every Step Implements the Same Contract

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

**Rules:**
- `timeoutSeconds()` returns `null` for no timeout, or an integer for seconds.
- `maxAttempts()` declares how many times the step may be retried on failure.
- Steps MUST be independently testable without Laravel.

### 3. WorkflowContext Is Immutable

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

**Rules:**
- Steps never mutate `$context` directly.
- Steps return `StepResult::complete(['key' => 'value'])` and the engine merges updates.
- Context keys use `snake_case` for consistency.
- Signal data is prefixed with `_signal` and `_signal_data` to avoid collisions.

### 4. StepResult Has Three Outcomes

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

**Outcome meanings:**
- `complete` — Step finished successfully. Engine advances to the next step.
- `await` — Step is waiting for an external signal. Engine parks the workflow.
- `fail` — Step failed definitively. Workflow instance is marked failed.

### 5. Branching Via Context Flags

When a workflow needs conditional paths, a routing step writes boolean flags into context. Later steps read those flags and skip if not needed.

**Routing step:**
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

**Consuming step:**
```php
final class AwaitManualApproval implements WorkflowStepContract
{
    public function execute(WorkflowContext $context): StepResult
    {
        if (!$context->get('requires_manual_review', false)) {
            return StepResult::complete();
        }

        $signal = $context->get('_signal');

        if ($signal === 'timeout') {
            return StepResult::complete([
                'approval_decision' => 'auto_approved',
                'auto_approved_reason' => 'timeout',
            ]);
        }

        if ($signal === 'manager_decision') {
            return StepResult::complete([
                'approval_decision' => $context->get('_signal_data.decision'),
            ]);
        }

        return StepResult::await('manager_decision');
    }

    public function timeoutSeconds(): ?int { return 172_800; } // 48 hours
    public function maxAttempts(): int { return 1; }
}
```

**Rules:**
- Each step decides for itself whether to run or skip.
- The workflow definition stays flat and readable.
- Do not nest step lists or build directed graphs in PHP arrays.

### 6. Signal Handling and Resumption

When a signal arrives, the engine re-executes the awaiting step with the signal data merged into context.

**Step that handles signals:**
```php
final class AwaitPayment implements WorkflowStepContract
{
    public function execute(WorkflowContext $context): StepResult
    {
        // Already resolved — either paid or timed out
        if ($context->get('payment_status') !== null) {
            $status = $context->get('payment_status');

            if ($status === 'timed_out') {
                return StepResult::fail('Payment window expired. Order will be cancelled.');
            }

            return StepResult::complete();
        }

        // Signal just arrived
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

        // First time through — park and wait
        return StepResult::await('payment_succeeded');
    }

    public function timeoutSeconds(): ?int { return 604_800; } // 7 days
    public function maxAttempts(): int { return 1; }
}
```

**Key behavior:**
- The step runs **twice**: once to park, once to resume after the signal.
- On the second run, `_signal` and `_signal_data` are present in context.
- The step handles both the normal and timeout signals explicitly.

### 7. Timeout Mechanism

Every awaiting step declares its timeout. The engine handles the mechanics; the step handles the semantics.

**Engine behavior:**
1. When a step returns `await`, the engine dispatches a delayed job.
2. If the signal arrives before the job fires, the delayed job finds the workflow advanced and does nothing.
3. If the job fires first, it delivers a synthetic `timeout` signal.

**Delayed timeout job:**
```php
<?php
declare(strict_types=1);

namespace App\Jobs;

use App\Workflow\Domain\WorkflowEngine;
use App\Workflow\Domain\WorkflowRepositoryContract;
use Illuminate\Contracts\Queue\ShouldQueue;

final class TimeoutWorkflowStep implements ShouldQueue
{
    public function __construct(
        private readonly string $instanceId,
        private readonly string $expectedSignal,
    ) {}

    public function handle(WorkflowEngine $engine, WorkflowRepositoryContract $repository): void
    {
        $instance = $repository->findById($this->instanceId);

        if (!$instance->canReceiveSignal($this->expectedSignal)) {
            return;
        }

        $engine->signal(
            instanceId: $this->instanceId,
            signal: 'timeout',
            signalData: ['timed_out_signal' => $this->expectedSignal],
        );
    }
}
```

**Backstop sweep command:**
```php
<?php
declare(strict_types=1);

namespace App\Console\Commands;

use App\Workflow\Domain\WorkflowEngine;
use App\Workflow\Domain\WorkflowRepositoryContract;
use Illuminate\Console\Command;

final class ExpireStaleWorkflowSteps extends Command
{
    protected $signature = 'workflows:expire-stale';

    public function handle(WorkflowRepositoryContract $repository, WorkflowEngine $engine): void
    {
        $stale = $repository->findTimedOutInstances(now());

        foreach ($stale as $instance) {
            $engine->signal(
                instanceId: $instance->id,
                signal: 'timeout',
                signalData: [
                    'source' => 'sweep',
                    'timed_out_signal' => $instance->getAwaitingSignal(),
                ],
            );
        }
    }
}
```

**Rules:**
- The step owns timeout behavior, not the engine.
- A payment timeout might fail the workflow; a manual approval timeout might auto-approve.
- Runtime-configurable timeouts read from `SystemSettingsRepositoryContract`.
- Existing parked instances are NOT affected by settings changes.

### 8. Database Schema

Two tables do the heavy lifting:

**workflow_instances**
| Column | Type | Purpose |
|---|---|---|
| id | UUID / string | Primary key |
| workflow_name | string | Matches `WorkflowDefinitionContract::name()` |
| aggregate_id | string | Entity ID that triggered the workflow |
| aggregate_type | string | Entity type (e.g., 'order') |
| status | enum | running, awaiting, completed, failed, cancelled |
| current_step_index | int | Zero-based index into the step list |
| awaiting_signal | string\|null | Signal name when status = awaiting |
| context | json | Immutable context data |
| timeout_at | datetime\|null | When the timeout job fires |
| created_at | timestamp | |
| updated_at | timestamp | |

**workflow_signals**
| Column | Type | Purpose |
|---|---|---|
| id | auto-increment | |
| workflow_instance_id | FK | |
| signal | string | Signal name (e.g., 'payment_succeeded', 'timeout') |
| signal_data | json | Payload delivered with the signal |
| source | string | 'webhook', 'job', 'sweep', 'manual' |
| created_at | timestamp | |

**Rules:**
- `workflow_signals` is append-only. Never update or delete.
- The engine persists state before advancing to the next step.
- A failed queue job can be retried safely because the engine resumes from a known state.

## Testing Strategy

Every step is tested as a pure function: construct context, execute, assert result.

```php
<?php
declare(strict_types=1);

use App\Workflow\Domain\StepResult;
use App\Workflow\Domain\WorkflowContext;
use App\Workflows\OrderApproval\Steps\AwaitPayment;

it('awaits payment signal on first run', function () {
    $context = WorkflowContext::make(
        workflowInstanceId: 'wf_001',
        aggregateId: 'ord_001',
        aggregateType: 'order',
        initialData: [],
    );

    $step = new AwaitPayment();
    $result = $step->execute($context);

    expect($result->outcome)->toBe('await');
    expect($result->awaitSignal)->toBe('payment_succeeded');
});

it('completes when payment signal arrives', function () {
    $context = WorkflowContext::make(
        workflowInstanceId: 'wf_001',
        aggregateId: 'ord_001',
        aggregateType: 'order',
        initialData: [
            '_signal' => 'payment_succeeded',
            '_signal_data' => ['payment_id' => 'pay_123'],
        ],
    );

    $step = new AwaitPayment();
    $result = $step->execute($context);

    expect($result->outcome)->toBe('complete');
    expect($result->contextUpdates['payment_status'])->toBe('paid');
    expect($result->contextUpdates['payment_id'])->toBe('pay_123');
});

it('fails when payment times out', function () {
    $context = WorkflowContext::make(
        workflowInstanceId: 'wf_001',
        aggregateId: 'ord_001',
        aggregateType: 'order',
        initialData: [
            '_signal' => 'timeout',
            'payment_status' => 'timed_out',
        ],
    );

    $step = new AwaitPayment();
    $result = $step->execute($context);

    expect($result->outcome)->toBe('fail');
    expect($result->failReason)->toBe('Payment window expired. Order will be cancelled.');
});
```

**Coverage goals:**
- Every step tested for `complete`, `await`, and `fail` paths.
- Context flag skip logic tested (when applicable).
- Signal handling tested with and without signal data.
- Timeout behavior tested explicitly.

## Common Pitfalls

| Pitfall | Why It Hurts | Solution |
|---|---|---|
| Steps call `Event::dispatch()` | Couples step to Laravel; hard to test | Return `StepResult` only |
| Mutable context | Steps clobber each other's data | Immutable `with()` pattern |
| Nested step trees | Workflow definitions become unreadable | Flat list + context flags |
| Polling loops | Wastes resources; fragile | `await` + signal |
| Timeout logic in engine | Different steps need different timeout behavior | Step owns timeout semantics |
| Missing backstop sweep | Queue outages strand workflows forever | Scheduled `ExpireStaleWorkflowSteps` |
| Direct DB access in step | Couples step to schema; breaks testability | Inject repository interface |

## Directory Structure

```
src/Workflow/
  Domain/
    WorkflowEngine.php
    WorkflowDefinitionContract.php
    WorkflowStepContract.php
    WorkflowContext.php
    StepResult.php
    WorkflowRepositoryContract.php
    WorkflowInstance.php          # Eloquent or plain model
    WorkflowSignal.php
  Infrastructure/
    EloquentWorkflowRepository.php
    DatabaseWorkflowInstance.php
    Jobs/
      AdvanceWorkflow.php
      TimeoutWorkflowStep.php
  Application/
    StartWorkflow.php
    SignalWorkflow.php

src/Workflows/
  OrderApproval/
    OrderApprovalWorkflow.php
    Steps/
      ValidateOrderItems.php
      RouteApproval.php
      ReserveInventory.php
      AwaitComplianceClearance.php
      CreateInvoice.php
      AwaitPayment.php
      AwaitManualApproval.php
      NotifyFulfillmentTeam.php
```
