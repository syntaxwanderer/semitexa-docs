# Get Started — Semitexa

> 🤖 **For AI Agents:** A structured, step-by-step version of this guide with explicit rules and mapping is in [Get Started (for AI Agents)](../ai/GET_STARTED.md).  
> **Also:** [About Semitexa](../../README.md) (vision and philosophy) · [AI Reference](../../AI_REFERENCE.md) (for agents).

This guide helps you **install and run a Semitexa application** with minimal steps. The goal is simplicity: get the framework up and running so you can start building.

---

### What you need

- **PHP 8.4** or later  
- **Composer** (to install dependencies)  
- **Docker and Docker Compose** (Semitexa runs on Swoole inside Docker — this is the only supported way to run the app)

---

### Get the project

You can either start from an existing Semitexa project (for example, the [semitexa.dev](https://github.com/semitexa/semitexa.dev) repo) or create a new one.

- **From an existing project:** clone the repository and go to *Install dependencies* below.  
- **From scratch:** create a new folder, run `composer init`, then add the meta-package (it pulls in the framework and everything you need):

  ```bash
  composer require semitexa/ultimate
  ```

  Then continue.

---

### Install dependencies

In the project root:

```bash
composer install
```

After this, the `bin/semitexa` CLI and all framework packages under `vendor/semitexa/` are available.

---

### Initialize project files (if needed)

If the project does not yet have `docker-compose.yml`, `.env.example`, or `server.php`, generate them:

```bash
bin/semitexa init
```

This creates the minimal Docker setup, environment example, and entry script so you can run the app.

---

### Set up environment

Copy the example environment file and adjust if needed:

```bash
cp .env.example .env
```

The default HTTP port for the app is **9502**; you can change it in `.env` (`SWOOLE_PORT`).

---

### Run the application

Start the server (via Docker):

```bash
bin/semitexa server:start
```

Open your browser at **http://0.0.0.0:9502** (or the port you set in `.env`).

To stop the server:

```bash
bin/semitexa server:stop
```

---

### If something goes wrong

- **"docker-compose.yml not found"** — run `bin/semitexa init` to generate the project structure (including Docker files).  
- **Port or logs** — see the core package docs: `vendor/semitexa/core/docs/RUNNING.md` (or the same file in the `semitexa-core` package in your repo).

---

### What’s next?

- **Your first page (Twig)** — add a minimal HTML page with a Twig template: [A minimal working page](MINIMAL_PAGE.md). That guide shows one route, one Payload, one Handler, and **rendering with Twig** (no JSON). Same Payload‑first flow; the handler sets the layout handle and context, and the framework renders the template.
- **Add more routes** — see `vendor/semitexa/core/docs/ADDING_ROUTES.md`.  
- **Understand the stack** — read [About Semitexa](../../README.md) and the rest of the docs in this package.

The main aim of this guide is to get you from zero to a running app with as little friction as possible. For more detail and machine-friendly steps, use the [AI-oriented Get Started](../ai/GET_STARTED.md) or the core package documentation.
