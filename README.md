# Laravel Statecraft

This skill helps developers work with **Bounded Contexts + state machines + job orchestration** for bulletproof Laravel business logic. Uses standard Laravel directory conventions — no custom folder structure.

**This is not full Domain-Driven Design.** We adopt Bounded Contexts to split god entities, state machines to enforce transitions, and Laravel's native job system to orchestrate side effects. We skip aggregates, repositories, and value object ceremony unless genuinely needed.

## Install

Using [Skills](https://www.skills.sh/):

```bash
npx skills add https://github.com/b7s/skill-laravel-statecraft
```

## How It Works

### The Problem

You've seen this class with 15 nullable columns, half of them null, serving Billing, Fulfillment, Compliance, and Catalog all at once. That's a **God Entity**, and it's an architectural **lie**. Different contexts genuinely need different models.

### The Solution: Core Patterns

1. **Bounded Contexts** — Each context gets its own Eloquent model, its own status enum, its own events. No model serves two contexts. They communicate through explicit integration patterns.

2. **State Machines** — Eloquent models own their transitions. Each transition validates state, mutates status, and returns a typed domain event. No status checks in controllers.

3. **Job Orchestration** — Orchestrates side effects using Laravel's native job chains, batches, and queues. No custom workflow infrastructure needed.

### Application Service: Actions + Services

**Actions** (`app/Actions/{Context}/`) — Perform ONE atomic operation (create invoice, generate PDF, send email). Wrap mutations in `DB::transaction()`. Do NOT call other actions.

**Services** (`app/Services/{Context}/`) — Two types:
1. **Orchestrator Services**: Call multiple actions to complete workflows
2. **Logic Services**: Pure calculations, validation, external APIs (no actions)

**Rule:** Services orchestrate. Actions execute. Actions never call other actions.

### Context Integration

Contexts communicate through integration patterns:

| Pattern | When |
|---|---|
| **Customer/Supplier** | Upstream publishes events; downstream subscribes (primary pattern) |
| **Shared Primitives** | Share string IDs, int amounts via primitive types + validation |
| **Anti-Corruption Layer** | Protect from external/unstable upstream systems (advanced, rarely needed) |

**Contexts:** Document relationships inline in `EventServiceProvider`.  

### Domain First, Deployment Second

Bounded Contexts are a domain concept. All contexts live in one monolith. Microservices are a deployment decision made later — not a design decision. Bounded contexts make it easier if you ever need to separate them into microservices.

### Mandatory Quality Gates

Every feature requires:
- **Tests** — Pest tests for all actions, transitions, and event listeners
- **Static Analysis** — PHPStan level 6 compliance (change if you want)
- **Code Style** — Laravel Pint formatting
- **Quality Metrics** — b7s/catraca checks for Security, DRY, SRP violations

**IMPORTANT:** Tests and quality checks must run automatically after every change and fix any issue.

## Directory Structure

```
app/
├── Models/                      # Eloquent models with transition methods
├── Enums/{Context}/             # Status enums (one per context per model)
├── Events/{Context}/            # Domain events
├── Exceptions/                  # Typed exceptions
├── Actions/{Context}/           # One action per file, flat folder
├── Listeners/{Context}/         # Event listeners
├── Jobs/{Context}/              # Queued jobs for async operations
├── Services/{Context}/          # Domain services (when needed)
├── Infrastructure/{Context}/ACL/# Anti-Corruption Layer (advanced, rarely needed)
└── Http/
    ├── Controllers/
    └── Requests/                # Form Requests (validate before action)
```

**Note:** No custom `app/Domain/` or `app/ValueObjects/` folders. Use Laravel primitives (string, int, float, bool, etc.) with validation. Create custom casts only when validation/transformation logic is complex and shared.

## Files

| File | What It Covers |
|---|---|
| [SKILL.md](SKILL.md) | Main skill: directives, quality gates, directory structure, examples |
| [references/action-service-pattern.md](references/action-service-pattern.md) | Actions + Services: orchestration vs calculation, sync vs async patterns |
| [references/bounded-context-pattern.md](references/bounded-context-pattern.md) | God entities, context isolation, testing per context |
| [references/cross-context-comments.md](references/cross-context-comments.md) | Cross-context comment conventions in code |
| [references/integration-patterns.md](references/integration-patterns.md) | Customer/Supplier, ACL, shared primitives — with examples |
| [references/state-machine-pattern.md](references/state-machine-pattern.md) | Eloquent model transitions, enums, events, testing |
| [references/job-orchestration-pattern.md](references/job-orchestration-pattern.md) | Job chains, batches, and async coordination |
| [references/php-rules.md](references/php-rules.md) | PHP/Laravel coding standards and best practices |
| [references/quality-gates.md](references/quality-gates.md) | Testing requirements, PHPStan, Pint, Catraca setup |

## Usage

This is an agent skill. Install it under your tool's skill directory (e.g., `.agents/skills/laravel-statecraft/`, `.opencode/skills/laravel-statecraft/`, `.claude/skills/laravel-statecraft/`).

## Credits

- [brunots.dev](https://brunots.dev)
- Eric Evans — *Domain-Driven Design* (Bounded Contexts, Context Maps, Integration Patterns)
- Vaughn Vernon — *Implementing Domain-Driven Design* (Aggregates, Events, ACL)
- Robert C. Martin — *Clean Code* (SRP, DRY, meaningful names, small functions, guard clauses)
- Fabio Akita — [Open Source Best Practices & LLM](https://akitaonrails.com/en/2026/05/30/open-source-best-practices-llm-the-minimum/)
- Steve McDougall — [The Art of Keeping Business Logic Honest](https://www.juststeveking.com/articles/the-art-of-keeping.business-logic-honest/)
