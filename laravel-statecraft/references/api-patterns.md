# API Patterns Reference

## Overview

Production Laravel APIs need predictable, safe, and observable behavior beyond domain logic. This reference covers three critical API-level concerns: **error responses (Problem+JSON)**, **idempotency**, and **route versioning**.

---

## Error Responses (RFC 9457 Problem+JSON)

APIs must return a **single, predictable error shape** for every failure. When errors are inconsistent, client-side error handling becomes a map of special cases — one endpoint uses `errors`, another `error`, another `message`, another a `200` with `success: false`. Every special case is a place the client goes wrong. Consistent errors mean one client-side function that reads `status`, pulls `detail`, maybe extracts `errors` on 422s. That function works for every endpoint.

Use **RFC 9457 Problem+JSON** with these fields:

| Field | Type | Description |
|---|---|---|
| `type` | `string` | URI identifying the error type |
| `title` | `string` | Short, human-readable title |
| `status` | `int` | HTTP status code (mirrors the actual response status) |
| `detail` | `string` | Human-readable explanation specific to this occurrence |
| `errors` | `array` | *(optional)* Per-field validation errors: `field => string[]` |

```json
{
    "type": "https://httpstatuses.com/422",
    "title": "Unprocessable Entity",
    "status": 422,
    "detail": "Cannot pay invoice from status 'draft'",
    "errors": {
        "status": ["Invalid transition from draft to paid."]
    }
}
```

### Exception Configuration in bootstrap/app.php

In Laravel 11+, exceptions are handled through `bootstrap/app.php` using `->withExceptions()`. There is no separate `Handler.php` class — the configuration lives alongside everything else in the application bootstrap.

```php
// bootstrap/app.php
->withExceptions(function (Exceptions $exceptions) {
    $exceptions->render(function (ValidationException $e) {
        return problem(
            status: 422,
            title: 'Unprocessable Entity',
            detail: 'The request data did not pass validation.',
            extra: ['errors' => $e->errors()],
        );
    });

    $exceptions->render(function (AuthenticationException $e) {
        return problem(
            status: 401,
            title: 'Unauthorized',
            detail: 'Authentication is required to access this resource.',
        );
    });

    $exceptions->render(function (ModelNotFoundException $e) {
        return problem(
            status: 404,
            title: 'Not Found',
            detail: 'The requested resource does not exist.',
        );
    });

    $exceptions->render(function (NotFoundHttpException $e) {
        return problem(
            status: 404,
            title: 'Not Found',
            detail: 'The requested endpoint does not exist.',
        );
    });

    $exceptions->render(function (HttpException $e) {
        return problem(
            status: $e->getStatusCode(),
            title: title_for_status($e->getStatusCode()),
            detail: $e->getMessage() ?: title_for_status($e->getStatusCode()),
        );
    });

    $exceptions->render(function (Throwable $e) {
        if (!app()->environment('production')) {
            return null;
        }

        return problem(
            status: 500,
            title: 'Internal Server Error',
            detail: 'An unexpected error occurred. Please try again later.',
        );
    });
})
```

The `Content-Type` header on every error response is `application/problem+json`, not `application/json`. This is part of the RFC 9457 spec, and it matters for clients that want to detect Problem+JSON responses programmatically. The fallback `Throwable` handler returns `null` in non-production environments, which lets Laravel's default exception rendering show the full stack trace during development. In production, it returns a generic 500. Internal exception details must never leak into production API responses. The render callbacks are evaluated in registration order, and Laravel uses the first match — specific exception types are registered before the generic `HttpException` handler, which is before the catch-all `Throwable` handler. Order matters.

### The `problem()` Helper and `title_for_status()`

These live in `app/Support/helpers.php` (or any autoloaded file of your choosing):

```php
<?php

declare(strict_types=1);

use Illuminate\Http\JsonResponse;

function problem(
    int $status,
    string $title,
    string $detail,
    array $extra = [],
): JsonResponse {
    return response()->json(array_merge([
        'type' => 'https://httpstatuses.com/' . $status,
        'title' => $title,
        'status' => $status,
        'detail' => $detail,
    ], $extra), $status, [
        'Content-Type' => 'application/problem+json',
    ]);
}

function title_for_status(int $status): string
{
    return match ($status) {
        400 => 'Bad Request',
        401 => 'Unauthorized',
        403 => 'Forbidden',
        404 => 'Not Found',
        405 => 'Method Not Allowed',
        409 => 'Conflict',
        422 => 'Unprocessable Entity',
        429 => 'Too Many Requests',
        500 => 'Internal Server Error',
        503 => 'Service Unavailable',
        default => 'Error',
    };
}
```

### Domain Exceptions as First-Class Errors

Domain exceptions are not unexpected failures — they are known, named outcomes of domain operations. `InvalidKeyTransitionException` when a key transition is attempted that the lifecycle does not allow. `InvalidKeyScopeException` when an operation is attempted with an insufficient scope. They should produce informative 422 or 403 responses, not 500s.

Register them in `withExceptions` alongside the framework exceptions:

```php
$exceptions->render(function (InvalidKeyTransitionException $e) {
    return problem(
        status: 422,
        title: 'Invalid State Transition',
        detail: $e->getMessage(),
    );
});

$exceptions->render(function (InvalidKeyScopeException $e) {
    return problem(
        status: 403,
        title: 'Forbidden',
        detail: $e->getMessage(),
    );
});
```

The exception carries the human-readable explanation of what went wrong, and the handler maps it to the right HTTP status and Problem+JSON shape. The action class that throws `InvalidKeyTransitionException` does not need to know anything about HTTP — it throws a domain exception, the handler turns it into a contract-compliant response.

### Domain Exception to HTTP Status Mapping

| Domain Exception | HTTP Status | Rationale |
|---|---|---|
| `InvalidTransitionException` | `422` | Well-formed but semantically invalid action |
| `InvalidKeyScopeException` | `403` | Insufficient scope for operation |
| `NotFoundException` / `ModelNotFoundException` | `404` | Resource does not exist |
| `ValidationException` | `422` | Input fails business rules |
| `AuthenticationException` | `401` | Caller not authenticated |
| `AuthorizationException` | `403` | Caller authenticated but not authorised |
| `DomainException` (catch-all) | `422` | Generic business rule violation |
| Everything else | `500` | Unexpected programmer error |

**Rule:** Domain exceptions are **not unexpected failures**. They are named, known outcomes and deserve an informative `422` (not a generic `500`).

### Testing the Error Layer

Error shapes are part of your contract — they deserve the same test coverage as success paths. In Pest, a small helper reduces repetition:

```php
// tests/Support/assertions.php
function assertProblemJson(
    \Illuminate\Testing\TestResponse $response,
    int $status,
    string $title,
): void {
    $response
        ->assertStatus($status)
        ->assertHeader('Content-Type', 'application/problem+json')
        ->assertJsonStructure(['type', 'title', 'status', 'detail'])
        ->assertJsonFragment(['status' => $status, 'title' => $title]);
}
```

Then in feature tests:

```php
it('returns a problem json response for invalid scope', function () {
    $key = ApiKey::factory()->create([
        'scopes' => [KeyScope::Read],
        'status' => KeyStatus::Active,
    ]);

    $response = $this->withToken($key->raw_value)
        ->postJson('/api/v1/keys');

    assertProblemJson($response, 403, 'Forbidden');
});

it('returns per-field validation errors', function () {
    $response = $this->actingAs(User::factory()->create())
        ->postJson('/api/v1/keys', [
            'name' => '',
            'scopes' => ['invalid_scope'],
        ]);

    assertProblemJson($response, 422, 'Unprocessable Entity');
    $response->assertJsonPath('errors.name.0', 'The name field is required.');
});

it('returns a 404 for missing keys', function () {
    $response = $this->actingAs(User::factory()->create())
        ->getJson('/api/v1/keys/' . Str::ulid());

    assertProblemJson($response, 404, 'Not Found');
});
```

The assertions are explicit and readable. Because the tests run against the actual exception configuration, they catch regressions if the handler's behaviour changes.

---

## Idempotency Keys

### The Problem

A network drop after you return `201 Created` but before the client receives it means the client retries. Without idempotency protection, you happily process the same operation twice.

### The Solution

Accept a client-generated `Idempotency-Key` header on state-changing endpoints. Store the original response and replay it on any retry that carries the same key within a reasonable window.

### Middleware Implementation

```php
<?php
declare(strict_types=1);

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Cache;
use Symfony\Component\HttpFoundation\Response;

final class IdempotencyKey
{
    public function handle(Request $request, Closure $next): Response
    {
        $key = $request->header('Idempotency-Key');

        if ($key === null) {
            return $next($request);
        }

        $cacheKey = 'idempotency:' . $key;

        // Check if we already processed this key
        if (Cache::has($cacheKey)) {
            return response()->json(
                Cache::get($cacheKey . ':body'),
                Cache::get($cacheKey . ':status', 200)
            );
        }

        $response = $next($request);

        // Only cache successful state-changing responses (2xx)
        if ($response->isSuccessful()) {
            Cache::put($cacheKey, true, now()->addDay());
            Cache::put($cacheKey . ':status', $response->getStatusCode(), now()->addDay());
            Cache::put($cacheKey . ':body', $response->getContent(), now()->addDay());
        }

        return $response;
    }
}
```

### Rules

- Only apply to **state-changing** endpoints (`POST`, `PUT`, `PATCH`, `DELETE`).
- `Idempotency-Key` header is **optional** but **recommended** for write operations.
- Keys should be unique (UUID v4 recommended).
- Cache the response for a reasonable window (default: 24 hours).
- Do NOT cache non-successful responses.
- Include the idempotency key in your API documentation and client SDKs.

### Testing

```php
it('returns cached response on idempotency key replay', function () {
    $key = (string) Str::uuid();

    $response1 = postJson('/api/v1/keys', [
        'name' => 'Production API Key',
    ], ['Idempotency-Key' => $key]);

    expect($response1->status())->toBe(201);

    $response2 = postJson('/api/v1/keys', [
        'name' => 'Production API Key',
    ], ['Idempotency-Key' => $key]);

    expect($response2->status())->toBe(201)
        ->and($response2->json('id'))->toBe($response1->json('id')); // Same resource returned
});
```

---

## Route Versioning

### Version from Day One

Adding `/v1` to a production API is itself a breaking change (every consumer must update their URLs). Version from day one so the prefix is free.

### Version Is Config-Driven, Never Hardcoded

The current API version lives in **one place**: `config/app.php`, read from `env()` with a typed `config()` call and a `v1` default. It must never appear as a string literal in a route file, a controller, a URL generator, or a route name. Setting it once in config means a typo cannot silently route half your API to the wrong prefix, and changing it is a one-line deploy (no code edit, no missed file).

```php
// config/app.php
'actual_version' => env('APP_ACTUAL_VERSION', 'v1'),
```

```dotenv
# .env
APP_ACTUAL_VERSION=v1
```

```php
// routes/api.php
// Directive 19 — typed config() helper, default 'v1'.
$actualVersion = config()->string('app.actual_version', 'v1');

// "current" routes — the ones that have not been superseded — use $actualVersion.
Route::prefix($actualVersion)->group(function () {
    Route::post('/keys', \App\Http\Controllers\Keys\V1\IssueKeyController::class);
    Route::delete('/keys/{key}', \App\Http\Controllers\Keys\V1\RevokeKeyController::class);
    Route::post('/invoices', \App\Http\Controllers\Billing\V1\CreateInvoiceController::class);
});

// When v2 ships, the **previous** version becomes an explicit, literal
// prefix block (it is now pinned history, not "current"), and
// $actualVersion moves to v2 for the new code. Never edit a retired
// version's block after it ships — that's the whole point.
//
// Route::prefix('v1')->group(function () {
//     Route::post('/keys', \App\Http\Controllers\Keys\V1\IssueKeyController::class);
// });
//
// Route::prefix($actualVersion)->group(function () {
//     Route::post('/keys', \App\Http\Controllers\Keys\V2\IssueKeyController::class);
// });
```

**Rules:**
- `$actualVersion` is read **once** at the top of every route file from `config()->string('app.actual_version', 'v1')` — never `'v1'` as a literal in a route. The literal only lives in `config/app.php` and `.env`.
- All new, unversioned "current" routes belong under `$actualVersion`.
- When a v2 endpoint differs from v1, pin the v1 block with a literal `'v1'` prefix (it is now retired history, not the current version) and add a new `$actualVersion` block for v2 — never modify a retired block.
- The config key makes it trivial to find which version is "current" across every route file and prevents typo bugs (`'v1'` vs `'V1'` vs `'v01'`) — config is the single source of truth.
- The `config()->string(...)` form (not `config('app.actual_version')`) is **mandatory** per Directive 19 — the typed accessor is itself the version-source-of-truth contract.
- You can group all routes under the correct version using `prefix`. The config key on top tells the AI agent which is the latest version — only one place to bump, no risk of leaving a stale literal somewhere.

### Namespace by Version — Models, Controllers, Request, Resources, Actions, Jobs

The version boundary must be **visible in the file tree**, not just in the URL. The same concept that can host two `IssueKeyController`s without collision must also host two `ApiKey` models, two `IssueKeyData` payloads, and two `IssueKey` actions. **Bounded Context already owns every other layer; version sits *inside* the context, as the innermost namespace segment**, so grouping remains by context first and version second — never version-as-top-level-folder.

```
app/
├── Models/{Context}/V{N}/                 # e.g. app/Models/Keys/V1/ApiKey.php
├── Enums/{Context}/V{N}/
├── Data/{Context}/V{N}/
├── Actions/{Context}/V{N}/
├── Listeners/{Context}/V{N}/
├── Jobs/{Context}/V{N}/
├── Services/{Context}/V{N}/
└── Http/
    ├── Controllers/{Context}/V{N}/         # e.g. app/Http/Controllers/Keys/V1/IssueKeyController.php
    ├── Requests/{Context}/V{N}/
    └── Resources/{Context}/V{N}/
```

Example versioned namespaces:

```
App\Models\Keys\V1\ApiKey
App\Models\Keys\V2\ApiKey                  # different invariants, different file
App\Enums\Keys\V1\ApiKeyStatus
App\Data\Keys\V1\IssueKeyData
App\Actions\Keys\V1\IssueKey
App\Actions\Keys\V2\IssueKey                # different behaviour, different file
App\Http\Controllers\Keys\V1\IssueKeyController
App\Http\Controllers\Keys\V2\IssueKeyController
App\Http\Requests\Keys\V1\IssueKeyRequest
App\Http\Resources\Keys\V1\ApiKeyResource
```

This makes the version boundary **visible in code and in the file tree**. You physically cannot put v2 logic in a v1 controller or model because they are separate files in separate folders. When a breaking change is needed, you clone the slice of the Bounded Context you touch into a new `V{N}` folder and leave the original untouched — the same model, action, and controller pattern duplicated at the version level, mirroring the Bounded-Context split itself.

### The Sunset Pattern

Versioning lets you introduce breaking changes without breaking existing integrations — but versions accumulate. If you never retire old versions, you maintain v1, v2, v3 indefinitely. The Sunset pattern (RFC 8594) solves this: a standardised way to tell consumers a version is approaching retirement, giving them time and information to migrate before it disappears.

The `Sunset` header carries the retirement date. The `Deprecation` header marks when the version was officially deprecated. Together they give consumers a timeline:

```
Deprecation: Mon, 01 Jul 2077 00:00:00 GMT
Sunset: Sat, 31 Dec 2077 23:59:59 GMT
```

The `HttpSunset` middleware adds these headers to deprecated route groups, receiving its dates through Laravel's route middleware parameters:

```php
<?php

declare(strict_types=1);

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

final class HttpSunset
{
    public function handle(
        Request $request,
        Closure $next,
        string $deprecationDate,
        string $sunsetDate,
    ): Response {
        $response = $next($request);

        $response->headers->set('Deprecation', $deprecationDate);
        $response->headers->set('Sunset', $sunsetDate);
        $response->headers->set(
            'Link',
            '</docs/migration/v1-to-v2>; rel="successor-version"'
        );

        return $response;
    }
}
```

Apply it to a deprecated route group by passing the dates as middleware parameters:

```php
Route::prefix('v1')
    ->middleware([
        'sunset:Mon\, 01 Jul 2077 00:00:00 GMT,Sat\, 31 Dec 2077 23:59:59 GMT',
    ])
    ->group(function () {
        require __DIR__ . '/api/v1/keys.php';
    });
```

Register the alias in `bootstrap/app.php`:

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->alias([
        'sunset' => \App\Http\Middleware\HttpSunset::class,
    ]);
})
```

The `Link` header pointing to a migration guide is the part teams most often skip — and it matters. A consumer whose monitoring picks up a `Sunset` header can follow the link directly to migration docs. The header becomes actionable, not just informational. Well-behaved API clients log warnings, alert the team, open tickets. The API communicates the timeline; the consumer has the runway to act on it.

**Rule:** No version retires without at least **six months of notice**.

### Version Decision Matrix

| Change | Strategy | Example |
|---|---|---|
| Add field | Any version | `email_verified_at` added to `UserResource` |
| Change field meaning | New version | `status` now uses different enum values |
| Remove field | New version with Sunset on old | Remove `legacy_id` from `OrderResource` |
| Change pagination | New version with Sunset on old | Offset → Cursor |
| Fix bug (not a breaking change) | Same version | Correct `total` calculation |
