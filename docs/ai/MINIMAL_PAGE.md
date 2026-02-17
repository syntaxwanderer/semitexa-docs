# A minimal working page — Semitexa (for AI Agents)

> 👤 **For humans:** A friendlier, narrative version of this guide is in [A minimal working page (for humans)](../hm/MINIMAL_PAGE.md).  
> **Also:** [About Semitexa](../../README.md) (vision) · [AI Reference](../../AI_REFERENCE.md) (philosophy) · [Get Started](GET_STARTED.md) (install & run). Full routes reference: `vendor/semitexa/core/docs/ADDING_ROUTES.md`.

This document describes how to create a **minimal working module** in Semitexa: one route, one Payload, one Handler, one response. It emphasizes that the **Payload is the shield** — the single place where request data is accepted, validated, and normalized; the handler only receives that Payload and can trust it.

---

## Purpose

- Create a minimal working page (one route) inside a Semitexa module.
- Establish the mental model: **Payload first** (contract + validation), **Handler second** (logic + response).
- Provide unambiguous steps and file paths so agents and scripts can reproduce the result.

---

## Principle: Payload as the shield

In Semitexa, the **Payload is the only door** for that request. All incoming data is accepted, filtered, and validated in the Payload. The handler never sees raw, unchecked input — it only ever receives the Payload instance. So you define the contract and validation once in the Payload; from then on the handler can trust the data. No back doors, no scattered checks.

---

## Scope / When to use

- Adding the first page (or a new page) in a new or existing module.
- Demonstrating or following the Payload → Handler → Response flow.
- When you need the exact file layout and attribute usage for a minimal route.

---

## Prerequisites

- A Semitexa project already installed and runnable (see [Get Started](GET_STARTED.md)).
- New routes live **only in modules** (`src/modules/`, `packages/`, or `vendor/`); not in project `src/` (namespace `App\`).

---

## Steps

### 1. Create the module directory

Example: `src/modules/Website/` (or `Minimal`, `App`, etc.). All new routes for this “app” will live in this module.

### 2. Add `composer.json` inside the module

So the framework recognises it as a Semitexa module and registers autoload:

```json
{
  "name": "semitexa/module-website",
  "type": "semitexa-module",
  "autoload": {
    "psr-4": {
      "Semitexa\\Modules\\Website\\": "."
    }
  }
}
```

Run **`composer dump-autoload`** in the **project root** after adding or changing this file.

### 3. Create the Payload (the shield)

Put the HTTP request DTO in **`Application/Payload/Request/`** with namespace `Semitexa\Modules\{ModuleName}\Application\Payload\Request\`. This class defines path, methods, and which Resource is used for the response. It is the **single place of truth** for this route’s input contract. Use **protected** properties and **getters/setters** so the framework can hydrate from query/body; implement **ValidatablePayload** so invalid data never reaches the handler.

Example — `src/modules/Website/Application/Payload/Request/MinimalPagePayload.php` with one validated parameter `name` (e.g. **GET /minimal?name=World**):

```php
<?php

declare(strict_types=1);

namespace Semitexa\Modules\Website\Application\Payload\Request;

use Semitexa\Core\Attributes\AsPayload;
use Semitexa\Core\Contract\RequestInterface;
use Semitexa\Core\Contract\ValidatablePayload;
use Semitexa\Core\Http\PayloadValidationResult;
use Semitexa\Core\Validation\Trait\LengthValidationTrait;
use Semitexa\Modules\Website\Application\Resource\MinimalPageResource;

#[AsPayload(path: '/minimal', methods: ['GET'], responseWith: MinimalPageResource::class)]
class MinimalPagePayload implements RequestInterface, ValidatablePayload
{
    use LengthValidationTrait;

    protected string $name = '';

    public function getName(): string
    {
        return $this->name;
    }

    public function setName(string $name): void
    {
        $this->name = trim($name);
    }

    public function validate(): PayloadValidationResult
    {
        $errors = [];
        $this->validateLength('name', $this->name, 1, 100, $errors);
        return new PayloadValidationResult(empty($errors), $errors);
    }
}
```

- **Hydration:** the framework fills `name` from the query (e.g. `?name=World`) via `setName()`.
- **Validation:** `name` must be 1–100 characters. If validation fails, the handler is **not** called and the client receives **422 Unprocessable Entity** with `{ "errors": { "name": ["Length must be at least 1."] } }` (or similar). Only valid data reaches the handler.

### 4. Create the Resource (response type)

Put the Response DTO in **`Application/Resource/`**. For a minimal JSON response you can use a small class with `AsResource` and `ResponseFormat::Json`.

Example — `src/modules/Website/Application/Resource/MinimalPageResource.php`:

```php
<?php

declare(strict_types=1);

namespace Semitexa\Modules\Website\Application\Resource;

use Semitexa\Core\Attributes\AsResource;
use Semitexa\Core\Http\Response\GenericResponse;
use Semitexa\Core\Http\Response\ResponseFormat;

#[AsResource(handle: 'minimal-page', format: ResponseFormat::Json)]
class MinimalPageResource extends GenericResponse
{
}
```

### 5. Create the Handler

Put the handler in **`Application/Handler/Request/`** with namespace `Semitexa\Modules\{ModuleName}\Application\Handler\Request\`. It receives the **Payload** (already validated) and the response object; it must return a `ResponseInterface`. It does not need to re-validate input — it trusts the Payload.

Example — `src/modules/Website/Application/Handler/Request/MinimalPageHandler.php`:

```php
<?php

declare(strict_types=1);

namespace Semitexa\Modules\Website\Application\Handler\Request;

use Semitexa\Core\Attributes\AsPayloadHandler;
use Semitexa\Core\Contract\HandlerInterface;
use Semitexa\Core\Contract\RequestInterface;
use Semitexa\Core\Contract\ResponseInterface;
use Semitexa\Core\Response;
use Semitexa\Modules\Website\Application\Payload\Request\MinimalPagePayload;
use Semitexa\Modules\Website\Application\Resource\MinimalPageResource;

#[AsPayloadHandler(payload: MinimalPagePayload::class, resource: MinimalPageResource::class)]
final class MinimalPageHandler implements HandlerInterface
{
    public function handle(RequestInterface $request, ResponseInterface $response): ResponseInterface
    {
        $payload = (MinimalPagePayload) $request;
        $name = $payload->getName();
        return Response::json([
            'page' => 'minimal',
            'message' => 'Hello, ' . $name . '!',
        ]);
    }
}
```

The handler uses the Payload’s **validated** `name` only — no `isset()`, no re-validation. If the request had failed validation, this code would not run.

### 6. Sync the registry

After adding or changing Payload classes, run:

```bash
bin/semitexa registry:sync:payloads
```

(or **`bin/semitexa registry:sync`** to sync payloads and contracts). Routes are generated in `src/registry/Payloads/`; without this step the new route will not exist. `composer install` / `composer update` run this automatically.

### 7. Reload the app

Restart the application (e.g. `bin/semitexa server:stop` then `bin/semitexa server:start`) so the new route is discovered. Then open **GET /minimal?name=World** (or your chosen path). Without `name` or with invalid length the client gets **422** and the handler is not run.

---

## Rules & constraints

- **Payload first:** Define path, methods, and response type in the Payload. Put validation in the Payload so the handler only sees valid data.
- **Handler trusts Payload:** Do not re-validate or parse raw request in the handler; use the Payload instance only.
- **Modules only:** Do not add new routes in project `src/` (namespace `App\`). Use a module under `src/modules/`, `packages/`, or an installed package.
- **After Payload changes:** Always run **`bin/semitexa registry:sync:payloads`** (or **`bin/semitexa registry:sync`**) so routes are regenerated.

---

## Mapping (where to read more)

| Goal | Document or command |
|------|----------------------|
| Add routes / module layout | `vendor/semitexa/core/docs/ADDING_ROUTES.md` |
| Payload validation (traits, 422, hydration) | `vendor/semitexa/core/docs/PAYLOAD_VALIDATION.md` |
| Payload/Handler attributes | `vendor/semitexa/core/docs/attributes/` |
| HTML pages (Twig, layouts) | `vendor/semitexa/core/docs/ADDING_ROUTES.md` (Responses: JSON and HTML) |
| Module structure (Payload/Handler/Resource) | `vendor/semitexa/core/docs/MODULE_STRUCTURE.md` (if present), ADDING_ROUTES.md |

---

## Summary for agents

1. **Module:** Create `src/modules/{Name}/`, add `composer.json` with `"type": "semitexa-module"` and PSR-4; run `composer dump-autoload`.
2. **Payload (shield):** `Application/Payload/Request/{Name}Payload.php` — `#[AsPayload(path, methods, responseWith)]`, implements `RequestInterface` and `ValidatablePayload`. Use protected properties + getters/setters for hydration; `validate()` returns `PayloadValidationResult`. Use traits e.g. `LengthValidationTrait`, `NotBlankValidationTrait` (see `vendor/semitexa/core/docs/PAYLOAD_VALIDATION.md`).
3. **Resource:** `Application/Resource/{Name}Resource.php` — `#[AsResource(handle, format)]`, extends `GenericResponse`.
4. **Handler:** `Application/Handler/Request/{Name}Handler.php` — `#[AsPayloadHandler(payload, resource)]`, implements `HandlerInterface`; `handle(RequestInterface, ResponseInterface): ResponseInterface`; trust the Payload, return response.
5. **Sync:** `bin/semitexa registry:sync:payloads` (or `registry:sync`).
6. **Reload** the app and hit the route.

Payload = single place of truth for that request. Handler = logic and response only.
