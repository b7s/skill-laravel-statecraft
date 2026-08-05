# Security & Access-Control Checks — IDOR Prevention Across Access Boundaries

Statecraft makes domain logic bulletproof, but a correct state machine is worthless if the wrong actor can reach it. This reference defines the **mandatory access-control layer** every bounded context must enforce before any transition, query, or mutation runs.

> **OWASP alignment:** A01 Broken Access Control (OWASP Top 10). OWASP Multi-Tenant Application Security §3 covers the tenant-isolation special case.

## Scope: every app has an access boundary — not always a tenant

The skill talks about IDOR in tenant terms because that's the canonical, hardest case (cross-tenant leaks under per-tenant DB or schema tenancy). But **the system does not always have a tenant.** What every system DOES have is an **access boundary** — the wall that decides which rows an authenticated actor may touch. The same defence-in-depth applies regardless of what the wall is:

| App topology | The access boundary | A leak looks like… |
|---|---|---|
| Multi-tenant, per-tenant database (`stancl/tenancy` multi-DB) | Tenant key (resolved via subdomain/header/JWT) | Tenant B's user reads/edits tenant A's invoice |
| Multi-tenant, schema or column-based (`tenant_id` on shared tables) | Tenant id column | Same — binding returns the wrong row |
| Single-tenant, user-owned resources | The owning user (`user_id` on rows) | User B edits user A's `users/{user}/orders/{order}` |
| Org/team-scoped (GitHub-style) | The organisation/team id | Org B's member reads org A's private repo |
| Platform / central admin context | "Is this a platform-staff route?" + impersonation stamp | Tenant-staff route accessed without platform role; impersonation Don't-impersonate self-service |
| Public read-only API | None (no authn) | Modify verbs must 401/403; the IDOR suite is irrelevant |

**The IDOR sweep applies wherever a boundary exists** and an authenticated attacker could reach a foreign boundary's row by guessing its id. It is most familiar as "cross-tenant IDOR" but the identical pattern proves user-bucketed ownership in a single-tenant app, org isolation, and platform-admin impersonation. **No boundary → no suite** (you still need rate limits + input validation, but cross-boundary IDOR is not a risk).

The trait below is written **boundary-agnostic**: you swap four small hooks to match your topology. The canonical consumer examples show both the multi-tenant (per-tenant DB) and single-tenant (user-owned) variants.

## Why this matters in Statecraft

Bounded contexts owning models like `Invoice`, `Order`, or `Scorecard` MUST guarantee those models never leak across their access boundary. The state machine (`markPaid()`, `cancel()`, `approve()`) and the action layer (`MarkInvoicePaid`) both operate on a model resolved by route-model binding — if that binding trusts the request's resource id without scoping it to the acting actor, **a transition applied to user A's invoice can be triggered by user B**, or a transition on tenant A's record by tenant B's user. The transition is then "correct" by the machine's rules but serves the wrong actor.

The check must live at the **data-access layer**, not in the controller. A controller-level `Gate::check('own', $model)` or a `where('owner_id', ...)` in the controller is insufficient because:

- A second entry point (job, listener, console command, queued transition) bypasses the HTTP gate.
- A future route group reuses the model binding without the gate.
- Route-model binding already resolved the model; the gate is a re-check, not a filter.

## The Three Layers of Access Control

```
HTTP Request
    │
    ▼
1. Policy / Gate (controller authorisation) — who may call this verb on this model class?
    │
    ▼
2. Boundary-Scoped Binding (route-model binding) — only resolve rows that belong to the acting actor
    │
    ▼
3. Transition Guard (model) — only own invariants + assert the acting actor still owns the row
```

Layers 1 and 2 are HTTP-facing; **layer 3 is mandatory for every entry point (jobs, listeners, commands, tests, and HTTP).**

## 1. Policy / Gate — Controller Authorisation

```php
// app/Policies/InvoicePolicy.php
final class InvoicePolicy
{
    public function update(User $user, Invoice $invoice): bool
    {
        // Multi-tenant (column-based):
        return $invoice->tenant_id === $user->tenant_id
            && $user->can('invoices.update');

        // — OR — single-tenant, user-owned:
        // return $invoice->user_id === $user->id
        //     && $user->can('invoices.update');
    }
}

// Controller — let Laravel resolve the model, then authorise the *verb*
class InvoiceController extends Controller
{
    public function update(UpdateInvoiceRequest $request, Invoice $invoice, UpdateInvoice $action)
    {
        $this->authorize('update', $invoice); // throws 403 if not allowed

        $invoice = $action($invoice, $request->payload());

        return new InvoiceResource($invoice);
    }
}
```

This is necessary but NOT sufficient — keep reading.

## 2. Boundary-Scoped Route-Model Binding — the Real Defence

A per-tenant database (multi-DB tenancy, e.g. `stancl/tenancy`) makes this automatic: route-model binding runs against the acting tenant's connection, so a foreign tenant's id **cannot resolve** — Laravel returns 404. **A foreign-tenant id 404-ing is your guarantee** — but the guarantee is configured, not proven; the suite (`TestsCrossTenantAccess`) proves it end-to-end.

When the boundary isn't a separate database — schema/column tenancy, single-tenant `user_id` ownership, or org id — you MUST scope binding explicitly.

### Schema / column tenancy — scoped binding

```php
// AppServiceProvider::boot()
public function boot(): void
{
    // Strict mode rejects lazy-loaded relations so leaks by relation access fail loudly.
    // Useless on single-tenant per-user ownership if your model has cross-user relations.
    Model::shouldBeStrict();
}
```

OR scope per-binding when strict mode is too aggressive:

```php
// routes/api.php — scoped binding for route-model params sharing the boundary key
Route::put('invoices/{invoice}', [InvoiceController::class, 'update'])
    ->scopeBindings();

// Or a custom binding that injects the acting actor's boundary key from the acting context.
// Replace `TenantContext::currentId()` with `Auth::id()`, `OrgContext::currentId()`, etc.
public function boot(): void
{
    Route::bind('invoice', function (string $value): Invoice {
        return Invoice::query()
            ->where('id', $value)
            ->where('tenant_id', TenantContext::currentId()) // or `->where('user_id', Auth::id())`
            ->firstOrFail(); // 404 when the id does not belong to the acting actor
    });
}
```

The key contract: **a foreign actor's resource id yields 404, never the row**, never 403. A 403 on a resolvable binding is an information leak (it confirms the id exists in another boundary).

## 3. Transition Guard — Model-Level Ownership Assertion

Even after binding, every transition method MUST re-check ownership defensively, because jobs/listeners can be invoked outside the HTTP gate. The shape of the guard mirrors the boundary. `TenantContext::currentId()` is the multi-tenant example; `Auth::id()` or `OrgContext::currentId()` are the single-tenant / org variants.

```php
class Invoice extends Model
{
    public function markPaid(string $paymentId): InvoicePaidData
    {
        // Guard: assert the acting actor still owns this row.
        // Throw a typed domain exception — never a 500. Example uses tenant boundary;
        // single-tenant apps swap to: $this->user_id !== Auth::id()
        if ($this->tenant_id !== TenantContext::currentId()) {
            throw new AccessDeniedException('Invoice does not belong to the acting actor.');
        }

        if ($this->status !== InvoiceStatus::Pending) {
            throw new InvalidTransitionException('Cannot pay from this status');
        }

        $this->status = InvoiceStatus::Paid;
        $this->payment_id = $paymentId;
        $this->save();

        return InvoicePaidData::fromEvent($this, $paymentId);
    }
}
```

## 4. IDOR Regression Suite — the Mandatory Test (boundary-agnostic)

Policies and scoped bindings are config; the *contract* is proven by an automated sweep that fires every show/update/destroy route as a **cross-boundary attacker** and asserts **404** (or 422).

### Where to add this

- **Trait:** `tests/Support/TestsCrossTenantAccess.php` (project root `tests/`). The trait name is `TestsCrossTenantAccess` even in single-tenant apps — "cross-tenant" is the most readable name; in single-tenant apps the "tenant" is the user bucket.
- **Test file:** `tests/Feature/Security/CrossTenantIdorTest.php` (or your project's security test namespace).
- **Invoked by:** `uses(TestsCrossTenantAccess::class)` at the top of the pest file (or attach the trait to `tests/TestCase.php` so `pest-plugin-phpstan` analyses it in TestCase context — see Hit List below).

### Generic trait skeleton — boundary-agnostic, four swappable hooks

The trait is written with four overridable methods. The default implementation targets `stancl/tenancy` (multi-tenant, per-tenant DB). Single-tenant / org-scoped apps override these four methods; the sweep, the catalogue, the fixtures — everything else — is unchanged.

```php
<?php

declare(strict_types=1);

namespace Tests\Support;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Foundation\Testing\TestCase;

trait TestsCrossTenantAccess
{
    /**
     * Declarative catalogue of routes to probe. Set per test via setTenantIdorRoutes().
     * Kept as an instance property — see Hit List "Static vs. instance catalogue binding".
     *
     * @var array<string, array{
     *   model: class-string<Model>,
     *   factory: callable(object $self): Model,
     *   methods: list<string>,
     *   body?: array<string, mixed>,
     *   params?: callable(Model $record, object $self): array<int, Model>,
     * }>
     */
    private array $idorRoutes = [];
    private static ?object $actors = null;

    // ─── The four swappable hooks ────────────────────────────────────────
    // Override these in a project-specific subclass or on tests/TestCase to match
    // your access-boundary topology. Defaults assume `stancl/tenancy` multi-tenant.

    /**
     * Enter the access boundary of `$actor` — make subsequent DB/route resolution
     * scoped to that actor. Multi-tenant default: `tenancy()->initialize($tenant)`.
     * Single-tenant user-owned: no-op (the acting user is set via `actingAs()`).
     */
    protected function enterActorContext(object $actor): void
    {
        tenancy()->initialize($actor);
    }

    /**
     * Exit the access boundary — restore the default connection / context so the
     * next actor's setup is clean. Multi-tenant default: `tenancy()->end()`.
     * Single-tenant: no-op.
     */
    protected function exitActorContext(): void
    {
        if (tenancy()->initialized) {
            tenancy()->end();
        }
    }

    /**
     * Headers the attacker's request must carry to be routed into actor B's
     * boundary. Multi-tenant (header-driven tenancy): `['X-Tenant-Id' => $key]`.
     * Subdomain tenancy: use `withServerParameter(...)`. Single-tenant: `[]`.
     *
     * @return array<string, string>
     */
    protected function actorRequestHeaders(object $actor): array
    {
        return ['X-Tenant-Id' => (string) $actor->getTenantKey()];
    }

    /**
     * Build the two opposing actors and their owning users. Return an object with
     * `{ownerA, userA, ownerB, userB}`. Default uses `Tenant` + `TenantSeeder` +
     * a `Role::Owner` enum; override for single-tenant (`User::factory()->create()`
     * pairs) or org-scoped (`Org::factory()` + member users) topologies.
     */
    protected function buildOpposingActors(): object
    {
        $ownerA = \App\Models\Tenant::query()->create($this->actorDefaults('alpha'));
        $this->enterActorContext($ownerA);
        \Database\Seeders\TenantSeeder::seedDefaults();
        $userA = \App\Models\User::factory()->create(['is_active' => true]);
        $userA->assignRole(\App\Enums\Access\Role::Owner->value);
        $this->exitActorContext();

        $ownerB = \App\Models\Tenant::query()->create($this->actorDefaults('beta'));
        $this->enterActorContext($ownerB);
        \Database\Seeders\TenantSeeder::seedDefaults();
        $userB = \App\Models\User::factory()->create(['is_active' => true]);
        $userB->assignRole(\App\Enums\Access\Role::Owner->value);
        $this->exitActorContext();

        return (object) [
            'ownerA' => $ownerA, 'userA' => $userA,
            'ownerB' => $ownerB, 'userB' => $userB,
        ];
    }

    /** @return array{name:string,slug:string,status:string} */
    protected function actorDefaults(string $slug): array
    {
        return ['name' => ucfirst($slug).' Inc', 'slug' => $slug, 'status' => 'trial'];
    }

    // ─── The non-swappable mechanics ──────────────────────────────────────

    /** Lazily build + cache the two actors. */
    private function ensureActorsBind(): object
    {
        if (self::$actors instanceof \stdClass) {
            return self::$actors;
        }

        return self::$actors = $this->buildOpposingActors();
    }

    /** Set the catalogue from a pest beforeEach. Instance property — never static. */
    private function setTenantIdorRoutes(array $routes): void
    {
        $this->idorRoutes = $routes;
    }

    /** Create a record in actor A's boundary, then exit the boundary (resets default connection). */
    private function createInOwnerA(callable $factory): Model
    {
        $actors = $this->ensureActorsBind();
        $this->enterActorContext($actors->ownerA);
        $record = $factory($this);
        $this->exitActorContext();

        return $record;
    }

    /** The sweep itself — every catalogue route/verb probed as a cross-boundary attacker. */
    private function sweepIdor(): void
    {
        $actors = $this->ensureActorsBind();

        /** @var array<string, array{model:class-string<Model>,factory:callable,methods:list<string>,body?:array<string,mixed>,params?:callable}> $routes */
        $routes = $this->idorRoutes;

        foreach ($routes as $routeName => $spec) {
            $record = $this->createInOwnerA($spec['factory']);

            foreach ($spec['methods'] as $verb) {
                // Attacker is actor B's user, authenticated + boundary on B.
                $this->enterActorContext($actors->ownerB);
                $routeParams = isset($spec['params'])
                    ? ($spec['params'])($record, $this)
                    : $record;

                $uri = route($routeName, $routeParams, false);

                $request = $this->actingAs($actors->userB, 'sanctum')
                    ->withHeaders($this->actorRequestHeaders($actors->ownerB));
                $body = $spec['body'] ?? [];
                $response = match ($verb) {
                    'GET'    => $request->getJson($uri),
                    'PUT'    => $request->putJson($uri, $body),
                    'PATCH'  => $request->patchJson($uri, $body),
                    'DELETE' => $request->deleteJson($uri),
                    default  => throw new \InvalidArgumentException("Unknown verb [{$verb}] for route [{$routeName}]."),
                };
                $status = $response->status();
                $this->exitActorContext();

                // 404 or 422 passes (binding missed / request shape rejected).
                // 200 means the row leaked across the boundary.
                // 403 means an authz gate fired while the binding STILL resolved the foreign row
                // (information leak). BOTH fail.
                expect(in_array($status, [404, 422], true))
                    ->toBeTrue("IDOR leak on [{$routeName}] [{$verb}]: expected 404/422, got {$status}.");
            }
        }
    }

    private function resetIdorTenants(): void
    {
        self::$actors = null;
        $this->idorRoutes = [];
        $this->exitActorContext();
    }
}
```

### Single-tenant / user-owned variant of the hooks

The trait is identical — only the four hooks change. Here is what an override looks like on `tests/TestCase.php` for a single-tenant app where the boundary is the owning user (no `tenancy()` helper exists):

```php
abstract class TestCase extends BaseTestCase
{
    use TestsCrossTenantAccess;

    // No tenant connection to enter/exit — the acting user is set via actingAs().
    protected function enterActorContext(object $actor): void
    {
        // no-op
    }

    protected function exitActorContext(): void
    {
        // no-op
    }

    // No tenant header on the request.
    /** @return array<string, string> */
    protected function actorRequestHeaders(object $actor): array
    {
        return [];
    }

    // Actors are just users: actor A owns the record, user B is the attacker.
    protected function buildOpposingActors(): object
    {
        return (object) [
            'ownerA' => User::factory()->create(),
            'userA'  => User::factory()->create(),
            'ownerB' => User::factory()->create(),
            'userB'  => User::factory()->create(),
        ];
    }
}
```

> The same pattern covers org-scoped apps: `enterActorContext()` sets `OrgContext::current($org)` (or a request header), `actorRequestHeaders()` returns `['X-Org-Id' => $org->id]`, `buildOpposingActors()` returns two orgs + their member users. The sweep, the catalogue, and the assertions are unchanged.

### Generic Pest consumer (multi-tenant example)

```php
<?php

declare(strict_types=1);

use App\Models\Billing\Invoice;
use Database\Factories\Billing\InvoiceFactory;
use Tests\Support\TestsCrossTenantAccess;

// IMPORTANT: also `use TestsCrossTenantAccess;` on tests/TestCase.php so
// pest-plugin-phpstan analyses the trait in TestCase context.
uses(TestsCrossTenantAccess::class);

beforeEach(function (): void {
    $this->resetIdorTenants();

    $this->setTenantIdorRoutes([
        'api.v1.invoices.show' => [
            'model'   => Invoice::class,
            'factory' => fn (object $t) => InvoiceFactory::new()->create(),
            'methods' => ['GET'],
        ],
        'api.v1.invoices.update' => [
            'model'   => Invoice::class,
            'factory' => fn (object $t) => InvoiceFactory::new()->create(),
            'methods' => ['PUT'],
            'body'    => ['notes' => 'attacker'],
        ],
        'api.v1.invoices.destroy' => [
            'model'   => Invoice::class,
            'factory' => fn (object $t) => InvoiceFactory::new()->create(),
            'methods' => ['DELETE'],
        ],

        // Nested route: invoices/{invoice}/payments/{payment}
        'api.v1.payments.show' => [
            'model'   => Payment::class,
            'factory' => function (object $t): Payment {
                $invoice = InvoiceFactory::new()->create();
                $payment = Payment::factory()->for($invoice)->create();
                $payment->setRelation('invoice', $invoice); // pre-load parent

                return $payment;
            },
            'methods' => ['GET'],
            'params'  => fn (Payment $payment, object $t) => [$payment->invoice, $payment],
        ],
    ]);
});

afterEach(function (): void {
    $this->resetIdorTenants();
});

it('blocks cross-tenant IDOR on every show / update / destroy route', function (): void {
    $this->sweepIdor();
});

it('allows same-tenant owner to reach the record (sanity)', function (): void {
    $actors = $this->ensureActorsBind();
    $record = $this->createInOwnerA(fn (object $t) => InvoiceFactory::new()->create());

    // tenancy is on ownerA here; actingAs userA → 200
    $this->actingAs($actors->userA, 'sanctum')
        ->getJson(route('api.v1.invoices.show', $record))
        ->assertOk()
        ->assertJsonPath('data.id', $record->getKey());

    tenancy()->end();
});
```

### Single-tenant Pest consumer (user-owned resources)

Identical catalogue; the routes are user-scoped (`users/{user}/orders/{order}`), no tenant seeders, no `X-Tenant-Id` header. Both the catalogue and the sanity test shrink accordingly.

```php
beforeEach(function (): void {
    $this->resetIdorTenants();

    $this->setTenantIdorRoutes([
        // Order routes are scoped by their owning user.
        'api.v1.users.orders.show' => [
            'model'   => Order::class,
            'factory' => fn (object $t) => OrderFactory::new()->create(),
            'methods' => ['GET'],
            // Route: users/{user}/orders/{order} — bind the owning user (createInOwnerA
            // returns the Order; its user_id points at ownerA, but we pre-load anyway).
            'params'  => fn (Order $order, object $t) => [$order->user, $order],
        ],
    ]);
});
```

### Kill switch: when the suite must be updated

A new boundary-scoped route lands without a catalogue row → the sweep stays green but coverage silently regresses. To prevent this, assert that every `show`/`update`/`destroy` route scoped to the boundary appears in the catalogue. The marker differs per topology: `tenant` middleware (multi-tenant), a `scoped` binding (`->scopeBindings()`), or a convention (every route under `users/{user}/...`).

```php
// Multi-tenant variant — looks for the `tenant` middleware.
it('catalogue covers every tenant-scoped show / update / destroy route', function (): void {
    $router = app(\Illuminate\Routing\Router::class);
    /** @var \Illuminate\Support\Collection<int, \Illuminate\Routing\Route> $tenantRoutes */
    $tenantRoutes = collect($router->getRoutes())
        ->filter(fn ($r) => in_array('tenant', $r->gatherMiddleware(), true))
        ->filter(fn ($r) => in_array($r->methods[0], ['GET', 'PUT', 'PATCH', 'DELETE'], true))
        ->filter(fn ($r) => $r->getName() !== null);

    $covered = collect(array_keys($this->idorRoutes));

    $missing = $tenantRoutes
        ->filter(fn ($r) => ! $covered->contains($r->getName()))
        ->map(fn ($r) => $r->getName());

    expect($missing)->toBeEmpty('Catalogue missing boundary-scoped routes: '.$missing->implode(', '));
});
```

## Hit List — Gotchas Encountered in Practice

### pest-plugin-phpstan skips traits in `uses()`

The official `pestphp/pest-plugin-phpstan` extension's `TestClosureThisTypeExtension::toClassObjectTypes()` **explicitly skips traits** (`isTrait() → continue`). So passing your trait to `uses(YourTrait::class)` in a pest file makes phpstan report every trait method as "undefined" on `$this` AND flags the trait as "used zero times".

**Fix:** put `use TestsCrossTenantAccess;` directly on `tests/TestCase.php`. Pest re-applying the same trait via `uses()` in the test-file subclass is a legal no-op in PHP (a child re-using a parent's trait is allowed). This is also the natural place to override the four swappable hooks for single-tenant / org-scoped topologies (the override lives next to `use TestsCrossTenantAccess;`).

### Static vs. instance catalogue binding

A `self::$idorRoutes` static set by `YourTrait::setTenantIdorRoutes(...)` binds to the **trait class** — but `$this->sweepIdor()` reads from the **pest wrapper class**. Result: the catalogue silently reads empty and the sweep runs with **0 assertions** (everything "passes").

**Fix:** catalogue is an instance property (`private array $idorRoutes = []`). Side-steps late-static-binding entirely.

### Nested routes + boundary-context exit

Routes like `invoices/{invoice}/payments/{payment}` need BOTH parents in `route()`. After `exitActorContext()` the boundary's connection / context is gone, so a lazy `$payment->invoice` access throws ("Database connection [tenant] not configured" for multi-DB tenancy, or null for column/org tenancy).

**Fix:** the `params` callback returns `[parent, leaf]` and the factory **pre-loads** the parent via `$child->setRelation('parent', $parent)` so the in-memory relation cache is read, not the boundary-scoped DB.

### Route name shapes

`apiResource` routes are **unprefixed** (`invoices.show`). Explicit `Route::get('...')->name('...')` routes carry their literal name (often `api.v1.invoices.show`). Verify with `php artisan route:list` — catalogue keys must match the registered names **verbatim**. The kill-switch test above surfaces mismatches.

**Version segment in route names is also config-driven (Gate 18).** The `api.v1...` string you see in `route:list` is built at registration time from `config()->string('app.actual_version', 'v1')` — never write `'v1'` as a literal when you *name* a route. Register versioned names so the segment derives from the same config key as the prefix:

```php
// routes/api.php
$actualVersion = config()->string('app.actual_version', 'v1');

Route::prefix($actualVersion)
    ->middleware(['request-id'])
    ->name("api.{$actualVersion}.")
    ->group(function () {
        Route::get('invoices/{invoice}', [InvoiceController::class, 'show'])->name('invoices.show');
        // registered as: api.v1.invoices.show  (when APP_ACTUAL_VERSION=v1)
    });
```

The **catalogue keys in tests are fixture data, not source code** — they copy `route:list` verbatim, so they contain the literal `api.v1...` that was registered. That is correct and required (the test must assert against the name Lumen actually produced). The Gate 18 contract applies to *route registration*, not to test fixtures quoting the registry.

### 403 is a leak, not a pass

A 403 on a route whose binding still **resolved the foreign row** confirms the id exists in another boundary — an information leak. The sweep asserts `404 || 422` only. If your policy design returns 403, either scope the binding so the row is not found (preferred) or accept the sweep will flag it.

### Don't assume tenancy is always on

The default `enterActorContext()` calls `tenancy()->initialize()`. If your project doesn't use `stancl/tenancy` (single-tenant, org-scoped, or platform-admin impersonation flows), calling `tenancy()` throws a "helper not found" fatal. Override the four hooks on `tests/TestCase.php` before invoking the suite — there is no tenancy to enter in those topologies. The presence of a `TestsCrossTenantAccess.php` file does NOT imply multi-tenancy; "cross-tenant" is the historical name for "cross-boundary".

## When to use this

| Scenario | Required suite? | Boundary key |
|---|---|---|
| Multi-tenant per-tenant DB (`stancl/tenancy` multi-DB) | **Yes — mandatory** | tenant key |
| Multi-tenant schema / column-based | **Yes — even more critical** (binding needs manual scoping) | `tenant_id` column |
| Single-tenant, user-owned resources (`users/{user}/...`) | **Yes** | owning user id |
| Org/team-scoped (GitHub-style) | **Yes** | org/team id |
| Platform / central admin context (impersonation flows) | **Yes — impersonation Don't-impersonate self-service** | impersonated-vs-actor stamp |
| Single-tenant, all users share all rows | No boundary → no suite | — |
| Public read-only API (no auth) | No — but still enforce rate limits + input validation; modify verbs must 401/403 | — |

## Run requirements

```
pest --fail-on-risky --parallel tests/Feature/Security
```

Add the IDOR suite to the project's quality-gate checklist alongside `pest --parallel`, `phpstan`, `pint`, and `catraca`. A regression here is a **P0 vulnerability**, not a styling nit.
