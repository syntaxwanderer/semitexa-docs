# A minimal working page — Semitexa

> 🤖 **For AI Agents:** A structured, step-by-step version with explicit rules and mapping is in [A minimal working page (for AI Agents)](../ai/MINIMAL_PAGE.md).  
> **Also:** [About Semitexa](../../README.md) (vision and philosophy) · [Get Started](GET_STARTED.md) (install and run).

This guide walks you through creating a **minimal working page** in Semitexa: one route, one Payload, one Handler, and an **HTML page rendered with Twig**. The main idea: the **Payload is the shield** — the single place where request data is accepted, filtered, and validated. By the time your handler runs, it only sees that Payload and can trust it.

---

### Payload as the shield

In Semitexa, every request is first shaped into a **Payload**. That Payload is the only door for that route: all input is accepted, validated, and normalized there. The handler never sees raw request data — it only receives the Payload instance. So you define the contract and validation once; after that, the handler can focus on logic and response. No back doors, no scattered checks.

---

### What we’re building

We’ll add a single route **GET /minimal?name=World** that returns an **HTML page rendered with Twig** and **validates** the query parameter `name` (1–100 characters). You’ll create:

- A **module** (folder + `composer.json`).
- A **Payload** — the shield: path, method, and response type.
- A **Resource** — the response type (here: Layout, for Twig).
- A **Twig template** — the page view (e.g. `minimal.html.twig`).
- A **Handler** — receives the Payload, sets the render context, and returns the response so the framework renders the template.

Order matters: **Payload first**, then Handler. So the flow is clear: shield first, then logic, then Twig.

---

### Step 1: Create the module

Create a directory for your module, for example **`src/modules/Website/`**. Inside it, add a **`composer.json`** so the framework recognises it as a Semitexa module:

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

Then run **`composer dump-autoload`** in the **project root**.

---

### Step 2: Create the Payload (the shield)

The Payload is the **single place of truth** for this route: path, methods, and which Resource is used. Put it in **`Application/Payload/Request/`** inside the module. Use **protected** properties and **getters/setters** so the framework can fill them from the query or body; implement **ValidatablePayload** so that invalid data never reaches the handler.

Example — **`src/modules/Website/Application/Payload/Request/MinimalPagePayload.php`** with one validated parameter **`name`** (e.g. **GET /minimal?name=World**):

```php
<?php

declare(strict_types=1);

namespace Semitexa\Modules\Website\Application\Payload\Request;

use Semitexa\Core\Attributes\AsPayload;
use Semitexa\Core\Contract\PayloadInterface;
use Semitexa\Core\Contract\ValidatablePayload;
use Semitexa\Core\Http\PayloadValidationResult;
use Semitexa\Core\Validation\Trait\LengthValidationTrait;
use Semitexa\Modules\Website\Application\Resource\MinimalPageResource;

#[AsPayload(path: '/minimal', methods: ['GET'], responseWith: MinimalPageResource::class)]
class MinimalPagePayload implements PayloadInterface, ValidatablePayload
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

- **Hydration:** the framework fills `name` from the query string (e.g. `?name=World`) via `setName()`.
- **Validation:** `name` must be between 1 and 100 characters. If it’s missing or too long, the handler is **never called** — the client gets **422 Unprocessable Entity** with a JSON body like `{ "errors": { "name": ["Length must be at least 1."] } }`. Only valid data passes through the shield.

---

### Step 3: Create the Resource

The Resource describes how the response is rendered (e.g. JSON or HTML). Put it in **`Application/Resource/`**. For an HTML page with Twig use **`ResponseFormat::Layout`** and a **handle** that matches your template name (e.g. `minimal` → `minimal.html.twig`).

Example — **`src/modules/Website/Application/Resource/MinimalPageResource.php`**:

```php
<?php

declare(strict_types=1);

namespace Semitexa\Modules\Website\Application\Resource;

use Semitexa\Core\Attributes\AsResource;
use Semitexa\Core\Http\Response\GenericResponse;
use Semitexa\Core\Http\Response\ResponseFormat;

#[AsResource(handle: 'minimal', format: ResponseFormat::Layout)]
class MinimalPageResource extends GenericResponse
{
}
```

---

### Step 4: Create the Twig template

Put the template in **`Application/View/templates/`** inside the same module. The file name must be **`{handle}.html.twig`** (here `minimal.html.twig`). Extend your module’s base layout so the page has a common shell (nav, footer). The template receives a **`response`** variable with the context the handler sets.

Example — **`src/modules/Website/Application/View/templates/minimal.html.twig`**:

```twig
{% extends "@project-layouts-Website/base.html.twig" %}
{% block title %}{{ response.title|default('Minimal page') }}{% endblock %}
{% block main %}
  <h1>{{ response.heading|default('Minimal page') }}</h1>
  <p>{{ response.message|default('')|raw }}</p>
{% endblock %}
{% block footer %}{{ response.footer|default('')|raw }}{% endblock %}
```

(If your module is not `Website`, replace `Website` in `@project-layouts-Website` with your module name. The base layout must exist in that module or be generated with `bin/semitexa layout:generate`.)

---

### Step 5: Create the Handler

The Handler receives the **Payload** (already validated) and the resource object. It does not need to check or parse the request again — it trusts the Payload. Put it in **`Application/Handler/Request/`**. You can type-hint both parameters as your Payload and Resource classes (e.g. `MinimalPagePayload`, `MinimalPageResource`); the framework only passes those instances, so no casts are needed. For Layout (Twig) responses the framework passes the Resource instance, which implements **LayoutRenderableInterface**. Set the render handle and context on it, then return it; the framework will render the template.

Example — **`src/modules/Website/Application/Handler/Request/MinimalPageHandler.php`**:

```php
<?php

declare(strict_types=1);

namespace Semitexa\Modules\Website\Application\Handler\Request;

use Semitexa\Core\Attributes\AsPayloadHandler;
use Semitexa\Core\Contract\HandlerInterface;
use Semitexa\Core\Contract\ResourceInterface;
use Semitexa\Modules\Website\Application\Payload\Request\MinimalPagePayload;
use Semitexa\Modules\Website\Application\Resource\MinimalPageResource;

#[AsPayloadHandler(payload: MinimalPagePayload::class, resource: MinimalPageResource::class)]
final class MinimalPageHandler implements HandlerInterface
{
    public function handle(MinimalPagePayload $payload, MinimalPageResource $resource): ResourceInterface
    {
        $name = $payload->getName();
        $message = 'Hello, ' . htmlspecialchars($name) . '!';
        $resource->setRenderHandle('minimal');
        $resource->setRenderContext([
            'title' => 'Minimal page',
            'heading' => 'Minimal page',
            'message' => $message,
            'footer' => 'Semitexa — Payload as the shield.',
        ]);
        return $resource;
    }
}
```

The handler simply uses the **validated** `name` — no `isset()`, no manual checks. If validation had failed, this code would not run. The framework then renders `minimal.html.twig` with that context (in Twig as `response.*`).

---

### Step 6: Sync the registry and reload

After adding or changing Payload classes, run:

```bash
bin/semitexa registry:sync:payloads
```

(or **`bin/semitexa registry:sync`**). This regenerates the route registry. Then restart the app (e.g. **`bin/semitexa server:stop`** and **`bin/semitexa server:start`**). Open **GET /minimal?name=World** — you should see an HTML page with “Hello, World!” rendered by Twig. Try **GET /minimal** without `name` or with a too-long value — you’ll get **422** and the handler won’t run.

---

### If something goes wrong

- **Route not found (404)?** Run **`bin/semitexa registry:sync:payloads`** and restart the app. Routes are built from generated classes in `src/registry/Payloads/`.
- **Class not found?** Ensure the module has a valid `composer.json` with `"type": "semitexa-module"` and run **`composer dump-autoload`** in the project root.
- **More details:** See `vendor/semitexa/core/docs/ADDING_ROUTES.md` for the full reference and HTML/Twig pages.

---

### What’s next?

- **More validation:** See `vendor/semitexa/core/docs/PAYLOAD_VALIDATION.md` for traits (`NotBlankValidationTrait`, `EmailValidationTrait`, etc.) and the full hydration/validation pipeline; invalid requests never reach the handler.
- **More routes:** Add more Payload + Resource + Handler + template in the same module; each route has its own shield (Payload).
- **Layouts and slots:** Same Payload → Handler flow with layout frames and `layout_slot()` — see `vendor/semitexa/core/docs/ADDING_ROUTES.md`.

The minimal page is the pattern: **Payload first (shield), Handler second (logic), Twig (HTML).** Everything that reaches your handler has passed through the Payload — that’s by design.
