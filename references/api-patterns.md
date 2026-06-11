# API Patterns Reference

## Overview

Production Laravel APIs need predictable, safe, and observable behavior beyond domain logic. This reference covers two critical API-level concerns: **idempotency** and **route versioning**.

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

### Namespace Controllers by Version

```
App\Http\Controllers\Keys\V1\IssueKeyController
App\Http\Controllers\Keys\V2\IssueKeyController
```

This makes the version boundary **visible in code**. You physically cannot put v2 logic in a v1 controller because they are separate files.

### Route Registration

Use a version variable so the current API version is defined in **one place**. When v2 arrives, you add a new block — the v1 block stays untouched.

```php
// routes/api.php
$actualVersion = 'v1';

Route::prefix($actualVersion)->group(function () {
    Route::post('/keys', \App\Http\Controllers\Keys\V1\IssueKeyController::class);
    Route::delete('/keys/{key}', \App\Http\Controllers\Keys\V1\RevokeKeyController::class);
    Route::post('/invoices', \App\Http\Controllers\Billing\V1\CreateInvoiceController::class);
});

// When v2 ships, add a second block — v1 stays as-is — change the $actualVersion to new version and old as hardcoded or create a new var 
// Route::prefix('v1')->group(function () {
//     Route::post('/keys', \App\Http\Controllers\Keys\V1\IssueKeyController::class);
// });

// Route::prefix($actualVersion)->group(function () {
//     Route::post('/keys', \App\Http\Controllers\Keys\V2\IssueKeyController::class);
// });
```

**Rules:**
- `$actualVersion` is set **once** at the top of the file.
- All new unversioned routes belong under `$actualVersion`.
- When a v2 endpoint differs from v1, add an explicit `'v2'` prefix block — never modify the v1 block.
- The variable makes it trivial to find which version is "current" and prevents typo bugs (`'v1'` vs `'V1'` vs `'v01'`).
- You can group all routes under the correct version using `prefix`. The variable on top will tell the AI agent which is the latest version.

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
