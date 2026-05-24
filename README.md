# Laravel Statecraft

State machines + workflow engines for bulletproof Laravel/PHP business logic.

## What It Does

A two-layer pattern that keeps domain logic explicit, auditable, and resilient:

- **State Machine Layer** — Entities own their valid transitions and emit domain events.
- **Workflow Engine Layer** — Orchestrates side effects, async operations, and multi-step processes.

## When to Use

- Entities with 5+ statuses and non-trivial transition graphs.
- Different transitions need different side effects.
- Async operations (payments, approvals, webhooks) that can fail halfway.
- You need an audit trail of who triggered what and when.

## Structure

```
laravel-statecraft/
├── SKILL.md                              Main skill: directives, quality gates, examples
└── references/
    ├── state-machine-pattern.md          Entity-owned transitions, enums, events, testing
    ├── workflow-engine-pattern.md        Steps, context, signals, timeouts, DB schema
    └── business-rules.md                 PHP/Laravel coding standards for domain logic
```

## Usage

This is an agent skill. Install it under your tool's skill directory (e.g., `.agents/skills/laravel-statecraft/`, `.opencode/skills/laravel-statecraft/`, `.claude/skills/laravel-statecraft/`).

## Credits

Pattern based on [The Art of Keeping Business Logic Honest](https://www.juststeveking.com/articles/the-art-of-keeping-business-logic-honest/) by Steve McDougall. Coding standards adapted from b7s project conventions.
