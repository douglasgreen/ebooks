# Book Outline: *Symfony in Depth: From First Request to Production*

**Working title:** *Symfony in Depth: From First Request to Production*
**Target audience:** Developers with solid PHP (8.2+) and basic web knowledge who want to build professional applications with Symfony 7.4 LTS / 8.x.
**Pedagogical approach:** Each chapter ends with exercises; a running project (a multi-tenant SaaS invoicing app) is built incrementally across Parts III–V, then hardened in Part VI.

---

# Front Matter

## Who This Book Is For

This book is written for developers who already know PHP and want to move from writing scripts and small websites to building professional, maintainable applications with Symfony — the framework that powers a large share of the PHP web, from SaaS platforms to e-commerce engines to government services.

You will get the most out of this book if you:

- Have built real things in PHP (not just "Hello World") and are comfortable with modern language features such as typed properties, enums, readonly classes, and attributes.
- Understand how the web works at a basic level: what happens between a browser sending a request and a server sending a response, and why status codes and headers matter.
- Want to understand *why* Symfony is designed the way it is — dependency injection, events, bundles — not just how to copy-paste recipes.

This book is **not** a PHP tutorial. If terms like "interface," "inheritance," or "HTTP POST" are unfamiliar, spend a week with a good PHP 8 book first and come back. It is also not a quick-start: if you want a working app in twenty minutes, the official Symfony documentation's "Getting Started" guide will serve you better. Here, we trade speed for depth — every feature is introduced with the context you need to use it well in production.

## Prerequisites

| Requirement | Minimum | Recommended |
|---|---|---|
| PHP | 8.2 (the floor for Symfony 7.4 LTS) | The latest stable release (8.3 or 8.4), so you can follow along on both the 7.4 and 8.x lines |
| Composer | 2.x | Latest 2.x |
| HTTP knowledge | Methods, status codes, headers, cookies | Familiarity with REST conventions helps in Part V |
| OOP | Classes, interfaces, inheritance, basic dependency injection | Experience with any DI container is a bonus |
| Tooling | A terminal, a code editor (VS Code, PhpStorm, or similar), Git basics | Docker (used in Chapter 24) |

No prior Symfony experience is assumed. If you have used another framework — Laravel, CakePHP, or even Spring — you'll find the concepts transfer well; we flag the differences where they matter.

## How the Book Is Organized

The book is divided into seven parts that follow the lifecycle of a real application: from understanding what Symfony *is*, to building features, to exposing APIs, to shipping and operating in production.

- **Part I — Foundations (Chapters 1–4).** What Symfony actually is (components vs. bundles vs. framework), how it models HTTP with `Request`/`Response`, how routes map URLs to code, and the full request lifecycle from kernel boot to response.
- **Part II — Core Architecture (Chapters 5–8).** The machinery underneath: dependency injection, the configuration system, events, and how bundles extend the framework. This is the part that separates people who use Symfony from people who understand it.
- **Part III — Building Web Applications (Chapters 9–14).** The daily-driver toolkit: Twig templates, forms, validation, security, Doctrine ORM, and frontend asset integration.
- **Part IV — Beyond the Browser (Chapters 15–18).** Console commands, email, asynchronous processing with Messenger, and scheduled jobs and webhooks — the parts of your app that run without a user watching.
- **Part V — APIs (Chapters 19–21).** REST APIs with the Serializer component, API Platform for resource-driven APIs, and stateless authentication patterns.
- **Part VI — Quality and Production (Chapters 22–24).** Testing strategies, debugging and performance tooling, caching, and deployment: web servers, Docker, CI/CD, and monitoring.
- **Part VII — Advanced Topics (Chapters 25–28).** Workflows for state machines, specialized components (UID, Lock, Semaphore, RateLimiter), internationalization, and how to contribute to Symfony itself.

Four appendices round out the book: a cheat sheet (A), a component reference table with use cases (B), a glossary (C), and further resources (D).

**The running project.** Starting in Part III, we build a single application incrementally: a multi-tenant SaaS invoicing app (working title: *InvoiceHub*) where organizations manage customers, invoices, and payments. Each chapter adds a real slice of it — the login system in Chapter 12, the invoice entities in Chapter 13, the email notifications in Chapter 16, the public API in Part V — and Part VI hardens the whole thing with tests, caching, and a production deployment. You can follow along chapter by chapter, or use the project as a reference architecture for your own work.

**Exercises.** Every chapter ends with exercises ranging from quick checks to open-ended challenges (marked ⭐ when they stretch you). Solutions are available in the book's companion repository.

**Suggested paths.** New to Symfony? Read straight through — the parts are sequenced so each builds on the last. Already comfortable with Symfony and here for a specific topic? Part II is worth skimming regardless, then jump ahead: API work lives in Part V (with Chapters 5, 12, and 13 as prerequisites), and deployment questions are answered in Chapter 24.

## Conventions

**Code style.** All code follows PSR-12 and uses `declare(strict_types=1)`. We write modern PHP: constructor property promotion, readonly properties, enums, first-class callable syntax, and attributes — the same style you should adopt in your own projects on PHP 8.2+.

**Configuration: attributes first.** Where Symfony offers a choice of configuration formats, we use **PHP attributes** for anything attached to code (routes, controllers, event subscribers, validation constraints, Doctrine mappings) and **YAML** under `config/` for bundle and service configuration — the default layout of the current project skeleton. XML configuration was deprecated in Symfony 7.4 and appears nowhere in this book; PHP array configuration is supported and mentioned where relevant, but YAML keeps our listings readable.

**Notation.**
- `Inline code` marks class names, methods, files, commands, and configuration keys.
- **Bold** marks user interface elements (buttons, menu items).
- Four kinds of margin boxes appear throughout: **Tip** (a useful shortcut), **Note** (context worth knowing), **Caution** (a common pitfall), and **Symfony 8** (behavior that differs between the 7.4 LTS line and the 8.x line — read these if you're on either end of that range).

**Code listings.** Listings are trimmed for clarity; ellipses (`// …`) mark omitted code, and comments in the margin explain non-obvious lines. The complete running project, plus all exercise solutions, is available in the companion GitHub repository (see Appendix D), tagged per chapter so you can check out exactly the state the book describes at any point.

## Setting Up Your Environment

You'll need three tools: PHP, Composer, and the Symfony CLI. All are free and cross-platform; instructions below cover macOS and Linux, with notes for Windows.

### 1. Install PHP

Check whether you already have a suitable version:

```bash
php -v
```

You need **8.2 or higher**; we recommend the latest stable release (8.3 or 8.4) so the book works on both Symfony lines. If you don't have it, or your version is old:

- **macOS:** `brew install php` (Homebrew keeps you current; `brew upgrade php` later).
- **Debian/Ubuntu:** `sudo apt install php-cli php-xml php-mbstring php-curl php-sqlite3 php-intl` — or, better, use a version manager such as [shivammathur/php](https://github.com/shivammathur/php) via Docker so you can switch PHP versions per project.
- **Windows:** Download the zip from php.net, or use Laravel Herd / XAMPP. The Symfony CLI also bundles a portable PHP if you're in a pinch.

### 2. Install Composer

Composer is PHP's package manager — think of it as `npm` for PHP. Verify with `composer --version`; install it via your package manager (`brew install composer`, `sudo apt install composer`) or the official installer:

```bash
curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
```

### 3. Install the Symfony CLI

The [Symfony CLI](https://symfony.com/download) is a small binary that makes day-to-day work dramatically easier: it creates projects, runs a development server with automatic reloads, opens the console, and can expose your local app over a secure tunnel. Install it with:

```bash
curl -sS https://get.symfony.com/cli/installer | php
```

(On Windows, `winget install SymfonyCLI` works too.) Verify with `symfony --version`. The CLI is optional — everything it does can be done with PHP and Composer alone — but the rest of this book assumes you have it.

### 4. Create Your First Project

Let's create a throwaway project to confirm your setup:

```bash
symfony new hello-symfony --full
cd hello-symfony
symfony serve
```

`--full` installs the "full recipe": Doctrine, Twig, forms, validation, security, and testing tools — the same starting point we use for the running project in Part III. `symfony serve` starts a development server; open <http://localhost:8000> and you should see Symfony's welcome page.

While it's running, try the console from a second terminal:

```bash
bin/console about
```

That command prints your PHP version, Symfony version, environment, and key configuration — it will become your first diagnostic tool. When you're done, press `Ctrl+C` in the server terminal.

### 5. Tour the Directory Layout

Here is what `symfony new --full` gives you, annotated:

```
hello-symfony/
├── bin/
│   └── console              # Entry point for all CLI commands
├── config/
│   ├── bundles.php          # Which bundles are enabled (per environment)
│   ├── packages/            # One YAML file per bundle: framework.yaml,
│   │                        #   doctrine.yaml, security.yaml, …
│   ├── routes/              # Route imports (e.g. webhooks, attributes)
│   ├── services.yaml        # Service definitions and autowiring rules
│   └── preload.php          # Optional opcache.preload bootstrap
├── public/
│   └── index.php            # The *only* file the web server may serve
├── src/
│   ├── Controller/          # Your controllers (one per feature area)
│   ├── Entity/              # Doctrine entities (empty until Chapter 13)
│   ├── Repository/          # Custom repositories
│   └── Kernel.php           # The application kernel — rarely touched
├── templates/               # Twig templates, organized by controller
├── tests/                   # Unit and functional tests
├── var/                     # Cache and logs (git-ignored, auto-created)
├── .env                     # Committed environment defaults
├── .env.local               # Your local overrides (git-ignored)
├── .env.test                # Settings for the test environment
├── composer.json            # PHP dependencies
└── phpunit.xml.dist         # Test runner configuration
```

Two ideas to internalize now, because they recur all book: **`public/` is the only directory exposed to the web server** — everything else sits behind it — and **configuration lives in `config/`, code lives in `src/`**. Chapter 6 dissects both.

### 6. Understanding `.env` Files

Symfony loads environment variables from a small hierarchy of files, in this order (later files win):

1. `.env` — committed defaults shared by the whole team
2. `.env.local` — your personal overrides; **never committed**
3. `.env.$APP_ENV` — e.g. `.env.test`, loaded when `APP_ENV=test`
4. `.env.$APP_ENV.local` — e.g. `.env.dev.local`, also never committed

A typical `.env` from a fresh project looks like this:

```dotenv
###> symfony/framework-bundle ###
APP_ENV=dev
APP_SECRET=ChangeMeToSomethingLongAndRandom
###< symfony/framework-bundle ###

###> doctrine/doctrine-bundle ###
DATABASE_URL="sqlite:///%kernel.project_dir%/var/data.db"
###< doctrine/doctrine-bundle ###

###> symfony/messenger ###
MESSENGER_TRANSPORT_DSN=doctrine://default?auto_setup=0
###< symfony/messenger ###

###> symfony/mailer ###
MAILER_DSN=null://null
###< symfony/mailer ###
```

Three things to notice. First, `APP_ENV` selects the environment (`dev`, `test`, or `prod`) and drives which bundles load, how errors are displayed, and which `.env` files apply. Second, `APP_SECRET` signs cookies and tokens — it must be a long random string in production (the CLI generates one for you when you deploy). Third, real secrets never belong in committed files: in production they come from the actual environment (your PaaS or container orchestrator), which always takes precedence over any `.env` file. We revisit this hierarchy in Chapter 6 and its production implications in Chapter 24.

## A Note on Versions

This book targets **Symfony 7.4 LTS** — released November 2025, requiring PHP 8.2+, with bug fixes through November 2028 and security fixes through November 2029 — and the **8.x line** (8.0 and 8.1, both requiring PHP 8.4+). The two lines are close cousins: nearly everything in this book works on both, and where behavior or defaults differ, a **Symfony 8** box tells you exactly what changes.

Which should you install? If you're starting a long-lived commercial project and want the longest runway, create your project on 7.4 (`symfony new my-app --full` then pin `symfony/symfony: ^7.4` in `composer.json`). If you want the newest features and can run PHP 8.4, use the current stable (8.1). Either way, keep your PHP version at or above what your Symfony line requires, and check <https://symfony.com/releases> for the latest patch releases — minor versions ship every six months (May and November), and upgrading between them is a routine `composer update`, as Chapter 1 explains in detail.

## Part I — Foundations

### Chapter 1. The Symfony Ecosystem

**What you will learn in this chapter:**

- What Symfony actually is — components, bundles, and the framework as three distinct layers
- The design decisions that explain most of what you will see in the rest of this book
- How releases work: minor versions every six months, LTS every two years, and a support model you can plan around
- How to upgrade Symfony without pain, from routine `composer update` to a full major-version migration
- A guided tour of the component library, and where each piece fits

Ask three experienced developers "what is Symfony?" and you will get three different answers. One will hand you a `Request` object and start talking about HTTP. Another will open `config/bundles.php` and talk about FrameworkBundle. The third will run `symfony new` and talk about the project skeleton that appears. All three are right — because "Symfony" names three different things at three different layers of abstraction, and conflating them is the single most common source of confusion for people coming to the framework. This chapter untangles the layers, explains the release model you will live with for years, and gives you a map of the component library before we start using any of it.

#### A Short History

Symfony began in 2005, when Fabien Potencier — then running the French agency Sensio — kept rewriting the same boilerplate for client projects: routing tables, request handling, template glue, form processing. He extracted the reusable parts into a set of PHP classes. In 2007 those classes became **Symfony 1.0**, a full MVC framework that was, alongside CakePHP and CodeIgniter, one of the first modern PHP frameworks. It was popular through the late 2000s, but it had a structural flaw: the pieces were entangled. You took the whole framework or nothing.

The pivotal moment came with **Symfony 2.0 in October 2011**, a ground-up rewrite driven by one idea: *every feature should be a standalone library that works without the rest of Symfony*. The DI container became its own component. HTTP handling became its own component. PSR-0 autoloading arrived, and the "framework" shrank to a thin layer of bundles sitting on top of independent libraries. That architecture — components underneath, bundles in the middle, an optional framework on top — is still the architecture today. Everything else in this book is elaboration on it.

The subsequent majors were mostly about discipline:

- **3.0 (2015)** removed every piece of code deprecated during the 2.x line and hardened a rule that has never been broken since: *no backward-compatibility breaks within a major version*.
- **4.0 (November 2017)** reorganized the project layout into the `config/` + `src/` + `public/` skeleton you created in the front matter, and was followed in 2018 by **Symfony Flex**, the recipe system that automates bundle installation and configuration.
- **5.0 (2019)** and **6.0 (November 2021, the first line to require PHP 8.0)** continued the pattern: each major removed the previous line's deprecations and raised the PHP floor.
- **7.0 (May 2024, requiring PHP 8.2+)** fixed the release rhythm to a strict May/November cadence — the one described next.

The current landscape has two supported lines: **Symfony 7.4 LTS**, released in November 2025 for long-lived projects, and the **8.x line** (8.0 and 8.1), which requires PHP 8.4+ and carries the newest features. This book targets both.

> **Note.** The name comes from *symphony* — many independent parts playing in harmony, each one usable on its own. It is also a hint about governance: Sensio was acquired by the Drupal Association in 2015, and Symfony is now developed by an international team of core contributors, with commercial support available from several vendors. The project's health does not depend on any single company.

#### Design Philosophy

You do not need to memorize a manifesto, but six decisions made around 2011 explain most of what you will encounter in this book. When a Symfony feature seems oddly shaped, one of these is usually the reason.

**1. Components first.** Every feature ships as a library with its own namespace (`Symfony\Component\X`), its own tests, its own documentation page, and its own `UPGRADE` file. You can `composer require symfony/validator` in a project that contains no Symfony framework at all. The consequence is a small attack surface per feature and the freedom to adopt pieces incrementally — you are never forced to take the whole framework.

**2. The framework is optional glue.** A "Symfony application" is really *a set of bundles plus your code*. Bundles wire components together with sensible defaults; you opt in bundle by bundle. An app that only needs a REST API can skip Twig, forms, and sessions entirely — and the container will not even compile those services.

**3. Dependency injection is the backbone.** Controllers, validators, repositories, mailers — everything is a *service* resolved from a container, and autowiring makes the wiring invisible in day-to-day code. Chapter 5 dissects this; until then, just notice that you will rarely write `new` for anything interesting.

**4. Events are the extension points.** Rather than subclassing framework internals to hook in your behavior, Symfony publishes events at every interesting moment in the request lifecycle, and you *listen*. This is why cross-cutting concerns — logging, security checks, tenant resolution — can be added without touching core code. Chapter 7 is built around this idea.

**5. Standards compliance.** Symfony follows PSR-4 autoloading, exposes PSR-3 logger interfaces throughout, and bridges to PSR-7, PSR-15, and PSR-18 where relevant. It plays well with the wider PHP ecosystem, which matters when you integrate third-party libraries or outgrow the framework's defaults.

**6. Backward compatibility is a contract.** Within a major version, Symfony never breaks your code — it deprecates first, warns loudly in development and test environments, and removes only at the next major. This promise is what makes "upgrade every six months" a realistic strategy instead of a terrifying one. The release model section below explains the mechanics.

#### Components, Bundles, and the Framework

##### The Three Layers

Here are the three terms, precisely:

- **A component** is a standalone library published as `symfony/<name>` on Packagist, namespaced `Symfony\Component\<Name>`. It knows nothing about the rest of Symfony (or only minimal shared contracts). `HttpFoundation`, which models HTTP requests and responses, is the canonical example.
- **A bundle** is a package that integrates one or more components *into* the framework. Concretely, a bundle provides three things: a **configuration tree** (the options you can set in YAML under `config/packages/`), **service definitions** (which services to register and how to wire them), and **compiler passes** (hooks that modify the service container while it compiles). `FrameworkBundle`, for instance, is what turns the `HttpKernel`, `Routing`, `DependencyInjection`, `Config`, `EventDispatcher`, and `Console` components into a working application.
- **The framework** is what `symfony new` creates: the project skeleton plus Flex plus a set of *recipes* that install and configure bundles for you. It is a curated default stack, not a separate technology.

A useful analogy: components are ingredients, bundles are recipes that combine them, and the framework is the restaurant's default menu. You can buy the ingredients for your own kitchen (a non-Symfony project), write new recipes (your own bundle — Chapter 8), or just order from the menu.

##### Prove It: A Component Without a Framework

Words are cheap; let's demonstrate component independence in ten lines. In a scratch directory:

```bash
mkdir standalone-demo && cd standalone-demo
composer require symfony/http-foundation
```

Now create `demo.php`:

```php
<?php

declare(strict_types=1);

use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;

require __DIR__.'/vendor/autoload.php';

// Request::create() builds a request from scratch — handy in tests,
// and perfect for proving this component needs no framework around it.
$request = Request::create('/invoices/42', 'GET');

$response = new Response(
    sprintf('%s %s — handled without a framework', $request->getMethod(), $request->getPathInfo()),
    200,
    ['Content-Type' => 'text/plain; charset=utf-8'],
);

$response->send();
```

Run it:

```bash
php demo.php
### GET /invoices/42 — handled without a framework
```

No kernel, no bundle, no `config/` directory, no Flex. Just a library doing its job. In a real web entrypoint — the `public/index.php` you will dissect in Chapter 4 — the same component is populated from the actual HTTP request via `Request::createFromGlobals()`. The component does not care which world it lives in; that is the whole point of the layering.

> **Tip.** Two commands will become second nature. `composer show symfony/*` lists every Symfony component installed in your project (with versions), and `bin/console about` prints your PHP version, Symfony version, environment, and key configuration at a glance. When anything seems off, start there.

##### What a Bundle Actually Contains

You will meet bundles constantly, so it is worth knowing what one *is* before Chapter 8 takes one apart. A bundle is a class extending `Symfony\Component\HttpKernel\Bundle\Bundle` — often almost empty — plus the metadata that makes it useful:

1. **A configuration tree.** The bundle declares which options exist in its YAML file (for example, `framework.session.handler_id`), with defaults and validation. This is why typos in `config/packages/` produce precise errors instead of silent misbehavior.
2. **Service definitions.** The bundle tells the container which services to register, what their arguments are, and which tags they carry. Tags matter more than they look — they are how components discover your code (a service tagged `kernel.event_subscriber`, for instance, is wired into the event dispatcher automatically).
3. **Compiler passes.** Optional hooks that run while the container compiles, letting a bundle adjust or remove services based on configuration. This is the most powerful — and most advanced — of the three; Chapter 8 covers it properly.

> **Note.** Not everything in a Symfony app is a Symfony component. **Twig** (templating) and **Doctrine** (the ORM) are independent projects with their own release cycles; **Monolog** (logging) likewise. They integrate through bundles — `TwigBundle`, `DoctrineBundle`, `MonologBundle` — which is exactly the extension mechanism you will build yourself in Chapter 8. When you read third-party bundle documentation, you are reading about one of these integration seams.

##### Why the Distinction Matters

The three-layer model is not taxonomy for its own sake. It tells you where to look when things go wrong and what is possible:

- **Component-level problem** (a `Response` behaves oddly)? That is a library issue — reproducible in isolation, fixable by upgrading one package.
- **Wiring problem** (a service is missing, a tag is ignored, a config key is rejected)? That lives at the bundle/container level — Chapters 5 and 6.
- **Skeleton problem** (a file is in the wrong place, a recipe misconfigured something)? That is Flex territory, and `php bin/recipes:update` often fixes it.

It also means Symfony's pieces travel: you will find `HttpFoundation`, `Validator`, and `Serializer` inside projects that are not Symfony applications at all.

#### The Release Model

##### The Cadence

Since Symfony 7.0, releases follow a fixed rhythm:

- **Minor versions ship every six months — in May and November.** Each minor adds new features and may introduce *deprecations*, but it never breaks existing code. Upgrading from 7.2 to 7.3 to 7.4 is, by design, a routine `composer update`.
- **Major versions ship every two years** (7.0 in 2024, 8.0 in 2026). A major removes everything that was deprecated during the previous line, may change defaults, and raises the minimum PHP version. This is the only moment when code changes are required.
- **The `.4` minor of each line is the LTS** — 4.4, 5.4, 6.4, and now **7.4**. LTS releases get a substantially longer support window than regular minors.

##### Support Windows

Two kinds of fixes flow into a released version, and they have different lifespans:

- **Bug fixes** correct incorrect behavior. They stop first.
- **Security fixes** patch vulnerabilities. They continue longer, because an unpatched vulnerability in an old line is a real risk even for teams that no longer upgrade.

For **7.4 LTS**, the windows are: **bug fixes until November 2028, security fixes until November 2029**. Regular minor versions (say, 7.3) receive a shorter window than the LTS. Patch releases within a line — 7.4.1, 7.4.2, and so on — are drop-in: they contain only fixes and require no code changes.

| Line | Released | PHP requirement | Support |
|---|---|---|---|
| **7.4 LTS** | November 2025 | 8.2+ | Bug fixes until November 2028; security fixes until November 2029 |
| **8.x** (8.0, 8.1) | 2026 | 8.4+ | Current stable line |

> **Symfony 8.** The 8.x line requires PHP 8.4 and removes everything that was deprecated during the 7.x line. If you are on 7.4 LTS, none of this affects you until you *choose* to move — which is precisely the point of an LTS: it is a runway, not a dead end. When you do move, start with the `UPGRADE-8.0` notes shipped inside each component and the "Upgrade to 8.0" guide in the official documentation.

##### Choosing a Line

The front matter already gave you the short version; here is the reasoning. If you are starting a long-lived commercial project and want the longest runway with the fewest forced migrations, create your project on **7.4 LTS** — you will have security fixes for four years and can time the jump to 8.x around your own calendar. If you want the newest features and can run PHP 8.4, use the current stable of the **8.x line**. Either way, this book works: nearly everything is identical across the two lines, and where behavior or defaults differ, a **Symfony 8** box tells you exactly what changes.

#### Upgrading Symfony

##### Patch and Minor Upgrades Are Routine

A **patch upgrade** (7.4.3 → 7.4.4) contains only bug fixes:

```bash
composer update
```

That is the entire procedure. A **minor upgrade** (7.2 → 7.3, or 7.3 → 7.4) is almost as cheap, because no backward-compatibility breaks are allowed within a major version:

```bash
composer update symfony/* --with-all-dependencies
```

New features appear; your code keeps working. The one thing to watch for is **deprecation warnings**. Symfony's deprecation policy is simple and strict: *code deprecated during a major line is removed in the next major*. Anything you use that was deprecated somewhere in 7.x will be gone in 8.0 — but until then, every time your code touches it, Symfony emits an `E_USER_DEPRECATED` notice in the `dev` and `test` environments. Those notices are not noise; they are your to-do list for the next major, generated for you by the framework.

> **Caution.** Do not silence deprecation warnings. In production they are hidden by default (correctly — you do not want clients reading your upgrade notes), but in development and test they are visible *on purpose*. A project that upgrades with zero outstanding deprecations upgrades in an afternoon; a project that ignores the warnings for two years discovers its to-do list the week before a deadline, when every warning has compounded into a migration. Treat a deprecation notice like a compiler warning: fix it now, or track it deliberately.

##### Major Upgrades: The Workflow

When you do decide to cross a major boundary — 7.4 → 8.0 in your case — the work is bounded and mechanical if you follow the sequence:

1. **Get to the latest patch of your current line.** `composer update` until 7.4.x is fully current. You want the smallest possible delta.
2. **Fix outstanding deprecations.** Run the application, exercise the main flows, run the test suite, and check `var/log/dev.log`. Every deprecation you see is something 8.0 will remove.
3. **Raise your PHP requirement if needed.** The 8.x line requires PHP 8.4: update the `php` constraint in `composer.json`, your CI configuration, and your deployment targets before touching Symfony.
4. **Bump the Symfony constraints** in `composer.json` (for example, `^7.4` → `^8.0` for the packages you use).
5. **Run `composer update`.** Read any conflict messages carefully — they usually point at a third-party package that has not yet released an 8.x-compatible version.
6. **Apply mechanical patches.** The official `symfony/upgrade` package installs a set of scripts that apply the most common renames and signature changes automatically:

   ```bash
   composer require symfony/upgrade --dev

   # Add type declarations to your codebase, targeting your minimum PHP version
   SYMFONY_PATCH_TYPE_DECLARATIONS="force=1&php=8.2" ./vendor/bin/patch-type-declarations
   ```

   The package also ships per-version patch scripts for the larger mechanical changes of each major. Run them with a dry run first, review the diff, and commit only what looks right — these are aids, not oracles.
7. **Sync your bundle recipes.** Flex remembers which recipe version installed each configuration file; this command compares against the latest and patches your `config/` files accordingly:

   ```bash
   php bin/recipes:update
   ```

8. **Run the full test suite and read the upgrade notes.** Some changes are not hard breaks but *behavior* shifts — a default that changed, a deprecation that now throws. The `UPGRADE-8.0` file in each component and the "Upgrade to 8.0" page in the official documentation list them. Your tests catch the rest.

> **Tip.** Upgrade minors regularly — every May and November, as part of your normal release cadence. The cost of a major upgrade is proportional to how far behind you are: a team that stays current treats majors as a formality (steps 4–7 above, mostly automated), while a team two years behind faces a project. This is the single highest-leverage habit in Symfony maintenance.

One last note: third-party bundles — `DoctrineBundle`, `TwigBundle`, and friends — keep pace with the framework but ship their own upgrade notes. `composer outdated` shows you what is lagging, and most well-maintained bundles release an 8.x-compatible version within weeks of a new major.

#### Tour of the Component Library

Symfony ships on the order of fifty components. You do not need to memorize them — you need to know *where to look*. Here is the map, grouped by job, with the chapter where each component earns its keep in this book. To explore your own project's slice of it, run `composer show symfony/*`; the full reference lives at <https://symfony.com/doc/current/components/>.

##### The HTTP Layer

| Component | Job | Where in this book |
|---|---|---|
| `HttpFoundation` | `Request`/`Response` objects, sessions, cookies, file uploads | Chapter 2 |
| `HttpKernel` | The request lifecycle: kernel events, middleware, exception handling | Chapters 4, 7 |
| `Routing` | Maps URLs to code; generates URLs from route names | Chapter 3 |

##### Core Machinery

| Component | Job | Where in this book |
|---|---|---|
| `DependencyInjection` | The service container: definitions, autowiring, autoconfiguration | Chapter 5 |
| `Config` | Configuration trees and parameters behind everything in `config/` | Chapter 6 |
| `EventDispatcher` | Publish/subscribe events that glue the whole system together | Chapter 7 |
| `Console` | CLI commands — the engine behind `bin/console` | Chapter 15 |
| `Yaml`, `Dotenv` | Parse the YAML configuration and `.env` files you will edit constantly | Chapter 6 |
| `Finder`, `Filesystem`, `Process` | File discovery, safe file operations, running external processes | Throughout (e.g., Chapter 24) |
| `OptionsResolver` | Validates option arrays — the engine under console commands and many services | Chapter 15 |
| `VarExporter` | Serializes PHP values into container code at compile time | Chapter 5 |
| `Clock` | A swappable "now" so time-dependent code is testable | Chapter 22 |

##### Data and Validation

| Component | Job | Where in this book |
|---|---|---|
| `Serializer` | Objects ↔ arrays/JSON; the heart of all API work | Chapter 19 |
| `Validator` | Constraint-based validation for objects, forms, and APIs | Chapter 11 |
| `PropertyInfo` | Introspects property types and metadata (used by forms and the serializer) | Chapters 10, 19 |
| `PropertyAccess` | Reads and writes nested paths like `address.city` | Chapter 10 |
| `Mime` | MIME type detection for uploads and attachments | Chapters 2, 16 |
| `Uid` | UUIDs and ULIDs as first-class values | Chapter 26 |

##### Security

| Component | Job | Where in this book |
|---|---|---|
| `Security` | Firewalls, authenticators, voters, access control | Chapters 12, 21 |
| `PasswordHasher` | Pluggable password hashing (argon2id by default) | Chapter 12 |
| `Lock` | Distributed locks so only one process runs a given task | Chapters 18, 26 |
| `Semaphore` | Counting semaphores for concurrency limits | Chapter 26 |
| `RateLimiter` | Token-bucket rate limiting for APIs and login attempts | Chapters 19, 26 |

##### Asynchronous Processing and Messaging

| Component | Job | Where in this book |
|---|---|---|
| `Messenger` | Async jobs: messages, handlers, transports, retries | Chapter 17 |
| `Mailer` | Email with pluggable transports (SMTP, API providers, null) | Chapter 16 |
| `Notifier` | SMS, push, and other notification channels beyond plain email | Chapter 16 |
| `Scheduler` | Cron-style scheduled tasks | Chapter 18 |
| `Webhook` | Receiving and verifying events from external services | Chapter 18 |

##### Presentation and Internationalization

| Component | Job | Where in this book |
|---|---|---|
| Twig (via `TwigBundle`) | The template engine — a third-party library integrated as a bundle | Chapter 9 |
| `AssetMapper` | Versioned asset references with hot reload in development | Chapter 14 |
| `HtmlSanitizer` | Safe rendering of user-supplied HTML | Chapter 9 |
| `Translation` | Translation catalogs and locale negotiation | Chapter 27 |
| `Workflow` | State machines for entities that move through states | Chapter 25 |

##### Debugging and Observability

| Component | Job | Where in this book |
|---|---|---|
| `VarDumper` | `dump()` and the data behind the debug toolbar | Chapter 23 |
| `ErrorHandler` | Turns exceptions into detailed dev error pages and production-safe responses | Chapters 4, 23 |
| `Stopwatch` | Timing measurements that feed the profiler | Chapter 23 |
| `HttpClient` | Outbound HTTP (PSR-18) for calling external APIs | Throughout (e.g., Chapter 18)

> **Note.** Underneath all of this sits a small package you will rarely touch directly: `symfony/contracts`. It holds the shared interfaces — service locator, event dispatcher, logger, translation — that let components talk to each other, and to non-Symfony code, without depending on each other's implementations. When you see an interface in a component whose job seems to be *only* declaring a contract, it probably lives here.

#### In Summary

- **Symfony is three layers:** components (standalone libraries), bundles (the integration seam: configuration trees, services, compiler passes), and the framework (the curated stack `symfony new` creates). The layering *is* the design — it is why you can use pieces in isolation and extend the whole without forking it.
- **The 2011 rewrite made every feature standalone**, and the backward-compatibility contract — no breaks within a major, deprecations removed only at the next major — has held ever since.
- **Releases are predictable:** minors every six months (May and November), majors every two years, LTS on the `.4` minor. 7.4 LTS receives bug fixes until November 2028 and security fixes until November 2029; the 8.x line requires PHP 8.4.
- **Upgrades are cheap if you stay current:** `composer update` for patches and minors, deprecation warnings as your migration to-do list, and for majors the `symfony/upgrade` patch scripts plus `php bin/recipes:update`.
- **The component tour is your map.** Every chapter from here on deepens one region of it; when you are lost, the table above tells you which chapter owns the question.

#### Exercises

1. Run `bin/console about` in your `hello-symfony` project and identify the Symfony version, PHP version, and environment. Then run `composer show | grep '^symfony/'` and count the installed components. Which three do you recognize from the tour above?
2. In two or three sentences each, explain the difference between a component and a bundle, naming one of each from your project. Where would *your* code live in that picture — and where does it not belong?
3. Reproduce the standalone demo from this chapter: in a scratch directory, `composer require symfony/http-foundation`, write a script that builds a `Request` with `Request::create()`, inspects its query parameters, and sends a `Response`. What does running it with plain `php` prove about how Symfony is packaged?
4. **Deprecation audit.** Load a few pages of `hello-symfony` in the dev environment, then check `var/log/dev.log` for deprecation notices. How many does a fresh project emit, and where do they originate — your code, or dependencies? (A fresh skeleton should be nearly clean; if it is not, you now know what to investigate.)
5. ⭐ Pick one component you have never used — `Lock`, `Process`, `Stopwatch`, `Uid` are good candidates. Read its README and documentation page, then write a ten-line standalone script demonstrating it. No framework allowed: no kernel, no bundle, no `config/`.
6. ⭐ **Upgrade drill.** Create a throwaway project pinned to an older 7.x minor (for example, `composer require symfony/framework-bundle:7.2.*`), then walk it forward one minor at a time to the latest 7.4 using only `composer update` and deprecation warnings. Write down every step that required more than a composer command. Then read the `UPGRADE-8.0` notes for `HttpFoundation` and list three changes that would affect your code.
7. ⭐ **Ecosystem map.** For a project of your own — or InvoiceHub, if you are reading ahead — draw the three layers: which components it uses, which bundles integrate them (including third-party ones like `DoctrineBundle`), and what Flex recipes installed for you. Mark where your own code sits in each layer.

Solutions for all exercises are in the companion repository (see Appendix D), tagged per chapter so you can check out exactly the state this chapter describes.

### Chapter 2 — HTTP Fundamentals with HttpFoundation

Every web application, no matter how elaborate, is a conversation in one language: HTTP. A browser sends a *request* — a method, a URL, some headers, maybe a body — and the server answers with a *response* — a status code, headers, and content. Everything else in this book (routing, controllers, security, APIs) is machinery for turning one side of that conversation into the other.

Raw PHP exposes that conversation as five scattered superglobals (`$_GET`, `$_POST`, `$_FILES`, `$_COOKIE`, `$_SERVER`) plus a pile of functions (`header()`, `echo`). It works, but it's global state: hard to test, easy to misread, and impossible to reason about once your application grows. Symfony's answer is the **HttpFoundation** component, which packages an incoming request into a single `Request` object and an outgoing response into a single `Response` object.

In this chapter you will learn to:

- read every part of an incoming `Request` — method, URI, parameters, headers, cookies, files;
- distinguish the three kinds of request parameters (`query`, `request`, `attributes`) and know which ones you can trust;
- build `Response` objects with the right status codes, including redirects, JSON, and file downloads;
- manage sessions, cookies, and flash messages;
- handle file uploads safely;
- determine a client's real IP address when your app sits behind a proxy.

We'll work in the `hello-symfony` project you created in the front matter. The running invoicing application arrives in Part III; for now, small self-contained examples are all we need.

---

#### 2.1 One Object Instead of Five Superglobals

The web server (PHP's built-in dev server, PHP-FPM behind nginx, FrankenPHP — Chapter 24 compares them) is configured to hand *every* URL to a single file: `public/index.php`, the **front controller**. In a fresh project it looks like this:

```php
<?php

use App\Kernel;

require_once dirname(__DIR__).'/vendor/autoload_runtime.php';

return function (array $context) {
    return new Kernel($context['APP_ENV'], (bool) $context['APP_DEBUG']);
};
```

That's the whole file. Symfony's *Runtime* component takes over from here: it reads `APP_ENV` and `APP_DEBUG` from the environment, boots the kernel, and — the step that matters for this chapter — builds a `Request` object from PHP's superglobals via `Request::createFromGlobals()`. The kernel then walks the request through your application and expects a `Response` back.

So the mental model for the rest of the book is: **your code receives a `Request`, returns a `Response`**. Nothing else crosses that boundary.

> **Note.** HttpFoundation is a standalone component. You can `composer require symfony/http-foundation` in any PHP project — framework or not — and use `Request`, `Response`, cookies, and file uploads without the rest of Symfony. Other frameworks (Laravel, for instance) build on it too. When you learn it here, you're learning a general-purpose HTTP toolkit, not a Symfony quirk.

Objects instead of superglobals buy you three things:

1. **A single source of truth.** `$request->query->get('page')` is unambiguous; there's no question whether the value came from `$_GET`, `$_REQUEST`, or a header.
2. **Testability.** You can build requests in code — `Request::create('/invoices?page=2', 'GET')` — and pass them to your controllers without starting a server. Chapter 22 leans on this constantly.
3. **Framework independence of the HTTP layer.** The same `Response` object works whether it's sent over a real connection, through Symfony's test client, or serialized for a queue.

---

#### 2.2 Anatomy of a Request

Let's look at a concrete request and find each piece on the `Request` object:

```http
POST /invoices?page=2&sort=date HTTP/1.1
Host: app.example.com
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)
Cookie: PHPSESSID=xk2m9…; theme=dark
Content-Type: application/x-www-form-urlencoded

amount=99.50&currency=EUR
```

Here is where each part lives once Symfony has wrapped it:

| Part of the HTTP message | On the `Request` object |
|---|---|
| Method (`POST`) | `$request->getMethod()` |
| Host, scheme, port | `getHost()`, `getScheme()`, `getPort()` |
| Path (`/invoices`) | `getPathInfo()` |
| Query string (`page=2&sort=date`) | `getQueryString()`, or parsed: `$request->query` |
| Headers | `$request->headers` (a `HeaderBag`) |
| Cookies | `$request->cookies` (a `ParameterBag`) |
| Form-encoded body | `$request->request` (a `ParameterBag`) |
| Uploaded files | `$request->files` (a `FileBag`) |
| Raw body, unparsed | `getContent()`; JSON/XML: `getPayload()` |
| Server variables (`REMOTE_ADDR`, …) | `$request->server` (a `ServerBag`) |
| Route and framework data | `$request->attributes` (a `ParameterBag`) |

The first three rows are worth a quick controller you can paste into your project and hit with `curl`:

```php
<?php

declare(strict_types=1);

namespace App\Controller;

use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Routing\Attribute\Route;

#[Route('/request-info', name: 'app_request_info')]
class RequestInfoController
{
    public function __invoke(Request $request): JsonResponse
    {
        return new JsonResponse([
            'method'     => $request->getMethod(),      // 'POST'
            'uri'        => $request->getUri(),         // 'https://app.example.com/invoices?page=2&sort=date'
            'path'       => $request->getPathInfo(),    // '/invoices'
            'query'      => $request->query->all(),     // ['page' => '2', 'sort' => 'date']
            'post'       => $request->request->all(),   // ['amount' => '99.50', 'currency' => 'EUR']
            'user_agent' => $request->headers->get('User-Agent'),
            'cookies'    => $request->cookies->all(),   // ['PHPSESSID' => 'xk2m9…', 'theme' => 'dark']
            'client_ip'  => $request->getClientIp(),
        ]);
    }
}
```

Two method-level helpers are worth knowing now because they matter for caching and retries later in the book:

```php
$request->isMethodSafe();      // true for GET, HEAD, OPTIONS, TRACE — "no side effects"
$request->isMethodIdempotent(); // safe methods plus PUT and DELETE — "repeating it is harmless"
```

A proxy that may retry a request should only retry idempotent ones; your HTTP cache (Chapter 23) will only store safe ones. When you design endpoints in Part V, these two booleans are part of the contract.

> **Note.** HTML forms can only submit `GET` or `POST`, which is why Symfony offers *method override*: a hidden `_method=PUT` field makes a form POST behave like a PUT. This feature is **off by default** in Symfony 7+ (`framework.http_method_override: false`). If you maintain legacy forms that rely on it, enable the option — and since 7.4 you can restrict which methods may be overridden with `framework.allowed_http_method_override: ['PUT', 'PATCH', 'DELETE']`. Overriding *to* `GET`, `HEAD`, `CONNECT`, or `TRACE` is deprecated in 7.4 and gone in 8.0.

---

#### 2.3 Three Kinds of Parameters: `query`, `request`, `attributes`

This is the distinction that separates people who use Symfony from people who understand it, so let's be precise. A `Request` carries three parameter bags, and they are **not interchangeable**:

| Bag | Property | Comes from | Set by | Trust level |
|---|---|---|---|---|
| Query parameters | `$request->query` (`InputBag`) | The URL: `/invoices?page=2` | The client | Untrusted input |
| Request body | `$request->request` (`ParameterBag`) | Form-encoded POST body | The client | Untrusted input |
| Attributes | `$request->attributes` (`ParameterBag`) | Nowhere in the HTTP message | The router, listeners, your code | Framework-internal |

**Query parameters** are everything after the `?`. Because they live in the URL, they're visible to users, shareable, bookmarkable — and therefore the natural place for *filters* (which page, which sort order) rather than for data that mutates state. The query bag is an `InputBag`, which adds filtering on top of plain lookup:

```php
// ?page=2   → 2 (an int)
// ?page=abc → 1 (the default — FILTER_VALIDATE_INT rejected it)
// no page   → 1
$page = $request->query->get('page', 1, true, \FILTER_VALIDATE_INT);
```

The signature is `get($key, $default = null, $trim = false, $filter = FILTER_DEFAULT, $filterOptions = null)`. When the filter rejects a value, you get the default — never a half-validated string. You'll use this pattern for every pagination and filter parameter in the running project.

**The request body** (`$request->request`) holds form-encoded data submitted with `POST` — the `amount=99.50&currency=EUR` from our example. It's the same bag type, same trust level: whatever a client types into a form is input to be validated (Chapter 11), not data to be believed.

**Attributes** are different in kind. They never come from the client at all. The router fills two of them on every matched request — `_route` and `_controller` — and listeners or your own code can add more as the request travels through the kernel (Chapter 7 shows how). Attributes are the framework's way of passing data *along* the request pipeline:

```php
$route = $request->attributes->get('_route'); // 'app_invoices_list'
```

> **Caution.** `$request->query` and `$request->request` are untrusted input, full stop. Validate them before use, and never derive an *authorization* decision from them — "the URL says `?admin=1`" is not a permission. Attributes, by contrast, are set by code you control; that's why route placeholders (`/invoices/{id}`) land in attributes rather than the query bag: they're part of the matched route, not free-form client input.

One more source of body data deserves a mention now and a full chapter later (Part V): **JSON bodies**. Form-encoded data goes into `$request->request`; JSON does not. For `Content-Type: application/json` requests, use the payload bag:

```php
// PUT /api/invoices/42 with body {"amount": 99.50, "currency": "EUR"}
$payload = $request->getPayload()->all(); // ['amount' => 99.5, 'currency' => 'EUR']
```

`getPayload()` parses JSON (and XML) bodies regardless of HTTP method and returns an `InputBag`, so the same filtering works. If you need the body completely raw — webhooks in Chapter 18 do this for signature verification — `getContent()` gives you the exact string the client sent.

> **Symfony 8.** Two behaviors around request parameters differ between the lines:
>
> 1. The convenience method `$request->get('key')` — which searched attributes, then query, then body in that order — is **deprecated in 7.4 and removed in 8.0**. Its ambiguity (where did this value come from?) is exactly what the three bags exist to prevent. Always read from a specific bag, as this chapter does.
> 2. Symfony 7.4 can parse form-encoded bodies for `PUT`, `PATCH`, and `DELETE` into `$request->request` — but only when running on PHP 8.4, which provides the underlying `request_parse_body()` function. On Symfony 8.x (PHP 8.4 is required) this always works; on 7.4 with PHP 8.2/8.3, only `POST` bodies are parsed, so use `getPayload()` or `getContent()` for other methods.

---

#### 2.4 Headers and Cookies

Headers live in a `HeaderBag`, which is **case-insensitive** — HTTP headers are, per the RFC, case-insensitive, and the bag enforces that:

```php
$request->headers->get('User-Agent');   // same as 'user-agent', 'USER-AGENT'
$request->headers->all();               // ['user-agent' => [...], 'cookie' => [...], ...]
```

Two request-side helpers save you from parsing header values by hand:

```php
$request->getAcceptableContentTypes(); // ['text/html', 'application/xhtml+xml', '*/*']
$request->getPreferredFormat('html');  // 'html' — the first Accept type Symfony understands
```

You'll meet both again in Part V when APIs negotiate response formats.

Writing headers is a matter of decorating your `Response` (Section 2.7):

```php
$response->headers->set('X-Request-Id', $id);
$response->headers->set('Cache-Control', 'no-store');
```

**Cookies** are how the stateless HTTP protocol remembers things. Reading is just another bag:

```php
$theme = $request->cookies->get('theme'); // 'dark' — or null if the client didn't send it
```

Writing a cookie means attaching a `Cookie` object to the response:

```php
use Symfony\Component\HttpFoundation\Cookie;

$cookie = new Cookie(
    name:   'theme',
    value:  'dark',
    expire: time() + 86_400 * 30, // 30 days; 0 (the default) means "session cookie"
);
$response->headers->setCookie($cookie);
```

Look at the constructor's defaults and you'll see Symfony's security posture: `$httpOnly` is `true` (JavaScript cannot read the cookie — a major XSS mitigation), `$sameSite` is `'lax'` (the browser won't send it on cross-site requests — a CSRF mitigation), and `$secure` is `null`, which means *decide at send time*: when the response is prepared for an HTTPS request, Symfony marks the cookie `Secure` automatically. Pass `true` or `false` explicitly only when you have a specific reason.

To delete a cookie, send it again with a past expiry — `clearCookie()` does that for you:

```php
$response->headers->clearCookie('theme');
```

> **Caution.** Cookies are sent by the browser on *every* request to your domain, so keep them small and few. Never store secrets in cookies you don't sign (signing and encryption of cookies is covered with security in Chapter 12), and remember that `HttpOnly` protects against script access only — it does nothing against a server-side response injection.

---

#### 2.5 File Uploads

When a form submits with `enctype="multipart/form-data"`, PHP stashes each uploaded file in a temporary location and describes it in `$_FILES`. Symfony wraps those descriptions in `UploadedFile` objects, reachable through the files bag:

```php
$file = $request->files->get('document'); // UploadedFile|null — null if no file was sent
```

For multiple files, name the form field `attachments[]` and you get an array of `UploadedFile`s back.

The `UploadedFile` API has a subtle but important split between what the *client claims* and what the *server verifies*:

| Method | Tells you | Trust it? |
|---|---|---|
| `getClientOriginalName()` | The filename the client sent (`"invoice (1).pdf"`) | **No** — display only |
| `getClientMimeType()` | The MIME type the client claimed | **No** |
| `getMimeType()` | Server-side guess from file content (finfo) | Yes |
| `guessExtension()` | Extension derived from the *guessed* MIME type | Yes |
| `getSize()` | Size in bytes | Yes |
| `isValid()` | Actually uploaded over HTTP, no PHP errors | — |
| `move($dir, $name = null)` | Moves the temp file to its final home; returns a `File` | — |

A client can name a file whatever it likes and claim any MIME type it likes. A file called `invoice.pdf` with a claimed type of `application/pdf` might be a PHP script. So the rules for handling uploads are:

1. **Validate with server-side evidence** — `getMimeType()` (content-based) plus an extension allowlist, never the client's claims.
2. **Generate your own filename.** The client's name is for display; on disk you store under a random name.
3. **Store outside `public/`** (or serve through a controller) when the file is private — invoices are exactly that kind of file.
4. **Respect PHP's limits.** `upload_max_filesize` and `post_max_size` in `php.ini` cap uploads; `UploadedFile::getMaxFilesize()` returns the effective limit so you can show it to users instead of letting them hit a cryptic failure.

Section 2.9 puts all four rules into a working controller. For now, here is the core of an upload handler:

```php
$file = $request->files->get('document');

if (!$file instanceof UploadedFile || !$file->isValid()) {
    // No file, or PHP rejected it (too large, partial upload, …)
    throw $this->unprocessableEntityError($file?->getErrorMessage() ?? 'No file uploaded.');
}

if ('application/pdf' !== $file->getMimeType()) {
    throw $this->unprocessableEntityError('Only PDF files are allowed.');
}

$stored = $file->move($uploadDir, bin2hex(random_bytes(16)).'.pdf'); // random name, safe extension
```

(`throw $this->…` is `AbstractController` sugar for returning a JSON error response; you'll see plain status codes in the next section.)

> **Tip.** When `move()` fails it throws specific exceptions — `IniSizeFileException`, `FormSizeFileException`, `PartialFileException`, and so on — each with a user-friendly message. Catching them (or at least logging them) turns "500 Internal Server Error" into "your file exceeds the 2 MB limit."

---

#### 2.6 Client IP Addresses and Trusted Proxies

Ask "who is calling?" and PHP's honest answer is `REMOTE_ADDR`: the IP address of the machine that opened the TCP connection. In development, that's `127.0.0.1`. In production, it's usually **not your user** — it's a load balancer, CDN, or reverse proxy sitting in front of your app.

Proxies cope by appending the previous hop to an `X-Forwarded-For` header as the request passes through:

```
X-Forwarded-For: 203.0.113.7, 10.0.0.4
```

Read right-to-left from the server's perspective: `10.0.0.4` is the proxy that connected to you; `203.0.113.7` is what *that* proxy was told the client was. The problem: **any client can send an `X-Forwarded-For` header of its own**, so the leftmost value is forgeable unless you know which hops are honest.

Symfony's solution is the *trusted proxies* configuration. You declare which IPs are your infrastructure; Symfony then walks the `X-Forwarded-For` chain from the server outward, only accepting values contributed by trusted hops, and skipping reserved (private) ranges. What remains is your real client:

```yaml
# config/packages/framework.yaml
framework:
    # …
    trusted_proxies: '127.0.0.1,PRIVATE_SUBNETS'
    trusted_headers: ['x-forwarded-for', 'x-forwarded-proto']
```

`PRIVATE_SUBNETS` (available since 7.2) is a shortcut for all RFC 1918 and loopback ranges — handy when your proxy's IP changes between deploys. Since 7.2 you can also supply the same settings via the `SYMFONY_TRUSTED_PROXIES` and `SYMFONY_TRUSTED_HEADERS` environment variables, which is often cleaner on a PaaS where the load balancer's address isn't known at deploy time. (`trusted_headers: ['x-forwarded-all']` trusts every forwarded header; list specific names to trust fewer.)

With that in place:

```php
$ip = $request->getClientIp(); // '203.0.113.7' — or null if nothing trustworthy was found
```

> **Caution.** Misconfigured trusted proxies fail in two opposite ways, and both are nasty to find in production:
>
> - **Trust too little** (your proxy isn't listed): every user appears to come from the proxy's IP. Per-IP rate limiting (Chapter 26), audit logs, geo-based logic, and "one session per client" rules all silently break.
> - **Trust too much** (a broad range, or trusting headers your proxy doesn't set): clients can spoof their IP — bypassing rate limits, polluting analytics, and forging the origin of webhook calls.
>
> When you deploy in Chapter 24, configuring this correctly is a checklist item, not an afterthought.

Two small utilities round out the topic. `IpUtils::anonymize('203.0.113.7')` returns `'203.0.113.*'` — useful when you log IPs and want to be gentle about personal data. And if you're using HttpFoundation *without* FrameworkBundle, the same configuration exists as a static call: `Request::setTrustedProxies(['127.0.0.1'], Request::TRUSTED_PROXIES_HEADER_X_FORWARDED_FOR | Request::TRUSTED_PROXIES_HEADER_X_FORWARDED_PROTO)`.

---

#### 2.7 Anatomy of a Response

A response is, per the HTTP spec, three things: a status line, headers, and an optional body. Symfony mirrors that exactly:

```php
use Symfony\Component\HttpFoundation\Response;

$response = new Response('<h1>Invoice paid</h1>', Response::HTTP_OK);
//                        content              status (200)
```

The third constructor argument is a header array. You'll rarely need it, because the class constants make status codes self-documenting: `Response::HTTP_CREATED` (201), `Response::HTTP_NO_CONTENT` (204), `Response::HTTP_BAD_REQUEST` (400), and so on. Here are the ones you'll actually use in this book's application:

| Status | Meaning | When you'll use it |
|---|---|---|
| `200 OK` | Success, content follows | The default for GETs |
| `201 Created` | A resource was created | POSTing a new invoice (Part V) |
| `204 No Content` | Success, nothing to send back | DELETE of a draft |
| `301 Moved Permanently` | This URL is gone forever | Domain migrations |
| `302 Found` | Temporary redirect | The workhorse; default for `RedirectResponse` |
| `303 See Other` | "Follow up with a GET" | Explicit POST/redirect pattern |
| `304 Not Modified` | Your cached copy is still fresh | HTTP caching (Chapter 23) |
| `400 Bad Request` | Malformed input | Unparseable JSON body |
| `401 Unauthorized` | Not authenticated | Missing/expired credentials (Ch. 12, 21) |
| `403 Forbidden` | Authenticated, but not allowed | Voter denies access (Chapter 12) |
| `404 Not Found` | No such resource | Unknown invoice ID |
| `405 Method Not Allowed` | Resource exists, wrong verb | POST to a GET-only route |
| `409 Conflict` | State conflict | Duplicate payment reference |
| `422 Unprocessable Entity` | Well-formed, semantically invalid | API validation failures (Part V) |
| `429 Too Many Requests` | Slow down | Rate limiting (Ch. 19, 26) |
| `500 Internal Server Error` | We broke | Anything uncaught |
| `503 Service Unavailable` | Temporarily down | Maintenance mode (Chapter 24) |

Choosing the *right* code is part of designing an API: a client that gets `409` knows to re-fetch and reconcile; a client that gets `500` knows to retry later. Status codes are your error taxonomy.

**Redirects** deserve their own paragraph because the whole web-app flow of this book depends on them:

```php
use Symfony\Component\HttpFoundation\RedirectResponse;

return new RedirectResponse($this->generateUrl('app_documents')); // 302
```

The pattern is called **POST/redirect/GET**: a form POSTs, your handler does its work, and instead of rendering a page you return a 302. The browser follows it with a fresh GET — so pressing *Refresh* after a successful submission re-runs the harmless GET, not the mutating POST. Every form in Part III ends this way.

**JSON responses** are a small subclass:

```php
use Symfony\Component\HttpFoundation\JsonResponse;

return new JsonResponse(['id' => 42, 'status' => 'paid'], Response::HTTP_CREATED);
```

It sets `Content-Type: application/json` for you and encodes the data with sensible defaults — notably, it escapes `<`, `>`, `'`, `&`, and `"` (the `DEFAULT_ENCODING_OPTIONS` constant), which makes the JSON safe to embed in HTML. Adjust with `setEncodingOptions()` when you have a reason. Part V is devoted to JSON done properly: serialization groups, format negotiation, error formats.

**File downloads** use `BinaryFileResponse`, which streams a file from disk and sets the right headers for you:

```php
use Symfony\Component\HttpFoundation\BinaryFileResponse;

return new BinaryFileResponse($path, Response::HTTP_OK, [
    'Content-Disposition' => 'attachment; filename="invoice-42.pdf"',
]);
```

It also understands `Range` requests, so browsers can resume large downloads. For generating exports on the fly — a CSV of ten thousand invoices — there's `StreamedResponse`, which calls your callback in chunks instead of buffering the whole body in memory:

```php
use Symfony\Component\HttpFoundation\StreamedResponse;

return new StreamedResponse(function () {
    $out = fopen('php://output', 'w');
    fputcsv($out, ['id', 'total']);
    // … write rows as you fetch them …
    fclose($out);
}, Response::HTTP_OK, ['Content-Type' => 'text/csv; charset=utf-8']);
```

Finally, two methods complete the picture. `Response::prepare(Request $request)` normalizes a response before it goes out — fixing up status codes, `Content-Length`, and conditional-request handling (the 304s in the table above). `Response::send()` writes headers and body to the client. **You almost never call either**: the kernel does both at the end of the request lifecycle, which Chapter 4 dissects event by event.

---

#### 2.8 Sessions and Flash Messages

HTTP has no memory, so "logged-in user" or "items in cart" must be reconstructed on every request. The standard mechanism is a **session**: the server generates an opaque ID, hands it to the browser in a cookie, and stores data keyed by that ID (by default, in PHP session files). Every subsequent request brings the cookie back, and Symfony reattaches the same storage.

You get the session from the request:

```php
$session = $request->getSession();

$session->set('cart', ['invoice_42' => 99.50]);
$cart    = $session->get('cart', []);   // second argument: default when absent
$session->has('cart');
$session->remove('cart');
$session->all();                        // everything stored
$session->clear();                      // wipe the slate
```

The session's behavior is configured in `framework.yaml`:

```yaml
# config/packages/framework.yaml
framework:
    session:
        enabled: true
        handler_id: null        # null = PHP's native file storage; swap for Redis later (Ch. 23)
        cookie_secure: auto     # Secure flag whenever the request arrives over HTTPS
        cookie_samesite: lax
```

`cookie_secure: auto` is the setting to remember: it makes the session cookie `Secure` in production (HTTPS) and omits it locally, where your dev server speaks plain HTTP — no per-environment fiddling.

**Flash messages** are the small UX trick that makes redirects feel right. A flash is a message stored in the session that survives *exactly one* request — which is precisely the hop between your POST handler and the redirect target:

```php
// In a controller (AbstractController helper):
$this->addFlash('success', 'Invoice paid.');

// Or, without AbstractController:
$request->getSession()->getFlashBag()->addFlash('success', 'Invoice paid.');
```

And in Twig, where every template in this book renders them:

```twig
{% for label, messages in app.flashes %}
    {% for message in messages %}
        <div class="flash flash-{{ label }}">{{ message }}</div>
    {% endfor %}
{% endfor %}
```

The semantics are read-once: the flash bag is consumed when the next request renders, so the message appears exactly one time. Refresh the page and it's gone — which is why flashes pair naturally with the POST/redirect pattern. (In tests you can `peek()` at flashes without consuming them; Chapter 22 uses this to assert on them.)

> **Caution.** A session is per-user state, not a database. Don't store large blobs or data you need to query — that's Doctrine's job (Chapter 13). And don't treat the session ID as a permanent identity: Symfony regenerates it when a user's privileges change (login, in Chapter 12) to prevent *session fixation*, where an attacker plants a known ID and waits for the victim to log in.

> **Symfony 8.** In 8.0, several legacy options of `NativeSessionStorage` (`referer_check`, `use_only_cookies`, `sid_length`, `sid_bits_per_character`, and friends) were removed. If you ever need that level of control over PHP's session engine, configure the corresponding `session.*` ini directives directly instead.

---

#### 2.9 Putting It All Together: A Document Inbox

Let's assemble everything from this chapter into one small feature: a page where you upload a PDF (say, a signed invoice), see a confirmation flash, and download it again. Create `src/Controller/DocumentController.php`:

```php
<?php

declare(strict_types=1);

namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\BinaryFileResponse;
use Symfony\Component\HttpFoundation\File\UploadedFile;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

#[Route('/documents')]
class DocumentController extends AbstractController
{
    #[Route('', name: 'app_documents', methods: ['GET', 'POST'])]
    public function index(Request $request): Response
    {
        if ($request->isMethod('POST')) {
            return $this->handleUpload($request);
        }

        return $this->render('documents/index.html.twig', [
            'last_document' => $request->getSession()->get('last_document'),
        ]);
    }

    #[Route('/download', name: 'app_documents_download')]
    public function download(Request $request): Response
    {
        $path = $request->getSession()->get('last_document');

        if (null === $path || !is_file($path)) {
            return new Response('Nothing to download yet.', Response::HTTP_NOT_FOUND);
        }

        return new BinaryFileResponse($path, Response::HTTP_OK, [
            'Content-Disposition' => 'attachment; filename="document.pdf"',
        ]);
    }

    private function handleUpload(Request $request): Response
    {
        $file = $request->files->get('document');

        if (!$file instanceof UploadedFile || !$file->isValid()) {
            $this->addFlash('danger', 'No valid file was uploaded.');

            return $this->redirectToRoute('app_documents'); // 302
        }

        // Trust the server-side MIME guess, never the client's claim.
        if ('application/pdf' !== $file->getMimeType()) {
            $this->addFlash('danger', 'Only PDF files are allowed.');

            return $this->redirectToRoute('app_documents');
        }

        $dir = $this->getParameter('kernel.project_dir').'/var/uploads';
        if (!is_dir($dir)) {
            mkdir($dir, 0o775, true);
        }

        // Generate our own name; the client's is for display only.
        $stored = $file->move($dir, bin2hex(random_bytes(16)).'.pdf');

        $request->getSession()->set('last_document', (string) $stored);
        $this->addFlash('success', sprintf('Uploaded "%s".', $file->getClientOriginalName()));

        return $this->redirectToRoute('app_documents'); // the flash survives exactly this hop
    }
}
```

And `templates/documents/index.html.twig`:

```twig
{% extends 'base.html.twig' %}

{% block title %}Documents{% endblock %}

{% block body %}
    {% for label, messages in app.flashes %}
        {% for message in messages %}
            <div class="flash flash-{{ label }}">{{ message }}</div>
        {% endfor %}
    {% endfor %}

    <h1>Document inbox</h1>

    <form method="post" enctype="multipart/form-data">
        <input type="file" name="document" accept="application/pdf" required>
        <button type="submit">Upload</button>
    </form>

    {% if last_document %}
        <p><a href="{{ path('app_documents_download') }}">Download the last document</a></p>
    {% endif %}
{% endblock %}
```

Start the server (`symfony serve`) and walk through what just happened, because every line maps to something in this chapter:

1. **GET `/documents`** — the router matches the attribute route, the session is empty, and Twig renders the form. No flash yet.
2. **POST `/documents`** (multipart) — `isMethod('POST')` routes into `handleUpload()`. The file arrives in `$request->files`; we validate it with the *server-side* MIME guess, move it under a random name into `var/uploads/` (outside `public/`, so nobody can fetch it by URL), and record its path in the session.
3. **The 302** — `redirectToRoute()` sends a redirect; the flash message is stashed in the session for exactly one more request.
4. **GET `/documents` again** — the browser follows the redirect, Twig renders, and the flash loop prints "Uploaded *invoice.pdf*" once. Refresh: gone.
5. **GET `/documents/download`** — the session tells us where the file lives; `BinaryFileResponse` streams it back with a `Content-Disposition` header, so the browser saves rather than renders.

Notice what we did *not* do: no raw superglobals, no `header()` calls, no manual cookie handling. The `Request` came in fully assembled, and everything out — status codes, redirect, flash, file stream — was expressed as objects. That's the HttpFoundation contract, and every chapter from here on builds on it. Chapter 3 shows how URLs find their way to controllers like this one; Chapter 4 opens the hood on the kernel event pipeline that carries the `Request` through your application and turns your returned `Response` into bytes on the wire.

---

#### Exercises

1. **Request echo.** Add a controller at `/debug-request` returning a `JsonResponse` with the method, path, all query parameters, the `User-Agent` header, and the client IP. Verify with `curl 'http://localhost:8000/debug-request?foo=bar&baz=1'`.

2. **Filtered input.** Extend it to read `?page=` using `InputBag` filtering (default `1`, `FILTER_VALIDATE_INT`). Confirm that `?page=abc` yields the integer `1`, and `?page=07` yields `7` — not strings.

3. **Cookies.** Create a GET route `/visit` that reads a `visits` cookie, increments it, sets the cookie again (expiring in one day), and returns the new count as plain text. Visit three times with `curl -v` (watch the `Set-Cookie`/`Cookie` headers) or your browser's devtools.

4. **Flash without a form.** Add a POST route that adds a flash message and redirects back to a page that renders flashes. Confirm the message appears exactly once — refresh and it must be gone. What happens if you add *two* flashes of different labels in one request?

5. **Upload guardrails.** In the document inbox, reject files larger than 2 MB with a specific flash message (check `getSize()`), and display `UploadedFile::getMaxFilesize()` in the form so users know the server limit before they try.

6. **Downloads with the original name.** Serve the last uploaded file with a `Content-Disposition` that uses the *original* client filename instead of the stored random one (store it in the session alongside the path). What breaks when the original name contains non-ASCII characters or quotes? Research RFC 5987's `filename*` parameter and fix it.

7. ⭐ **Trusted proxies.** Put a reverse proxy (nginx, Caddy, or the Docker setup from Chapter 24) in front of `symfony serve`, configure `trusted_proxies` and `trusted_headers`, and verify that `getClientIp()` returns your machine's address rather than `127.0.0.1`. Then remove the configuration and explain, precisely, what you observe and why.

8. ⭐ **Craft requests in code.** Write a PHPUnit test — no server, no browser — that builds a request with `Request::create('/documents', 'POST')`, attaches an `UploadedFile` created in test mode (`new UploadedFile($path, 'test.pdf', 'application/pdf', null, true)`), invokes your controller directly, and asserts that the response is a 302 redirect and that a success flash was added. This is the testing style of Chapter 22, arriving early on purpose: if you can build requests by hand, HttpFoundation has clicked.

---

# Part I — Foundations

## Chapter 3: Routing

> **In this chapter, you will learn:**
> - How to define routes using attributes on controller classes and methods
> - How to use placeholders, requirements, and default values
> - How Symfony compiles and matches routes at runtime
> - How to issue redirects and control route priority
> - How to generate URLs in Twig templates and PHP code
> - How to debug your routing table from the console

Routing is the front door of every Symfony application. Before a controller runs, before a form is validated, before a query hits the database, Symfony's router decides *which* controller handles the incoming request. Get routing right and the rest of the application has a predictable, testable structure. Get it wrong and you end up with ambiguous URLs, 404 storms, and a maintenance burden that compounds with every new feature.

This chapter assumes you have completed the environment setup in the Front Matter and have a working Symfony project. All examples use a project skeleton created with:

```bash
symfony new routing-demo --starter=webapp
cd routing-demo
```

We will build a small set of routes throughout the chapter that later becomes the skeleton of the multi-tenant invoicing application in Part III.

---

### 3.1 Defining Routes with Attributes

Symfony 7 uses PHP attributes (the `#[Route]` attribute) as the canonical way to declare routes. XML and YAML route configuration still exist for edge cases and legacy bundles, but in a modern application you will almost never touch them.

The attribute lives in the `Symfony\Component\Routing\Attribute` namespace:

```php
use Symfony\Component\Routing\Attribute\Route;
```

#### 3.1.1 The Minimal Route

The simplest possible route is a single attribute on a controller method:

```php
// src/Controller/HomeController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class HomeController extends AbstractController
{
    #[Route('/')]
    public function index(): Response
    {
        return $this->render('home/index.html.twig');
    }
}
```

When the user visits `https://app.example.com/`, Symfony's router matches the path `/` to this action and dispatches the `index` method.

The first (positional) argument of `#[Route]` is the **path**. You can also use the named `path` argument, which is useful when you want to add other options on the same line:

```php
#[Route(path: '/home', name: 'home')]
```

#### 3.1.2 Route Names

Every route should have a unique name. The name is the identifier you use for URL generation (Section 3.5) and for referencing routes in other parts of the codebase. Omit the name and Symfony generates one automatically (e.g., `_home`), but explicit names are far more readable:

```php
#[Route('/dashboard', name: 'app_dashboard')]
public function dashboard(): Response
{
    // ...
}
```

**Convention:** Prefix names with a module or domain segment (`app_`, `invoice_`, `tenant_`). This prevents collisions as the project grows and makes `debug:router` output scannable.

#### 3.1.3 Class-Level Route Prefix

When a controller owns several routes under a common prefix, move the prefix to the class-level attribute. Every method-level route is then *appended* to the class prefix:

```php
#[Route('/tenant/{tenantSlug}')]
class TenantController extends AbstractController
{
    #[Route('', name: 'tenant_show')]
    public function show(string $tenantSlug): Response
    {
        // /tenant/acme
    }

    #[Route('/invoices', name: 'tenant_invoices')]
    public function invoices(string $tenantSlug): Response
    {
        // /tenant/acme/invoices
    }

    #[Route('/invoices/new', name: 'tenant_invoice_new')]
    public function newInvoice(string $tenantSlug): Response
    {
        // /tenant/acme/invoices/new
    }
}
```

The class-level attribute can also define `name` as a prefix for all method-level names. If you write `name: 'tenant_'` at the class level and `name: 'show'` at the method level, the full name becomes `tenant_show`. This keeps names DRY:

```php
#[Route('/tenant/{tenantSlug}', name: 'tenant_')]
class TenantController extends AbstractController
{
    #[Route('', name: 'show')]
    // Full name: tenant_show

    #[Route('/invoices', name: 'invoices')]
    // Full name: tenant_invoices
}
```

#### 3.1.4 Multiple Routes per Method

A single action can be reachable from several URLs. Pass an array of `Route` attributes:

```php
#[Route('/invoices', name: 'invoice_list')]
#[Route('/billing', name: 'billing_alias')]
public function listInvoices(): Response
{
    // Both URLs dispatch here.
    // The route name available via $request->attributes->get('_route')
    // depends on which URL was actually requested.
}
```

This is a legitimate pattern for legacy URL compatibility or for offering human-friendly aliases alongside canonical paths. Use it sparingly—every additional route increases the compiled matcher's size.

---

### 3.2 Placeholders, Requirements, and Defaults

#### 3.2.1 Placeholders (Route Variables)

Curly braces in the path define **variables**. The values are extracted from the request path and made available as route attributes:

```php
#[Route('/invoices/{id}', name: 'invoice_show')]
public function show(int $id): Response
{
    // $id is injected by the framework from the URL.
    // /invoices/42 → $id = 42
}
```

Symfony's `Router` component maps the extracted values to method parameters by name. The parameter must exist in the action signature (or be available as a route attribute). Type coercion happens automatically: the string `"42"` from the URL becomes the integer `42` because the parameter is typed `int`.

**Tip:** Always type-hint route parameters. Beyond the convenience of coercion, it makes the contract explicit and allows the framework to reject malformed input early.

#### 3.2.2 Inline Requirements

You can constrain a placeholder directly in the path using a colon and a regular expression:

```php
#[Route('/invoices/{id:\d+}', name: 'invoice_show')]
```

This means the route will only match if `{id}` is one or more digits. A request to `/invoices/abc` will not match this route (it will fall through to the next matching route or produce a 404).

Inline requirements are convenient for simple cases, but they become unreadable for complex patterns. For anything beyond `\d+` or `[a-z0-9]+`, use the dedicated `requirements` argument.

#### 3.2.3 The `requirements` Argument

The `requirements` option accepts an associative array mapping variable names to regex patterns:

```php
#[Route(
    path: '/tenant/{tenantSlug}/invoices/{invoiceNumber}',
    name: 'invoice_show',
    requirements: [
        'tenantSlug' => '[a-z0-9]{2,32}',
        'invoiceNumber' => '\d{4}-\d{5}',
    ]
)]
public function show(string $tenantSlug, string $invoiceNumber): Response
{
    // /tenant/acme/invoices/2024-00001 matches
    // /tenant/Acme/invoices/2024-00001 does NOT match (uppercase)
}
```

Requirements are compiled into the route's matcher. They are not validation in the sense of the Validator component—they determine whether the route *matches at all*. If a URL fails the requirement, the router simply tries the next route.

**Common patterns:**

| Pattern | Meaning |
|---------|---------|
| `\d+` | One or more digits |
| `\d{4}` | Exactly four digits |
| `[a-z0-9-]+` | URL-safe slug |
| `[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}` | UUID v4 |
| `(?:draft\|sent\|paid)` | Enumerated values |

#### 3.2.4 Defaults

When a placeholder appears in the path but should be optional, you supply a default value. The route will match with or without the variable:

```php
#[Route(
    path: '/invoices',
    name: 'invoice_list',
    defaults: ['status' => 'all']
)]
public function list(string $status): Response
{
    // /invoices → $status = 'all'
    // (There is no /invoices/{status} here—defaults alone don't add a placeholder.)
}
```

To make a placeholder truly optional in the URL, use a default *and* mark the variable as optional with a trailing pattern. Symfony does not natively support optional trailing placeholders the way some frameworks do, but you can simulate it with two routes:

```php
#[Route('/invoices', name: 'invoice_list_all')]
public function listAll(): Response
{
    return $this->redirectToRoute('invoice_list', ['status' => 'all']);
}

#[Route('/invoices/{status}', name: 'invoice_list',
    requirements: ['status' => 'draft|sent|paid|overdue']
)]
public function list(string $status): Response
{
    // /invoices/draft
    // /invoices/sent
}
```

This is the idiomatic Symfony approach: define explicit routes and redirect the catch-all to the specific one. It keeps the URL surface predictable and the router fast.

#### 3.2.5 The Full Attribute Signature

For reference, here is the complete set of commonly used arguments:

```php
#[Route(
    path: '/invoices/{id}',          // URL pattern
    name: 'invoice_show',            // Unique identifier
    methods: ['GET', 'HEAD'],        // HTTP methods (default: all)
    requirements: ['id' => '\d+'],   // Regex constraints
    defaults: ['page' => 1],         // Default values for variables
    options: ['compiler_class' => null], // Advanced compiler config
    stateless: false,                // Hints to the profiler (no session)
)]
```

The `methods` argument restricts which HTTP verbs the route responds to. If a `POST` arrives at a route defined with `methods: ['GET']`, Symfony returns a 405 Method Not Allowed (or, if another route matches the same path with `POST`, it dispatches that one).

**Best practice:** Always declare `methods` explicitly. A route that accepts all methods is a latent source of bugs and security issues.

---

### 3.3 Route Compilation and Matching

#### 3.3.1 What Happens at Compile Time

When Symfony boots (or when you run `cache:clear`), the **RouteCompiler** transforms each `#[Route]` declaration into a `CompiledRoute` object. This object contains:

- A compiled regular expression for matching the path
- A list of variable names and their positions
- Default values
- The route name and associated metadata (methods, requirements, etc.)

All compiled routes are stored in a single `CompiledUrlMatcher` (or `CompiledExpressionLanguageUrlMatcher` if you use expression language). This matcher is the hot path: on every request, the router walks the compiled tree to find a match.

The compiler performs several optimizations:

1. **Segment extraction.** Fixed path segments (like `/invoices`) are pulled out of the regex and matched as simple string comparisons, which are faster than backtracking.
2. **Variable reordering.** If two consecutive variables are unbounded (e.g., `{a}/{b}` with no requirements), the compiler may merge them into a single capture with a split, reducing backtracking.
3. **Tree structure.** Routes are organized in a trie (prefix tree) by their fixed segments. Matching `/invoices/42` first checks the `/invoices` prefix, then descends.

You do not need to manipulate this process directly, but understanding it explains *why* certain route patterns are faster than others and why the order of definitions matters.

#### 3.3.2 Route Priority and Matching Order

Routes are matched **in the order they are loaded**. Within a single controller, that means top-to-bottom in the file. Across the project, it means the order in which Symfony discovers your controllers (alphabetically by file path within `src/Controller/`).

This creates a practical rule:

> **More specific routes must be defined before more general routes.**

```php
class InvoiceController extends AbstractController
{
    // ✅ Specific: matches /invoices/42
    #[Route('/invoices/{id:\d+}', name: 'invoice_show')]
    public function show(int $id): Response { /* ... */ }

    // ✅ General: matches /invoices/new, /invoices/export, etc.
    #[Route('/invoices/{action}', name: 'invoice_action',
        requirements: ['action' => 'new|export|archive']
    )]
    public function action(string $action): Response { /* ... */ }
}
```

If you reversed these, `/invoices/42` would match the second route (with `action = "42"`), fail the requirement, and fall through to a 404—or worse, if the second route had no requirement, it would silently dispatch the wrong action.

**There is no explicit priority field.** You control ordering purely by definition order. If you find yourself needing to reorder routes across different files, it is usually a sign that the URL structure needs flattening.

#### 3.3.3 Expression Language in Routes

Symfony supports **ExpressionLanguage** in route paths and defaults, giving you dynamic route construction without a controller:

```php
#[Route(
    path: '/invoices/{id}',
    name: 'invoice_show',
    defaults: ['_controller' => 'App\Controller\InvoiceController::show']
)]
```

A more interesting use is conditional defaults:

```php
#[Route(
    path: '/reports/{year}',
    name: 'report_show',
    defaults: [
        'year' => 'date("Y")',  // Defaults to current year if omitted
    ]
)]
```

Expression language is evaluated at match time with a context that includes `request`, `context` (the router context), and `service` (the service locator). It is powerful but should be used judiciously—each expression adds a small runtime cost and makes the route harder to reason about statically.

---

### 3.4 Redirects

#### 3.4.1 RedirectResponse

The most common redirect is a temporary (302) or permanent (301) HTTP redirect. In a controller, return a `RedirectResponse`:

```php
use Symfony\Component\HttpFoundation\RedirectResponse;

#[Route('/old-invoices', name: 'legacy_invoices')]
public function legacyInvoices(): RedirectResponse
{
    return $this->redirectToRoute('invoice_list');
}
```

The `AbstractController::redirectToRoute()` helper (available in any controller extending `AbstractController`) builds the URL from a route name and optional parameters, then wraps it in a `RedirectResponse`. This is the **preferred** approach over hard-coding URLs:

```php
// ✅ Good—uses the route name
return $this->redirectToRoute('invoice_show', ['id' => 42]);

// ❌ Fragile—breaks if the path changes
return new RedirectResponse('/invoices/42');
```

#### 3.4.2 Redirect Status Codes

| Method | Status | Use case |
|--------|--------|----------|
| `$this->redirectToRoute()` | 302 | Temporary redirect (default) |
| `$this->redirect('/path', 301)` | 301 | Permanent redirect (SEO, URL cleanup) |
| `$this->redirect('/path', 307)` | 307 | Temporary, preserves HTTP method |
| `$this->redirect('/path', 308)` | 308 | Permanent, preserves HTTP method |

For a multi-tenant app, you will frequently redirect unauthenticated users to a login page or a tenant-selection page:

```php
#[Route('/tenant/{tenantSlug}', name: 'tenant_show')]
public function show(string $tenantSlug): Response|RedirectResponse
{
    $tenant = $this->tenantRepository->findBySlug($tenantSlug);

    if (!$tenant) {
        return $this->redirectToRoute('tenant_index');
    }

    // Redirect to the tenant's dashboard (the "real" landing page)
    return $this->redirectToRoute('tenant_dashboard', [
        'tenantSlug' => $tenantSlug,
    ]);
}
```

#### 3.4.3 Route Aliases via `toRoute`

You can define a route that is nothing but a redirect to another route, using the `to_route` default. This is useful for URL shortening or A/B testing:

```php
#[Route('/i/{id:\d+}', name: 'invoice_short',
    defaults: ['_controller' => 'Symfony\\Bundle\\FrameworkBundle\\Controller\\RedirectController::redirectAction']
)]
```

Or, more idiomatically, just write a one-line controller action that calls `redirectToRoute`. The `to_route` mechanism in YAML routing config does not have a direct attribute equivalent, so in practice a controller action is clearer.

---

### 3.5 URL Generation

#### 3.5.1 The `path()` Helper in Templates

In Twig, the `path()` function generates a URL from a route name and parameters:

```twig
{# base: templates/base.html.twig #}
<nav>
    <a href="{{ path('app_dashboard') }}">Dashboard</a>
    <a href="{{ path('invoice_list', { status: 'draft' }) }}">Draft Invoices</a>
    <a href="{{ path('invoice_show', { id: invoice.id }) }}">View</a>
</nav>
```

`path()` produces a **relative** URL by default (e.g., `/invoices/42`). Use `url()` instead when you need an **absolute** URL (including scheme and host), such as in email links or API responses:

```twig
{# In an email template #}
<p>Your invoice is ready: {{ url('invoice_show', { id: invoice.id }) }}</p>
{# Produces: https://app.example.com/invoices/42 #}
```

#### 3.5.2 The `UrlGeneratorInterface` in PHP

In controllers, services, and any other PHP code, inject `UrlGeneratorInterface`:

```php
use Symfony\Component\Routing\Generator\UrlGeneratorInterface;

class InvoiceNotifier
{
    public function __construct(
        private readonly UrlGeneratorInterface $urlGenerator,
    ) {}

    public function buildNotificationUrl(int $invoiceId): string
    {
        return $this->urlGenerator->generate(
            'invoice_show',
            ['id' => $invoiceId],
            UrlGeneratorInterface::ABSOLUTE_URL
        );
    }
}
```

The third argument controls the URL type:

| Constant | Result |
|----------|--------|
| `UrlGeneratorInterface::ABSOLUTE_PATH` | `/invoices/42` (default) |
| `UrlGeneratorInterface::ABSOLUTE_URL` | `https://app.example.com/invoices/42` |
| `UrlGeneratorInterface::NETWORK_PATH` | `/invoices/42` (without query string) |
| `UrlGeneratorInterface::NETWORK_HOST` | `example.com` |
| `UrlGeneratorInterface::RELATIVE_PATH` | `invoices/42` (no leading slash) |

#### 3.5.3 Generating URLs with Extra Parameters

If you need to append query parameters that are not part of the route definition, pass them in the parameters array. Symfony appends them as a query string:

```php
$url = $this->urlGenerator->generate('invoice_list', [
    'status' => 'draft',
    'page' => 3,
    'sort' => 'created_at:desc',
]);
// /invoices/draft?page=3&sort=created_at%3Adesc
```

This is the standard pattern for preserving pagination and filters across links (e.g., a "next page" link that keeps the current filter applied).

#### 3.5.4 Current Route in Templates

Twig exposes the current route name via the `app` variable (available when using `framework: true` in Twig config, which is the default):

```twig
{% if app.request.attributes.get('_route') == 'invoice_show' %}
    <span class="breadcrumb active">Invoice</span>
{% endif %}
```

This is useful for highlighting the active navigation item.

---

### 3.6 Debugging Routes

#### 3.6.1 The `debug:router` Command

The single most useful command when working with routes:

```bash
$ php bin/console debug:router
```

Example output:

```
-------------------------- -------- ----------------------------------------
  Route name                 Methods  URI Pattern
-------------------------- -------- ----------------------------------------
  app_dashboard              GET      /dashboard
  tenant_show                GET      /tenant/{tenantSlug}
  tenant_invoices            GET      /tenant/{tenantSlug}/invoices
  invoice_show               GET      /invoices/{id}
  invoice_list               GET      /invoices/{status}
  invoice_list_all           GET      /invoices
  legacy_invoices            ANY      /old-invoices
-------------------------- -------- ----------------------------------------
```

You can filter by name or path:

```bash
$ php bin/console debug:router invoice
$ php bin/console debug:router --filter=/tenant
```

Add `-v` (verbose) to see the full route definition including requirements, defaults, and the controller:

```bash
$ php bin/console debug:router -v invoice_show

Route:          invoice_show
Path:           /invoices/{id}
Methods:        GET
Requirements:   id = \d+
Defaults:       _controller = App\Controller\InvoiceController::show
```

#### 3.6.2 The Web Profiler

When you have the Web Profiler enabled (default in `dev`), every request profile page includes a **Router** panel that shows:

- Which route matched
- The matched route name and path
- The extracted route attributes (variables)
- The time spent in route matching

This is invaluable when a request is hitting the wrong route or when you are unsure whether a variable was extracted correctly.

#### 3.6.3 Troubleshooting Checklist

When a route does not match as expected, work through this list:

1. **Is the controller in the right directory?** Symfony's autoconfiguration scans `src/Controller/`. A controller in `src/Service/` will not be registered as a route.
2. **Is the route name spelled correctly in `debug:router` output?** A typo in `redirectToRoute('invoice_shwo')` throws a `RouteNotFoundException`.
3. **Do the requirements reject the URL?** A request to `/invoices/abc` will not match a route requiring `\d+`. Check with `debug:router -v`.
4. **Is another route matching first?** Use the profiler to see which route actually matched. Remember: order matters.
5. **Is the HTTP method allowed?** A `POST` to a `GET`-only route produces a 405. Check the Methods column in `debug:router`.
6. **Is the route in a bundle that is not registered?** If the controller is in a bundle, confirm the bundle is listed in `config/bundles.php`.

#### 3.6.4 The `Router` Service for Programmatic Matching

In rare cases (e.g., a middleware that inspects the URL before the router runs), you may need to match a URL programmatically:

```php
use Symfony\Component\Routing\Matcher\UrlMatcherInterface;

class RouteInspector
{
    public function __construct(
        private readonly UrlMatcherInterface $urlMatcher,
    ) {}

    public function inspect(string $pathInfo, string $method): array
    {
        return $this->urlMatcher->matchRequest(
            new Request($pathInfo, [], [], [], [], ['REQUEST_METHOD' => $method])
        );
    }
}
```

The returned array contains the route name (`_route`), the controller (`_controller`), and all extracted variables. This is mostly useful for testing and for building custom routing logic (e.g., tenant resolution based on subdomain or path prefix).

---

### 3.7 Putting It Together: An Invoicing Route Table

Let us sketch the route table for the multi-tenant invoicing application we will build in Part III. This is not yet functional code—the controllers are stubs—but it demonstrates how the routing concepts compose:

```php
// src/Controller/TenantController.php
#[Route('/tenant/{tenantSlug}', name: 'tenant_')]
class TenantController extends AbstractController
{
    #[Route('', name: 'dashboard', methods: ['GET'])]
    public function dashboard(string $tenantSlug): Response
    {
        return $this->render('tenant/dashboard.html.twig');
    }

    #[Route('/settings', name: 'settings', methods: ['GET', 'POST'])]
    public function settings(string $tenantSlug): Response
    {
        // ...
    }
}

// src/Controller/InvoiceController.php
#[Route('/tenant/{tenantSlug}/invoices', name: 'invoice_')]
class InvoiceController extends AbstractController
{
    #[Route('', name: 'list', methods: ['GET'])]
    public function list(string $tenantSlug, string $status = 'all'): Response
    {
        // /tenant/acme/invoices
        // /tenant/acme/invoices?status=draft
    }

    #[Route('/new', name: 'new', methods: ['GET', 'POST'])]
    public function new(string $tenantSlug): Response
    {
        // /tenant/acme/invoices/new
    }

    #[Route('/{id:\d+}', name: 'show', methods: ['GET'])]
    public function show(string $tenantSlug, int $id): Response
    {
        // /tenant/acme/invoices/42
    }

    #[Route('/{id:\d+}/edit', name: 'edit', methods: ['GET', 'POST'])]
    public function edit(string $tenantSlug, int $id): Response
    {
        // /tenant/acme/invoices/42/edit
    }

    #[Route('/{id:\d+}', name: 'delete', methods: ['DELETE'])]
    public function delete(string $tenantSlug, int $id): Response
    {
        // DELETE /tenant/acme/invoices/42
    }
}
```

Notice the structure:

- **Tenant is a first-class URL segment.** Every resource lives under `/tenant/{tenantSlug}/`, which will make tenant isolation trivial in later chapters (security voters, Doctrine filters, cache keys).
- **Specific routes come before general ones.** `/invoices/new` is defined before `/invoices/{id}` so that the literal `new` is not captured as an ID.
- **HTTP methods are declared explicitly.** The `delete` action uses `DELETE`, which in practice will be sent as a `POST` with a `_method` override (covered in Chapter 10, Forms).
- **Names are prefixed by the class-level `name: 'invoice_'`.** The full names are `invoice_list`, `invoice_new`, `invoice_show`, etc.

Run `php bin/console debug:router` after adding these controllers and verify that all routes appear with the expected methods and requirements.

---

### 3.8 Common Pitfalls

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Forgetting `methods` | Route matches any verb, including `DELETE` on a list endpoint | Always specify `methods` |
| General route before specific route | `/invoices/new` dispatches to the `{id}` route | Reorder: specific first |
| Hard-coded URLs in controllers | Breaks when the path changes | Use `redirectToRoute()` / `path()` |
| Missing route name | `RouteNotFoundException` at generation time | Give every route an explicit `name` |
| Using `Annotation\Route` import | Deprecation warning in 7.x, removed in 8.x | Use `Symfony\Component\Routing\Attribute\Route` |
| Unbounded variable adjacent to another variable | Ambiguous match (e.g., `/a/b/c` — where does `{x}` end?) | Add a requirement or restructure the path |
| Forgetting the class-level prefix | All routes lose the prefix; URLs are wrong | Check that the class attribute is present and correctly formatted |

---

### 3.9 Exercises

**Exercise 1: Route table audit**

Create a new controller, `src/Controller/ReportController.php`, with the following routes. After each one, run `php bin/console debug:router report` and verify the output.

1. `GET /reports` → `report_index`
2. `GET /reports/{year:\d{4}}` → `report_yearly`
3. `GET /reports/{year:\d{4}}/{month:\d{2}}` → `report_monthly`
4. `GET /reports/export` → `report_export`
5. `GET /reports/export/{year:\d{4}}` → `report_export_yearly`

**Question:** Does route 4 (`/reports/export`) conflict with route 2 (`/reports/{year}`)? What happens when a user visits `/reports/export`? Explain why, and confirm by testing in the browser or with `curl`.

**Exercise 2: URL generation in a service**

Create a service `App\Service\InvoiceLinkBuilder` that takes `UrlGeneratorInterface` in its constructor and exposes:

```php
public function invoiceUrl(int $tenantId, int $invoiceId): string;
// Returns absolute URL to invoice_show

public function listUrl(int $tenantId, string $status, int $page): string;
// Returns absolute URL to invoice_list with query params
```

Write a unit test (you will formalize testing in Chapter 22, but a quick test now reinforces the concept) that asserts the generated URLs match expected strings.

**Exercise 3: Redirect chain**

Add a route `GET /invoices` that redirects to `GET /tenant/{tenantSlug}/invoices` using the *first* tenant in the database (assume a simple `Tenant` entity with a `slug` field and a `TenantRepository`). What happens if no tenants exist? Handle that case with a redirect to `tenant_create` (a route you will build in Part III; for now, just reference the name).

**Exercise 4: Requirements challenge**

Define a route for `/invoices/{invoiceNumber}` where `invoiceNumber` must match the pattern `INV-\d{4}-\d{5}` (e.g., `INV-2024-00001`). Write two test URLs that should match and two that should not. Verify with `debug:router -v` and by making requests with `curl`.

**Exercise 5: Priority investigation**

Create two routes in the same controller:

```php
#[Route('/api/{version}/invoices', name: 'api_invoices')]
public function apiV1(string $version): Response { /* ... */ }

#[Route('/api/v2/invoices', name: 'api_invoices_v2')]
public function apiV2(): Response { /* ... */ }
```

Which route matches a request to `/api/v2/invoices`? What if you swap the definition order? Explain the result in terms of route compilation order.

---

### Summary

| Concept | Key takeaway |
|---------|-------------|
| `#[Route]` attribute | The primary way to define routes in Symfony 7 |
| Class-level prefix | DRYs up common path segments and name prefixes |
| Placeholders | Extract URL segments into typed method parameters |
| Requirements | Regex constraints that determine whether a route matches at all |
| Defaults | Supply values for variables not present in the URL |
| Route order | Specific routes before general ones; no explicit priority field |
| `redirectToRoute()` | Generate redirect URLs from route names, never hard-code paths |
| `path()` / `url()` in Twig | Generate relative/absolute URLs in templates |
| `UrlGeneratorInterface` in PHP | Generate URLs in services and controllers |
| `debug:router` | Your first stop when a route does not behave as expected |

In the next chapter, we turn to what happens *after* the router finds its match: the controller layer and the full request lifecycle from kernel boot to response send.
---

## Chapter 4 — Controllers and the Request Lifecycle

> **In this chapter you will learn to:**
> - Write the four flavors of Symfony controller and choose the right one for each job.
> - Follow a single HTTP request from `public/index.php` all the way to the bytes sent back to the browser, and name every kernel event along the way.
> - Return HTML, JSON, files, streams, and redirects — and know exactly what each one does under the hood.
> - Hook into the lifecycle with event listeners and subscribers to solve cross-cutting problems.
> - Debug routing and controller resolution with the console and the profiler.

By now you can build a `Request` and a `Response` by hand (Chapter 2) and you can map a URL to a controller with the `#[Route]` attribute (Chapter 3). This chapter is the missing middle: what actually *runs* between those two objects, and how you write the code that does.

> **A note on the running project.** The multi-tenant invoicing app we build in Parts III–V does not start until Chapter 9. The examples in this chapter are deliberately self-contained, but they are flavored with invoicing so the patterns land naturally when we return to them.

---

### 4.1 Anatomy of a Controller

A controller is, at bottom, **any PHP callable that turns a `Request` into a `Response`**. That could be a bare function, a closure, or — by far the most common case — a method on a class. The contract is simple:

1. Symfony hands you a `Request` (and usually a few arguments).
2. You do the work for this one page.
3. You return a `Response`.

Here is the minimal method controller, building directly on Chapter 3:

```php
// src/Controller/LuckyNumberController.php
namespace App\Controller;

use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class LuckyNumberController
{
    #[Route('/lucky/{max}', name: 'app_lucky', methods: ['GET'])]
    public function number(int $max): Response
    {
        $number = random_int(0, $max);

        return new Response(
            '<html><body>Lucky number: '.$number.'</body></html>'
        );
    }
}
```

A few things to notice:

- The controller **must** return a `Response` (or one of its subclasses). We cover the full menu of return types in §4.6.
- The `{max}` placeholder in the route (Chapter 3) becomes the `int $max` argument by name. Symfony fills in controller arguments for you — §4.5 goes deep on this.
- The class is named `*Controller` by convention, but the name is not load-bearing.
- We did **not** extend any base class here. A controller does not *require* a base class — it only requires that it's resolvable. That is worth a moment of attention, because it's where most of the "controller types" discussion lives (§4.3).

The one method you will almost always want from `AbstractController` is `render()`, which renders a Twig template and wraps it in a `Response` for you (Twig itself is Chapter 9):

```php
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;

class LuckyNumberController extends AbstractController
{
    #[Route('/lucky/{max}', name: 'app_lucky', methods: ['GET'])]
    public function number(int $max): Response
    {
        return $this->render('lucky/number.html.twig', [
            'number' => random_int(0, $max),
            'max'    => $max,
        ]);
    }
}
```

#### 4.1.1 The `AbstractController` base class

`AbstractController` is optional, but it is the default choice for a reason. It gives you a small set of convenience methods — `render()`, `generateUrl()`, `json()`, `redirect*()`, `binaryFile()`, `createNotFoundException()`, `isGranted()`, `getUser()` — so you do not have to wire up the underlying services yourself.

How does it get those services? It uses a **lazy service locator**. When you call `$this->render()`, the base class resolves the `twig` service *at that moment* and calls it. It does not inject every service into the controller up front; it fetches only what each helper needs. That keeps the controller lightweight and avoids a long constructor.

You should learn to read `AbstractController` as a reference manual. Every helper is a few lines, and each one shows you the "real" service behind the shortcut. When you outgrow the helpers, you swap them for direct service injection (§4.3.3).

The core helpers, and what they wrap:

| Method | Returns | Wraps |
|---|---|---|
| `render($view, $params)` | `Response` | the `twig` service |
| `generateUrl($route, $params)` | `string` | the `router` service |
| `redirect($url)` | `RedirectResponse` | — |
| `redirectToRoute($route, $params, $status)` | `RedirectResponse` | `generateUrl()` + `redirect()` |
| `json($data, $status, $headers, $context)` | `Response` | `json_encode` + JSON headers |
| `stream($callback, $status, $headers)` | `StreamedResponse` | — |
| `binaryFile($path, $name, ...)` | `BinaryFileResponse` | — |
| `file($path, $disposition, ...)` | `BinaryFileResponse` | — |
| `createNotFoundException($message)` | throws `NotFoundHttpException` | — |
| `isGranted($attribute, $subject)` | `bool` | the security `AuthorizationChecker` |
| `getUser()` | `UserInterface\|null` | the security `TokenStorage` |

> **Tip** The security helpers (`isGranted()`, `getUser()`) are used heavily from this chapter forward, but they are fully explained in Chapter 12. For now, treat them as "ask the security layer a question."

---

### 4.2 A Slightly Larger, Realistic Controller

Before we split controllers into types, look at a controller that resembles what you will actually write in the invoicing app: route parameters, a service dependency, an error case, and a template.

```php
// src/Controller/InvoiceController.php
namespace App\Controller;

use App\Repository\InvoiceRepository;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class InvoiceController extends AbstractController
{
    public function __construct(
        private readonly InvoiceRepository $invoices,
    ) {
    }

    #[Route('/invoices', name: 'app_invoice_index', methods: ['GET'])]
    public function index(): Response
    {
        $invoices = $this->invoices->findBy(['status' => 'unpaid']);

        return $this->render('invoice/index.html.twig', ['invoices' => $invoices]);
    }

    #[Route(
        '/invoices/{invoice}',
        name: 'app_invoice_show',
        requirements: ['invoice' => '\d+'],
        methods: ['GET'],
    )]
    public function show(int $invoice): Response
    {
        $invoiceEntity = $this->invoices->find($invoice)
            ?? throw $this->createNotFoundException('Invoice not found.');

        return $this->render('invoice/show.html.twig', ['invoice' => $invoiceEntity]);
    }
}
```

Three patterns are doing the heavy lifting here, and each is a topic in this chapter:

- **Constructor injection** (`private readonly InvoiceRepository $invoices`) — this only works because the controller is a *service*. That is §4.3.
- **Route parameter → typed argument** (`{invoice}` → `int $invoice`) — covered in §4.5.
- **The `?? throw` idiom** for "not found" — PHP 8.0's null-coalescing throw, paired with `createNotFoundException()` to produce a proper 404. The full error model is §4.6.4.

---

### 4.3 Controller Types and How They're Instantiated

This is the section that surprises people, because "controller" hides four distinct shapes. The differences are not about *what* a controller does, but about *how Symfony instantiates it* and *how it gets its dependencies*. Get this straight and the rest of the book gets easier.

#### 4.3.1 The four flavors

**1. Method controller (the default).** One class, several action methods, a `#[Route]` on each method. This is what you've seen throughout. It is the right default for a resource with several actions (index, show, create, edit, delete).

**2. Invokable controller.** One class, exactly one action, defined as `__invoke()`, with the `#[Route]` on the *class*:

```php
// src/Controller/HelloController.php
namespace App\Controller;

use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

#[Route('/hello/{name}', name: 'app_hello', methods: ['GET'])]
class HelloController
{
    public function __invoke(string $name = 'World'): Response
    {
        return new Response(sprintf('Hello %s!', $name));
    }
}
```

Invokable controllers are the backbone of the **ADR pattern** (Action–Domain–Responder): each action is its own small, focused class. That makes a single endpoint easier to read, test, and reason about. They are a natural fit for one-off routes that do not belong to a resource (a webhook receiver, a checkout-confirmation page, an import trigger).

**3. Service controller.** A controller that is registered in the service container and therefore gets **constructor injection** like any other service. The `InvoiceController` in §4.2 is one.

**4. Value-object (prototype) controller.** The same idea as a service controller, but the service is configured with `shared: false`, so Symfony builds a **fresh instance per request** instead of reusing one singleton:

```yaml
# config/services.yaml
services:
    App\Controller\InvoiceController:
        shared: false
```

#### 4.3.2 Why instantiation matters: shared vs. fresh

A service controller is, by default, a **shared singleton**: the container creates the instance once and hands the same object to every request. A value-object controller is **per-request**: a brand-new object on each request.

This distinction is only dangerous if you store *per-request* data in an instance property:

```php
// DANGEROUS in a shared (singleton) controller
class BadController extends AbstractController
{
    private ?Invoice $lastInvoice = null;   // ← state that leaks between requests

    public function show(int $id): Response
    {
        $this->lastInvoice = $this->repo->find($id);
        // ...
    }
}
```

With `shared: true` (the default), the *same* object serves every request, so `$lastInvoice` from request A can be observed in request B. That is a bug waiting to happen (and a concurrency footgun under PHP-FPM workers).

You have two clean fixes, and the professional one is usually the first:

1. **Do not store request data in properties.** Pass everything through method arguments (route params, the `Request`, and services). A stateless singleton controller is perfectly safe and marginally cheaper because it is instantiated only once.
2. **If a controller genuinely needs mutable per-request state**, make it a value object with `shared: false`.

> **Rule of thumb** Keep controllers thin and stateless; push real logic into services. When you do that, the singleton-vs-prototype question mostly takes care of itself. Reach for `shared: false` only when a controller is deliberately carrying request-scoped state.

> **History note.** Before "service controllers" existed, every controller was instantiated fresh per request with its dependencies injected — in other words, the value-object model was the *default*. Modern Symfony flipped the default to shared services for performance, and gives you `shared: false` to opt back into per-request instances.

#### 4.3.3 Getting dependencies in: constructor injection vs. argument injection

There are **two** ways a service reaches your controller code, and you can mix them.

**Constructor injection** — dependencies are fixed for the object's lifetime. Works only when the controller is a service. Best for services used by *every* action of that controller.

```php
public function __construct(
    private readonly InvoiceRepository $invoices,
    private readonly LoggerInterface $logger,
) {
}
```

**Argument (action) injection** — you type-hint a service as an *argument of the action method*, and Symfony injects it just for that call. This works whether or not the controller is a service:

```php
#[Route('/lucky/{max}', name: 'app_lucky')]
public function number(int $max, RandomNumberService $random): Response
{
    return new Response((string) $random->draw($max));
}
```

Both forms use the same autowiring you will study in depth in Chapter 5. To see what is injectable, run:

```bash
$ php bin/console debug:autowiring
```

When a plain type-hint is not enough, the `#[Autowire]` attribute narrows the choice — pick a specific service, or inject a container parameter:

```php
use Symfony\Component\DependencyInjection\Attribute\Autowire;

public function number(
    int $max,
    #[Autowire(service: 'monolog.logger.request')] LoggerInterface $logger,
    #[Autowire('%app.lucky.max%')] int $defaultMax,
): Response {
    // ...
}
```

#### 4.3.4 How a controller becomes a service

A controller does **not** need to be a service to run. But if you want constructor injection, it must be one. There are three opt-in mechanisms, and they all do the same thing internally: apply the **`controller.service_arguments`** tag.

1. **Extend `AbstractController`** — with the default `services.yaml` (autowiring + autoconfiguration on), subclasses are registered as services for you. This is why it is the path of least resistance.
2. **Put `#[Route]` on the class** — Symfony automatically tags the class with `controller.service_arguments`, even if it does not extend `AbstractController`.
3. **Use the `#[AsController]` attribute** — the explicit, self-documenting way to say "this class is a controller service":

```php
use Symfony\Component\HttpKernel\Attribute\AsController;
use Symfony\Component\Routing\Attribute\Route;

#[AsController]
class ReportController
{
    public function __construct(private InvoicePdfGenerator $pdf) {}

    #[Route('/reports', name: 'app_reports')]
    public function index(): Response
    {
        // ...
    }
}
```

4. **The `controller.service_arguments` tag directly** — for the rare case where you want full manual control:

```yaml
# config/services.yaml
App\Controller\:
    resource: '../src/Controller/'
    tags: ['controller.service_arguments']
```

What the tag actually does: it marks the service **public and non-lazy**. Public, because the controller resolver fetches controllers from the container by service ID at runtime — and private services cannot be fetched that way. Non-lazy, because the resolver calls the controller immediately after fetching it, so there is no point wrapping it in a proxy.

> **Security detail — the controller allowlist.** For security, Symfony keeps an allowlist of controller types that may handle a request (this matters for fragment rendering and ESI). Anything using `#[AsController]`, anything extending `AbstractController`, anything tagged `controller.service_arguments`, and the built-in `TemplateController` are allowed automatically. If you, as a bundle author, register a controller that matches none of these, you explicitly register it with the `controller_resolver` service's `allowControllers()` method. You will not hit this often, but it explains why the framework is picky about *how* a controller is declared.

#### 4.3.5 Choosing a controller type

| Situation | Reach for |
|---|---|
| A resource with several actions (CRUD) | Method controller, `AbstractController` |
| One self-contained endpoint (webhook, one-off page) | Invokable controller (ADR) |
| Several actions sharing the same dependencies | Service controller + constructor injection |
| Controller must hold mutable per-request state | Value-object controller (`shared: false`) |
| Only one action needs a given service | Argument injection (type-hint the method) |

> **Tip** When in doubt: method controller, `AbstractController`, argument injection for the occasional dependency, constructor injection only for dependencies shared by every action. That combination is stateless, testable, and unremarkable — which is exactly what you want 90% of the time.

---

### 4.4 The Request Lifecycle: From `index.php` to the Wire

Now the heart of the chapter. Everything a Symfony web app does to answer a request is orchestrated by the **HttpKernel** component, and it is *entirely event-driven*. The kernel's `handle()` method contains almost no business logic of its own; it **dispatches events**, and the work is done by **listeners** attached to those events. If you can picture the pipeline, you can insert almost anything anywhere.

#### 4.4.1 The front controller

Every request hits a single PHP file, `public/index.php`. It does essentially three things, wrapped by the Symfony Runtime:

```php
// Conceptually, the front controller:
$request  = Request::createFromGlobals();   // build a Request from globals
$response = $kernel->handle($request);       // run the event pipeline
$response->send();                           // emit headers + body
$kernel->terminate($request, $response);     // dispatch kernel.terminate
```

The `handle()` call is where the pipeline runs. `send()` writes the response to the client. `terminate()` runs *after* the client has what it needs — the ideal place for work that must not slow down the response.

#### 4.4.2 The pipeline, end to end

```
Browser              Web server            Symfony Kernel — handle()
  |                        |                          |
  |  HTTP Request          |                          |
  |----------------------->|                          |
  |                        | Request::createFromGlobals
  |                        |------------------------->|
  |                        |                          |
  |                        |   1. kernel.request        routing, locale, session,
  |                        |                          (security may short-circuit here)
  |                        |   2. resolve controller    look up _controller -> callable
  |                        |   3. kernel.controller     inspect or replace the controller
  |                        |   4. kernel.controller_arguments  build the method's arguments
  |                        |   5. CALL THE CONTROLLER
  |                        |   6. kernel.view           ONLY if the result is not a Response
  |                        |   7. kernel.response       mutate headers, add cookies, ...
  |                        |   8. kernel.finish_request reset per-subrequest globals
  |                        |                          |
  |  HTTP Response         |                          |
  |<---------------------- |<--------------------------|
  |                        |   9. kernel.terminate      AFTER send()
```

Let's walk it.

**Step 1 — `kernel.request`.** The first event inside `handle()`. Listeners here either (a) *add information to the `Request`*, or (b) *return a `Response` immediately*, in which case the pipeline skips straight to `kernel.response` and the controller is never resolved.

The most important built-in listener at this point is the **RouterListener**: it matches the URL against your routes (Chapter 3) and stores the result — including the `_controller` value and any route parameters — on the `Request`'s `attributes` bag. Other early listeners set the session, the locale, and so on.

> **Note — short-circuiting.** If *any* `kernel.request` listener sets a response, propagation stops: lower-priority listeners on the same event do not run. This is how a security layer can produce a 403 or a redirect-to-login without ever touching your controller.

**Step 2 — resolve the controller.** The **ControllerResolver** reads `_controller` from the request attributes and turns that string (e.g. `App\Controller\InvoiceController::show`) into a real PHP callable. This is where a service controller is fetched from the container, or where a plain class is instantiated.

**Step 3 — `kernel.controller`.** Dispatched after the controller is resolved but *before* it runs. Two canonical uses: read custom attributes off the controller, or **replace the controller entirely** via `$event->setController($callable)`. The framework's `#[Cache]` attribute listener lives here — it reads caching directives from the controller and applies them to the response.

**Step 4 — `kernel.controller_arguments`.** Dispatched just before the call. This is where Symfony resolves the method's arguments — mapping route parameters to named arguments, injecting the `Request`, injecting services. It is also the one place where you can rewrite the exact arguments the controller will receive.

**Step 5 — the controller runs.** Your method executes and returns something.

**Step 6 — `kernel.view` (conditional).** Dispatched **only if** the controller did *not* return a `Response`. The whole point of this event is to convert a non-`Response` return (a string, an array, an object) into a real `Response` via `$event->setResponse(...)`. If you always return a `Response` from your controllers — which is the norm — you will never see this event fire.

**Step 7 — `kernel.response`.** Dispatched once a `Response` exists (from the controller, or from a `kernel.view` listener). This is the classic place to mutate the response: add HTTP headers, set cookies, adjust the status. Many cross-cutting concerns land here.

**Step 8 — `kernel.finish_request`.** Dispatched after `kernel.response`. Its job is to *reset global state* — most notably the locale — which matters when sub-requests are involved. You rarely write listeners for it yourself.

**Step 9 — `kernel.terminate`.** Dispatched *after* `handle()` returns and the response has been sent. Anything slow or non-critical — background bookkeeping, logging, "you are now offline" analytics — belongs here, because it can no longer delay the user.

**The side branch — `kernel.exception`.** At *any* point, if a throwable escapes, the kernel dispatches `kernel.exception`. Listeners can catch it, replace it, or set a response (turning the error into a normal response). The framework's `ErrorListener` uses this to turn exceptions into the status-coded error pages you are used to seeing. The status-code rules: if the response is a client/server error or redirect, its code wins; otherwise, if the exception implements `HttpExceptionInterface`, the exception's status is used; otherwise it is a 500.

#### 4.4.3 The event class reference

Every kernel event is a subclass of `KernelEvent`, which gives you `getRequest()`, `getKernel()`, `isMainRequest()`, and `getRequestType()`. The concrete event for each hook is:

| Event name | Event class | Typical use |
|---|---|---|
| `kernel.request` | `RequestEvent` | Routing; add data to the request; short-circuit with an early response |
| `kernel.controller` | `ControllerEvent` | Read controller attributes; replace the controller |
| `kernel.controller.arguments` | `ControllerArgumentsEvent` | Rewrite the controller's arguments |
| `kernel.view` | `ViewEvent` | Turn a non-`Response` return into a `Response` |
| `kernel.response` | `ResponseEvent` | Mutate the response (headers, cookies, status) |
| `kernel.finish_request` | `FinishRequestEvent` | Reset global state between (sub)requests |
| `kernel.terminate` | `TerminateEvent` | Post-response, non-critical work |
| `kernel.exception` | `ExceptionEvent` | Catch and recover from errors |

Each corresponds to a `KernelEvents` constant, so you register listeners by constant rather than string: `KernelEvents::RESPONSE`, `KernelEvents::REQUEST`, and so on.

> **Since Symfony 8.1** the post-resolution events expose a `controllerMetadata` property, and `ControllerEvent`/`ResponseEvent` add `getAttributes()` / `getControllerAttributes()` and `evaluate()` helpers so listeners can read and evaluate controller attributes without doing their own reflection. If you are on 7.4 LTS you will not have these; on 8.x they make attribute-driven listeners cleaner. The core pipeline is identical across both.

#### 4.4.4 Priorities: controlling listener order

Listeners on the same event run in **priority order: higher numbers run first**, default is `0`, negative numbers run later. This is how the framework sequences, say, routing before locale handling.

You do not need to memorize the built-in priorities — the console is the source of truth:

```bash
$ php bin/console debug:event-dispatcher kernel.request
```

That prints every listener registered for the event, in the order it will run, with its priority. A few you will recognize (illustrative defaults — check your own app):

| Listener | Priority | Role |
|---|---|---|
| `ValidateRequestListener` | 256 | Reject malformed requests very early |
| `SessionListener` | 128 | Attach the session to the request |
| `RouterListener` | 32 | Match the route, set `_controller` |
| `LocaleListener` | 16 | Set the locale from the route/session/negotiation |
| `ProfilerListener` | 0 | Start profiling (when enabled) |

> **Tip** The single most useful debugging habit in this chapter: when something in the pipeline is not behaving, run `debug:event-dispatcher <event>` and read the ordered list. It turns "why is my listener not running?" into a 5-second lookup.

---

### 4.5 Injecting Controller Arguments

A controller method can receive a surprising number of things, and Symfony fills them in for you based on the method signature. Here is the family, in roughly the order you will use them.

**Route parameters.** A `{invoice}` in the route becomes a same-named argument. Type the argument and Symfony casts it:

```php
#[Route('/invoices/{invoice}', name: 'app_invoice_show', requirements: ['invoice' => '\d+'])]
public function show(int $invoice): Response { /* ... */ }
```

**The `Request` object.** Type-hint `Request` to read the query string, headers, uploaded files, etc. (Chapter 2):

```php
public function index(Request $request): Response
{
    $page = $request->query->getInt('page', 1);
    // ...
}
```

**Services.** Type-hint any autowirable service (§4.3.3).

**Query parameters, individually — `#[MapQueryParameter]`.** Pull a single query string value into a typed, optionally filtered argument:

```php
use Symfony\Component\HttpKernel\Attribute\MapQueryParameter;

public function dashboard(
    #[MapQueryParameter] int $page = 1,
    #[MapQueryParameter] string $sort = 'created',
): Response {
    // ...
}
```

It supports scalars, arrays, `\BackedEnum`, and UIDs, and accepts a `filter` (a PHP `FILTER_*` constant) for validation:

```php
#[MapQueryParameter(filter: FILTER_VALIDATE_REGEXP, options: ['regexp' => '/^\w+$/'])]
string $firstName,
```

**The whole query string into a DTO — `#[MapQueryString]`.** Map all (or a keyed subset of) query parameters onto an object, with optional validation:

```php
use Symfony\Component\HttpKernel\Attribute\MapQueryString;
use App\Model\InvoiceFilter;

public function index(
    #[MapQueryString] InvoiceFilter $filter,
): Response {
    // $filter is a validated object built from the query string
}
```

`InvoiceFilter` is a plain DTO with validation constraints; Symfony denormalizes the query string into it and returns a 404 by default if validation fails (overridable via the attribute's `validationFailedStatusCode` and `validationGroups`).

> **The wider family.** `#[MapQueryParameter]` and `#[MapQueryString]` are the most commonly used, but they belong to a group of "map this part of the request into an argument" attributes: `#[MapRequestPayload]` (map a JSON/FORM body into a validated object — central to APIs in Part V), `#[MapUploadedFile]` (map an upload), and `#[MapRequestHandler]`. Learn the pattern — *declare an attribute, get a typed, validated value* — and the rest are mechanical.

#### 4.5.1 A note on sub-requests

Every one of these events and arguments works on the **main request**, but Symfony can also handle **sub-requests** internally (rendering a fragment of one page inside another, for example). That is why events carry `isMainRequest()`. When you write a listener, guard with `if (!$event->isMainRequest()) { return; }` unless you specifically want the behavior to run for sub-requests too. It is the single most common source of "why is my code running twice?" confusion.

---

### 4.6 Returning Things: The Response Menu

A controller's job ends by returning a response. Here is the full menu and when to reach for each.

#### 4.6.1 HTML — `Response` and `render()`

The default. `render()` wraps a rendered Twig template in a 200 `Response`. For raw HTML or text without a template, return `new Response($content, $status, $headers)`.

#### 4.6.2 Redirects — `RedirectResponse`

```php
// Redirect to a route (preferred: survives URL changes)
return $this->redirectToRoute('app_invoice_index');

// Route + parameters, a permanent 301, or a fragment anchor
return $this->redirectToRoute('app_invoice_show', ['invoice' => 42]);
return $this->redirectToRoute('app_invoice_index', [], Response::HTTP_MOVED_PERMANENTLY);
return $this->redirectToRoute('app_invoice_show', ['_fragment' => 'totals']);

// Redirect to the current route — the classic Post/Redirect/Get move
return $this->redirectToRoute($request->attributes->get('_route'));

// Redirect to an absolute external URL
return $this->redirect('https://pay.example.com/charge');
```

> **Danger** `redirect()` performs **no validation** of its target. If the destination can come from user input, you are open to an open-redirect attack. Validate or use `redirectToRoute()` with a known route name instead (see the OWASP Unvalidated Redirects cheat sheet).

#### 4.6.3 JSON — `json()` and `createJsonResponse()`

For APIs (fully developed in Part V), `json()` encodes the data, sets the `Content-Type` to `application/json`, and returns a `Response`:

```php
return $this->json(['invoices' => $dto], 200);

// Control encoding and headers
return $this->json($data, 201, ['Cache-Control' => 'no-store'], [
    'json_encode_flags' => JSON_THROW_ON_ERROR,
    'decode_urls'       => true,
]);

// Lower-level variant when you want the raw pieces
return $this->createJsonResponse(['error' => 'Not found'], 404);
```

#### 4.6.4 Files — `BinaryFileResponse`

Serve a file from disk. `binaryFile()` forces a **download** (attachment); `file()` serves **inline** (e.g. an image or PDF rendered in the browser):

```php
use App\Service\InvoicePdfGenerator;

public function __construct(private InvoicePdfGenerator $pdf) {}

#[Route('/invoices/{invoice}/pdf', name: 'app_invoice_pdf')]
public function download(int $invoice): Response
{
    $path = $this->pdf->generate($invoice);   // e.g. /tmp/invoice-42.pdf

    return $this->binaryFile($path, 'invoice-42.pdf');
}
```

#### 4.6.5 Streams — `StreamedResponse` and `stream()`

For output that is too large to hold in memory — a CSV export of thousands of invoices, a log dump, a generated report — do **not** build the whole body as a string. Stream it in chunks:

```php
#[Route('/invoices/export.csv', name: 'app_invoice_export')]
public function export(): Response
{
    return $this->stream(function (): void {
        echo "id,total,status\n";

        // fetch in batches so memory stays flat
        foreach ($this->invoices->iterableUnpaid() as $invoice) {
            echo sprintf("%d,%.2f,%s\n", $invoice->getId(), $invoice->getTotal(), $invoice->getStatus());
        }
    }, 200, [
        'Content-Type'        => 'text/csv; charset=UTF-8',
        'Content-Disposition' => 'attachment; filename="invoices.csv"',
    ]);
}
```

The callback is called after headers are sent, so it must not throw in a way that expects to modify the response — and it must not start a transaction or do anything that needs to roll back. Treat it as "the response is already on its way."

#### 4.6.6 Errors — throw, do not return

For "not found" and other HTTP error conditions, **throw an exception** rather than returning an error response. The `kernel.exception` pipeline (and `ErrorListener`) converts it into the right status and error page:

```php
use Symfony\Component\HttpKernel\Exception\AccessDeniedHttpException;
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;

// 404
$invoice = $this->invoices->find($id)
    ?? throw $this->createNotFoundException('Invoice not found.');

// 403
throw new AccessDeniedHttpException('You cannot view this invoice.');

// any 4xx/5xx with an explicit code
throw new NotFoundHttpException('Gone.', 410);
```

The mapping is automatic: an `HttpExceptionInterface` carries its status; anything else is a 500. This is why you never write `return new Response('Not found', 404);` in application code — you throw, and the framework renders the error page consistently (and you can restyle it once, in Chapter 23).

#### 4.6.7 The return-type cheat sheet

| You want to return | Use | Type |
|---|---|---|
| A Twig page | `$this->render('...')` | `Response` |
| Raw HTML/text | `new Response($content)` | `Response` |
| JSON | `$this->json($data)` | `Response` |
| A redirect to a route | `$this->redirectToRoute('...')` | `RedirectResponse` |
| A redirect to a URL | `$this->redirect('...')` | `RedirectResponse` |
| A file download | `$this->binaryFile($path, $name)` | `BinaryFileResponse` |
| An inline file | `$this->file($path)` | `BinaryFileResponse` |
| A large/streaming body | `$this->stream(fn() => ...)` | `StreamedResponse` |
| A 404 / 403 / error | `throw ...` | (exception) |

---

### 4.7 Hooking Into the Lifecycle

The pipeline is not just something to understand — it is an **extension point**. Cross-cutting concerns (things that touch many controllers but belong to none of them) become small, focused listeners. Two on-theme examples.

**Deriving the tenant on every request.** Our invoicing app is multi-tenant; a natural scheme is to resolve the tenant from the subdomain (`acme.app.example.com` → tenant `acme`) as early as possible, and stash it on the request for downstream code:

```php
// src/EventSubscriber/TenantResolutionSubscriber.php
namespace App\EventSubscriber;

use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpKernel\Event\RequestEvent;
use Symfony\Component\HttpKernel\KernelEvents;

class TenantResolutionSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [
            // run early: a higher number than the router
            KernelEvents::REQUEST => ['onKernelRequest', 64],
        ];
    }

    public function onKernelRequest(RequestEvent $event): void
    {
        if (!$event->isMainRequest()) {
            return;
        }

        $host   = $event->getRequest()->getHost();        // e.g. "acme.app.example.com"
        $tenant = explode('.', $host)[0] ?? 'default';

        $event->getRequest()->attributes->set('_tenant', $tenant);
    }
}
```

Because `EventSubscriberInterface` is autoconfigured (Chapter 7 in detail), this class is discovered and registered automatically — no YAML.

**Adding security headers to every response.** A textbook `kernel.response` listener:

```php
// src/EventSubscriber/SecurityHeadersSubscriber.php
namespace App\EventSubscriber;

use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\ResponseEvent;
use Symfony\Component\HttpKernel\KernelEvents;

class SecurityHeadersSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [KernelEvents::RESPONSE => 'onKernelResponse'];
    }

    public function onKernelResponse(ResponseEvent $event): void
    {
        if (!$event->isMainRequest()) {
            return;
        }

        $headers = $event->getResponse()->headers;
        $headers->set('X-Content-Type-Options', 'nosniff');
        $headers->set('Referrer-Policy', 'strict-origin-when-cross-origin');
    }
}
```

The other hooks, in one line each:

- `kernel.controller` — read a custom attribute off the controller, or swap the controller.
- `kernel.controller.arguments` — rewrite the arguments the controller will receive.
- `kernel.view` — convert a non-`Response` return into a `Response` (only fires when needed):

```php
public function onKernelView(ViewEvent $event): void
{
    $value = $event->getControllerResult();
    if ($value instanceof Response || !is_string($value)) {
        return;
    }
    $event->setResponse(new Response($value));
}
```

- `kernel.exception` — sanitize or replace a throwable before it becomes a response:

```php
public function onKernelException(ExceptionEvent $event): void
{
    $original = $event->getThrowable();
    // never leak internals
    $event->setThrowable(new \RuntimeException('Unexpected error', 0, $original));
}
```

- `kernel.terminate` — do slow, non-critical work *after* the client already has its response.

> **Note** Event listeners and subscribers, priorities, and the full `EventDispatcher` model are the subject of Chapter 7. The examples above are just enough to make the pipeline *feel* real; go back and read Chapter 7 to use them with confidence.

---

### 4.8 Debugging Controllers and the Pipeline

You will debug controllers constantly. These are the tools, in the order you will reach for them.

**"Is this route even registered / what does it map to?"**

```bash
$ php bin/console debug:router                 # list all routes
$ php bin/console debug:router app_invoice_show  # inspect one route
```

`debug:router` shows the path, the required methods, the defaults, the requirements, and the target controller. If your URL 404s, this is the first place to look — usually a method mismatch (`GET` vs `POST`) or a requirement the value does not satisfy.

**"What listeners run for this event, and in what order?"**

```bash
$ php bin/console debug:event-dispatcher kernel.request
$ php bin/console debug:event-dispatcher          # all events
```

**"What does the container see?"** When a constructor-injected dependency fails, you are in Chapter 5 territory, but the first signals are here:

```bash
$ php bin/console debug:container App\Controller\InvoiceController
$ php bin/console debug:autowiring InvoiceRepository
```

**The profiler.** In the `dev` environment, every response carries a profiler link. The **Request** panel shows the route, the controller, the request/response attributes, and the events fired; the **Time** panel shows where the milliseconds went. When a page is slow or a parameter is wrong, the profiler shows you the actual resolved controller and arguments — no guessing.

**Common pitfalls, pre-empted:**

| Symptom | Usual cause |
|---|---|
| Constructor dependency is `null` / "service not found" | The controller is not a service, so constructor injection never ran. Extend `AbstractController`, or use `#[AsController]` / `controller.service_arguments`. |
| Argument "has no value" / type error | A route parameter name does not match the argument name, or the requirement rejects the value. |
| Listener runs twice | You did not guard with `isMainRequest()` and a sub-request triggered it. |
| `kernel.view` never fires | You always return a `Response`, so there is nothing to convert — that's correct, not a bug. |
| Controller code not picking up edits | You are in `prod` (or a stale container). Re-run and clear the cache if needed. |

---

### 4.9 Putting It Together

Here is a compact, realistic controller that combines everything in this chapter: a service controller with constructor injection, a route parameter, an early 404, a query-mapped filter, and a choice between HTML and CSV depending on a request hint. (We will build a much richer version of this in the running project.)

```php
// src/Controller/InvoiceController.php
namespace App\Controller;

use App\Repository\InvoiceRepository;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class InvoiceController extends AbstractController
{
    public function __construct(
        private readonly InvoiceRepository $invoices,
    ) {
    }

    #[Route('/invoices', name: 'app_invoice_index', methods: ['GET'])]
    public function index(Request $request): Response
    {
        $status = $request->query->getAlnum('status', 'unpaid');
        $invoices = $this->invoices->findBy(['status' => $status]);

        // Same route, two representations: ?format=csv streams, otherwise render HTML.
        if ($request->query->getAlnum('format') === 'csv') {
            return $this->stream(function () use ($invoices): void {
                echo "id,total,status\n";
                foreach ($invoices as $invoice) {
                    echo sprintf("%d,%.2f,%s\n",
                        $invoice->getId(),
                        $invoice->getTotal(),
                        $invoice->getStatus(),
                    );
                }
            }, 200, [
                'Content-Type'        => 'text/csv; charset=UTF-8',
                'Content-Disposition' => 'attachment; filename="invoices.csv"',
            ]);
        }

        return $this->render('invoice/index.html.twig', ['invoices' => $invoices]);
    }

    #[Route(
        '/invoices/{invoice}',
        name: 'app_invoice_show',
        requirements: ['invoice' => '\d+'],
        methods: ['GET'],
    )]
    public function show(int $invoice): Response
    {
        $invoiceEntity = $this->invoices->find($invoice)
            ?? throw $this->createNotFoundException('Invoice not found.');

        return $this->render('invoice/show.html.twig', ['invoice' => $invoiceEntity]);
    }
}
```

Read it once with the pipeline in mind: `kernel.request` matches the route → the resolver fetches this (shared) service → `kernel.controller.arguments` binds `int $invoice` from the URL and injects `$request` → the method runs and returns a `Response` (or a `StreamedResponse`) → `kernel.response` lets your subscribers add headers → it's sent → `kernel.terminate` runs. One request, fully accounted for.

---

### 4.10 Exercises

1. **Warm-up.** Create a controller with two routes: `GET /healthz` returning the JSON `{"status":"ok"}`, and `GET /time` returning the current UTC time as plain text. Use `AbstractController` for one and a bare `new Response(...)` for the other. Which one is easier to extend later, and why?

2. **Controller types.** Convert the `LuckyNumberController` from §4.1 into an **invokable** controller (ADR). Then add a second action, `GET /lucky/even`, that only returns even numbers. Why does the invokable shape stop being a good fit here? What shape should you switch to?

3. **Shared vs. prototype.** Write a controller that stores the current `Request` in a private property (a deliberately bad practice, for demonstration). Run two requests in a row and observe the consequence of the default shared service. Then fix it two ways — (a) by removing the property, and (b) by setting `shared: false` — and explain which fix you would actually ship and why.

4. **Argument injection.** Add a route `GET /invoices/search` that accepts `?status=&from=&to=`. Use `#[MapQueryString]` to bind them to a small `InvoiceSearch` DTO with validation constraints. What does Symfony return when a required parameter is missing, and how do you change the status code?

5. **Return types.** Extend `InvoiceController::index` so that `?format=json` returns a JSON array (using `json()`), `?format=csv` streams (using `stream()`), and the default renders HTML. Make sure the CSV branch sets correct `Content-Type` and `Content-Disposition` headers.

6. **The pipeline.** Write a `kernel.response` subscriber that adds an `X-Response-Time` header measuring how long handling took (hint: record a `microtime(true)` on `kernel.request`, read it on `kernel.response`). Register it, hit a few routes, and confirm the header appears — and confirm it does **not** appear on a sub-request (add an `isMainRequest()` guard and observe).

7. **Short-circuit.** Write a `kernel.request` listener that returns a `402 Payment Required` response for any URL under `/premium/*` when the request has no `X-Premium-Token` header. Verify that your premium controller is *never* invoked (add a `dump()` or log line to prove it).

8. **Debugging.** Introduce a deliberate bug: a route parameter named `{invoiceId}` but a controller argument named `$invoice`. Use `debug:router` and the profiler to diagnose it, then fix it. Write one sentence describing the *signal* that told you it was a name mismatch rather than a missing route.

---

### 4.11 Key Takeaways

- A controller is any callable that maps a `Request` to a `Response`; a method on a class is the default, `__invoke()` is the ADR one-shot, and making it a **service** unlocks constructor injection.
- A service controller is a **shared singleton** by default; use `shared: false` for a **value-object** (per-request) instance, but the better habit is to keep controllers **stateless** and pass everything through arguments.
- Dependencies arrive two ways: **constructor injection** (for the object's lifetime, requires a service) and **argument injection** (per-call, works everywhere). `#[Autowire]` narrows the choice.
- The kernel is **event-driven**: `kernel.request` → resolve controller → `kernel.controller` → `kernel.controller.arguments` → call → (optionally `kernel.view`) → `kernel.response` → `kernel.finish_request` → send → `kernel.terminate`, with `kernel.exception` as the error branch.
- `kernel.view` only fires when a controller does **not** return a `Response`; everything else assumes you do.
- Return what fits: `render()` for HTML, `json()` for APIs, `redirect*()` for moves, `binaryFile()`/`file()` for files, `stream()` for large bodies, and **throw** for error status codes.
- Guard listeners with `isMainRequest()`; debug with `debug:router`, `debug:event-dispatcher`, `debug:container`, and the profiler.

You now own the middle of the stack — the part that turns a URL into a response. The next chapter turns the camera to the machinery that decides *which* service a controller can even talk to: the **Dependency Injection** container.

---

## Part II — Core Architecture

### Chapter 5. Dependency Injection

> *"In the Symfony world, almost every object you use is a service wired together by a container. Understanding how that container works — and how to shape it — is the single most useful thing you can learn about the framework's internals."*

---

#### 5.1 Why Dependency Injection Matters

If you have written any non-trivial PHP application, you have probably written code like this:

```php
class InvoiceService
{
    public function __construct(
        private EntityManagerInterface $em,
        private MailerInterface $mailer,
        private LoggerInterface $logger,
    ) {}

    public function sendInvoice(Invoice $invoice): void
    {
        // ...
    }
}
```

The question that immediately arises is: *who creates the `EntityManagerInterface`, the `MailerInterface`, and the `LoggerInterface`? And who passes them in?* The answer in Symfony is the **service container** — a carefully compiled object factory that resolves, wires, and caches every collaborator your application needs.

Dependency injection (DI) is the mechanism; the container is the *implementation*. In this chapter you will learn:

- How services are defined and resolved
- How autowiring and autoconfiguration eliminate most configuration
- How to use advanced patterns: decorators, aliases, service locators, and tags
- How the container compiles, what "private" really means, and how to debug problems

By the end you will be able to design the object graph of a Symfony application with confidence, and you will know exactly what happens between the moment a request hits `public/index.php` and the moment your controller is instantiated.

---

#### 5.2 The Service Container: A First Look

The container is a singleton object (available as `$container` in the compiled runtime) that can do two things:

1. **Create** a service from its definition (constructor arguments, factory, decorators, etc.)
2. **Return** a previously created instance (for shared, i.e. singleton, services)

Every service definition is a recipe: *given these arguments, call this factory or this constructor, and store the result under this identifier.* The identifier is the **service ID**, and for services that are autowired it is conventionally the fully-qualified class name (FQCN).

```text
┌─────────────────────────────────────────────────────────┐
│                    Service Container                     │
├─────────────────────────────────────────────────────────┤
│  ID                        │  Definition                │
├─────────────────────────────┼───────────────────────────┤
│  App\Service\InvoiceService │  class + autowired args   │
│  App\Repository\InvoiceRepo │  class + autowired args   │
│  Psr\Log\LoggerInterface    │  alias → monolog.logger   │
│  monolog.logger             │  factory → new Logger()   │
│  event_dispatcher           │  factory → new EventDisp()│
│  ...                       │  ...                      │
└─────────────────────────────┴───────────────────────────┘
```

You rarely interact with the container directly. Instead, you rely on **autowiring** (the container inspects your constructor and resolves each parameter) and **autoconfiguration** (the container assigns common tags and decorators based on the interfaces your service implements). The container is the engine; autowiring and autoconfiguration are the user experience on top of it.

---

#### 5.3 Defining Services

##### 5.3.1 The Default: Resource-Based Autowiring

In a modern Symfony application, the vast majority of your services require **zero explicit configuration**. This is because the `framework` bundle's `services` section includes a default resource registration:

```yaml
# config/services.yaml
services:
    # Default configuration for services in *this* file
    _defaults:
        autowire: true          # Automatically inject dependencies
        autoconfigure: true     # Automatically tag based on interfaces

    # Register every class in the App\ namespace
    App\:
        resource: '../src/'
        exclude:
            - '../src/DependencyInjection/'
            - '../src/Entity/'
            - '../src/Kernel.php'
            - '../src/Attributes/'
```

What this does:

- Scans every `.php` file under `src/`
- Registers each class as a service with its FQCN as the ID
- Enables autowiring and autoconfiguration for all of them
- Excludes directories that are not services (entities, attributes, etc.)

So if you write:

```php
// src/Service/InvoiceService.php
namespace App\Service;

use Doctrine\ORM\EntityManagerInterface;
use Psr\Log\LoggerInterface;

class InvoiceService
{
    public function __construct(
        private EntityManagerInterface $em,
        private LoggerInterface $logger,
    ) {}
}
```

…you have already "defined" the service. There is nothing else to do. The container will create it the first time something asks for it.

> **Convention note:** This book uses attribute-based configuration exclusively for routes, validation, and entity mapping. For service definitions, YAML (`config/services.yaml`) remains the idiomatic format in Symfony 7/8 because the DI configuration is inherently *infrastructure-level* and benefits from being centralized. You will see YAML for service configuration throughout this chapter, but the same definitions are expressible in XML if you prefer.

##### 5.3.2 Explicit Service Definitions

Sometimes a service needs a specific argument that cannot be autowired (a string, an integer, a callable, a specific instance among many implementations). You define it explicitly:

```yaml
# config/services.yaml
services:
    App\Service\InvoiceService:
        arguments:
            $logger: '@monolog.logger.invoice'   # bind a specific logger
```

Or, when the service is not in your `src/` namespace (a vendor class you want to reconfigure):

```yaml
services:
    App\Service\PaymentProcessor:
        class: App\Service\PaymentProcessor
        arguments:
            $apiKey: '%env(PAYMENT_API_KEY)%'
            $timeout: 30
```

> **Tip:** When you override arguments for an autowired service, you only need to specify the ones that differ. The rest are still resolved automatically.

##### 5.3.3 The `_defaults` Section

The `_defaults` key applies configuration to every service defined *in that file*:

```yaml
services:
    _defaults:
        autowire: true
        autoconfigure: true
        public: false        # all services are private by default

    App\:
        resource: '../src/'
        exclude:
            - '../src/Entity/'
```

You can also set defaults for a subset:

```yaml
services:
    App\Service\:
        resource: '../src/Service/'
        public: false
        autowire: true
        autoconfigure: true
```

##### 5.3.4 Service IDs and Naming

The service ID is the string you use to reference a service. For autowired services the ID is the FQCN:

```text
App\Service\InvoiceService
```

You can also use short IDs for frequently referenced services (common in Symfony's own bundles):

```yaml
services:
    my_invoice_service:
        class: App\Service\InvoiceService
```

The convention in modern Symfony is to use FQCNs for your own services and short IDs only when the framework or a bundle establishes them (e.g., `event_dispatcher`, `http_kernel`, `router`).

---

#### 5.4 Autowiring

##### 5.4.1 How It Works

Autowiring is a *type-driven* resolution strategy. When the container instantiates a service, it inspects the constructor (or factory method) and, for each parameter, looks for a service whose **type matches the parameter's type hint**:

```php
class InvoiceService
{
    public function __construct(
        private EntityManagerInterface $em,    // → finds the EM service
        private LoggerInterface $logger,       // → finds the logger service
        private InvoiceRepository $repo,       // → finds the repository
    ) {}
}
```

The resolution order is:

1. **Exact class match** — is there a service with ID `Doctrine\ORM\EntityManagerInterface`?
2. **Alias match** — is `Psr\Log\LoggerInterface` aliased to another service? (Yes: `monolog.logger`.)
3. **Implementation match** — is there exactly one service that implements `LoggerInterface`?
4. **Failure** — if multiple services match or none match, autowiring fails.

##### 5.4.2 When Autowiring Fails

Autowiring fails in predictable situations:

| Situation | Example | Fix |
|-----------|---------|-----|
| No service for the type | `private string $apiKey` | Pass it explicitly: `arguments: [$apiKey]` or use `bind` |
| Multiple implementations | `private LoggerInterface $logger` when 3 loggers exist | Use an alias or bind a specific one |
| Unresolvable interface | A vendor interface with no registered implementation | Register the implementation or use a service locator |

**Scalar and array parameters** cannot be autowired (there is no type to match). You must provide them:

```yaml
services:
    App\Service\PaymentProcessor:
        arguments:
            $apiKey: '%env(PAYMENT_API_KEY)%'
            $allowedRegions: ['eu', 'us', 'uk']
```

##### 5.4.3 The `bind` Directive

The `bind` configuration lets you inject values into *every* service that has a parameter of that type, without repeating the argument:

```yaml
services:
    _defaults:
        bind:
            string $env: '%kernel.environment%'
            int $cacheTtl: 3600
            App\Entity\Tenant: '@app.current_tenant'  # inject the current tenant
```

This is particularly powerful in the running project. In a multi-tenant SaaS app, you often need the "current tenant" available everywhere. Instead of injecting `TenantRepository` and querying on every call, you bind the resolved tenant:

```yaml
# config/services.yaml
services:
    _defaults:
        bind:
            ?App\Entity\Tenant: '@app.tenant_resolver'
```

Now any service with `public function __construct(protected Tenant $tenant)` gets the current tenant automatically.

> **Caution:** `bind` is powerful but invisible. A new developer reading a constructor sees a `Tenant` parameter and may not realize it is magically injected. Use it judiciously and document it.

##### 5.4.4 Autowiring and `final`

Autowiring works with `final` classes, interfaces, and abstract types. There is no difference in resolution. However, `final` classes have one practical advantage: they can never be accidentally replaced by a mock in a test that uses type-based resolution, making them slightly more predictable.

---

#### 5.5 Autoconfiguration

##### 5.5.1 What It Does

Autoconfiguration is a *tagging* mechanism. When a service is marked `autoconfigure: true`, the container inspects the interfaces it implements (and, in Symfony 7+, the attributes on the class) and applies pre-defined configuration:

| Interface / Attribute | Effect |
|-----------------------|--------|
| `EventSubscriberInterface` | Tagged as `kernel.event_subscriber` |
| `MessageHandlerInterface` (or `#[AsMessageHandler]`) | Tagged as `messenger.message_handler` |
| `CacheItemPoolInterface` | Tagged as `cache.pool` |
| `RateLimiterFactory` implementations | Tagged as `rate_limiter.factory` |
| `KernelEvents` listeners (via `#[AsEventListener]`) | Tagged as `kernel.event_listener` |
| `ValidatorInterface` / `ConstraintValidatorInterface` | Tagged as `validator.constraint_validator` |
| `Twig\Extension\ExtensionInterface` | Tagged as `twig.extension` |
| `Twig\RuntimeLoader\RuntimeLoaderInterface` | Tagged as `twig.runtime` |

So this class:

```php
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
class SendInvoiceEmailHandler
{
    public function __invoke(SendInvoiceEmail $message): void
    {
        // ...
    }
}
```

…is automatically tagged as a messenger message handler. You did not write a single line of DI configuration.

##### 5.5.2 Attributes vs. Interfaces

Symfony 7+ strongly prefers **attributes** over marker interfaces for autoconfiguration. The attribute `#[AsMessageHandler]` is the modern replacement for implementing `MessageHandlerInterface`. Both work, but attributes are:

- More explicit (you can configure options inline)
- Not polluted with extra interface methods
- Discoverable by static analysis tools more easily

```php
#[AsMessageHandler]
class Handler { /* ... */ }

// Equivalent to:
#[AsMessageHandler(fromTransport: 'async', method: 'process')]
class Handler {
    public function process(SendInvoiceEmail $message): void { /* ... */ }
}
```

##### 5.5.3 Custom Autoconfiguration

You can register your own autoconfiguration rules in a bundle's `DependencyInjection` extension or via a compiler pass. For the running project, imagine a custom interface for tenant-scoped services:

```php
// src/DependencyInjection/TenantAwarePass.php
namespace App\DependencyInjection;

use Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Definition;

class TenantAwarePass implements CompilerPassInterface
{
    public function process(ContainerBuilder $container): void
    {
        $tenantResolverId = 'app.tenant_resolver';

        foreach ($container->getDefinitions() as $id => $definition) {
            if ($definition->isDeprecated()) {
                continue;
            }

            // Add tenant resolver as a last argument to all tenant-aware services
            if ($definition->hasTag('app.tenant_aware')) {
                $definition->addArgument(new Reference($tenantResolverId));
            }
        }
    }
}
```

This is an advanced topic covered in Chapter 8, but it is worth knowing that the mechanism exists.

---

#### 5.6 Constructor Injection: The Core Pattern

##### 5.6.1 The Rule

**All dependencies go through the constructor.** No setters, no service locators in business logic, no static `::getInstance()`.

```php
class InvoiceService
{
    public function __construct(
        private EntityManagerInterface $em,
        private MailerInterface $mailer,
        private LoggerInterface $logger,
        private InvoiceNumberGenerator $numberGenerator,
    ) {}

    public function createAndSend(CreateInvoiceDto $dto): Invoice
    {
        $invoice = new Invoice();
        $invoice->setNumber($this->numberGenerator->next());
        // ...
        $this->em->persist($invoice);
        $this->em->flush();

        $this->mailer->send($this->buildEmail($invoice));
        $this->logger->info('Invoice created', ['id' => $invoice->getId()]);

        return $invoice;
    }
}
```

Why constructor injection?

- **Immutability of dependencies** — the object is fully configured at construction time.
- **Testability** — you can pass mocks without any reflection or container access.
- **Explicitness** — the constructor signature *is* the dependency contract.
- **Fail-fast** — if a dependency is missing, the object cannot be constructed at all.

##### 5.6.2 Constructor Property Promotion (PHP 8+)

PHP 8.0's constructor property promotion makes the pattern extremely concise:

```php
class InvoiceService
{
    public function __construct(
        private EntityManagerInterface $em,
        private MailerInterface $mailer,
        private readonly InvoiceNumberGenerator $numberGenerator,
    ) {}
}
```

Use `readonly` for dependencies that are truly immutable (value objects, generators). Use `private` without `readonly` when you might need to reassign (rare for services, but possible for test doubles).

##### 5.6.3 Avoiding God Constructors

A constructor with 10+ parameters is a design smell. It usually means the class has too many responsibilities. Refactor by extracting cohesive groups:

```php
// Before: 8 parameters
class InvoiceService {
    public function __construct(
        private EntityManagerInterface $em,
        private MailerInterface $mailer,
        private LoggerInterface $logger,
        private InvoiceNumberGenerator $gen,
        private PdfGenerator $pdf,
        private CurrencyConverter $currency,
        private TaxCalculator $tax,
        private NotificationService $notify,
    ) {}
}

// After: extract a "value object" or "context"
class InvoiceContext {
    public function __construct(
        public readonly PdfGenerator $pdf,
        public readonly CurrencyConverter $currency,
        public readonly TaxCalculator $tax,
    ) {}
}

class InvoiceService {
    public function __construct(
        private EntityManagerInterface $em,
        private MailerInterface $mailer,
        private LoggerInterface $logger,
        private InvoiceContext $context,
    ) {}
}
```

The container will autowire `InvoiceContext` automatically (all its own dependencies are resolvable).

##### 5.6.4 Optional and Null-able Dependencies

If a dependency is genuinely optional (e.g., a "primary" service that may not exist in all environments), use a nullable type:

```php
public function __construct(
    private ?FeatureFlagService $featureFlags = null,
) {}
```

The container will inject `null` if no service of that type is registered. This is preferable to a service locator in this case because the optionality is visible in the type signature.

> **Anti-pattern:** Do not use `?Type` as an excuse to avoid proper abstraction. If a dependency is optional in *some* environments but required in others, that is a configuration problem, not a type problem.

---

#### 5.7 The Service Locator

##### 5.7.1 When (and When Not) to Use It

A **service locator** is a small, *selective* container that knows about a limited set of services. It exists to break hard dependencies — situations where a service needs *one of many* implementations chosen at runtime.

```php
// BAD: hard dependency on all payment processors
class CheckoutService {
    public function __construct(
        private StripeProcessor $stripe,
        private PayPalProcessor $paypal,
        private BitcoinProcessor $bitcoin,
    ) {}
}

// BETTER: a locator of payment processors
class CheckoutService {
    public function __construct(
        private PaymentProcessorLocator $processors,  // ServiceLocatorInterface
    ) {}

    public function process(string $method, array $data): PaymentResult
    {
        $processor = $this->processors->get($method);  // 'stripe', 'paypal', 'bitcoin'
        return $processor->charge($data);
    }
}
```

##### 5.7.2 Defining a Service Locator

You define a service locator as a service with the `service-locator` tag:

```yaml
# config/services.yaml
services:
    payment_processor_locator:
        class: Symfony\Component\DependencyInjection\ServiceLocator
        public: true
        factory: ['Symfony\Component\DependencyInjection\ServiceLocator', 'locate']
        arguments:
            -
                stripe: '@App\Service\Payment\StripeProcessor'
                paypal: '@App\Service\Payment\PayPalProcessor'
                bitcoin: '@App\Service\Payment\BitcoinProcessor'
```

Then reference it in your code:

```php
use Symfony\Contracts\Service\ServiceLocatorInterface;

class CheckoutService
{
    public function __construct(
        private ServiceLocatorInterface $paymentProcessors,  // autowired to payment_processor_locator
    ) {}
}
```

Wait — autowiring will not pick up `payment_processor_locator` for a `ServiceLocatorInterface` parameter because there may be multiple locators. You need a `bind` or an alias:

```yaml
services:
    _defaults:
        bind:
            ServiceLocatorInterface $paymentProcessors: '@payment_processor_locator'
```

##### 5.7.3 The `ServiceLocatorFactory` (Dynamic Locators)

For more dynamic scenarios (e.g., a plugin system where processors are registered at runtime), Symfony provides `ServiceLocatorFactory`:

```yaml
services:
    App\Service\Payment\PaymentProcessorFactory:
        class: Symfony\Component\DependencyInjection\ServiceLocator
        factory: ['@service_locator', 'getLocators']
        arguments:
            - [
                'App\Service\Payment\PaymentProcessorInterface',
                '@App\Service\Payment\PaymentProcessorInterface',
            ]
```

This creates a locator that contains *all services tagged* `App\Service\Payment\PaymentProcessorInterface`.

##### 5.7.4 Rules of Thumb

| Use a service locator when… | Do NOT use it when… |
|----------------------------|--------------------|
| You need to pick one of N implementations at runtime | You have a fixed, known set of dependencies |
| You are building a plugin/strategy system | You can express the choice with a single interface + binding |
| You are in a *framework* or *abstract layer* that serves many backends | You are in business logic (use constructor injection) |

In the running invoicing project, the only place a service locator is justified is the payment processor dispatch. Everything else uses constructor injection.

---

#### 5.8 Decorators

##### 5.8.1 The Concept

A **decorator** wraps an existing service with additional behavior. The original service is *replaced* in the container, but the decorator receives the original as a reference, allowing it to delegate and add logic around it.

```yaml
services:
    # The "real" service
    App\Service\Payment\StripeProcessor:
        ~

    # The decorator
    App\Service\Payment\LoggingPaymentDecorator:
        decorates: 'App\Service\Payment\StripeProcessor'
        arguments:
            $inner: '@App\Service\Payment\StripeProcessor.inner'
```

The key mechanics:

- `decorates: <original_id>` tells the container: "when someone asks for `StripeProcessor`, give them my decorator instead."
- The original service is still created, but under the ID `<original_id>.inner`.
- The decorator receives the original via the `.inner` reference.

##### 5.8.2 A Practical Example: Caching a Repository

In the running project, you might want to add a short-lived cache to the `InvoiceRepository` for read-heavy report queries:

```php
// src/Repository/CachingInvoiceRepository.php
namespace App\Repository;

use App\Repository\InvoiceRepository as InnerRepository;
use Doctrine\ORM\EntityManagerInterface;

class CachingInvoiceRepository extends InnerRepository
{
    private array $cache = [];

    public function __construct(
        private InnerRepository $inner,
        private EntityManagerInterface $em,
        private int $ttl = 60,
    ) {
        parent::__construct($em);
    }

    public function findByTenant(string $tenantId): array
    {
        $cacheKey = "invoices:{$tenantId}";

        if (isset($this->cache[$cacheKey]) && $this->cache[$cacheKey]['exp'] > time()) {
            return $this->cache[$cacheKey]['data'];
        }

        $data = $this->inner->findByTenant($tenantId);

        $this->cache[$cacheKey] = ['data' => $data, 'exp' => time() + $this->ttl];

        return $data;
    }

    public function __call(string $method, array $args): mixed
    {
        return $this->inner->$method(...$args);
    }
}
```

```yaml
services:
    App\Repository\InvoiceRepository:
        ~

    App\Repository\CachingInvoiceRepository:
        decorates: 'App\Repository\InvoiceRepository'
        arguments:
            $inner: '@App\Repository\InvoiceRepository.inner'
            $em: '@doctrine.orm.entity_manager'
            $ttl: 60
```

Any service that type-hints `InvoiceRepository` now transparently gets the caching version. The decorator is invisible to consumers.

##### 5.8.3 Multiple Decorators and `on-invalid`

You can chain multiple decorators on the same service. They are applied in the order they are defined:

```yaml
services:
    App\Service\Payment\StripeProcessor:
        ~

    App\Service\Payment\LoggingPaymentDecorator:
        decorates: 'App\Service\Payment\StripeProcessor'
        arguments:
            $inner: '@App\Service\Payment\StripeProcessor.inner'

    App\Service\Payment\MetricsPaymentDecorator:
        decorates: 'App\Service\Payment\StripeProcessor'
        arguments:
            $inner: '@App\Service\Payment\StripeProcessor.inner'
```

With multiple decorators, the `.inner` reference in the *last* decorator points to the *previous* decorator's output. The order of definition in YAML matters.

> **Caution:** Deeply nested decorators are hard to debug. If you find yourself with 3+ decorators on one service, consider whether the responsibilities should be split into separate services.

---

#### 5.9 Aliases

An **alias** is an alternative name for a service. The most common use is mapping an interface to its implementation:

```yaml
services:
    App\Service\Payment\StripeProcessor:
        ~

    # Now anything that type-hints PaymentProcessorInterface gets StripeProcessor
    App\Service\Payment\PaymentProcessorInterface:
        alias: 'App\Service\Payment\StripeProcessor'
```

When a consumer does:

```php
public function __construct(
    private PaymentProcessorInterface $processor,  // → resolves to StripeProcessor
) {}
```

…the container follows the alias and injects the concrete service.

##### 5.9.1 Aliases vs. Decorators

| | Alias | Decorator |
|---|---|---|
| What it does | Renames a service | Replaces a service with a wrapper |
| Original still available? | Yes (under its original ID) | Yes (under `.inner`) |
| Use case | Interface → implementation | Add behavior (logging, caching, retry) |
| Consumer awareness | None | None |

##### 5.9.2 Public Aliases

Aliases are private by default (same as services). If you need to access an aliased service by its alias in code (rare), mark it public:

```yaml
services:
    my_payment_processor:
        alias: 'App\Service\Payment\StripeProcessor'
        public: true
```

In practice, you almost never need this. Prefer type-hinting the interface.

---

#### 5.10 Tags

##### 5.10.1 What Tags Are

Tags are *metadata* attached to a service definition. They do not change the service's identity or instantiation — they mark the service for *consumption by a compiler pass* or a *consumer locator*.

```yaml
services:
    App\Service\Notification\EmailChannel:
        tags:
            - { name: 'app.notification_channel', channel: 'email', priority: 10 }
    App\Service\Notification\SmsChannel:
        tags:
            - { name: 'app.notification_channel', channel: 'sms', priority: 5 }
```

A compiler pass (or a tagged-iterator locator) collects all services with the tag `app.notification_channel` and can act on them.

##### 5.10.2 Tagged Iterators and Locators

Since Symfony 4.3, you can autowire a *collection* of tagged services:

```php
use Symfony\Component\DependencyInjection\Attribute\TaggedIterator;

class NotificationDispatcher
{
    public function __construct(
        #[TaggedIterator('app.notification_channel')]
        private iterable $channels,  // all services tagged 'app.notification_channel'
    ) {}

    public function dispatch(Notification $notification): void
    {
        foreach ($this->channels as $channel) {
            $channel->send($notification);
        }
    }
}
```

The `#[TaggedIterator]` attribute (or the `!tagged_iterator` YAML syntax) creates a lazy iterator that yields each tagged service, sorted by `priority` if present.

Similarly, `#[TaggedLocator]` gives you a `ServiceLocatorInterface` of all tagged services, keyed by a tag attribute (e.g., the `channel` key above):

```php
class NotificationDispatcher
{
    public function __construct(
        #[TaggedLocator(tag: 'app.notification_channel', indexAttribute: 'channel')]
        private ServiceLocatorInterface $channels,
    ) {}

    public function sendVia(string $channel, Notification $n): void
    {
        $this->channels->get($channel)->send($n);
    }
}
```

##### 5.10.3 Common Built-in Tags

| Tag | Purpose |
|-----|---------|
| `kernel.event_subscriber` | Register an event subscriber |
| `kernel.event_listener` | Register a single event listener |
| `kernel.exception_listener` | Register an exception listener |
| `messenger.message_handler` | Register a message handler |
| `validator.constraint_validator` | Register a constraint validator |
| `twig.extension` | Register a Twig extension |
| `cache.pool` | Register a cache pool |
| `console.command` | Register a console command (auto for `extends Command`) |
| `routing.loader` | Register a custom route loader |
| `monolog.logger` | Tag a service as a named logger |

Most of these are applied automatically by autoconfiguration when you use the corresponding attribute or interface. You only write them manually when you need to configure options that cannot be expressed via an attribute.

##### 5.10.4 Tags in YAML (When Needed)

```yaml
services:
    App\EventSubscriber\InvoiceSubscriber:
        tags:
            - { name: 'kernel.event_subscriber' }

    App\Service\Notification\EmailChannel:
        tags:
            - { name: 'app.notification_channel', channel: 'email', priority: 10 }
```

##### 5.10.5 Tags and the Running Project

In the multi-tenant invoicing app, tags shine in a few places:

- **Notification channels** (email, SMS, in-app) are registered with a custom tag and collected via `#[TaggedLocator]`.
- **Invoice number generators** (per-tenant sequence, global ULID) are tagged and selected at runtime.
- **Audit log strategies** (database, external API) are tagged and dispatched to all that apply.

---

#### 5.11 Public vs. Private Services

##### 5.11.1 The Visibility Model

By default, **all services are private**. This is a critical design decision:

- **Private services** are removed from the compiled container if nothing references them. They cannot be fetched by ID from the container at runtime.
- **Public services** are always available via `$container->get('service_id')`.

```yaml
services:
    App\Service\InvoiceService:
        public: false   # default — will be removed if unused

    App\Controller\InvoiceController:
        public: true    # controllers must be public (the router instantiates them by ID)
```

##### 5.11.2 Why Private by Default?

1. **Compilation optimization** — unused services are stripped, reducing container size.
2. **Encapsulation** — it forces you to depend on services through type-hinting (constructor injection) rather than pulling them from the container by string ID.
3. **Refactoring safety** — if a service is private and nothing uses it, the compiler will not include it, so you get a clean signal that it is dead code.

##### 5.11.3 When to Make a Service Public

| Case | Reason |
|------|--------|
| Controllers | The router instantiates them by class name; they must be reachable by ID |
| Console commands (if not auto-registered) | `Application::add()` needs to find them |
| Services accessed from outside the container (e.g., in a test booting the kernel) | `test.client->getContainer()->get(...)` requires public |
| Services exposed to a service locator that uses string keys | The locator resolves by ID at runtime |
| Framework "entry point" services | `http_kernel`, `router`, `debug.debug` |

> **Rule of thumb:** If you find yourself making a service public "because I need it in a test," you probably should be injecting it into the class under test instead.

##### 5.11.4 The `public` Keyword in Attributes (Symfony 7+)

For services registered via attributes (e.g., `#[AsCommand]`, `#[AsMessageHandler]`), the service is private by default. If you need it public, use:

```php
use Symfony\Component\DependencyInjection\Attribute\AsCommand;

#[AsCommand(name: 'app:invoice:generate', public: true)]
class GenerateInvoiceCommand extends Command { /* ... */ }
```

---

#### 5.12 Container Compilation

##### 5.12.1 What Happens at Build Time

When you run `bin/console cache:clear` (or the first request in production), the container goes through a **compilation** phase:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Container Compilation Pipeline                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. Load service definitions (YAML, XML, PHP, attributes)           │
│  2. Register compiler passes (from bundles + your own)              │
│  3. Resolve autowiring (type → service ID)                         │
│  4. Apply autoconfiguration (interfaces → tags)                     │
│  5. Apply decorators (original → .inner, wrap with decorator)       │
│  6. Resolve aliases                                                │
│  7. Inline single-use private services (performance)               │
│  8. Remove unused private services (optimization)                  │
│  9. Remove unused private aliases                                  │
│ 10. Generate the compiled PHP container class                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

The output is a **single PHP file** (in `var/cache/<env>/Container<hash>/`) that contains `get*()` methods for each public service and inlined instantiation code for private ones.

##### 5.12.2 Inlining

Private services that are used by exactly one other service are **inlined**: their constructor call is embedded directly in the consumer's factory method. This eliminates a method call and a hash lookup per request.

```php
// Pseudocode of what the compiled container looks like:

public function getInvoiceService(): InvoiceService
{
    // Inlined: the logger is used only here, so it's constructed inline
    $logger = new \Monolog\Logger('invoice', [
        new \Monolog\Handler\StreamHandler('/var/log/invoice.log', \Monolog\Logger::INFO),
    ]);

    return new InvoiceService(
        $this->get('doctrine.orm.entity_manager'),  // shared, not inlined
        $logger,                                      // inlined
        $this->getInvoiceNumberGeneratorService(),    // used elsewhere, not inlined
    );
}
```

##### 5.12.3 Environment-Specific Containers

Each environment (`dev`, `test`, `prod`) has its own compiled container. This means:

- Different services can be defined per environment (via `config/services_dev.yaml`, etc.)
- The dev container includes debug services (profiler, var-dumper); the prod container does not
- Clearing the cache (`cache:clear`) recompiles the container for the current environment

```yaml
# config/services_dev.yaml
services:
    _defaults:
        autowire: true
        autoconfigure: true

    # Extra services only in dev
    App\Service\Dev\MockPaymentProcessor:
        ~
```

---

#### 5.13 Debugging the Container

##### 5.13.1 `debug:container`

The primary debugging tool is the `debug:container` console command:

```bash
# Show all services (public only by default)
bin/console debug:container

# Show a specific service
bin/console debug:container App\Service\InvoiceService

# Show a service and its dependencies
bin/console debug:container App\Service\InvoiceService -v

# Search for services by name
bin/console debug:container "*Invoice*"

# Show a service's tags
bin/console debug:container --tag=app.notification_channel

# Show the parameters (not services)
bin/console debug:container --parameter
```

Example output:

```
$ bin/console debug:container App\Service\InvoiceService -v

 ID                        App\Service\InvoiceService
 --                        ------------------------------------
 Class                     App\Service\InvoiceService
 Public                    no
 Synthetic                 no
 Tags                      -
 Lazy                        no
 Factory                   -
 Shared                    yes
 Calls                     -
 Arguments
 [0]  =>  @doctrine.orm.entity_manager
 [1]  =>  @monolog.logger
 [2]  =>  @App\Service\InvoiceNumberGenerator
```

##### 5.13.2 Common Problems and How to Diagnose Them

| Symptom | Likely cause | Diagnosis |
|---------|-------------|-----------|
| `Cannot autowire service "App\Foo": argument "$bar" of method "__construct()" references interface "BarInterface" but no such service exists` | No service implements the interface | Register an implementation or create an alias |
| `Cannot autowire service "App\Foo": argument "$apiKey" of method "__construct()" is type-hinted with "string"` | Scalar parameter, not autowirable | Add it to `arguments:` or use `bind` |
| `Cannot autowire service "App\Foo": argument "$processor" of method "__construct()" references class "PaymentProcessorInterface" but it has multiple aliases` | Ambiguous interface | Use an alias or bind a specific implementation |
| Service works in dev but fails in prod | Service is private and only referenced by ID (not type) | Make it public or inject by type |
| `Class "App\Service\Foo" does not exist` | Class not in `src/` or excluded by `exclude` pattern | Check the `resource` and `exclude` in `services.yaml` |

##### 5.13.3 The Debug Container (Dev Only)

In the `dev` environment, the container is wrapped in a **debug container** that records every service instantiation. You can inspect it via the Web Profiler:

- **Container panel** — shows all instantiated services, their creation time, memory usage, and the call stack that triggered instantiation.
- **Service search** — filter by class, tag, or ID.

This is invaluable for understanding *why* a service was created (e.g., "I didn't expect `InvoiceService` to be instantiated on this request — what pulled it in?").

##### 5.13.4 `bin/console lint:container`

This command validates the container for common misconfigurations without booting the full application:

```bash
bin/console lint:container
```

It checks for:
- Services referencing non-existent IDs
- Cyclic dependencies
- Autowiring failures
- Invalid tag usage

Run it in CI to catch configuration errors early.

---

#### 5.14 Factory Methods and `shared`

##### 5.14.1 Factory Methods

Sometimes a service should not be constructed via `new ClassName(...)`. You can specify a factory:

```yaml
services:
    App\Service\Payment\PaymentGatewayFactory:
        class: App\Service\Payment\PaymentGateway
        factory: ['@App\Service\Payment\PaymentGatewayFactory', 'create']
        arguments:
            - '%env(PAYMENT_GATEWAY)%'
```

Or with a static factory:

```yaml
services:
    app.config_loader:
        factory: ['App\Config\Loader', 'create']
        arguments:
            - '%kernel.project_dir%/config'
```

In attributes (for your own services), you can use the `#[Factory]` attribute or define the factory in YAML.

##### 5.14.2 `shared: false` (Prototypal Services)

By default, services are **shared** (singleton within a request). If you need a *new instance* every time the service is requested, set `shared: false`:

```yaml
services:
    App\Form\Type\InvoiceForm:
        shared: false
```

> **Use sparingly.** Unshared services defeat the purpose of the container's caching. The most common legitimate use is for **form types** (though Symfony's form system handles this for you via the form factory) or for services that hold per-request mutable state that must not leak between calls.

In the running project, the `TenantResolver` might be `shared: false` if it reads the tenant from the request and the request changes during sub-requests. However, a better design is to make it stateless and pass the tenant explicitly.

---

#### 5.15 Designing the Object Graph: A Practical Walkthrough

Let's design the DI for a slice of the running invoicing project to tie everything together.

##### The Services

```
InvoiceController
  ├── InvoiceService
  │     ├── EntityManagerInterface (shared)
  │     ├── MailerInterface (shared)
  │     ├── LoggerInterface (shared)
  │     ├── InvoiceNumberGenerator
  │     └── PaymentProcessorInterface → alias → StripeProcessor
  ├── InvoiceRepository (decorated by CachingInvoiceRepository)
  └── TenantResolver (bound via _defaults.bind)

SendInvoiceEmailHandler (#[AsMessageHandler])
  ├── MailerInterface
  └── Twig\Environment

NotificationDispatcher
  └── #[TaggedLocator('app.notification_channel')] → [EmailChannel, SmsChannel]
```

##### The Configuration

```yaml
# config/services.yaml
services:
    _defaults:
        autowire: true
        autoconfigure: true
        public: false
        bind:
            ?App\Entity\Tenant: '@app.tenant_resolver'

    App\:
        resource: '../src/'
        exclude:
            - '../src/DependencyInjection/'
            - '../src/Entity/'
            - '../src/Kernel.php'
            - '../src/Attributes/'

    # Payment processor alias
    App\Service\Payment\PaymentProcessorInterface:
        alias: 'App\Service\Payment\StripeProcessor'

    # Caching decorator for the repository
    App\Repository\CachingInvoiceRepository:
        decorates: 'App\Repository\InvoiceRepository'
        arguments:
            $inner: '@App\Repository\InvoiceRepository.inner'
            $em: '@doctrine.orm.entity_manager'
            $ttl: 60

    # Notification channels
    App\Service\Notification\EmailChannel:
        tags:
            - { name: 'app.notification_channel', channel: 'email' }

    App\Service\Notification\SmsChannel:
        tags:
            - { name: 'app.notification_channel', channel: 'sms' }
```

Notice: no explicit definition is needed for `InvoiceService`, `InvoiceController`, `SendInvoiceEmailHandler`, or `NotificationDispatcher`. Autowiring, autoconfiguration, and `bind` handle them all.

##### Verifying

```bash
$ bin/console debug:container App\Service\InvoiceService -v
$ bin/console debug:container --tag=app.notification_channel
$ bin/console lint:container
```

---

#### 5.16 Best Practices Summary

1. **Always use constructor injection.** No setters, no service locators in business logic.
2. **Let autowiring do the work.** Only write explicit `arguments` when a parameter cannot be type-resolved.
3. **Use `bind` for cross-cutting values** (current tenant, environment, feature flags) — but sparingly.
4. **Prefer attributes over interfaces** for autoconfiguration in Symfony 7+.
5. **Keep services private** unless there is a concrete reason to expose them.
6. **Use decorators for cross-cutting behavior** (logging, caching, retry) rather than mixing concerns.
7. **Use service locators only for runtime polymorphic dispatch** (N-choose-1 patterns).
8. **Use `#[TaggedIterator]` / `#[TaggedLocator]`** for plugin-style collections.
9. **Run `lint:container` in CI** to catch configuration drift.
10. **Use the Web Profiler's Container panel** in development to understand instantiation order and detect surprises.

---

#### 5.17 Exercises

##### Exercise 1: Autowiring a New Service

1. Create a new service `App\Service\InvoicePdfGenerator` that depends on `Twig\Environment` and `Psr\Log\LoggerInterface`.
2. Verify that it is autowired with zero YAML configuration: run `bin/console debug:container App\Service\InvoicePdfGenerator`.
3. Add a `string $pdfFont` parameter. Observe the autowiring failure. Fix it using the `arguments` key.
4. Add an `int $maxPageSize` parameter. Fix it using `bind` in `_defaults`.

##### Exercise 2: Decorator Chain

1. Create `App\Service\Payment\RetryPaymentDecorator` that wraps `PaymentProcessorInterface` and retries failed charges up to 3 times with exponential backoff.
2. Configure it as a decorator of `App\Service\Payment\StripeProcessor`.
3. Write a unit test that mocks the inner processor to return a failure twice, then a success, and verify the decorator retries correctly.
4. Add a second decorator: `MetricsPaymentDecorator` that records timing. Verify the order of decoration and that both are applied.

##### Exercise 3: Tagged Locator for Notification Channels

1. Define a `NotificationChannelInterface` with a `send(Notification $n): void` method.
2. Create `EmailChannel` and `SmsChannel` implementations.
3. Tag both with `app.notification_channel` and a `channel` attribute.
4. Create a `NotificationDispatcher` that uses `#[TaggedLocator(tag: 'app.notification_channel', indexAttribute: 'channel')]`.
5. Write a functional test that dispatches a notification and verifies both channels are called.

##### Exercise 4: Service Locator for Strategy Dispatch

1. Create an `ExportFormatInterface` with a `render(Invoice $invoice): string` method.
2. Create `CsvExport` and `PdfExport` implementations.
3. Build a service locator containing both, keyed by format name.
4. Create an `InvoiceExporter` service that accepts the locator and a `string $format` parameter.
5. Verify that requesting an unknown format throws a meaningful `ServiceNotFoundException`.

##### Exercise 5: Debug a Broken Container

1. In a scratch Symfony project, define a service with a cyclic dependency (A depends on B, B depends on A).
2. Run `bin/console debug:container` and observe the error.
3. Resolve the cycle by extracting the shared dependency into a third service.
4. Repeat with an ambiguous autowiring failure (two services implement the same interface). Fix it with an alias.
5. Document each error message and the exact fix in a Markdown file.

##### Challenge: Refactor a "God Service"

Take the `InvoiceService` from the running project (at this point in the book, it may have accumulated several dependencies). Refactor it into:

- `InvoiceService` — orchestration (create, validate, dispatch)
- `InvoiceCalculator` — total, tax, discount logic
- `InvoiceNotifier` — email + notification dispatch

Verify that:
- All three are autowired without YAML changes (beyond the initial class registration).
- The public API of `InvoiceService` is unchanged (controllers still call the same methods).
- `bin/console lint:container` passes.
- A `debug:container -v` dump shows the new dependency graph clearly.

---

#### Chapter Summary

| Concept | Key Takeaway |
|---------|-------------|
| Service definitions | Most services need zero configuration thanks to resource-based autowiring |
| Autowiring | Type-hint-driven resolution; fails on scalars and ambiguous interfaces |
| Autoconfiguration | Interfaces and attributes → tags; eliminates boilerplate |
| Constructor injection | The default and preferred pattern; use `readonly` for immutable deps |
| Service locator | Only for runtime N-choose-1 dispatch; never in business logic |
| Decorators | Wrap services transparently; use for cross-cutting concerns |
| Aliases | Map interfaces to implementations; invisible to consumers |
| Tags | Metadata for compiler passes and tagged iterators/locators |
| Private by default | Services are stripped if unused; public only when necessary |
| Compilation | The container is a compiled PHP file; inlining optimizes hot paths |
| Debugging | `debug:container`, `lint:container`, and the Web Profiler are your friends |

The container is not a black box. It is a predictable, inspectable, and (when used well) invisible layer that lets you focus on your domain logic. In the next chapter, we turn to the **Configuration System** — the parameters, environment variables, and bundle configuration trees that feed the container's definitions.

### Chapter 6. The Configuration System

> *One codebase ships to a laptop, a CI runner, a staging box, and a fleet of production servers. The code stays the same; the configuration is what makes each deployment behave correctly. This chapter is about the machinery that makes that possible: where values come from, how they are typed and validated, and how they end up wired into the container you built in Chapter 5.*

By the end of Chapter 5 you can define services, autowire them, and tag them. But every real service needs *inputs* — a database URL, an API key, a template path, a feature flag. Chapter 6 explains the four layers Symfony provides for supplying those inputs, in the order you will reach for them:

1. **Environment variables** (the `.env` files) — for values that differ by *where* you deploy.
2. **Parameters** — named values you reuse across your own configuration.
3. **Bundle configuration** — the semantic, validated options each bundle exposes.
4. **Per-environment overrides** — small deltas applied only in `dev`, `prod`, `test`, or a custom environment.

We finish by walking the `config/` directory end to end, and by arming you with the console commands that let you *see* the configuration Symfony actually built.

A word on scope: Part II is about understanding the framework, so the examples here are small and self-contained. In Part III we will build all of this into the running project (*InvoiceHub*), where you'll configure the Doctrine connection, the mailer, security, and more.

---

#### 6.1 The big picture: two clocks

Before the details, one idea is worth internalizing because it explains most of what this chapter does. **Symfony resolves configuration on two different clocks:**

- **Compile time** — When you clear the cache (or on the first request after a configuration change), Symfony *compiles* all your configuration into a PHP service container and caches it on disk. Bundle options, parameters, service wiring, and route metadata are all frozen here. This is why a Symfony app is fast: the container is a pre-computed object graph, not something parsed per request.
- **Runtime** — A few values are deliberately *not* frozen. Environment variables referenced with the `%env(VAR)%` syntax are read once per request, not at compile time.

That single distinction answers the question every production developer eventually asks: *"How do I change the database password in production without redeploying?"* You put the password in an environment variable, reference it with `%env(DATABASE_URL)%`, and then changing the variable in your orchestrator takes effect on the very next request — no rebuild, no restart of the code.

```text
Real OS environment (PaaS / container orchestrator)      ← highest; never overridden
        │
.env                                        ┐
.env.local                                  │ loaded in this order; later files
.env.$APP_ENV                               │ win — but can never beat the real env
.env.$APP_ENV.local                         ┘
        │
        ▼
   %env(VAR)%  ────────────── resolved at RUNTIME (once per request)
        │
        ▼
   Parameters + Bundle configuration ── compiled into the container at BUILD time
```

Hold that picture in your head. Every section below is an elaboration of it.

---

#### 6.2 The `.env` file hierarchy, in depth

The front matter introduced the `.env` files; here we go deeper, because a surprising number of production incidents trace back to a misunderstanding of this hierarchy.

Symfony loads environment variables with the **Dotenv** component, using `Dotenv::loadEnv()`. Starting from the base `.env` file, it loads a chain of files where **later files override earlier ones**:

```text
1. .env                  committed defaults, shared by the whole team
2. .env.local            your personal overrides (git-ignored)
3. .env.$APP_ENV         e.g. .env.test, committed (per-environment defaults)
4. .env.$APP_ENV.local   e.g. .env.dev.local (git-ignored)
```

There is, however, a rule that outranks the entire chain, and it is the one that matters most in production:

> **Caution** — **The real environment always wins.** `loadEnv()` is called with `overrideExistingVars = false`, which means a variable that is *already* set in the actual OS environment (by your PaaS, your `docker run -e`, your CI secrets) is **never** overridden by any `.env` file. The `.env` files only *fill in gaps*. This is exactly what you want: your committed `.env` holds harmless defaults for development, while real credentials come from the deployment environment and cannot be clobbered by an accidental commit.

The variable that drives the whole mechanism is `APP_ENV` itself. When `loadEnv()` runs, it reads `APP_ENV` (falling back to `prod` if absent) and uses it to decide *which* files in the chain to load. So setting `APP_ENV=test` causes `loadEnv()` to load `.env` → `.env.local` → `.env.test` → `.env.test.local`.

##### The three variables that drive everything

Three variables are special and worth memorizing:

| Variable | Role |
|---|---|
| `APP_ENV` | Selects the *configuration environment*: which `config/packages/{env}/` files load, which `.env` chain applies, and how errors are rendered. One of `dev`, `prod`, `test` — or any custom name you create. |
| `APP_DEBUG` | Selects *debug mode*: verbose exception pages, the Web Profiler (Chapter 23), and per-request logging in `dev`. Kept as a separate variable from `APP_ENV` on purpose (see §6.4). |
| `APP_SECRET` | A long random string used to sign cookies and CSRF tokens. Must be unique per deployment. The Symfony CLI generates one for you at deploy time. |

You can override `APP_ENV` for a single command without editing any file — just prefix the command:

```bash
# Use the environment from .env (dev, in a fresh project)
$ php bin/console cache:clear

# Force this one command to run as if it were production
$ APP_ENV=prod php bin/console cache:clear
```

This is invaluable when you want to *pre-build* the production container locally before you deploy — a trick you will use again in Chapter 24.

##### Referencing environment variables in configuration

The bridge between the `.env` world and the configuration world is the `%env(VAR)%` syntax. Anywhere a configuration value appears — a bundle option, a service argument, a parameter — you can write:

```yaml
# config/packages/doctrine.yaml
doctrine:
    dbal:
        # Resolved at runtime, not compile time.
        url: '%env(DATABASE_URL)%'
```

And in `services.yaml`, environment variables work in arguments too:

```yaml
# config/services.yaml
services:
    app.stripe_client:
        class: App\StripeClient
        arguments:
            $apiKey: '%env(STRIPE_API_KEY)%'
```

You *can* read these variables directly from `$_ENV` or `$_SERVER` in PHP:

```php
$dsn = $_ENV['DATABASE_URL'];
```

But that bypasses everything else in this chapter. The `%env()%` form is what you want, because it is typeable, cacheable in the compiled container as a placeholder, and inspectable with the debug commands in §6.6. Reach for `$_ENV` only inside a `Kernel` override or very early bootstrap code.

> **Symfony 8** — Symfony 8.1 allows a `.` in environment variable names (for example, `FOO.BAR`). On the 7.4 LTS line a dot in an env var name is not supported. If you name variables with dots, pin your minimum version accordingly.

##### Environment variable processors

Environment variables are *strings*. Your application, of course, needs integers, booleans, arrays, enums, and the components of a DSN. Symfony's **env var processors** transform the raw string. You stack them by prefixing the variable name, reading right-to-left:

```yaml
framework:
    router:
        http_port: '%env(int:HTTP_PORT)%'
```

The full set of built-in processors is large; here are the ones you will actually use, with a worked example of each:

| Processor | What it does | Example |
|---|---|---|
| `env(string:FOO)` | Cast to string (explicit) | `'%env(string:APP_SECRET)%'` |
| `env(bool:FOO)` | Cast to boolean (`'1'`, `'true'`, `'on'`, `'yes'` → `true`) | `'%env(bool:APP_DEBUG)%'` |
| `env(not:FOO)` | Boolean, inverted | `safe_mode: '%env(not:APP_DEBUG)%'` |
| `env(int:FOO)` | Cast to integer | `'%env(int:HTTP_PORT)%'` |
| `env(float:FOO)` | Cast to float | `'%env(float:RATE_LIMIT)%'` |
| `env(json:FOO)` | JSON-decode to array or `null` | `'%env(json:ALLOWED_LANGUAGES)%'` |
| `env(csv:FOO)` | Split CSV string into array | `'%env(csv:CORS_ORIGINS)%'` |
| `env(const:FOO)` | Look up a PHP constant named in `FOO` | `'%env(const:HEALTH_METHOD)%'` |
| `env(file:FOO)` | Read the file whose path is `FOO` | `'%env(file:AUTH_FILE)%'` |
| `env(require:FOO)` | `require()` the file and return its value | `'%env(require:PHP_FILE)%'` |
| `env(trim:FOO)` | Trim surrounding whitespace (pairs with `file`) | `'%env(trim:file:LICENSE_FILE)%'` |
| `env(key:NAME:FOO)` | Pull one key out of an array in `FOO` | `'%env(key:database_password:json:SECRETS)%'` |
| `env(default:FALLBACK:FOO)` | Use parameter `FALLBACK` if `FOO` is unset | `'%env(default:raw_key:file:PRIVATE_KEY)%'` |
| `env(url:FOO)` | Parse a URL into its component array | see below |
| `env(enum:Enum:FOO)` | Cast a string to a `\BackedEnum` case | `'%env(enum:App\Enum\BillingPlan:DEFAULT_PLAN)%'` |
| `env(defined:FOO)` | `true` if `FOO` is set and non-empty | `'%env(defined:STRIPE_API_KEY)%'` |
| `env(shuffle:FOO)` | Shuffle an array (pairs with `csv`) | `'%env(shuffle:csv:REDIS_NODES)%'` |

Processors compose. The classic DSN pattern uses `url` together with `key` to extract individual components:

```yaml
# config/packages/doctrine_mongodb.yaml
doctrine_mongodb:
    clients:
        default:
            hosts:
                - { host: '%env(key:host:url:MONGODB_URL)%',
                    port: '%env(int:key:port:url:MONGODB_URL)%' }
            username: '%env(key:user:url:MONGODB_URL)%'
            password: '%env(key:pass:url:MONGODB_URL)%'
            database_name: '%env(key:path:url:MONGODB_URL)%'
```

This is why a single `DATABASE_URL="postgresql://user:pass@host:5432/db?serverVersion=15.4"` in your `.env` is enough to configure a full Doctrine connection — the bundle pulls each piece out of the URL with a processor.

> **Tip** — To read every processor you can use, run `php bin/console` and consult the "Environment Variable Processors" page of the Symfony docs, or simply look at how a mature bundle reads a DSN. When in doubt, `env(string:FOO)` is always a safe, explicit choice.

###### Giving an environment variable a default

Sometimes you want an environment variable to be *optional* — used if present, falling back to a sane default if not. You declare that default under the `parameters` key using the `env(NAME):` pseudo-parameter:

```yaml
# config/services.yaml
parameters:
    # If APP_TIMEZONE is unset, it will be 'UTC'.
    env(APP_TIMEZONE): 'UTC'

services:
    app.scheduler:
        arguments:
            $timezone: '%env(APP_TIMEZONE)%'
```

Now the app runs out of the box, and any deployment that sets `APP_TIMEZONE` overrides it.

###### In PHP configuration closures

When you configure a service in a PHP file (Chapter 5 showed the configurator API), the equivalent of `%env(FOO)%` is the `env()` helper imported from the Configurator namespace:

```php
// config/packages/stripe.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use App\StripeClient;

return function (ContainerConfigurator $container): void {
    $container->services()
        ->set('app.stripe_client', StripeClient::class)
        ->arg('$apiKey', env('STRIPE_API_KEY'));
};
```

Both forms produce the same runtime-resolved placeholder in the compiled container.

###### Custom processors

If none of the built-ins fit, you can add your own. Implement `EnvVarProcessorInterface` and register the class as a service (autoconfiguration tags it for you in a standard project):

```php
<?php

declare(strict_types=1);

namespace App\EnvVar;

use Symfony\Component\DependencyInjection\EnvVarProcessorInterface;

final class LowercaseProcessor implements EnvVarProcessorInterface
{
    public function getEnv(string $prefix, string $name, \Closure $getEnv): string
    {
        return strtolower($getEnv($name));
    }

    public static function getProvidedTypes(): array
    {
        // 'lowercase' => 'string'  →  usable as %env(lowercase:FOO)%
        return ['lowercase' => 'string'];
    }
}
```

You will rarely need this — the built-ins cover nearly every case — but it is good to know the extension point exists.

---

#### 6.3 Parameters

Environment variables are *deployment-specific* and *stringly-typed by nature*. **Parameters** are the other half of the story: named values that you define once and reuse across your own configuration. They live under the `parameters` key, by convention in `config/services.yaml`:

```yaml
# config/services.yaml
parameters:
    # Prefix your own parameters with 'app.' to keep them distinct from the
    # framework's own parameters.
    app.support_email: 'support@example.com'
    app.max_upload_size: 5242880                 # 5 MB, as an integer
    app.supported_locales: ['en', 'es', 'fr']     # an array

    # A PHP constant, resolved at build time:
    app.version: !php/const App\Version::RELEASE

    # An enum case:
    app.default_plan: !php/enum App\Enum\BillingPlan::Pro
```

Once defined, reference a parameter anywhere by wrapping its name in two percent signs:

```yaml
# config/packages/acme_notify.yaml
acme_notify:
    from_address: '%app.support_email%'
```

Parameters can be scalars, integers, floats, booleans, arrays, and — with the tag syntax shown above — PHP constants (`!php/const`), enum cases (`!php/enum`), and base64 binary content (`!!binary`).

##### Naming conventions that matter

Two conventions are worth adopting from day one:

- **Use the `app.` prefix** for everything you define. The framework and third-party bundles define their own parameters; the prefix keeps your namespace clean and self-documenting.
- **A leading dot means "compile-time only."** A parameter named `.mailer.transport` (note the leading `.`) is available *only while the container is being compiled*. It disappears from the final container. This is the tool for scratch values you need inside a compiler pass (Chapter 8). You should never try to inject a dot-prefixed parameter into a service.

##### Escaping a literal percent sign

Because `%name%` is the reference syntax, a value that genuinely contains a percent sign must escape it by doubling it:

```yaml
parameters:
    # Parsed as the literal string 'https://symfony.com/?foo=%s&bar=%d'
    url_pattern: 'https://symfony.com/?foo=%%s&amp;bar=%%d'
```

> **Caution** — **Parameters cannot be used to build import paths.** This is a hard limitation, not a typo you can route around:
>
> ```yaml
> # config/services.yaml
> imports:
>     - { resource: '%kernel.project_dir%/somefile.yaml' }   # ← does NOT work
> ```
>
> Imports are resolved *before* parameters, so `%…%` is treated literally and the file is not found. If you need to import from a variable location, compute the path in a PHP loader, not in a `imports:` list.

##### Enforcing that a required parameter is present

Parameters are not validated by default — a missing one simply resolves to `null`, which can fail far away from the cause. When a parameter is genuinely load-bearing, guard it in a compiler pass:

```php
$container->parameterCannotBeEmpty('app.private_key',
    'Did you forget to set a value for the "app.private_key" parameter?');
```

This throws when the parameter is `null`, `''`, or `[]` — giving you a clear, early error instead of a mysterious failure at runtime.

##### The built-in (kernel) parameters

Symfony seeds the container with a set of parameters you will reference constantly. You do not define these; they are created from your `Kernel` and `framework` configuration. The most useful:

| Parameter | Type | Default / meaning |
|---|---|---|
| `kernel.project_dir` | string | Absolute path to the project root (directory of `composer.json`) |
| `kernel.environment` | string | The active configuration environment (`dev`, `prod`, `test`, …) |
| `kernel.debug` | bool | Whether debug mode is on |
| `kernel.cache_dir` | string | `var/cache/{environment}` — per-environment cache |
| `kernel.build_dir` | string | Read-only build dir; separate from `cache_dir` for Docker/Lambda |
| `kernel.logs_dir` | string | `var/log` |
| `kernel.charset` | string | `UTF-8` |
| `kernel.container_class` | string | e.g. `App_KernelDevDebugContainer` |
| `kernel.secret` | string | `%env(APP_SECRET)%` |
| `kernel.default_locale` | string | Default locale (from `framework.default_locale`) |
| `kernel.bundles` | array | Map of bundle name → bundle class |
| `kernel.runtime_environment` | string | *Where* the app is deployed (distinct from `environment`) |
| `kernel.runtime_mode.web` / `.cli` / `.worker` | bool | Which runtime mode is active (FrankenPHP-related) |

`kernel.project_dir` is your best friend for building paths:

```yaml
services:
    app.logo_storage:
        arguments:
            $path: '%kernel.project_dir%/public/uploads/logos'
```

The `runtime_environment` vs. `environment` distinction is subtle and worth a sentence: `kernel.environment` selects *which configuration files* load; `kernel.runtime_environment` describes *where the process is deployed* (e.g. `staging`, `production`, `preview`). You can run the `prod` configuration on several different runtime environments — a refinement that mostly matters with long-running servers, and one you can safely ignore until Chapter 24.

> **Symfony 8** — Symfony 8.1 honors the standard `SOURCE_DATE_EPOCH` environment variable for reproducible container builds (it pins `container.build_time` to a fixed timestamp). Combined with a fixed `kernel.container_build_time` parameter, this makes the compiled container byte-for-byte reproducible. If you ship containers and care about reproducible builds, this is the hook to use.

##### Parameters vs. environment variables: which do I pick?

This is the most common confusion, so here is the decision rule, stated plainly:

| Use an **environment variable** when… | Use a **parameter** when… |
|---|---|
| The value differs by *deployment* (DB URL, API keys, secrets) | The value is the same everywhere but you want to say it *once* |
| You want to change it *without* touching config files or rebuilding | The value is part of your *design*, not your deployment |
| The value is sensitive and must never be committed | You want it to be a typed, named, reusable building block |
| It must be resolved at *runtime* | It can be baked in at *build time* |

In practice you use both together, and parameters frequently *reference* env vars:

```yaml
parameters:
    app.cache_backend: '%env(CACHE_DSN)%'   # parameter wraps an env var
framework:
    cache:
        app: '%app.cache_backend%'           # config references the parameter
```

---

#### 6.4 Per-environment configuration

You have *one* application, but it needs to behave differently in several places: verbose logging and the profiler in development, speed and error-only logging in production, and a deterministic, quiet setup in tests. **Configuration environments** are how Symfony expresses "same app, different behavior."

Every project ships with three: `dev`, `prod`, and `test`. All three share a large base of configuration; each environment layers a small set of *overrides* on top.

##### The load order

When Symfony builds the container for a given environment, it loads configuration in this exact order — **later files override earlier ones**:

```text
1. config/packages/*.<ext>                base config, shared by all environments
2. config/packages/<environment>/*.<ext>  per-environment overrides (e.g. packages/test/)
3. config/services.<ext>                  base service definitions
4. config/services_<environment>.<ext>    per-environment service overrides
```

Walk it concretely. `config/packages/framework.yaml` is loaded in *every* environment. In `test`, the (small) file `config/packages/test/framework.yaml` is loaded *after* it and overrides only the keys it mentions. Everything else in `framework.yaml` is untouched. This is why the per-environment files are tiny: you only ever write the *differences*.

```yaml
# config/packages/framework.yaml   (loaded everywhere)
framework:
    http_method_override: true
    handle_all_throwables: true
```

```yaml
# config/packages/test/framework.yaml   (loaded only in the "test" environment)
framework:
    test: true                    # enables the client / profiler stubs for testing
```

The per-environment `services_test.yaml` works the same way and is where you typically swap in mocks or trim services:

```yaml
# config/services_test.yaml
services:
    test.client:
        class: Symfony\Component\BrowserKit\Client
```

##### The `when@` keyword: overrides inside a single file

Splitting into `packages/{env}/` files is the classic approach, but there is a lighter-weight alternative: the `when@<env>` keyword, which lets you scope a block to an environment *within the same file*:

```yaml
# config/packages/webpack_encore.yaml
webpack_encore:
    output_path: '%kernel.project_dir%/public/build'
    strict_mode: true
    cache: false

when@prod:
    webpack_encore:
        cache: true              # cache enabled only in production

when@test:
    webpack_encore:
        strict_mode: false       # loose mode only in tests
```

Prefer `when@` for a *single* small toggle in a bundle you already configure in this file; prefer the `packages/{env}/` directory when a whole bundle needs a distinct treatment per environment. Both compile to the same thing.

In a **PHP** configuration closure you get the active environment directly as an argument named `$env`:

```php
// config/packages/my_package.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return function (ContainerConfigurator $container, string $env): void {
    $container->parameters()->set('app.is_test', 'test' === $env);
};
```

##### Selecting and creating environments

You select the environment with `APP_ENV` (see §6.2). To create a *new* one — say `staging`, so a client can preview the app — do exactly this:

1. Create the directory `config/packages/staging/`.
2. Drop in only the files that differ from the base (for example, `config/packages/staging/routing.yaml` to point staging at a different API).
3. Set `APP_ENV=staging` for that deployment.

That is all it takes. Symfony will load the base `config/packages/*.yaml` first and your `staging/` overrides second.

> **Tip** — When two environments are almost identical (say `staging` and `prod`), you can **symlink** files between the `config/packages/<env>/` directories to avoid duplication. Just be aware that a symlink is only as portable as your deployment filesystem — some PaaS environments do not preserve them.

---

#### 6.5 Bundle configuration trees

So far you have seen bundles *accept* configuration (`doctrine:`, `framework:`, `acme_notify:`). This section turns the camera around and shows you *how a bundle defines and validates the configuration it accepts* — which is exactly what you will do in Chapter 8 when you write your own bundle.

The mental model is a **contract** with two parts:

1. **Define the shape** — a *configuration tree* that declares every option, its type, its default, and its allowed values.
2. **Process the input** — take the (possibly merged) array the user gave you, validate it against the tree, fill in defaults, and use the result to wire up services.

##### The root key comes from the bundle name

The top-level key of a bundle's configuration is derived automatically: it is the **snake_case of the bundle class name with the `Bundle` suffix removed**.

- `AcmeNotifyBundle` → `acme_notify`
- `DoctrineBundle` → `doctrine`
- `TwigBundle` → `twig`

This is why `Acme\NotifyBundle` reads configuration under the `acme_notify:` key with no extra wiring.

##### The modern way: `AbstractBundle::configure()`

For a new bundle, the recommended approach is to put both halves on the bundle class itself, extending `AbstractBundle`:

```php
<?php

declare(strict_types=1);

namespace Acme\NotifyBundle;

use Symfony\Component\Config\Definition\Configurator\DefinitionConfigurator;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;
use Symfony\Component\HttpKernel\Bundle\AbstractBundle;

class AcmeNotifyBundle extends AbstractBundle
{
    // 1. Define the shape of the configuration.
    public function configure(DefinitionConfigurator $definition): void
    {
        $definition->rootNode()
            ->children()
                ->scalarNode('from')
                    ->isRequired()
                    ->setInfo('The "From" address for all outgoing mail')
                ->end()
                ->integerNode('max_per_hour')
                    ->defaultValue(100)
                    ->setInfo('Hard ceiling on messages per hour')
                ->end()
                ->arrayNode('templates')
                    ->prototype('scalar')->end()   // a list of strings
                ->end()
            ->end();
    }

    // 2. Process the merged config and wire it into the container.
    //    By the time this runs, $config is already merged + validated.
    public function loadExtension(
        array $config,
        ContainerConfigurator $container,
        ContainerBuilder $builder,
    ): void {
        $container->parameters()
            ->set('acme_notify.from', $config['from'])
            ->set('acme_notify.max_per_hour', $config['max_per_hour'])
            ->set('acme_notify.templates', $config['templates']);
    }
}
```

The corresponding user-facing configuration is now fully type-checked:

```yaml
# config/packages/acme_notify.yaml
acme_notify:
    from: 'notifications@example.com'
    max_per_hour: 500
    templates:
        - 'email/invoice_paid'
        - 'email/invoice_overdue'
```

Because `from` is `isRequired()`, *omitting* it fails the build with a precise error. Because `max_per_hour` is an `integerNode`, writing `max_per_hour: fast` fails with a type error. And because only `from`, `max_per_hour`, and `templates` are declared, typing a *typo* — `templats:` — also fails. **That is the point of the tree: it converts "it broke three services in at 3 a.m." into "unknown option `templats` under `acme_notify`" at deploy time.**

> **Note** — Both `configure()` and `loadExtension()` run **only at compile time**. The `$config` you receive in `loadExtension()` is the fully merged, defaulted, validated array — you never have to merge or validate it yourself.

##### The traditional way: `Configuration` + `Extension`

You will still encounter the older, separate-class approach in many existing bundles, and understanding it is useful. Here the shape lives in a `Configuration` class and the processing in an `Extension` class:

```php
<?php

declare(strict_types=1);

namespace Acme\NotifyBundle\DependencyInjection;

use Symfony\Component\Config\Definition\Builder\TreeBuilder;
use Symfony\Component\Config\Definition\ConfigurationInterface;

class Configuration implements ConfigurationInterface
{
    public function getConfigTreeBuilder(): TreeBuilder
    {
        $treeBuilder = new TreeBuilder('acme_notify');

        $treeBuilder->getRootNode()
            ->children()
                ->scalarNode('from')->isRequired()->end()
                ->integerNode('max_per_hour')->defaultValue(100)->end()
            ->end();

        return $treeBuilder;
    }
}
```

```php
<?php

declare(strict_types=1);

namespace Acme\NotifyBundle\DependencyInjection;

use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Extension\Extension;

class AcmeNotifyExtension extends Extension
{
    public function load(array $configs, ContainerBuilder $container): void
    {
        // $configs is an ARRAY OF ARRAYS (see below). processConfiguration()
        // merges them, applies defaults, and validates against the tree.
        $config = $this->processConfiguration(new Configuration(), $configs);

        $container->setParameter('acme_notify.from', $config['from']);
    }
}
```

##### The "array of arrays" — how overrides actually merge

There is one subtlety in `processConfiguration()` worth understanding, because it explains *why* per-environment overrides work at the bundle level. The `$configs` argument is **not** a flat array; it is an **array of arrays** — one entry per configuration source that mentioned your root key.

If `acme_notify:` appears in `config/packages/acme_notify.yaml` *and* in `config/packages/prod/acme_notify.yaml`, the extension receives:

```php
[
    // from config/packages/acme_notify.yaml
    ['from' => 'notifications@example.com', 'max_per_hour' => 100],
    // from config/packages/prod/acme_notify.yaml
    ['max_per_hour' => 5000],
]
```

`processConfiguration()` then **merges** these into a single, validated array, later entries winning. This is the mechanism that lets a bundle participate in the per-environment system without knowing anything about environments.

> **Caution** — **Merging arrays is not the same as concatenating lists.** For *associative* arrays (string keys), merging is deep and later keys win. For *lists* (integer-keyed arrays, like a `prototype` list), the second source **replaces** the first unless the node is declared with `append: true`. This is the single most common surprise when a per-environment file "silently discards" a base list. If you intend to *add* to a list in an environment, make that node append:
>
> ```php
> ->arrayNode('whitelist')
>     ->append()                       // lists are concatenated, not replaced
>     ->prototype('scalar')->end()
> ->end()
> ```

##### The node vocabulary (the ones you'll reach for)

You do not need to memorize the whole tree-builder API. These cover 95% of cases:

| Builder | Purpose |
|---|---|
| `scalarNode()` | A string (or a scalar). The default type. |
| `integerNode()` / `floatNode()` | Numeric, type-validated. |
| `booleanNode()` | `true` / `false`. |
| `enumNode()->values([...])` | A scalar constrained to a fixed set. |
| `arrayNode()` | A nested structure. |
| `->prototype('scalar')` | A *list* of scalars (like `templates:` above). |
| `->arrayPrototype()` | A *map* of identical sub-structures (like named cache pools). |
| `variableNode()` | Anything, no validation — a last resort. |
| `->defaultValue()` | Applied when the key is absent. |
| `->isRequired()` | Build fails if the key is absent. |
| `->info()` / `->example()` | Rendered as comments in `config:dump-reference` output. |
| `->beforeNormalizing()` / `->validate()` | Custom transforms and cross-field checks. |

`arrayPrototype()` is the workhorse for "configure many of the same thing." The `framework.cache` configuration is built exactly this way — a map of cache pools, each with the same shape. Study a core `Configuration` class (FrameworkBundle's or TwigBundle's) as a living reference when your own tree grows.

##### How configuration becomes services

The end goal of `loadExtension()` / `load()` is always the same: **turn config values into a configured container.** The most common patterns are:

- **Publish a parameter** (as in the examples above) that a service or other bundle can reference.
- **Replace an argument** on a service that the bundle's own service file declares with a placeholder:

```php
$definition = $container->getDefinition('acme_notify.mailer');
$definition->replaceArgument(0, $config['from']);
```

- **Conditionally register services** based on a flag:

```php
if ($config['max_per_hour'] > 0) {
    $container->register('acme_notify.throttle', ThrottleService::class)
        ->arg('$maxPerHour', $config['max_per_hour']);
}
```

> **Tip** — If your extension only needs to *process config and then* do the usual wiring, extend `ConfigurableExtension` instead of `Extension`. It calls `processConfiguration()` for you and hands you the merged array in `loadInternal()`, saving you the boilerplate.

##### One bundle configuring another: prepend

When two bundles are tightly related, one can *inject* configuration into another before that bundle's extension runs, via the `prepend()` method. FrameworkBundle uses this internally (for instance, to configure the profiler into the `framework` config). You will meet it properly in Chapter 8 when you build a bundle that depends on others; for now, just know the mechanism exists for "my bundle needs to add options to *your* bundle."

##### Seeing the default configuration: `config:dump-reference`

For any bundle, you can dump its entire default configuration tree — every option, every default, with your `info()` comments rendered as YAML comments:

```bash
$ php bin/console config:dump-reference acme_notify
acme_notify:
    # The "From" address for all outgoing mail
    from: null
    # Hard ceiling on messages per hour
    max_per_hour: 100
    templates: []
```

This is *the* command to reach for when you want to know what a bundle lets you configure and what its defaults are. It works automatically as long as the bundle's `Configuration` sits in the standard location (`<Bundle>/src/DependencyInjection/Configuration`) or you use the `AbstractBundle::configure()` style above.

---

#### 6.6 The `config/` directory, organized

Time to step back and look at the whole room. Here is the modern `config/` layout, annotated for what each part does (this is what `symfony new --full` gives you):

```text
config/
├── bundles.php            # Which bundles are enabled — per environment
├── services.yaml          # Service definitions, autowiring rules, and parameters
├── routes/                # Route imports (one file per bundle that provides routes)
├── packages/              # One YAML file per bundle's configuration
│   ├── framework.yaml
│   ├── doctrine.yaml
│   ├── security.yaml
│   ├── twig.yaml
│   ├── test/              # ← per-environment overrides (loaded after base)
│   │   └── framework.yaml
│   └── prod/              # ← (often empty; prod is mostly "the base + env vars")
│       └── framework.yaml
└── preload.php            # Optional OPcache.preload class list (performance)
```

The rules of thumb for "where does this go?" are simple and worth keeping on a sticky note:

| You want to… | Put it in… |
|---|---|
| Turn a bundle on/off (globally or per env) | `config/bundles.php` |
| Configure a bundle's semantic options | `config/packages/<bundle>.yaml` |
| Override a bundle option for one environment | `config/packages/<env>/<bundle>.yaml` (or `when@` in the base file) |
| Define a service, or a reusable parameter | `config/services.yaml` |
| Override/replace a service for one environment | `config/services_<env>.yaml` |
| Add routes from a bundle or directory | `config/routes/` |
| Preload classes for performance | `config/preload.php` |

##### `bundles.php` is environment-aware

`bundles.php` returns a map of bundle class to an array of environments (or `true` for all). This is how a bundle is enabled in some environments but not others — for example, the debug bundle in `dev` only:

```php
<?php

use Symfony\Bundle\FrameworkBundle\FrameworkBundle;

return [
    FrameworkBundle::class => ['all' => true],
    // Enabled in every environment except prod:
    // DebugBundle::class => ['dev' => true, 'test' => true],
];
```

##### `services.yaml` is the backbone

`services.yaml` is the file you will open most. It holds three things: `parameters:`, the `services:` definitions (the heart of Chapter 5), and the `imports:` that pull in other files. Let's look at how imports actually work, since they are how the whole directory is stitched together:

```yaml
# config/services.yaml
parameters:
    app.support_email: 'support@example.com'

services:
    # …service definitions from Chapter 5…
```

The `imports:` directive (from the **Config** component) can pull in other files, supports **globs**, and can tolerate missing files:

```yaml
imports:
    - { resource: 'legacy_config.php' }
    # Glob: load every YAML in a directory
    - { resource: '/etc/myapp/*.yaml' }
    # Silently skip if the file does not exist
    - { resource: 'optional_feature.yaml', ignore_errors: not_found }
    # Glob, but exclude specific files
    - { resource: 'services/*.yaml', exclude: ['services/legacy_*.yaml'] }
```

> **Note** — `reference.php` is a newer addition to the default layout in recent Symfony versions (the 8.x line). It is *generated* by Symfony and contains typed definitions that improve IDE autocompletion and static analysis **when you use PHP as your configuration format**. It is safe to commit, and if you rely on it you may add `config/` to your Composer `classmap`. It is not required and does nothing when you configure in YAML.

##### A note on formats

Symfony does not force a configuration format. You have two that matter:

- **YAML** — the default, concise, very readable. This book uses it for `config/`.
- **PHP** — returns arrays or closures; dynamic, and benefits from autocompletion and static analysis. Use it when a configuration value needs logic (the `$env`-aware closures above are a good example).

(An XML format existed historically but was deprecated in Symfony 7.4, which is why it appears nowhere in this book.) There is *no* performance difference: every format is compiled to the same PHP and cached before it ever runs.

---

#### 6.7 Debugging configuration

You will not always get configuration right on the first try. These five commands are your diagnostics; learn them now and you will save yourself hours of `var_dump` archaeology later (Chapter 23 covers the broader debugging toolkit).

##### 1. `config:dump-reference` — "what *can* I configure?"

```bash
$ php bin/console config:dump-reference framework
$ php bin/console config:dump-reference doctrine
```

Prints the full default configuration tree for a bundle, with comments. Your reference for what options exist and what they default to.

##### 2. `debug:config` — "what *am I* actually running with?"

```bash
$ php bin/console debug:config framework
$ php bin/console debug:config acme_notify
```

Unlike the previous command, this shows the configuration **as resolved for your current environment** — base values plus all your overrides and per-environment changes, merged and final. This is the command to run when you are sure you set something but the app isn't behaving as expected.

##### 3. `debug:container` — "is my parameter/service there?"

```bash
# Show one parameter's resolved value
$ php bin/console debug:container app.support_email

# List every environment variable, its default, and its real (runtime) value
$ php bin/console debug:container --env-vars

# Filter to one variable
$ php bin/console debug:container --env-var=DATABASE_URL
```

The `--env-vars` view is especially useful in production-style debugging because it shows, side by side, the *default* (from your `parameters`/`.env`) and the *real* value that will actually be used at runtime.

##### 4. `lint:yaml` — "is my YAML even valid?"

```bash
$ php bin/console lint:yaml config/
```

Catches syntax errors (bad indentation, tabs, stray characters) before they ever reach the compiler. Cheap, fast, run it in CI.

##### 5. Read the compiler's errors — they are precise

When the tree builder rejects your configuration, the error is usually excellent and points at the exact key:

```text
[Symfony\Component\Config\Definition\Exception\InvalidConfigurationException]
Invalid configuration for "acme_notify" in "acme_notify":
- Unrecognized option "templats" under "acme_notify". Available options are
  "from", "max_per_hour", "templates".
```

> **Caution** — **You are working against a compiled cache.** If you change configuration and see the *old* behavior, your first instinct should be `php bin/console cache:clear` (or, in `dev`, let Symfony detect the change and rebuild automatically). Configuration is frozen at build time; a stale cache is the classic cause of "I changed it, why isn't it working?" Remember the two-clock model from §6.1: structural config changes need a rebuild; pure `%env()%` values do not.

---

#### 6.8 Putting it together: one value, end to end

Let's trace a single value through every layer of this chapter to make the mental model concrete. Suppose a feature needs an outbound SMTP relay host that differs per deployment.

**1. Define the variable** (per-deployment, secret-ish) — in the real environment, or in `.env` for local development:

```dotenv
# .env  (a harmless default for development)
MAIL_RELAY_HOST=localhost
```

```bash
# In production, the orchestrator sets it for real:
MAIL_RELAY_HOST=smtp.mail.example.com
```

**2. Give it a typed default** in `services.yaml` (so the app runs even if unset):

```yaml
parameters:
    env(MAIL_RELAY_HOST): 'localhost'
    app.mail_relay: '%env(MAIL_RELAY_HOST)%'   # parameter wraps the env var
```

**3. Consume it** in a service (Chapter 5) — referencing the *parameter*, not the env var directly:

```yaml
services:
    app.mail_transport:
        class: App\MailTransport
        arguments:
            $host: '%app.mail_relay%'
```

**4. Validate and inspect** — `debug:container app.mail_relay` shows the resolved host; `debug:container --env-vars` shows that in production the *real* value came from the orchestrator, not your committed `.env`.

Now the whole journey is legible: a deployment-specific value lives in the environment, is typed by a processor, is exposed as a stable parameter, is injected into a service, and can be inspected without a debugger. Change `MAIL_RELAY_HOST` in production and the next request uses it — no rebuild, no restart of the code. That is the configuration system working exactly as designed.

---

#### 6.9 Chapter summary

- **Two clocks.** Structural configuration (bundle options, parameters, service wiring) is compiled into the container at build time. Environment variables referenced with `%env(VAR)%` are resolved at runtime. This is why secrets and deployment-specific values belong in env vars.
- **The `.env` chain** loads `.env` → `.env.local` → `.env.$APP_ENV` → `.env.$APP_ENV.local`, later files winning — but the **real OS environment always outranks all files**. `APP_ENV` picks the environment; `APP_DEBUG` and `APP_SECRET` complete the trio.
- **Env var processors** (`int:`, `bool:`, `json:`, `csv:`, `url:` + `key:`, `enum:`, …) turn strings into typed values, and compose. `env(NAME):` under `parameters` gives a variable a default.
- **Parameters** are named, reusable values; prefix your own with `app.`, and know that a leading dot means "compile-time only." They cannot build `imports:` paths. `parameterCannotBeEmpty()` guards load-bearing ones. The `kernel.*` parameters give you paths, the environment, and the secret.
- **Environments** share a base config and layer tiny per-environment overrides, loaded in a fixed order (`packages/*`, then `packages/{env}/*`, then `services.*`, then `services_{env}.*`). `when@` gives you inline per-environment blocks; a new environment is just a new `packages/{env}/` directory plus an `APP_ENV` value.
- **Bundle configuration trees** are a contract: define the shape (via `AbstractBundle::configure()` or a `Configuration` class), then process the merged input (via `loadExtension()` or `processConfiguration()`). The root key is the snake_case bundle name; unknown keys and wrong types fail the build loudly.
- **`config/`** is organized by concern: `bundles.php` (enable/disable), `packages/` (bundle options), `services.yaml` (services + parameters), `routes/`, and per-environment subdirectories.
- **Five debug commands** — `config:dump-reference`, `debug:config`, `debug:container` (+ `--env-vars`), `lint:yaml`, and reading the compiler's errors — let you see exactly what Symfony built. When config "doesn't take," suspect the compiled cache first.

In the next chapter we put a different kind of machinery to work: **events**, the framework's way of letting loosely-coupled components react to the same request pipeline you saw in Chapter 4 — and the natural home for the cross-cutting concerns that configuration alone cannot express.

---

#### Exercises

**Quick checks**

1. You set `APP_SECRET` in both your committed `.env` and your production orchestrator. Which value does production use, and why? What would have to change for the `.env` value to win?
2. Name the four files, in order, that `loadEnv()` loads when `APP_ENV=test`. Which of them are committed to version control?
3. What is the difference between `kernel.environment` and `kernel.runtime_environment`? Give one situation where they differ.
4. You write `max_items: '%env(int:MAX_ITEMS)%'` but `MAX_ITEMS` is never set anywhere. What value does the service receive, and how do you give it a fallback without forcing every deployment to set the variable?

**Practical**

5. Add a parameter `app.api_base_url` to `config/services.yaml`, then reference it from a service argument and from a bundle option. Use `debug:container app.api_base_url` to confirm it resolves.
6. Create a custom `staging` environment: add `config/packages/staging/` with one file that overrides a single `framework` option, set `APP_ENV=staging`, and run `debug:config framework` to prove the override took effect.
7. Define a bundle option with three node types (a required scalar, an integer with a default, and a list of strings). Intentionally configure each one wrongly (missing the required key, a non-integer, an unknown key) and record the three distinct error messages Symfony produces.
8. Find a DSN (for example `DATABASE_URL`) in your `.env` and extract two of its components into configuration using the `url` and `key` processors.

**Stretch ⭐**

9. Write a **custom env var processor** that validates a DSN (e.g. that the scheme is one of a fixed set) and returns a `false`-y sentinel if invalid. Wire it into a service and make `debug:container --env-vars` reveal your processor's effect. Where is the single tag that registers it, and what makes autoconfiguration apply it in a standard project?
10. Build a small `AbstractBundle` that exposes configuration, then have a *second* bundle use `prepend()` to inject a default value into the first bundle's configuration. Use `config:dump-reference` on both and `debug:config` to trace how the prepended value lands. (This is a dress rehearsal for Chapter 8.)
11. **Two clocks, demonstrated.** In a `dev` project, pick one value that flows through `%env()%` and one that is a plain parameter. Change each and observe *when* the app picks up the change — which required a `cache:clear` and which did not? Write a short note explaining the result using the compile-time/runtime distinction.
12. Reproduce the "list silently replaced" pitfall from the merge Caution: declare an append vs. a non-append list node, override it in a `packages/test/` file, and show with `debug:config` that one is concatenated and the other is replaced. Which single modifier changes the behavior?

### Chapter 7: Events and Middleware

*Symfony gives you a rich pipeline of built-in hooks that fire during every request, plus a general-purpose event system you can use for any domain logic. Mastering both is the difference between an application where every concern is tangled into controllers and one where each responsibility lives in its own testable, composable unit.*

---

#### 7.1 The Problem Events Solve

Imagine the invoicing app from Parts III–V. A single `POST /invoices` request triggers a cascade of responsibilities:

- Resolve which tenant the request belongs to.
- Verify the user's rate limit.
- Authenticate and authorize the caller.
- Validate and persist the invoice.
- Send a confirmation email.
- Log the request for auditing.
- Update usage counters for the billing module.

If you stuff all of this into a controller, the method becomes a 200-line wall of logic that no one can modify without fear. Worse, adding a *new* cross-cutting concern—say, a feature-flag check or a security header—requires touching every controller.

The **EventDispatcher** solves this by decoupling *what happened* from *what to do about it*. The domain code announces an event; independent listeners react. No listener needs to know the others exist, and adding or removing a concern never changes the code that fires the event.

Symfony implements two well-known design patterns here: the **Observer** pattern (listeners subscribe to events) and the **Mediator** pattern (the dispatcher coordinates without coupling). The component is PSR-14 compliant, so it interoperates with any standard-compliant event library.

In this chapter we will:

1. Build custom domain events for the invoicing app.
2. Register listeners and subscribers using PHP attributes.
3. Walk through every kernel event in the request lifecycle.
4. Implement cross-cutting concerns (tenant resolution, rate limiting, request logging) as event listeners.
5. Explore middleware-style patterns and know when to reach for events versus decorators.

---

#### 7.2 The EventDispatcher: Core Concepts

##### 7.2.1 Events, Listeners, and the Dispatcher

The vocabulary is small:

| Term | Role |
|---|---|
| **Event** | A data object (usually a subclass of `Symfony\Contracts\EventDispatcher\Event`) carrying context about what happened. |
| **Event name** | A string or class name that identifies the event. Listeners subscribe to this. |
| **Listener** | A callable that reacts to a specific event. |
| **Subscriber** | A class that declares *multiple* event/method/priority bindings in one place. |
| **Dispatcher** | The central registry. It holds every listener and, when an event fires, calls them in priority order. |

The dispatcher is a service in any Symfony application. You inject the interface, not the concrete class:

```php
// src/Service/InvoiceService.php
namespace App\Service;

use Symfony\Contracts\EventDispatcher\EventDispatcherInterface;

final class InvoiceService
{
    public function __construct(
        private readonly EventDispatcherInterface $dispatcher,
    ) {}

    public function create(array $data): Invoice
    {
        // ... build and persist the Invoice ...

        $this->dispatcher->dispatch(new InvoiceCreatedEvent($invoice));

        return $invoice;
    }
}
```

The call to `dispatch()` is the only coupling between the service and the rest of the system. Anything that cares about a newly created invoice can listen to `InvoiceCreatedEvent`; the service doesn't know or care what they do.

> **Convention:** In this book we type-hint `EventDispatcherInterface` from `Symfony\Contracts\EventDispatcher`. Use the fuller `Symfony\Component\EventDispatcher\EventDispatcherInterface` only when you need methods like `getListeners()` or `getListenerPriority()` for introspection.

##### 7.2.2 Two Ways to Name an Event

Symfony accepts two forms for the event identifier:

```php
// String name (traditional)
$dispatcher->dispatch(new MyEvent(), 'invoice.created');

// Class name (recommended since Symfony 5.3)
$dispatcher->dispatch(new InvoiceCreatedEvent());
// name defaults to InvoiceCreatedEvent::class
```

Using the class name as the event identifier is strongly preferred in modern code. It gives you:

- **Type safety.** The listener method can type-hint the event class, and PHP (or a static analyser) catches mismatches at compile time.
- **Refactoring.** Renaming the class updates every reference automatically.
- **Clarity.** The class *is* the contract; its properties document what data the event carries.

The rest of this chapter uses class-name events exclusively.

---

#### 7.3 Defining Custom Events

An event class is a plain value object. Extend `Symfony\Contracts\EventDispatcher\Event` (or, for kernel events, one of the `HttpKernel\Event\*` subclasses) and expose the data your listeners need.

##### 7.3.1 A Simple Domain Event

```php
// src/Event/InvoiceCreatedEvent.php
namespace App\Event;

use App\Entity\Invoice;
use Symfony\Contracts\EventDispatcher\Event;

final class InvoiceCreatedEvent extends Event
{
    public function __construct(
        public readonly Invoice $invoice,
    ) {}

    // Convenience accessor for templates or logs
    public function getInvoiceNumber(): string
    {
        return $this->invoice->number;
    }
}
```

That's the entire event. No base-class ceremony, no abstract methods. The `readonly` property and constructor promotion keep it concise.

##### 7.3.2 When You Need Mutability

Kernel events are *mutable* by design: a listener can call `$event->setResponse($newResponse)` or `$event->setController($myController)`. For your own domain events, decide whether listeners should be able to *modify* the carried data.

```php
// src/Event/InvoicePricedEvent.php
namespace App\Event;

use App\Entity\Invoice;
use Symfony\Contracts\EventDispatcher\Event;

final class InvoicePricedEvent extends Event
{
    private float $taxAmount = 0.0;

    public function __construct(
        public readonly Invoice $invoice,
    ) {}

    public function getTaxAmount(): float
    {
        return $this->taxAmount;
    }

    public function setTaxAmount(float $amount): void
    {
        $this->taxAmount = $amount;
    }
}
```

Here a `TaxCalculationListener` can compute the correct tax and write it back. If no listener needs to modify the event, keep all properties `readonly`—it signals intent and prevents accidental side-effects.

##### 7.3.3 Stopping Propagation

Most events extend `Event`, which inherits from `Event` (the base contract) and does **not** support stop-propagation. When you need a listener to prevent subsequent listeners from running, extend `StopPropagationEvent` instead:

```php
// src/Event/InvoiceSubmissionEvent.php
namespace App\Event;

use App\Entity\Invoice;
use Symfony\Component\EventDispatcher\StopPropagationEvent;

final class InvoiceSubmissionEvent extends StopPropagationEvent
{
    public function __construct(
        public readonly Invoice $invoice,
    ) {}

    public function getDispatcherName(): string
    {
        return 'invoice';  // required by StopPropagationEvent
    }
}
```

A listener can then call `$event->stopPropagation()` to short-circuit the remaining listeners. Use this sparingly; it makes the execution order matter and is harder to reason about.

---

#### 7.4 Listeners

A listener is any PHP callable that accepts the event object. Symfony supports several registration styles; the book standard is **attribute-based**.

##### 7.4.1 Invokable Listener (the Default)

When a class implements `__invoke()` and you tag it with `#[AsEventListener]`, Symfony calls that method. The event type is inferred from the parameter:

```php
// src/EventListener/InvoiceCreatedListener.php
namespace App\EventListener;

use App\Event\InvoiceCreatedEvent;
use App\Notification\EmailNotifier;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

#[AsEventListener]  // event inferred from the type-hint
final class InvoiceCreatedListener
{
    public function __construct(
        private readonly EmailNotifier $notifier,
    ) {}

    public function __invoke(InvoiceCreatedEvent $event): void
    {
        $this->notifier->send(
            to: $event->invoice->customer->email,
            template: 'invoice/confirmation.html.twig',
            context: ['invoice' => $event->invoice],
        );
    }
}
```

No YAML, no tag name to remember. The attribute lives on the class, the service is auto-configured, and the listener is wired the moment the container compiles.

##### 7.4.2 Named-Method Listener

When a class listens to multiple events, or you want an explicit event name, specify the `event` and optional `method`:

```php
namespace App\EventListener;

use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

#[AsEventListener(event: 'kernel.request', method: 'onRequest')]
#[AsEventListener(event: 'kernel.response', method: 'onResponse')]
final class RequestLoggingListener
{
    public function onRequest(): void   { /* ... */ }
    public function onResponse(): void  { /* ... */ }
}
```

You can also stack multiple `#[AsEventListener]` attributes on a single method:

```php
final class AuditListener
{
    #[AsEventListener(event: 'kernel.controller_arguments')]
    #[AsEventListener(event: 'kernel.view')]
    public function track(ControllerArgumentsEvent|ViewEvent $event): void
    {
        // handle both with one method
    }
}
```

##### 7.4.3 Closure Listeners (Programmatic)

For throwaway or test-scoped listeners you can register a closure directly:

```php
$dispatcher->addListener(
    InvoiceCreatedEvent::class,
    fn (InvoiceCreatedEvent $e) => $logger->info('Invoice created', [
        'number' => $e->getInvoiceNumber(),
    ]),
);
```

You will see this pattern in functional tests and in code that registers listeners conditionally at runtime.

##### 7.4.4 Registering via YAML (Legacy / Bundle Context)

The attribute style is preferred, but some bundles (or third-party code you don't control) still use the YAML tag:

```yaml
# config/services.yaml
services:
    App\EventListener\LegacyListener:
        tags:
            - { name: kernel.event_listener, event: kernel.response, method: onKernelResponse, priority: 10 }
```

Understand this format when reading bundle configuration, but write new code with attributes.

---

#### 7.5 Subscribers

An **event subscriber** is a class that implements `EventSubscriberInterface` and declares *all* its event bindings in a single static method. Use a subscriber when one class naturally owns several related listeners—e.g., a `SecuritySubscriber` that hooks into `kernel.request`, `kernel.exception`, and `kernel.response`.

```php
// src/EventSubscriber/TenantResolutionSubscriber.php
namespace App\EventSubscriber;

use App\Service\TenantResolver;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpKernel\Event\RequestEvent;
use Symfony\Component\HttpKernel\Event\TerminateEvent;
use Symfony\Component\HttpKernel\KernelEvents;

final class TenantResolutionSubscriber implements EventSubscriberInterface
{
    public function __construct(
        private readonly TenantResolver $resolver,
    ) {}

    public static function getSubscribedEvents(): array
    {
        return [
            // [event, method, priority]
            [KernelEvents::REQUEST,  'onRequest',  32],
            [KernelEvents::TERMINATE, 'onTerminate', 0],
        ];
    }

    public function onRequest(RequestEvent $event): void
    {
        if (!$event->isMainRequest()) {
            return;
        }

        $request = $event->getRequest();

        // Resolve tenant from subdomain, header, or path segment
        $tenant = $this->resolver->resolve($request);

        // Store on the request so downstream listeners/controllers can use it
        $request->attributes->set('_tenant', $tenant);
    }

    public function onTerminate(TerminateEvent $event): void
    {
        // Reset tenant-specific state (e.g., database connection hints)
        $this->resolver->reset();
    }
}
```

**Listener vs. Subscriber—when to use which?**

| Criterion | `#[AsEventListener]` | `EventSubscriberInterface` |
|---|---|---|
| One event, one class | ✓ | Overkill |
| Multiple events, one cohesive concern | Possible (stacked attributes) | ✓ Cleaner |
| Per-method priority on the *same* event | Requires multiple attributes | ✓ Natural with array of `[method, priority]` |
| Bundle-level encapsulation | Fine | ✓ Common in bundles |

There is no performance difference; the choice is about readability and grouping.

---

#### 7.6 Priorities and Execution Order

When multiple listeners subscribe to the same event, the dispatcher calls them in **descending priority** order (highest number first). The default priority is `0`.

```
Priority  64   TenantResolutionSubscriber::onRequest
Priority  32   RateLimitListener
Priority  10   AuthenticationListener
Priority   0   (default)
Priority  -10   AuditLogListener
Priority -256   Symfony's internal ResponseListener
```

A few rules to keep in mind:

- **Symfony's own listeners** occupy the range −256 to 256. Place your listeners outside that range (or between the values that matter to you) to avoid unexpected interactions.
- **Same priority, different listeners:** execution follows registration order, which is effectively alphabetical by service id. If order matters, set explicit priorities.
- **Subscribers with multiple methods on the same event** are ordered by the priority you specify in `getSubscribedEvents()`.

> **Tip:** Use round numbers (10, 20, 30 …) and leave gaps. If you later need a listener to run between two existing ones, you have room to insert a priority like 15 without renumbering.

##### 7.6.1 Stopping Propagation

Only events that extend `StopPropagationEvent` honour `stopPropagation()`. The most common case is `RequestEvent` (which *does* extend `StopPropagationEvent`):

```php
use Symfony\Component\HttpKernel\Event\RequestEvent;
use Symfony\Component\HttpFoundation\Response;

#[AsEventListener(event: KernelEvents::REQUEST, priority: 64)]
final class MaintenanceModeListener
{
    public function __invoke(RequestEvent $event): void
    {
        if (!$event->isMainRequest()) {
            return;
        }

        if (file_exists(\dirname(__DIR__, 2).'/var/maintenance')) {
            $event->setResponse(new Response('Site under maintenance.', 503));
            $event->stopPropagation();  // no further listeners run
        }
    }
}
```

Because `stopPropagation()` only affects listeners registered for the *same* event, it does not prevent `kernel.response` listeners from running later in the pipeline.

---

#### 7.7 Kernel Events: The Request Pipeline

The HttpKernel component dispatches a fixed sequence of events as each request flows through the framework. Understanding this sequence is the key to knowing *where* to hook in.

```
                         ┌──────────────────────────────────────────────────┐
                         │              HTTP Kernel Pipeline                │
                         └──────────────────────────────────────────────────┘

  Request arrives
       │
       ▼
  ┌─────────────────┐
  │  kernel.request │  ← RequestEvent (StopPropagationEvent)
  │  (resolve       │    Add attributes, short-circuit with a Response
  │   route, etc.)  │
  └───────┬─────────┘
          ▼
  ┌─────────────────────┐
  │  kernel.controller  │  ← ControllerEvent
  │  (controller        │    Swap the controller, inspect attributes
  │   resolved)         │
  └───────┬─────────────┘
          ▼
  ┌───────────────────────────┐
  │  kernel.controller_       │  ← ControllerArgumentsEvent
  │  arguments                │    Inspect / modify resolved arguments
  └───────┬───────────────────┘
          ▼
  ┌─────────────────┐
  │  Controller      │
  │  executes        │
  └───────┬─────────┘
          ▼
  ┌─────────────────┐
  │  kernel.view    │  ← ViewEvent  (only if controller didn't
  │  (optional)     │    return a Response)
  └───────┬─────────┘
          ▼
  ┌────────────────────┐
  │  kernel.response   │  ← ResponseEvent
  │  (Response ready)  │    Modify headers, cookies, body
  └───────┬────────────┘
          ▼
  ┌────────────────────────┐
  │  kernel.finish_request │  ← FinishRequestEvent
  │  (reset state)         │    Clean up sub-request state
  └───────┬────────────────┘
          ▼
  Response sent to client
          │
          ▼
  ┌────────────────────┐
  │  kernel.terminate  │  ← TerminateEvent
  │  (post-response)   │    Email, cleanup, metrics flush
  └────────────────────┘

  At any point, if an exception is thrown:
  ┌────────────────────┐
  │  kernel.exception  │  ← ExceptionEvent (StopPropagationEvent)
  │  (error handling)  │    Recover, modify the error response
  └────────────────────┘
```

Every kernel event object inherits from `KernelEvent`, which gives you:

- `getRequest(): Request` — the current `Request`.
- `getKernel(): KernelInterface` — the kernel instance.
- `isMainRequest(): bool` — `true` for the top-level request, `false` for sub-requests.
- `getRequestType(): int` — `HttpKernelInterface::MAIN_REQUEST` or `::SUB_REQUEST`.

> **Always guard with `isMainRequest()`** in listeners that should run only once per HTTP request. Sub-requests (created by fragments or `forward()`) re-enter the pipeline and will trigger your listener again.

##### 7.7.1 `kernel.request` (RequestEvent)

Fired *before* routing is fully resolved (in practice, after the router has matched the route). The event extends `StopPropagationEvent`, so a listener can short-circuit the entire pipeline by setting a response.

```php
#[AsEventListener(event: KernelEvents::REQUEST, priority: 50)]
final class RequestAttributeListener
{
    public function __invoke(RequestEvent $event): void
    {
        if (!$event->isMainRequest()) {
            return;
        }

        $request = $event->getRequest();

        // Attach a request-scoped correlation ID for distributed tracing
        $request->attributes->set(
            '_correlation_id',
            $request->headers->get('X-Correlation-Id', (string) Uuid::v7()),
        );
    }
}
```

##### 7.7.2 `kernel.controller` (ControllerEvent)

The controller callable has been resolved. You can **replace** it, inspect its PHP attributes, or initialise resources it will need.

```php
#[AsEventListener(event: KernelEvents::CONTROLLER)]
final class ControllerInspectionListener
{
    public function __invoke(ControllerEvent $event): void
    {
        // $event->getController() returns the callable
        // For an attribute route: [Controller::class, 'method'] or a Closure

        $controller = $event->getController();

        if (is_array($controller)) {
            [$class, $method] = $controller;
            // inspect $class, $method ...
        }
    }
}
```

##### 7.7.3 `kernel.controller_arguments` (ControllerArgumentsEvent)

Fired just before the controller is *called*. Value resolvers have already populated the arguments; you can inspect or modify them.

```php
#[AsEventListener(event: KernelEvents::CONTROLLER_ARGUMENTS)]
final class ArgumentAuditListener
{
    public function __construct(private readonly LoggerInterface $logger) {}

    public function __invoke(ControllerArgumentsEvent $event): void
    {
        $args = $event->getArguments();

        $this->logger->debug('Controller arguments', [
            'route'  => $event->getRequest()->attributes->get('_route'),
            'args'   => $args,
        ]);
    }
}
```

##### 7.7.4 `kernel.view` (ViewEvent)

Dispatched **only** when the controller returned something that is *not* a `Response` (e.g., a string, an array, or an object). The default `ControllerListener` in TwigBundle converts the return value into a rendered `Response`. You can intercept and transform it:

```php
#[AsEventListener(event: KernelEvents::VIEW, priority: -128)]
final class JsonSerializationListener
{
    public function __invoke(ViewEvent $event): void
    {
        $result = $event->getControllerResult();

        // If the controller returned a plain array for a JSON route,
        // convert it to a JsonResponse here
        if (is_array($result) && $event->getRequest()->isXmlHttpRequest()) {
            $event->setResponse(new JsonResponse($result));
        }
    }
}
```

##### 7.7.5 `kernel.response` (ResponseEvent)

The `Response` object is final and about to be sent. This is the canonical place to add headers, cookies, or modify the body.

```php
#[AsEventListener(event: KernelEvents::RESPONSE)]
final class SecurityHeadersListener
{
    public function __invoke(ResponseEvent $event): void
    {
        $response = $event->getResponse();

        $response->headers->set('X-Content-Type-Options', 'nosniff');
        $response->headers->set('X-Frame-Options', 'DENY');
        $response->headers->set('Referrer-Policy', 'strict-origin-when-cross-origin');
    }
}
```

##### 7.7.6 `kernel.finish_request` (FinishRequestEvent)

Fired after the response is built but *before* it is sent. Ideal for resetting per-request state that must not leak into the next request (in long-lived processes like FrankenPHP or RoadRunner):

```php
#[AsEventListener(event: KernelEvents::FINISH_REQUEST)]
final class LocaleResetListener
{
    public function __invoke(FinishRequestEvent $event): void
    {
        $parent = $event->getRequestStack()->getParentRequest();
        if (null === $parent) {
            return;
        }

        // Restore the locale to the parent request's locale
        $this->translator->setLocale($parent->getDefaultLocale());
    }
}
```

##### 7.7.7 `kernel.terminate` (TerminateEvent)

Fired *after* the response has been flushed to the client. Perfect for slow, non-blocking work: sending emails, flushing metrics, writing audit logs.

```php
#[AsEventListener(event: KernelEvents::TERMINATE)]
final class AuditLogSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [KernelEvents::TERMINATE => ['onTerminate', 0]];
    }

    public function __construct(
        private readonly AuditLogRepository $repo,
        private readonly LoggerInterface $logger,
    ) {}

    public function onTerminate(TerminateEvent $event): void
    {
        $request = $event->getRequest();

        // Skip sub-requests and static files
        if (!$request->attributes->has('_route')) {
            return;
        }

        $this->repo->save(new AuditLogEntry(
            method: $request->getMethod(),
            path:   $request->getPathInfo(),
            ip:     $request->getClientIp(),
            status: $event->getResponse()->getStatusCode(),
            // ...
        ));
    }
}
```

##### 7.7.8 `kernel.exception` (ExceptionEvent)

Fired the moment *any* uncaught throwable propagates during the request. You can:

- **Recover** by setting a custom `Response` (the error is swallowed).
- **Replace** the throwable with a friendlier one.
- **Decorate** the error response (e.g., add a `X-Request-Id` header to the 500 page).

```php
#[AsEventListener(event: KernelEvents::EXCEPTION, priority: 256)]
final class ExceptionRecoveryListener
{
    public function __construct(
        private readonly LoggerInterface $logger,
        private readonly TranslatorInterface $translator,
    ) {}

    public function __invoke(ExceptionEvent $event): void
    {
        $throwable = $event->getThrowable();
        $request   = $event->getRequest();

        $this->logger->error('Unhandled exception', [
            'exception' => $throwable::class,
            'message'   => $throwable->getMessage(),
            'route'     => $request->attributes->get('_route'),
        ]);

        // For API routes, return a structured JSON error
        if (str_starts_with($request->getPathInfo(), '/api/')) {
            $event->setResponse(new JsonResponse([
                'error'   => 'server_error',
                'message' => $this->translator->trans('error.generic_server'),
            ], 500));
        }
        // Otherwise, let Symfony's built-in error controller handle it
    }
}
```

> **Note:** The built-in Twig `ErrorListener` (priority 256) renders the error template. If you want to *augment* rather than *replace* it, use a lower priority (e.g., 128) so your listener runs after the error response is already built, and simply add headers.

---

#### 7.8 Cross-Cutting Concerns in the Invoicing App

Now we apply the patterns to the running multi-tenant SaaS project. Each concern is a small, focused listener or subscriber that could be moved to a bundle tomorrow without touching domain code.

##### 7.8.1 Tenant Resolution

The invoicing app is multi-tenant: each customer gets a subdomain (`acme.invoicing.example.com`). The tenant must be resolved *before* any controller runs, because Doctrine connections, cache keys, and permission checks all depend on it.

```php
// src/Service/TenantResolver.php
namespace App\Service;

use App\Repository\TenantRepository;
use Symfony\Component\HttpFoundation\Request;

final class TenantResolver
{
    public function __construct(
        private readonly TenantRepository $tenants,
    ) {}

    public function resolve(Request $request): ?Tenant
    {
        $host = $request->getHost();  // e.g. "acme.invoicing.example.com"

        // Strip the base domain
        $subdomain = strtok($host, '.') ?: null;

        if (null === $subdomain) {
            return null;  // base domain → admin / marketing pages
        }

        return $this->tenants->findOneBySlug($subdomain);
    }

    public function reset(): void
    {
        // Clear any tenant-specific doctrine connection hints
    }
}
```

```php
// src/EventSubscriber/TenantResolutionSubscriber.php
namespace App\EventSubscriber;

use App\Service\TenantResolver;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\RequestEvent;
use Symfony\Component\HttpKernel\Event\TerminateEvent;
use Symfony\Component\HttpKernel\Exception\AccessDeniedException;
use Symfony\Component\HttpKernel\KernelEvents;

final class TenantResolutionSubscriber implements EventSubscriberInterface
{
    public function __construct(
        private readonly TenantResolver $resolver,
    ) {}

    public static function getSubscribedEvents(): array
    {
        return [
            [KernelEvents::REQUEST,  'onRequest',  64],   // early!
            [KernelEvents::TERMINATE, 'onTerminate', 0],
        ];
    }

    public function onRequest(RequestEvent $event): void
    {
        if (!$event->isMainRequest()) {
            return;
        }

        $tenant = $this->resolver->resolve($event->getRequest());

        if (null === $tenant) {
            // Not a tenant subdomain—let the request through
            // (marketing pages, health checks, etc.)
            return;
        }

        // Make the tenant available to controllers via request attributes
        $event->getRequest()->attributes->set('_tenant', $tenant);
    }

    public function onTerminate(TerminateEvent $event): void
    {
        $this->resolver->reset();
    }
}
```

A controller can then type-hint the tenant through a value resolver:

```php
// src/Controller/InvoiceController.php
namespace App\Controller;

use App\Entity\Invoice;
use App\Entity\Tenant;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class InvoiceController extends AbstractController
{
    #[Route('/invoices', name: 'app_invoice_index')]
    public function index(Request $request, Tenant $tenant): Response
    {
        // $tenant is populated by a value resolver reading '_tenant'
        $invoices = $tenant->getInvoices();

        return $this->render('invoice/index.html.twig', [
            'invoices' => $invoices,
            'tenant'   => $tenant,
        ]);
    }
}
```

##### 7.8.2 Rate Limiting

The invoicing API must limit how many invoices a tenant can create per hour. A `kernel.request` listener with priority 32 (after tenant resolution, before authentication) is a natural fit:

```php
// src/EventListener/RateLimitListener.php
namespace App\EventListener;

use App\Entity\Tenant;
use App\Service\RateLimiter;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Event\RequestEvent;
use Symfony\Component\HttpKernel\KernelEvents;

#[AsEventListener(event: KernelEvents::REQUEST, priority: 32)]
final class RateLimitListener
{
    public function __construct(
        private readonly RateLimiter $rateLimiter,
    ) {}

    public function __invoke(RequestEvent $event): void
    {
        if (!$event->isMainRequest()) {
            return;
        }

        $request = $event->getRequest();
        $tenant  = $request->attributes->get('_tenant');

        if (!$tenant instanceof Tenant) {
            return;
        }

        // Only limit write operations
        if (!in_array($request->getMethod(), ['POST', 'PUT', 'PATCH', 'DELETE'], true)) {
            return;
        }

        $limit = $tenant->getRateLimitPerHour();  // e.g. 500
        $result = $this->rateLimiter->consume("tenant:{tenant->id}:writes", $limit);

        if (!$result->isAccepted()) {
            $response = new Response('Rate limit exceeded.', 429);
            $response->headers->set(
                'Retry-After',
                (string) ceil($result->getHeaders()->get('Retry-After', 60)),
            );
            $event->setResponse($response);
            $event->stopPropagation();
        }
    }
}
```

##### 7.8.3 Request Logging with Correlation ID

A single subscriber covers both `kernel.request` (start timer, log request) and `kernel.terminate` (log duration, flush):

```php
// src/EventSubscriber/RequestLoggingSubscriber.php
namespace App\EventSubscriber;

use Psr\Log\LoggerInterface;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\RequestEvent;
use Symfony\Component\HttpKernel\Event\TerminateEvent;
use Symfony\Component\HttpKernel\KernelEvents;

final class RequestLoggingSubscriber implements EventSubscriberInterface
{
    /** @var array<string, float> request-id → start timestamp */
    private array $timers = [];

    public static function getSubscribedEvents(): array
    {
        return [
            [KernelEvents::REQUEST,  'onRequest',  50],
            [KernelEvents::TERMINATE, 'onTerminate', 0],
        ];
    }

    public function __construct(
        private readonly LoggerInterface $logger,
    ) {}

    public function onRequest(RequestEvent $event): void
    {
        if (!$event->isMainRequest()) {
            return;
        }

        $request = $event->getRequest();
        $id      = $request->attributes->get('_correlation_id', bin2hex(random_bytes(8)));
        $request->attributes->set('_correlation_id', $id);

        $this->timers[$id] = microtime(true);

        $this->logger->info('Request started', [
            'correlation_id' => $id,
            'method'         => $request->getMethod(),
            'path'           => $request->getPathInfo(),
            'tenant'         => $request->attributes->get('_tenant')?->slug ?? 'public',
        ]);
    }

    public function onTerminate(TerminateEvent $event): void
    {
        $request = $event->getRequest();
        $id      = $request->attributes->get('_correlation_id');

        if (null === $id || !isset($this->timers[$id])) {
            return;
        }

        $duration = microtime(true) - $this->timers[$id];
        unset($this->timers[$id]);

        $this->logger->info('Request finished', [
            'correlation_id' => $id,
            'status'         => $event->getResponse()->getStatusCode(),
            'duration_ms'    => round($duration * 1000, 2),
        ]);
    }
}
```

> **Caveat:** Storing state in the subscriber instance (`$this->timers`) works in a traditional FPM setup (one request per process) but breaks under FrankenPHP or RoadRunner, where the worker handles many requests sequentially. In those environments, use the `RequestStack` or a per-request service to carry the start timestamp.

---

#### 7.9 Middleware-Style Patterns with Events

##### 7.9.1 The Pipeline Mental Model

Kernel events *are* a middleware pipeline. Each listener is a layer in an onion:

```
  ┌────────────────────────────────────────────────────────┐
  │  Request Logging (priority 50)                         │
  │  ┌────────────────────────────────────────────────┐    │
  │  │  Tenant Resolution (priority 64)               │    │
  │  │  ┌────────────────────────────────────────┐    │    │
  │  │  │  Rate Limiting (priority 32)           │    │    │
  │  │  │  ┌────────────────────────────────┐    │    │    │
  │  │  │  │  Authentication (priority 10)  │    │    │    │
  │  │  │  │  ┌──────────────────────────┐  │    │    │    │
  │  │  │  │  │  Controller             │  │    │    │    │
  │  │  │  │  └──────────────────────────┘  │    │    │    │
  │  │  │  └────────────────────────────────┘    │    │    │
  │  │  └────────────────────────────────────────┘    │    │
  │  └────────────────────────────────────────────────┘    │
  │  Response headers, security headers (priority 0)       │
  └────────────────────────────────────────────────────────┘
```

Higher-priority listeners wrap lower-priority ones. A listener that calls `stopPropagation()` peels off all remaining layers.

##### 7.9.2 When to Use Events vs. Decorators vs. Middleware

| Concern | Best tool | Why |
|---|---|---|
| Add HTTP headers to *every* response | Event listener (`kernel.response`) | One place, no controller changes |
| Wrap a specific service with cross-cutting logic (caching, logging) | **Decorator** (DI tag) | Tightly coupled to one service's interface |
| Run logic *between* routing and controller | Event listener (`kernel.request`) | Pipeline ordering, can short-circuit |
| Replace the controller under certain conditions | Event listener (`kernel.controller`) | Access to the resolved callable |
| Wrap the entire HTTP kernel (e.g., add a WAF) | Custom **HttpKernel middleware** or a `kernel.request` listener | Kernel-level, before routing |
| React to a domain event (invoice created) | Custom event + listener | Loose coupling, extensible |

The rule of thumb: **use events for pipeline concerns and domain notifications; use decorators for service-level concerns.** Mixing them is normal—e.g., a decorator on the `Cache` service *and* a `kernel.response` listener that sets `Cache-Control` headers.

##### 7.9.3 The "Middleware" Tag (Framework Internals)

Symfony's `HttpKernel` component internally uses a `HandlerStack` (a PSR-15-style middleware stack) for *some* layers (e.g., the profiler, the router). You don't typically add custom middleware at this level; instead, you use kernel events. However, if you need to wrap the *entire* kernel (for a reverse-proxy-style WAF or a global request-id header that must be present even on 404s before routing), you can decorate the `http_kernel` service:

```yaml
# config/services.yaml
services:
    App\Middleware\GlobalRequestIdMiddleware:
        decorates: http_kernel
        tags: [kernel.event_subscriber]  # or wrap via a compiler pass
```

In practice, a high-priority `kernel.request` listener (priority 256) covers 99% of these needs without touching the kernel service.

---

#### 7.10 Debugging and Testing Events

##### 7.10.1 Inspecting Listeners from the Console

The `debug:event-dispatcher` command is your first stop when something doesn't fire (or fires twice):

```bash
# List all listeners for a specific event
$ php bin/console debug:event-dispatcher kernel.response

# Show the listener graph (who is registered, in what order)
$ php bin/console debug:event-dispatcher

# Check whether a specific listener class is registered
$ php bin/console debug:event-dispatcher kernel.request --format=json
```

The output shows each listener's service id, method, priority, and connected channels (if you use them).

##### 7.10.2 Unit-Testing a Listener

Listeners are plain classes—test them directly with a mocked event:

```php
// tests/Unit/EventListener/RateLimitListenerTest.php
namespace App\Tests\Unit\EventListener;

use App\Entity\Tenant;
use App\EventListener\RateLimitListener;
use App\Service\RateLimiter;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Event\RequestEvent;
use Symfony\Component\HttpKernel\HttpKernelInterface;
use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\TestCase;

final class RateLimitListenerTest extends TestCase
{
    #[Test]
    public function itRejectsWhenRateLimitExceeded(): void
    {
        $request = Request::create('/api/invoices', 'POST');
        $tenant  = new Tenant();
        $tenant->id   = 1;
        $tenant->slug = 'acme';
        $request->attributes->set('_tenant', $tenant);

        $kernel    = $this->createMock(HttpKernelInterface::class);
        $event     = new RequestEvent($kernel, $request, HttpKernelInterface::MAIN_REQUEST);

        $limiter   = $this->createMock(RateLimiter::class);
        $limiter
            ->method('consume')
            ->willReturn(new \Symfony\Component\RateLimiter\Limit(500));  // exhausted

        $listener  = new RateLimitListener($limiter);
        $listener($event);

        $this->assertSame(Response::HTTP_TOO_MANY_REQUESTS, $event->getResponse()->getStatusCode());
        $this->assertTrue($event->isPropagationStopped());
    }
}
```

##### 7.10.3 Functional-Testing the Full Pipeline

To verify that listeners interact correctly end-to-end, use `WebTestCase`:

```php
// tests/Functional/TenantResolutionTest.php
namespace App\Tests\Functional;

use Symfony\Bundle\FrameworkBundle\KernelBrowser;
use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

final class TenantResolutionTest extends WebTestCase
{
    public function testTenantIsResolvedFromSubdomain(): void
    {
        self::bootKernel();

        /** @var KernelBrowser $client */
        $client = static::createClient();

        $crawler = $client->request('GET', 'http://acme.invoicing.example.com/invoices');

        self::assertResponseIsSuccessful();
        self::assertStringContainsString('Acme Corp', $client->getResponse()->getContent());
    }
}
```

---

#### 7.11 Channels (Advanced)

The `EventDispatcher` can partition events into **named channels**, so that a listener registered on channel `billing` never sees events dispatched on channel `notifications`. This is useful in large applications or bundles that want to avoid name collisions.

```php
// Dispatch on a channel
$dispatcher->dispatch(new InvoiceCreatedEvent($invoice), null, 'billing');

// Listen on a channel
#[AsEventListener(event: InvoiceCreatedEvent::class, channel: 'billing')]
final class BillingInvoiceListener { /* ... */ }
```

The default channel is `''` (empty string), so existing code is unaffected. Most applications will never need channels; they are primarily a concern for bundles that dispatch many events and want to avoid cross-interference.

---

#### 7.12 Common Pitfalls

| Pitfall | Symptom | Fix |
|---|---|---|
| Forgetting `isMainRequest()` guard | Listener runs 2× (main + sub-request) | Always guard in `kernel.request`, `kernel.response`, etc. |
| Storing request state in a singleton listener | Data leaks between requests under FrankenPHP | Use `RequestStack` or a `request`-scoped service |
| Relying on registration order at the same priority | Listener order changes after renaming a service | Set explicit priorities |
| Calling `stopPropagation()` on a plain `Event` | No effect (silently ignored) | Extend `StopPropagationEvent` instead |
| Heavy work in `kernel.request` | Slow initial response, no way to show a loading state | Move heavy work to `kernel.terminate` or a Messenger message |
| Catching all exceptions in a `kernel.exception` listener and returning 200 | Hides real errors from monitoring | Log the throwable; return a 500 unless the error is expected (e.g., 404) |

---

#### Exercises

**1. Correlation ID Header**
Add a listener on `kernel.response` (priority 10) that copies the `_correlation_id` request attribute into a response header `X-Correlation-Id`. Verify with `curl -v` that the header is present on both 200 and 500 responses.

**2. Slow-Request Logger**
Create a subscriber that records the start time in `kernel.request` and logs a warning in `kernel.terminate` whenever a request takes longer than 500 ms. Include the route name, HTTP method, and tenant slug in the log context.

**3. Maintenance Mode**
Write a listener on `kernel.request` (priority 256) that checks for the existence of a file `var/maintenance`. If the file exists *and* the request is not to the `/admin` route, respond with a 503 and a friendly template. Add a console command `app:enable-maintenance` that creates the file and `app:disable-maintenance` that removes it.

**4. Invoice Domain Events**
Refactor `InvoiceService` to dispatch `InvoiceCreatedEvent` and `InvoiceStatusChangedEvent` (a `StopPropagationEvent` with `oldStatus` and `newStatus` properties). Create three listeners:
   - `InvoiceEmailListener` – sends a notification email.
   - `InvoiceWebhookListener` – calls a customer-configured webhook URL.
   - `InvoiceAuditListener` – appends to an audit trail table.

   Write a unit test for each listener and a functional test that asserts all three side-effects occur when an invoice is created via `POST /api/invoices`.

**5. Channel Isolation**
Move the billing-related listeners from Exercise 4 onto a `billing` channel. Dispatch the events on that channel. Verify with `php bin/console debug:event-dispatcher` that the listeners appear under the `billing` channel and do not respond to events on the default channel.

**6. Debugging Challenge**
Add two listeners on `kernel.response`: one at priority 10 that adds header `X-First: yes`, and one at priority −10 that *removes* that header. Use `debug:event-dispatcher` to confirm the order, then change the priorities and verify the header's presence/absence changes accordingly.

---

#### Key Takeaways

- **Events decouple.** Domain code announces *what happened*; listeners decide *what to do*. Neither side knows about the other.
- **Class-name events** are the modern default. Extend `Event` for read-only data, `StopPropagationEvent` when a listener may short-circuit.
- **`#[AsEventListener]`** is the registration mechanism of choice. Use `EventSubscriberInterface` when one class owns multiple event bindings.
- **Kernel events** form a fixed pipeline: `request → controller → controller_arguments → view → response → finish_request → terminate`, with `exception` handling cross-cutting errors.
- **Guard with `isMainRequest()`** and set **explicit priorities** to keep execution order predictable.
- **Use events for pipeline and domain concerns; decorators for service-level concerns.** Knowing which tool fits which job is the mark of a mature Symfony architect.

*In the next chapter we step outside the request cycle and look at how Symfony itself is assembled: bundles, extension points, and compiler passes—the machinery that lets a few hundred independent packages compose into a single application.*

### Chapter 8. Bundles and Extension Points

> *By the end of this chapter you will be able to create, configure, and extend a custom Symfony bundle, register services through extension classes, modify the service container at compile time with compiler passes, and make informed decisions about when a bundle is the right packaging strategy versus a plain tagged service.*

---

#### 8.1 What Is a Bundle, Really?

You have already met bundles without naming them. Every `framework.yaml` entry that activates a component—`doctrine:`, `twig:`, `security:`—is a bundle declaring itself in your application. The `AppBundle` in a skeleton project (removed in Symfony 7.x in favor of flat `config/` directories) was the last vestige of the idea that *your* code needed a bundle wrapper to participate in the framework.

A **bundle** is a self-contained, reusable unit of functionality that:

1. Registers one or more **services** in the DI container.
2. Declares a **configuration schema** (a tree of options the consumer can tune).
3. Optionally ships **templates, translations, public assets, or routes**.
4. Exposes **extension points**—hooks that let the host application customize, extend, or replace its behavior.

Think of a bundle as a *plugin contract*. The framework (or your host application) loads it, calls its `Extension::load()`, and the bundle wires itself into the kernel. The framework never needs to know what the bundle does internally; it only needs the contract: *"Here is a config array; here is a ContainerBuilder; make it work."*

##### Why Bundles Matter in Symfony 7.x

Modern Symfony has shifted toward **attribute-driven, zero-config** setups. You no longer *need* a bundle to define a service (`config/services.yaml` handles that). So the question becomes: when *is* a bundle the right tool?

The short answer: when you are building **reusable, multi-app functionality** that needs its own configuration namespace, its own set of services, and a clean upgrade path. We will formalize this decision in §8.6.

---

#### 8.2 Anatomy of a Bundle

A bundle is a PHP namespace directory containing a small number of files. Let us dissect a minimal but complete bundle called `InvoiceBundle`, which we will build incrementally in this chapter.

```
src/
  InvoiceBundle/
    DependencyInjection/
      Configuration.php
      InvoiceExtension.php
      Compiler/
        InvoicePass.php
    Service/
      InvoiceGenerator.php
      InvoiceRepository.php
    EventSubscriber/
      InvoiceEventsSubscriber.php
    Resources/
      config/
        services.xml          # optional; we use attributes instead
    InvoiceBundle.php
    composer.json
```

##### 8.2.1 The Bundle Class

The bundle class is the entry point. In Symfony 7.x it extends `Symfony\Component\HttpKernel\Bundle\Bundle`:

```php
// src/InvoiceBundle/InvoiceBundle.php

namespace App\InvoiceBundle;

use Symfony\Component\HttpKernel\Bundle\Bundle;

class InvoiceBundle extends Bundle
{
    public function getPath(): string
    {
        return \dirname(__DIR__);
    }
}
```

> **Why override `getPath()`?** By default the framework derives the path from the class name (`App\InvoiceBundle` → `src/InvoiceBundle`). If your bundle lives outside the conventional `src/` layout (e.g., a monorepo package), you must provide an explicit path. In a standard app skeleton, the override is unnecessary—delete it.

The parent `Bundle` class already implements `getContainerExtension()`, which conventionally looks for a class named `...\DependencyInjection\{BundleNameWithoutBundle}Extension`. So for `InvoiceBundle`, it expects `InvoiceExtension`. If you follow that naming, you do not need to override the method.

##### 8.2.2 The Extension Class

The **Extension** is where the bundle *does its work*. It receives the raw configuration array (parsed from YAML, XML, PHP, or attributes) and a `ContainerBuilder`, and its job is to register services, parameters, and tags.

We will write it in full in §8.3. For now, note the contract:

```php
namespace App\InvoiceBundle\DependencyInjection;

use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\HttpKernel\DependencyInjection\Extension;

class InvoiceExtension extends Extension
{
    public function load(array $configs, ContainerBuilder $container): void
    {
        // 1. Merge configuration
        // 2. Register services
        // 3. Set parameters
    }
}
```

##### 8.2.3 The Configuration Tree

The `Configuration` class defines the *shape* of the options a consumer can pass. It extends `Symfony\Component\Config\Definition\ConfigurationInterface` (via `TreeBuilder`):

```php
namespace App\InvoiceBundle\DependencyInjection;

use Symfony\Component\Config\Definition\Builder\TreeBuilder;
use Symfony\Component\Config\Definition\ConfigurationInterface;

class Configuration implements ConfigurationInterface
{
    public function getConfigTreeBuilder(): TreeBuilder
    {
        $treeBuilder = new TreeBuilder('invoice');
        $rootNode = $treeBuilder->getRootNode();

        $rootNode
            ->children()
                ->scalarNode('currency')
                    ->defaultValue('USD')
                    ->info('ISO 4217 currency code used for new invoices.')
                ->end()
                ->booleanNode('auto_numbering')
                    ->defaultValue(true)
                    ->info('Assign sequential invoice numbers automatically.')
                ->end()
                ->arrayNode('tax_rates')
                    ->useAttributeAsKey('country')
                    ->arrayPrototype()
                        ->scalarNode('rate')->isRequired()->end()
                    ->end()
                    ->defaultValue(['US' => ['rate' => 0.0], 'DE' => ['rate' => 0.19]])
                ->end()
            ->end();

        return $treeBuilder;
    }
}
```

This is the same mechanism you used in Chapter 6 for `framework.yaml` or custom app configuration. The difference: this tree is *scoped to the bundle's namespace* (`invoice:` in `config/packages/invoice.yaml`).

##### 8.2.4 Service Definitions

In a bundle you can define services in several ways:

| Method | When to use |
|--------|------------|
| `services.yaml` / `services.xml` in `Resources/config/` | Static, non-configurable services |
| Programmatic registration in `Extension::load()` | Services that depend on config values or conditional logic |
| Attribute-based autoconfiguration in the host app | Quick, local services (no bundle needed) |

For a *reusable* bundle, programmatic registration in the Extension gives you the most control, because you can read the merged configuration and build service definitions dynamically.

---

#### 8.3 Building the Invoice Bundle, Step by Step

Let us create the bundle in our running project (the multi-tenant SaaS invoicing app). Even though we are still in Part II, we will sketch the bundle now and flesh it out in Parts III–V.

##### Step 1: Register the Bundle in `config/bundles.php`

```php
// config/bundles.php
return [
    // ...
    App\InvoiceBundle\InvoiceBundle::class => ['all' => true],
];
```

Symfony will now instantiate `InvoiceBundle`, call `getContainerExtension()`, find `InvoiceExtension`, and call `load()`.

##### Step 2: Declare the Configuration

We already saw the `Configuration` class above. Place it at `src/InvoiceBundle/DependencyInjection/Configuration.php`.

##### Step 3: Write the Extension

```php
// src/InvoiceBundle/DependencyInjection/InvoiceExtension.php

namespace App\InvoiceBundle\DependencyInjection;

use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Definition;
use Symfony\Component\DependencyInjection\Loader\YamlFileLoader;
use Symfony\Component\DependencyInjection\Reference;
use Symfony\Component\Config\FileLocator;
use Symfony\Component\HttpKernel\DependencyInjection\Extension;
use App\InvoiceBundle\Service\InvoiceGenerator;
use App\InvoiceBundle\Service\InvoiceRepository;
use App\InvoiceBundle\EventSubscriber\InvoiceEventsSubscriber;

class InvoiceExtension extends Extension
{
    public function load(array $configs, ContainerBuilder $container): void
    {
        $configuration = new Configuration();
        $config = $this->processConfiguration($configuration, $configs);

        // Store merged config as parameters for later use
        $container->setParameter('invoice.currency', $config['currency']);
        $container->setParameter('invoice.auto_numbering', $config['auto_numbering']);
        $container->setParameter('invoice.tax_rates', $config['tax_rates']);

        // Load static service definitions (subscribers, etc.)
        $loader = new YamlFileLoader(
            $container,
            new FileLocator(\dirname(__DIR__).'/Resources/config')
        );
        $loader->load('services.yaml');

        // Register config-dependent services programmatically
        $generatorDef = new Definition(InvoiceGenerator::class);
        $generatorDef->setArguments([
            new Reference('invoice.repository'),
            $config['currency'],
            $config['auto_numbering'],
            $config['tax_rates'],
        ]);
        $generatorDef->addTag('invoice.generator');
        $container->setDefinition('invoice.generator', $generatorDef);
    }

    public function getAlias(): string
    {
        return 'invoice';
    }
}
```

##### Step 4: Static Services File

```yaml
# src/InvoiceBundle/Resources/config/services.yaml
services:
    invoice.repository:
        class: App\InvoiceBundle\Service\InvoiceRepository
        autowire: true
        autoconfigure: true
        public: false

    invoice.events_subscriber:
        class: App\InvoiceBundle\EventSubscriber\InvoiceEventsSubscriber
        tags:
            - { name: kernel.event_subscriber }
```

> **Convention:** Bundle service IDs are prefixed with the bundle alias (`invoice.`) to avoid collisions.

##### Step 5: Consume the Bundle

In your application's `config/packages/invoice.yaml`:

```yaml
invoice:
    currency: EUR
    auto_numbering: true
    tax_rates:
        DE: { rate: 0.19 }
        FR: { rate: 0.20 }
        US: { rate: 0.08 }
```

Done. The container now has `invoice.generator`, `invoice.repository`, and `invoice.events_subscriber` wired in. You can inject `InvoiceGenerator` into any controller or service.

---

#### 8.4 Extension Classes: Deep Dive

##### 8.4.1 The `load()` Lifecycle

When the kernel boots, the bundle's extension is invoked **once** during container compilation. The sequence is:

1. The framework collects all bundle extensions.
2. Each extension's `getConfiguration()` returns the `Configuration` tree.
3. Config values from *every* `config/packages/invoice.yaml` (across environments) are merged.
4. `processConfiguration()` validates and normalizes the merged array against the tree.
5. `load()` receives the final, validated array.

Because this happens at **compile time** (cached in `var/cache/`), you cannot use runtime data (e.g., `$_SERVER['HTTP_HOST']`) inside `load()`. If you need runtime behavior, use a service or a compiler pass that reads a parameter set at runtime.

##### 8.4.2 Conditional Service Registration

Extensions are the right place to *conditionally* register services:

```php
public function load(array $configs, ContainerBuilder $container): void
{
    $config = $this->processConfiguration(new Configuration(), $configs);

    if ($config['enable_pdf_export']) {
        $pdfDef = new Definition(\App\InvoiceBundle\Service\PdfExporter::class);
        $pdfDef->setAutowired(true);
        $pdfDef->setAutoconfigured(true);
        $container->setDefinition('invoice.pdf_exporter', $pdfDef);
    }
}
```

This is more robust than `if (class_exists(...))` checks in a `services.yaml`, because it is driven by *explicit configuration* rather than class discovery.

##### 8.4.3 Overriding Bundle Services

A consumer application can *override* any private service defined by a bundle. Because bundle services are private by default, the host app's `config/services.yaml` entry wins:

```yaml
# config/services.yaml (host app)
services:
    invoice.generator:
        class: App\InvoiceBundle\Service\CustomInvoiceGenerator  # replaces the default
```

> **Caution:** This is a *last resort*. If you need to change behavior, prefer the extension's configuration or a compiler pass. Overriding by ID is fragile across bundle upgrades.

---

#### 8.5 Compiler Passes

A **compiler pass** is a callback that runs *after* all services (from all bundles) have been registered but *before* the container is compiled into the final optimized service locator. Compiler passes let you:

- **Tag** services that match a pattern.
- **Remove** or **replace** service definitions.
- **Rewire** dependencies (e.g., swap a concrete class for an interface alias).
- **Validate** the container graph (fail fast on misconfiguration).

##### 8.5.1 Writing a Compiler Pass

```php
// src/InvoiceBundle/DependencyInjection/Compiler/InvoicePass.php

namespace App\InvoiceBundle\DependencyInjection\Compiler;

use Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface;
use Symfony\Component\DependencyInjection\ContainerBuilder;

class InvoicePass implements CompilerPassInterface
{
    public function process(ContainerBuilder $container): void
    {
        // Auto-tag all services implementing InvoiceGeneratorInterface
        if (!$container->hasDefinition('invoice.generator_registry')) {
            return;
        }

        $registryDef = $container->getDefinition('invoice.generator_registry');

        foreach ($container->findTaggedServiceIds('invoice.generator') as $id => $tags) {
            $registryDef->addMethodCall('register', [
                new \Symfony\Component\DependencyInjection\Reference($id, \Symfony\Component\DependencyInjection\Reference::IGNORE_ON_INVALID_REFERENCE),
            ]);
        }
    }
}
```

##### 8.5.2 Registering the Pass

Compiler passes must be registered in the **Extension**, not in the Bundle class (this was changed in Symfony 4.2+ to decouple extension from bundle lifecycle):

```php
// In InvoiceExtension:

use Symfony\Component\DependencyInjection\Extension\Extension as BaseExtension;
use App\InvoiceBundle\DependencyInjection\Compiler\InvoicePass;

class InvoiceExtension extends BaseExtension
{
    public function load(array $configs, ContainerBuilder $container): void
    {
        // ... service registration as before ...

        // Register the compiler pass
        $container->addCompilerPass(new InvoicePass());
    }
}
```

> **Note:** In Symfony 7.x, `addCompilerPass()` on `ContainerBuilder` is the canonical way. The old `Bundle::build()` method (which accepted a `ContainerBuilder` directly) is deprecated.

##### 8.5.3 Pass Ordering

Multiple compiler passes from multiple bundles can interact. Symfony provides a `PassConfig` with three types:

| Type | When it runs | Use case |
|------|-------------|----------|
| `TYPE_REMOVE` | Early, before service resolution | Remove unused services, validate configuration |
| `TYPE_BEFORE_OPTIMIZATION` | After all services are registered, before optimization | Tagging, rewiring, building registries |
| `TYPE_AFTER_OPTIMIZATION` | After inlining and optimization | Final adjustments (rarely needed) |

You can set the priority and type:

```php
$container->addCompilerPass(new InvoicePass(), PassConfig::TYPE_BEFORE_OPTIMIZATION, 10);
```

##### 8.5.4 A Practical Example: Strategy Pattern via Tags

Suppose your invoicing app supports multiple tax calculation strategies (VAT, GST, flat). Rather than a single hardcoded calculator, you use a **strategy pattern** driven by tags:

```yaml
# src/InvoiceBundle/Resources/config/services.yaml
services:
    invoice.tax.vat:
        class: App\InvoiceBundle\Tax\VatCalculator
        tags:
            - { name: invoice.tax_calculator, alias: vat, priority: 10 }

    invoice.tax.gst:
        class: App\InvoiceBundle\Tax\GstCalculator
        tags:
            - { name: invoice.tax_calculator, alias: gst, priority: 5 }
```

The compiler pass collects all `invoice.tax_calculator` tags and injects them into a `TaxCalculatorRegistry`:

```php
class TaxCalculatorPass implements CompilerPassInterface
{
    public function process(ContainerBuilder $container): void
    {
        if (!$container->hasDefinition('invoice.tax_registry')) {
            return;
        }

        $registryDef = $container->getDefinition('invoice.tax_registry');
        $calculators = [];

        foreach ($container->findTaggedServiceIds('invoice.tax_calculator') as $id => $tags) {
            foreach ($tags as $tag) {
                $calculators[$tag['alias']] = new Reference($id);
            }
        }

        // Sort by priority (descending)
        uasort($calculators, fn($a, $b) => 0); // References; sort handled at runtime

        $registryDef->addArgument($calculators);
    }
}
```

The host application can now *add* its own calculator without modifying the bundle:

```yaml
# config/services.yaml
services:
    app.tax.custom:
        class: App\Service\CustomTaxCalculator
        tags:
            - { name: invoice.tax_calculator, alias: custom, priority: 100 }
```

This is the **open/closed principle** in action: the bundle is closed for modification but open for extension via tags.

---

#### 8.6 When to Write a Bundle vs. a Plain Service

This is the question every Symfony developer eventually faces. The temptation to "bundle-ify" everything is strong, but in Symfony 7.x the default should be **no bundle**.

##### Decision Matrix

| Criterion | Plain service (in `src/`) | Bundle |
|-----------|---------------------------|--------|
| Used in a single app | ✓ | |
| Simple logic, no config namespace needed | ✓ | |
| Reused across 3+ projects | | ✓ |
| Needs its own `config/packages/xxx.yaml` | | ✓ |
| Ships templates, migrations, or public assets | | ✓ |
| Exposes extension points (tags, interfaces) for consumers | | ✓ |
| Published as a Composer package | | ✓ |
| Internal domain module in a large monolith | ✓ (or a "bundle" if it has config) | ✓ |

##### Guidelines

1. **Default to a service.** If your functionality is a single class with a few dependencies, put it in `src/Service/` and autowire it. No bundle needed.

2. **Bundle when you have a configuration surface.** If your feature needs 5+ tunable options and you do not want to pollute `framework.yaml`, give it its own namespace via a bundle.

3. **Bundle when you ship reusable infrastructure.** A pagination engine, a multi-tenant router, a notification hub—these have enough moving parts (services, events, config) to justify the packaging overhead.

4. **Do NOT bundle a controller or a form.** These are application-level concerns. A bundle should be *framework-agnostic* (it can depend on the `HttpKernel` component but should not assume a specific controller structure).

5. **In a monolith, "pseudo-bundles" are fine.** Some teams organize a large app into internal bundles (`CustomerBundle`, `OrderBundle`, `BillingBundle`) even if they are never published. This enforces boundaries and makes it easy to extract a bundle later. The cost is minimal: a `Bundle` class + an `Extension` that does almost nothing.

##### Anti-Patterns

- **One class per bundle.** If your bundle contains exactly one service with no config, delete the bundle and use autowiring.
- **Bundles that override other bundles' services by ID.** This creates hidden coupling. Use tags and interfaces instead.
- **Bundles that depend on application-specific services.** A bundle should define its own abstractions (interfaces) and let the host app provide implementations. If your bundle's `Extension::load()` references `App\Entity\Customer`, you have a leak.

---

#### 8.7 Bundle Resources and Conventions

##### 8.7.1 Templates

If your bundle ships Twig templates, place them under:

```
src/InvoiceBundle/Resources/views/
    invoice/
        pdf.html.twig
        email.html.twig
```

In your app, reference them with the `@` syntax (deprecated) or the bundle's namespace (preferred in 7.x):

```twig
{# Preferred: use the bundle's template namespace #}
{% extends 'invoice::pdf.html.twig' %}
```

> **Symfony 7.x note:** The `@BundleName` prefix is removed. Use the bundle's registered template path, which is resolved automatically. If you use Twig's `TwigBundle` path resolver, templates in `Resources/views/` are available as `bundle_alias::template.html.twig`.

##### 8.7.2 Migrations

Doctrine migrations shipped in a bundle should go in:

```
src/InvoiceBundle/Migrations/
    Version20250115120000_CreateInvoiceTable.php
```

Configure the path in your `doctrine.yaml`:

```yaml
doctrine:
    dbal:
        migrations_paths:
            DoctrineMigrations: '%kernel.project_dir%/migrations'
            InvoiceBundleMigrations: '%kernel.project_dir%/src/InvoiceBundle/Migrations'
```

##### 8.7.3 Translations

```
src/InvoiceBundle/Resources/translations/
    messages.en.yaml
    messages.de.yaml
```

These are automatically picked up by the `Translator` component.

##### 8.7.4 Public Assets

```
src/InvoiceBundle/Resources/public/
    css/invoice-print.css
    js/invoice-interactions.js
```

Register the path in `framework.yaml` (or use AssetMapper from Chapter 14):

```yaml
framework:
    assets:
        packages:
            invoice:
                json_manifest_path: null
                paths:
                    - '%kernel.project_dir%/src/InvoiceBundle/Resources/public'
```

---

#### 8.8 Testing a Bundle

Because a bundle is a reusable package, it ships its own test suite. Two key patterns:

##### 8.8.1 Unit Tests for Services

Standard PHPUnit—mock dependencies, assert behavior:

```php
// tests/Unit/InvoiceBundle/InvoiceGeneratorTest.php

namespace App\Tests\Unit\InvoiceBundle;

use App\InvoiceBundle\Service\InvoiceGenerator;
use App\InvoiceBundle\Service\InvoiceRepository;
use PHPUnit\Framework\TestCase;

class InvoiceGeneratorTest extends TestCase
{
    public function testItAppliesCorrectTaxRate(): void
    {
        $repo = $this->createMock(InvoiceRepository::class);
        $repo->method('getNextNumber')->willReturn(42);

        $generator = new InvoiceGenerator($repo, 'EUR', true, ['DE' => ['rate' => 0.19]]);

        $invoice = $generator->create(['country' => 'DE', 'subtotal' => 100.0]);

        self::assertEquals(119.0, $invoice->getTotal());
        self::assertSame(42, $invoice->getNumber());
    }
}
```

##### 8.8.2 Functional Tests for the Extension

Verify that the Extension registers the expected services:

```php
// tests/Functional/InvoiceBundle/ExtensionTest.php

namespace App\Tests\Functional\InvoiceBundle;

use App\InvoiceBundle\DependencyInjection\InvoiceExtension;
use Symfony\Bundle\FrameworkBundle\Test\KernelTestCase;
use Symfony\Component\DependencyInjection\ContainerBuilder;

class ExtensionTest extends KernelTestCase
{
    public function testServicesAreRegistered(): void
    {
        $container = new ContainerBuilder();
        $extension = new InvoiceExtension();
        $extension->load([['currency' => 'EUR']], $container);

        $container->compile();

        self::assertTrue($container->has('invoice.generator'));
        self::assertTrue($container->has('invoice.repository'));
    }
}
```

---

#### 8.9 Debugging Bundles

##### Viewing Registered Services

```bash
$ bin/console debug:container invoice.
```

This lists all services whose ID starts with `invoice.`, showing their type, tags, and public/private status.

##### Inspecting a Specific Service

```bash
$ bin/console debug:container invoice.generator
```

Shows constructor arguments, tags, and which services depend on it.

##### Dumping the Configuration Tree

If you pass an invalid option, the framework throws a clear error:

```
[RuntimeException]
Unrecognized option "currency_code" under "invoice"
```

To see the *full* accepted schema:

```bash
$ bin/console debug:config invoice
```

##### Clearing the Cache

Bundle changes (new services, config schema changes) require a cache clear:

```bash
$ bin/cache:clear
```

In development, Symfony detects config file changes automatically, but compiler pass changes sometimes require a manual clear.

---

#### 8.10 Summary

| Concept | Key takeaway |
|---------|-------------|
| Bundle class | Thin entry point; extends `Bundle`; provides path and extension |
| Extension | Does the work: merges config, registers services, adds compiler passes |
| Configuration tree | Defines the schema; validated at compile time |
| Compiler pass | Modifies the container graph (tag, rewire, remove) before finalization |
| Tags | Open extension points; consumers add services without modifying the bundle |
| Bundle vs. service | Default to service; bundle when you need config namespace, reusability, or packaging |

A well-designed bundle is **boring**: a small extension, a clear config tree, a handful of tagged services, and a compiler pass that wires them together. The magic is in the *contract*—the tags and interfaces that let the host application extend the bundle without forking it.

---

#### 8.11 Exercises

##### Exercise 8.1 — Feature Flag Bundle

Create a `FeatureFlagBundle` that:

1. Accepts a configuration tree:
   ```yaml
   feature_flag:
       flags:
           new_checkout: { enabled: true,  environments: ['prod'] }
           beta_api:    { enabled: false, environments: ['dev', 'staging'] }
   ```
2. Registers a `FeatureFlagService` with a `isEnabled(string $flag, string $environment): bool` method.
3. Exposes a Twig function `feature_flag('new_checkout')` via a compiler pass that tags a Twig extension.
4. Includes a unit test for the service and a functional test for the extension.

##### Exercise 8.2 — Compiler Pass for a Registry

Extend the `InvoiceBundle` from this chapter:

1. Define an interface `InvoiceEventSubscriberInterface` with `onBeforeGenerate(InvoiceDTO $invoice): void`.
2. Write a compiler pass that collects all services tagged `invoice.event_subscriber` and injects them into an `InvoiceEventDispatcher` service.
3. Verify in a functional test that two tagged subscribers are both called in priority order.

##### Exercise 8.3 — Bundle vs. Service Decision

You are joining a team building an e-commerce platform. They have:

- A `PromotionEngine` (applies discount codes, 300 lines, no config)
- A `MultiTenantRouter` (resolves subdomain → tenant context, 12 config options, used in 4 projects)
- A `ShippingCalculator` (calculates shipping cost, 5 config options, used in 1 project)

For each, decide: **bundle** or **plain service**? Justify your choice in three sentences each, referencing the decision matrix in §8.6.

##### Exercise 8.4 — Breaking a Bundle

Intentionally break the `InvoiceBundle`:

1. Add a typo in the config tree (change `currency` to `curreny` in `Configuration.php` but not in the YAML).
2. Run `bin/console debug:config invoice` and note the error message.
3. Remove the `invoice.` prefix from the service ID in `services.yaml`.
4. Run `bin/console debug:container` and observe the collision (or lack thereof).
5. Fix both issues and confirm `bin/console cache:clear` resolves the cache.

---

*In the next chapter we leave the kernel's internals and turn to the user-facing layer: Twig templates, rendering, and the form/validation pipeline that turns user input into domain objects.*

### Chapter 9 — Twig Templating

> *You've built controllers that return responses, wired up routing, and mastered the dependency-injection container. But an invoicing app that only emits JSON isn't useful to the accountants who will actually use it. It's time to render HTML that people can read.*

Twig is Symfony's default templating engine, and it has become one of the most widely used template languages in the PHP ecosystem. Its design philosophy is deceptively simple: keep logic out of templates, make the syntax readable to non-developers, and provide a rich set of primitives—filters, functions, macros, inheritance—that let you build complex pages without writing a single line of PHP in a view.

In this chapter you'll learn:

- How to write Twig templates and use the core syntax (variables, conditionals, loops, operators)
- How template inheritance and `embed`/`include` let you structure multi-page applications
- How to reuse markup with macros and how to override form rendering with form themes
- How autoescaping, the sandbox, and the `security.policy` protect you from XSS and template injection
- How template caching works, how to debug templates with the web profiler, and how to extend Twig with custom filters and functions

By the end, you'll have the complete set of templates for the invoice list, invoice detail, and line-item editing screens of our running project, and you'll understand the mechanics that keep those templates fast and safe in a multi-tenant environment.

---

#### 9.1 Rendering Your First Template

If you followed the running project through Part II, you already have a `Tenant` entity, a `Controller` that fetches invoices, and a route `/tenant/{tenantSlug}/invoices`. Let's look at what the controller actually returns.

```php
// src/Controller/InvoiceController.php
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;

class InvoiceController extends AbstractController
{
    public function list(string $tenantSlug, InvoiceRepository $repo): Response
    {
        $tenant = $this->tenantResolver->resolve($tenantSlug);

        $invoices = $repo->findBy(['tenant' => $tenant], ['issuedAt' => 'DESC']);

        return $this->render('invoice/list.html.twig', [
            'tenant'   => $tenant,
            'invoices' => $invoices,
        ]);
    }
}
```

`$this->render()` is a convenience method on `AbstractController` that:

1. Resolves the template name against the configured template directory (by default `templates/`).
2. Renders it through the `Twig\Environment` service, passing the array as context.
3. Wraps the resulting HTML string in a `Response` object with `Content-Type: text/html`.

The file `templates/invoice/list.html.twig` starts as bare Twig:

```twig
{# templates/invoice/list.html.twig #}
<h1>Invoices for {{ tenant.name }}</h1>

<table>
  <thead>
    <tr><th>Number</th><th>Date</th><th>Total</th><th>Status</th></tr>
  </thead>
  <tbody>
    {% for invoice in invoices %}
      <tr>
        <td>{{ invoice.number }}</td>
        <td>{{ invoice.issuedAt|date('Y-m-d') }}</td>
        <td>{{ invoice.total|number_format(2, '.', ' ') }} €</td>
        <td>{{ invoice.status }}</td>
      </tr>
    {% endfor %}
  </tbody>
</table>
```

That's the entire mental model: `{{ }}` prints an expression, `{% %}` is a statement (control flow, imports, block definitions), and `{# #}` is a comment. Everything between those delimiters is Twig; everything else is literal HTML (or whatever markup you're producing).

> **Convention note.** Throughout this book we use attribute-based configuration and the `templates/` directory. You'll see `.html.twig` as the file extension; Symfony's `TwigBundle` is configured to look there by default, but any directory or suffix works if you override the `framework: twig:` config.

##### 9.1.1 The Twig Environment

Under the hood, `render()` resolves the service `twig`—an instance of `Twig\Environment`. That object holds:

| Member | Role |
|--------|------|
| A `FilesystemLoader` (or chain of loaders) | Maps a template name to a file path |
| A cache directory (`var/cache/twig` in a standard Symfony app) | Stores compiled PHP for each template |
| A set of `Twig\Filter`, `Twig\Function`, `Twig\NodeVisitor` instances | The extensions that add `|date`, `|trans`, the `sandbox` node-visitor, etc. |
| The autoescape policy | Decides which output formats are escaped by default |

You rarely talk to the environment directly in application code, but knowing it exists makes the rest of this chapter click. When you run `bin/console cache:clear`, the compiled templates in `var/cache/twig` are deleted; on the next request each template is recompiled into a PHP class like `__TwigTemplate_a3f8c…`.

##### 9.1.2 Expressions and the "Twig sandbox" of operators

Twig expressions support a deliberately limited operator set:

```twig
{{ invoice.total }}                  {# variable / method call / property #}
{{ invoice.total + invoice.tax }}    {# arithmetic #}
{{ invoice.total is defined }}       {# definition test #}
{{ invoice.status == 'paid' }}       {# comparison #}
{{ invoice.status in ['paid','sent'] }} {# membership #}
{{ invoice.issuedAt|date('m/d/Y') }} {# pipe a value through a filter #}
{{ invoice.lineItems|length }}       {# built-in filter #}
```

There is no `new`, no arbitrary function calls, no `::` (double-colon) static calls, and no array destructuring assignment. This restriction is not an accident; it's what lets Twig safely evaluate expressions in a sandboxed context (more in §9.6).

When you write `{{ invoice.total }}`, Twig resolves it in this order:

1. If `Invoice` has a `getTotal()` method → call it.
2. If it has an `isTotal()` method → call it.
3. If it has a `total` property (public or via `__get`) → read it.
4. If the value is an array, treat `total` as a key.

This "magic getter" chain is why Doctrine entities with standard getters render without any extra configuration.

---

#### 9.2 Control Flow

Twig's statement syntax mirrors what you'd read in plain English:

```twig
{# if / elseif / else #}
{% if invoice.isOverdue %}
  <span class="badge badge-danger">Overdue</span>
{% elseif invoice.status == 'sent' %}
  <span class="badge badge-info">Awaiting payment</span>
{% else %}
  <span class="badge badge-success">Paid</span>
{% endif %}

{# for loop with loop variable #}
{% for line in invoice.lineItems %}
  <tr class="{{ cycle(['row-a', 'row-b'], loop.index0) }}">
    <td>{{ loop.index }}.</td>
    <td>{{ line.description }}</td>
    <td>{{ line.quantity }} × {{ line.unitPrice|number_format(2) }}</td>
  </tr>
{% else %}
  {# executed when the collection is empty #}
  <tr><td colspan="3">No line items yet.</td></tr>
{% endfor %}

{# set / assign #}
{% set taxRate = tenant.vatRate ?? 0.20 %}
{% set totalWithTax = invoice.total * (1 + taxRate) %}

{# do — execute an expression for side effects (rarely needed) #}
{% do someService.notify(invoice) %}
```

The `loop` variable is available inside every `{% for %}` block and exposes `loop.index` (1-based), `loop.index0` (0-based), `loop.first`, `loop.last`, `loop.length`, and `loop.parent` (useful in nested loops).

> **Tip — the `cycle` filter.** The `cycle()` function above returns successive values from a list based on the current index. It's a small convenience that saves you from writing `loop.index0 % 2 == 0 ? 'row-a' : 'row-b'`.

##### 9.2.1 Whitespace control

Twig output includes the literal whitespace between tags, which can inflate HTML and break inline elements. Two mechanisms keep output tight:

```twig
{# Trim a single side with the `-` modifier #}
{% for line in items -%}
  <li>{{ line.name }}</li>
{%- endfor %}

{# Global option: trim_block_tag_start / trim_block_tag_end (deprecated in Twig 3.x) #}
{# The modern equivalent is the `whitespace` config on the Twig environment #}
```

In Symfony's default configuration, `trim_block_tag_end` is `true` and `trim_block_tag_start` is `false`. If you find extra blank lines in your rendered HTML, add the `-` modifier or wrap the block:

```twig
{%- for line in items %}
<li>{{ line.name }}</li>
{%- endfor %}
```

---

#### 9.3 Inheritance

A SaaS invoicing app has dozens of pages. You don't want to repeat the `<html>`, navigation bar, flash-message container, and footer on every one. Twig's inheritance mechanism solves this with a **base template** that defines *blocks*, and **child templates** that *override* those blocks.

##### 9.3.1 The base layout

```twig
{# templates/base.html.twig #}
<!DOCTYPE html>
<html lang="{{ app.request.locale|default('en') }}">
<head>
  <meta charset="utf-8">
  <title>{% block title %}Acme Invoicing{% endblock %}</title>
  <link rel="stylesheet" href="{{ asset('assets/build/app.css') }}">
  {% block stylesheets %}{% endblock %}
</head>
<body>

  <nav class="topbar">
    <a href="{{ path('dashboard') }}">{{ tenant.name }}</a>
    <ul>
      <li><a href="{{ path('invoice_list', {tenantSlug: tenant.slug}) }}">Invoices</a></li>
      <li><a href="{{ path('tenant_settings') }}">Settings</a></li>
    </ul>
  </nav>

  {# Flash messages — set by controllers, rendered once here #}
  <div class="flash-area">
    {% for label, messages in app.flashes %}
      {% for message in messages %}
        <div class="alert alert-{{ label }}">{{ message }}</div>
      {% endfor %}
    {% endfor %}
  </div>

  <main class="container">
    {% block body %}{% endblock %}
  </main>

  <footer>
    <p>© {{ "now"|date('Y') }} Acme Invoicing</p>
  </footer>

  <script src="{{ asset('assets/build/app.js') }}"></script>
  {% block javascripts %}{% endblock %}
</body>
</html>
```

Key points:

- `{% block name %}…{% endblock %}` marks a region that a child template can replace.
- `app` is a special Twig variable automatically injected by `TwigBundle`; `app.request`, `app.user`, `app.flashes`, and `app.environment` are always available.
- `path()` is a Twig function (registered by the `Router` integration) that generates a URL from a route name. You'll see it used constantly.
- `asset()` is provided by the `AssetMapper` component (Ch. 14) and resolves a logical path to a versioned URL.

##### 9.3.2 A child template

```twig
{# templates/invoice/list.html.twig #}
{% extends 'base.html.twig' %}

{% block title %}Invoices – {{ tenant.name }}{% endblock %}

{% block body %}
  <h1>Invoices</h1>
  <p class="muted">{{ invoices|length }} invoice(s)</p>

  <table class="table">
    <thead>
      <tr>
        <th>{{ 'number'|trans }}</th>
        <th>{{ 'issued_at'|trans }}</th>
        <th>{{ 'total'|trans }}</th>
        <th>{{ 'status'|trans }}</th>
      </tr>
    </thead>
    <tbody>
      {% for invoice in invoices %}
        <tr>
          <td>
            <a href="{{ path('invoice_show', {tenantSlug: tenant.slug, number: invoice.number}) }}">
              {{ invoice.number }}
            </a>
          </td>
          <td>{{ invoice.issuedAt|date('F j, Y') }}</td>
          <td class="num">{{ invoice.total|number_format(2, '.', ' ') }} {{ tenant.currency }}</td>
          <td>{{ invoice.statusLabel }}</td>
        </tr>
      {% else %}
        <tr><td colspan="4">{{ 'no_invoices'|trans }}</td></tr>
      {% endfor %}
    </tbody>
  </table>
{% endblock %}
```

`{% extends %}` must be the **first** statement in the child template. After that, the child only needs to define the blocks it overrides; everything else is inherited verbatim.

##### 9.3.3 Inheriting from a child — and calling the parent

Sometimes a child template needs to *add* content rather than *replace* it:

```twig
{% block body %}
  <h1>Invoices</h1>

  {# Render whatever the parent (or a higher-level template) put in this block #}
  {{- parent() -}}

  <p class="note">Showing the most recent 50. <a href="{{ path('invoice_list', {page: 2}) }}">Next →</a></p>
{% endblock %}
```

`parent()` is only available inside a block that is being overridden. It returns the content the parent template defined for that block (which itself may be empty).

##### 9.3.4 Multi-level inheritance

You can chain: `invoice/show.html.twig` → `invoice/base.html.twig` → `base.html.twig`. The innermost child wins for any given block. This is how we keep invoice-specific navigation (a tabs bar for *List / Detail / Settings*) in a shared `invoice/base.html.twig` without polluting the global layout.

```twig
{# templates/invoice/base.html.twig #}
{% extends 'base.html.twig' %}

{% block body %}
  <div class="tabs">
    <a href="{{ path('invoice_list', {tenantSlug: tenant.slug}) }}"
       class="{{ app.request.attributes.get('_route') == 'invoice_list' ? 'active' }}">
      All
    </a>
    {# child templates add more tabs by calling parent() and appending #}
    {{ block('invoice_tabs') }}
  </div>

  <div class="tab-content">
    {{- parent() -}}
  </div>
{% endblock %}
```

##### 9.3.5 `include` and `embed`

Inheritance is for *structure* (one template fills in another's skeleton). For *reusable fragments* you have two other tools:

**`include`** renders another template in the *current* context and inlines the result:

```twig
{# Simple: passes current context implicitly #}
{% include 'invoice/summary.html.twig' %}

{# Explicit context #}
{% include 'invoice/summary.html.twig' with {compact: true} %}

{# Ignore if the template doesn't exist #}
{% include 'invoice/notes.html.twig' ignore missing %}
```

**`embed`** combines inclusion with a mini-inheritance: you render a template *as if* it inherited from a block, without creating a separate file for the child:

```twig
{# Embed the 'form' block of a form theme, overriding just the label #}
{% embed 'form/form.html.twig' %}
  {% block label %}
    <span class="required">Amount</span>
  {% endblock %}
{% endembed %}
```

`embed` is especially handy when you want to reuse a complex component (a card, a modal) but tweak one field. It compiles to a single template, so there's no runtime cost of loading two files.

> **When to use which:**
> - *Inheritance* → page-level layout (one base, many children).
> - *include* → small, self-contained fragments (a pagination widget, a badge).
> - *embed* → you need the block-inheritance semantics of a template but don't want to create a child file.

---

#### 9.4 Macros

A **macro** is a named, reusable block of markup with parameters—Twig's answer to "component" before frameworks added explicit component systems.

##### 9.4.1 Defining and using macros

```twig
{# templates/_macros.html.twig #}

{% macro invoice_summary(invoice, options = {}) %}
  {% set showLineItems = options.showLineItems|default(false) %}

  <div class="invoice-card">
    <h3>{{ invoice.number }} — {{ invoice.issuedAt|date('Y-m-d') }}</h3>
    <p class="amount">{{ invoice.total|number_format(2, '.', ' ') }} {{ invoice.tenant.currency }}</p>
    <p class="status">{{ invoice.statusLabel }}</p>

    {% if showLineItems %}
      <ul class="line-items">
        {% for line in invoice.lineItems %}
          <li>{{ line.description }}: {{ line.subtotal|number_format(2) }}</li>
        {% endfor %}
      </ul>
    {% endif %}
  </div>
{% endmacro %}

{% macro badge(status) %}
  <span class="badge badge-{{ status|lower }}">{{ status|title }}</span>
{% endmacro %}
```

To use them in another template you must `import`:

```twig
{# templates/invoice/list.html.twig #}
{% import '_macros.html.twig' as macro %}

{% extends 'base.html.twig' %}
{% block body %}
  {# … #}
  <div class="summary">
    {{ macro.invoice_summary(latestInvoice, {showLineItems: true}) }}
  </div>

  <div>{{ macro.badge('Overdue') }}</div>
{% endblock %}
```

A few rules:

- Macros **cannot** use `extends`; they live in their own file (or the top of a file that has no `extends`).
- They **cannot** access the outer context implicitly. If you need `tenant`, pass it in.
- You can `from 'file' import macro_name` to pull a single macro, or `import 'file' as alias` to grab the whole module.
- If a macro file itself needs `extends` (rare), define the macros at the very top before the `extends` line, or split into two files.

##### 9.4.2 `_self` and calling macros within the same template

When a macro is defined in the same file where you use it, you reference it via `_self`:

```twig
{# templates/invoice/detail.html.twig #}
{% macro line_row(line) %}
  <tr>
    <td>{{ line.description }}</td>
    <td class="num">{{ line.quantity }}</td>
    <td class="num">{{ line.unitPrice|number_format(2) }}</td>
    <td class="num">{{ line.subtotal|number_format(2) }}</td>
  </tr>
{% endmacro %}

{% extends 'invoice/base.html.twig' %}
{% block body %}
  <table>
    {% for line in invoice.lineItems %}
      {{ _self.line_row(line) }}
    {% endfor %}
  </table>
{% endblock %}
```

##### 9.4.3 `with` context in macros

By default, a macro receives *only* the arguments you passed. If you want it to also see the calling template's context (e.g., `tenant` for currency), add the `with context` modifier:

```twig
{% macro invoice_summary(invoice) with context %}
  {# `tenant` is now available without being passed as an argument #}
  <p>{{ invoice.total|number_format(2) }} {{ tenant.currency }}</p>
{% endmacro %}
```

Use this sparingly; explicit arguments keep macros testable and predictable.

---

#### 9.5 Filters and Functions

Filters (pipe syntax `|`) and functions (call syntax `()`) are where Twig becomes genuinely expressive. Symfony ships dozens; here are the ones you'll use daily in the invoicing app.

##### 9.5.1 Date and number filters

```twig
{{ invoice.issuedAt|date('Y-m-d H:i') }}
{{ invoice.dueAt|date('F j, Y') }}
{{ "now"|date('Y') }}                          {# pass a string; null defaults to now #}
{{ invoice.issuedAt|date('U') }}               {# Unix timestamp #}
{{ invoice.issuedAt|date('Y-m-d', 'Europe/Paris') }} {# explicit timezone #}

{{ invoice.total|number_format(2, '.', ' ') }} {# 1 234.56 #}
{{ invoice.total|number_format(2, ',', '.') }} {# 1,234.56 #}
{{ line.quantity|int }}                        {# cast to int #}
{{ invoice.total|round(2, 'ceil') }}
```

##### 9.5.2 String and array filters

```twig
{{ invoice.number|upper }}
{{ tenant.name|lower }}
{{ tenant.name|title }}
{{ tenant.description|u.truncate(80, '…') }}   {# from the String component (symfony/string) #}

{{ lineItems|length }}
{{ ['a','b','c']|join(', ') }}
{{ invoice.lineItems|column('description') }}  {# extract one key from each element #}
{{ invoice.lineItems|map(l => l.subtotal)|reduce((sum, v) => sum + v, 0) }}
```

> **The `u.` prefix** (e.g., `|u.truncate`, `|u.slug`, `|u.fold`) comes from the `symfony/string` component and is auto-registered by `TwigBundle`. It gives you Unicode-aware string manipulation without loading an external library.

##### 9.5.3 Logic and comparison filters

```twig
{{ invoice.status|default('draft') }}          {# null-safe default #}
{{ invoice.status ? 'Yes' : 'No' }}            {# ternary (also works as filter: |ternary) #}
{{ invoice.tags|default([])|join(', ') }}
{{ [1,2,3]|filter(v => v > 1) }}               {# anonymous function (Twig 3.0+) #}
{{ [1,2,3,4,5]|slice(1, 3) }}                  {# [2,3,4] #}
{{ invoice.lineItems|sort(compare: a => a.subtotal, order: 'desc') }}
```

##### 9.5.4 The `trans` filter and translation

```twig
{{ 'invoice.total'|trans }}
{{ 'line_items.count'|trans({'%count%': lineItems|length}, 'Invoices') }}
```

`trans` looks up a message in the translation catalog for the current locale. You'll see the full mechanics in Ch. 27 (I18N); for now, know that every user-facing string in your templates should go through `|trans` so the app is localisable from day one.

##### 9.5.5 Writing a custom filter

The invoicing app needs a `|currency` filter that formats a number with the tenant's currency symbol. You register it in a service tagged `twig.extension`:

```php
// src/Twig/CurrencyExtension.php
use Twig\Extension\AbstractExtension;
use Twig\TwigFilter;

class CurrencyExtension extends AbstractExtension
{
    public function getFilters(): array
    {
        return [
            new TwigFilter('currency', $this->format(...), [
                'needs_environment' => false,
                'is_safe' => ['html'],
            ]),
        ];
    }

    public function format(float $amount, string $currencyCode): string
    {
        $symbol = match ($currencyCode) {
            'EUR' => '€',
            'USD' => '$',
            'GBP' => '£',
            default => $currencyCode.' ',
        };

        return number_format($amount, 2, '.', ' ')." {$symbol}";
    }
}
```

```php
// src/Twig/CurrencyExtension.php (continued, or a separate config)
#[AsExtension]  // Symfony 6.4+ / 7.x: auto-register via attribute
class CurrencyExtension extends AbstractExtension { /* … */ }
```

With the `#[AsExtension]` attribute (or the legacy `tags: [{twig.extension: ~}]` in XML/YAML), the extension is picked up by autowiring and registered automatically. Now in any template:

```twig
{{ invoice.total|currency(tenant.currencyCode) }}
```

A custom **function** is analogous:

```php
public function getFunctions(): array
{
    return [
        new TwigFunction('invoice_status_color', fn(string $s) => match($s) {
            'paid' => 'green', 'overdue' => 'red', 'sent' => 'blue', default => 'gray',
        }),
    ];
}
```

```twig
<span class="dot dot-{{ invoice_status_color(invoice.status) }}"></span>
```

##### 9.5.6 The `is` operator and tests

Tests are boolean predicates that read naturally:

```twig
{% if invoice.total is constant('App\\Entity\\Invoice::STATUS_PAID') %}
{% if invoice.issuedAt is same as someDate %}
{% if invoice.lineItems|length is odd %}
{% if invoice.dueAt is greaterthan("now") %}
```

You can also write custom tests:

```php
new TwigTest('overdue', fn(Invoice $inv): bool => $inv->isOverdue())
```

```twig
{% if invoice is overdue %}
```

---

#### 9.6 Form Themes

Forms (Ch. 10 in detail) are rendered in Twig through a **form theme**—a set of templates that control the HTML for each field type. Symfony ships a default Bootstrap-free theme, but you'll almost always customise at least one or two blocks.

##### 9.9.1 How form rendering works

When you write `{{ form_row(form.invoice) }}` in a template, Twig calls a chain of "renderer" functions (`form_widget`, `form_label`, `form_errors`, `form_row`) that look for a block matching the field's *block prefix*. For a field of type `text` inside a form named `invoiceForm`, the lookup order is:

1. `form_widget_text` (type-specific)
2. `form_widget` (generic widget)
3. The base theme's implementation

You override at the most specific level you need:

```twig
{# templates/form/theme.html.twig #}
{% extends 'form_div_layout.html.twig' %}

{# Customise all text inputs #}
{% block form_widget_text %}
  {% set attr = attr|merge({class: 'input-field ' ~ (errors|length ? 'has-error' : '')}) %}
  {{- parent() -}}
{% endblock %}

{# Customise a single field by its form name #}
{% block invoiceForm_amount %}
  {{ form_label(form) }}
  {{ form_widget(form, {attr: {placeholder: '0.00'}}) }}
  {{ form_errors(form) }}
{% endblock %}
```

Then in `config/packages/twig.yaml`:

```yaml
twig:
    form_themes: ['form/theme.html.twig']
```

Or per-view:

```twig
{% form_theme form 'form/theme.html.twig' %}
```

##### 9.9.2 Form theme blocks you'll override most

| Block | Purpose |
|-------|---------|
| `form_widget` | The `<input>` / `<select>` / `<textarea>` element |
| `form_label` | The `<label>` |
| `form_errors` | The `<ul>` of validation errors |
| `form_row` | The wrapper `<div>` that combines the above |
| `form_widget_checkbox` / `form_widget_radio` | Checkboxes and radio groups |
| `form_widget_collection` | Repeating sub-forms (e.g., line items) |

For the invoicing app's line-item collection (a form-within-a-form that the user can add/remove rows of), you'll override `form_widget_collection`:

```twig
{% block form_widget_collection %}
  <div class="line-items-collection">
    {% for child in form %}
      {{ form_row(child) }}
      <button type="button" class="btn-remove-line" data-index="{{ loop.index }}">×</button>
    {% endfor %}
    <button type="button" class="btn-add-line">+ Add line</button>
    {{ form_widget(form.prototype) }} {# hidden template for JS to clone #}
  </div>
{% endblock %}
```

This is the standard pattern: render the existing children, render a hidden `prototype` that JavaScript clones for new rows.

---

#### 9.7 Security: Autoescaping and the Sandbox

##### 9.7.1 Autoescaping

Twig's autoescaper wraps every `{{ expression }}` output in `htmlspecialchars()` by default, using the `html` charset. This neutralises `<script>`, `onerror=`, and other HTML-injection vectors **for you**, as long as you don't opt out.

The escape strategy depends on the template's *output format*, determined by the file extension:

| Extension | Escape strategy |
|-----------|----------------|
| `.html.twig`, `.twig` | `html` |
| `.js.twig` | `js` |
| `.css.twig` | `css` |
| `.txt.twig`, `.xml.twig` | `html` (safe for both) |
| `.json.twig` | (no autoescape; use `|json_encode`) |

You can force a different strategy inline:

```twig
{{ value|e('js') }}       {# escape for a JS string context #}
{{ value|e('css') }}      {# escape for a CSS value #}
```

##### 9.7.2 The `safe` flag — and why you should fear it

```twig
{# DANGEROUS: bypasses autoescaping entirely #}
{{ rawHtml|raw }}
{{ rawHtml|e('none') }}
```

`|raw` (alias `|e('none')`) tells Twig "I promise this string is already safe." You should reach for it **only** when:

- The content comes from your own code (not user input), or
- You have explicitly sanitised it (e.g., through a WYSIWYG sanitizer like `HTMLPurifier`), or
- You're outputting a trusted, static asset (a logo SVG).

A common safe pattern:

```twig
{# User-authored rich text that was sanitised in the controller #}
{{ line.notesHtml|raw }}   {# only if $line->getNotesHtml() was purified upstream #}
```

> **Rule of thumb:** if a value can trace its provenance back to a user-supplied field, it should never pass through `|raw`.

##### 9.7.3 The Twig sandbox

Autoescaping handles *output encoding*. The **sandbox** handles *what expressions are allowed to execute*. It's a `NodeVisitor` that, when enabled, throws a `SecurityPolicy` violation if a template tries to:

- Call a method not in the allow-list
- Access a property not in the allow-list
- Instantiate a class
- Use a filter or function not in the allow-list

This matters in the invoicing app because we consider the possibility of letting a tenant's admin upload a "custom report template." You'd render it with the sandbox on:

```php
// config/packages/twig.yaml
twig:
    sandbox:
        enabled: true
        global_template_allowed_methods: ['__toString']
        allowed_methods:
            'App\\Entity\\Invoice': ['getNumber', 'getTotal', 'getStatus', 'isOverdue']
            'App\\Entity\\Tenant': ['getName', 'getCurrency', 'getVatRate']
            'DateTimeImmutable': ['format', 'diff']
        allowed_properties: []
        allowed_tags: ['if', 'for', 'set', 'block']
        allowed_filters: ['date', 'number_format', 'trans', 'currency', 'upper', 'lower', 'length', 'join']
```

```twig
{# In a sandboxed template #}
{% for invoice in invoices %}
  {{ invoice.number }} — {{ invoice.total|currency(tenant.currency) }}
{% endfor %}

{# This would throw SecurityError: #}
{# {{ invoice.serialize() }}  — 'serialize' not in allowed_methods #}
{# {{ constant('App\\Entity\\Invoice::DRAFT') }} — 'constant' not in allowed_tags #}
```

You can enable the sandbox selectively per-request via the `twig.sandbox` service or by wrapping a specific render call:

```php
use Twig\Environment;

public function renderCustomReport(Template $tpl): string
{
    /** @var Environment $twig */
    $twig = $this->container->get('twig');
    // In production, ensure sandbox is ON globally or use a dedicated
    // Twig environment instance with sandbox enabled.
    return $twig->render('tenant/report.html.twig', ['template' => $tpl]);
}
```

The sandbox is **not** a general-purpose execution sandbox. A determined attacker with a very long template and clever string manipulation can still cause DoS (long loops, large allocations). Treat user-supplied templates as untrusted and set timeouts on the rendering process.

##### 9.7.4 XSS in attributes and JS contexts

Autoescaping with the `html` strategy covers text content and HTML attributes. But if you interpolate into a `<script>` block or a `data-*` attribute that JavaScript will `JSON.parse`, you need to be deliberate:

```twig
{# Safe: Twig's html escape handles the attribute context #}
<input value="{{ customer.name }}">

{# UNSAFE: the escaped string is now a JS literal, not a JSON one #}
<script>
  var name = "{{ customer.name }}";   {# "Bob \"O'Brien" breaks out #}
</script>

{# Safe: use |json_encode (which is a function, not a filter, in older Twig;
   in Twig 3.x it's a filter) #}
<script>
  var name = {{ customer.name|json_encode|e('js') }};
</script>
```

The `|json_encode` filter produces a valid JSON string (with quotes), and the subsequent `|e('js')` escapes any `</script>` sequence that might close the tag early.

---

#### 9.8 Template Caching

Every `.twig` file is compiled once into a PHP class and cached on disk. The cache directory in a standard Symfony app is `var/cache/{environment}/twig/` (e.g., `var/cache/prod/twig/`).

##### 9.8.1 How the cache key works

The compiled filename is a hash of the *template source*. Twig checks the source file's modification time (`mtime`) against the cached version. If the source is newer, it recompiles. In production (`APP_ENV=prod`), the check is cheap (a single `filemtime()` call per template per request). In development (`APP_ENV=dev`), Symfony's `debug` flag forces Twig into `auto_reload` mode, so you see changes immediately.

You can tune the behaviour in `config/packages/twig.yaml`:

```yaml
twig:
    # null = auto (on in dev, off in prod)
    auto_reload: null
    # Directory for compiled templates
    cache: '%kernel.build_dir%/twig'   # defaults to var/cache/{env}/twig
    # If true, recompile even if mtime is unchanged (useful in Docker dev)
    strict_variables: true            # throw on undefined vars instead of empty string
```

`strict_variables: true` is a good idea in dev: it surfaces typos like `{{ inoice.total }}` immediately instead of silently rendering an empty string.

##### 9.8.2 Invalidation

- **`bin/console cache:clear`** wipes the entire `var/cache/{env}/` tree, including compiled templates.
- **`bin/console cache:warmup`** pre-compiles all templates found by the bundle's `DependencyInjection\CompilerPass` that registers template paths. This is important in production Docker builds: you warm the cache in the image so the first real request doesn't pay the compilation cost.

```dockerfile
# Dockerfile (production stage, excerpt)
COPY --from=build /app /app
RUN php bin/console cache:warmup --env=prod
```

##### 9.8.3 HTTP caching of rendered pages

Template caching speeds up *server-side* rendering. If you also want to cache the *response* at the HTTP level (CDN, reverse proxy, or Symfony's own `HttpCache`), that's a separate concern covered in Ch. 23. The two are complementary: even when the HTTP cache is warm and the template is never rendered, you still want the template cache to be valid so that cache-miss requests are fast.

---

#### 9.9 Debugging Templates

##### 9.9.1 The Web Profiler

In `dev` environment, the toolbar at the top of every page includes a **Twig** panel. Click it and you see:

- Every template rendered during the request, in order, with its render time.
- The context (variables) available in each template — click a template to expand.
- Whether each template was served from cache or freshly compiled.
- Any deprecation notices from Twig.

This is the single most useful debugging tool for template work. If a page looks wrong, the Twig panel tells you exactly which template rendered which block and what values it received.

##### 9.9.2 Enabling Twig debug output

Add this to a template temporarily:

```twig
{# Dump the full context #}
{{ dump() }}

{# Dump a specific variable with more depth #}
{{ dump(invoice, {maxItems: 20, maxDepth: 4}) }}
```

`dump()` uses Symfony's `VarDumper` and produces a collapsible HTML table. In production it's disabled by default (the `debug` flag is off), so it's safe to leave a few stray `{{ dump() }}` calls in code that gets deployed—they simply render nothing.

##### 9.9.3 The `twig:debug` console command

```bash
php bin/console debug:twig
php bin/console debug:twig 'invoice/list.html.twig'   {# details for one template #}
```

Lists all registered extensions, filters, functions, and tags, and shows which template files are known to the loader. Useful when a filter "should" exist but doesn't (a missing bundle, a typo in the extension name).

##### 9.9.4 Common pitfalls

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| "Variable `x` does not exist" | Typo, or the variable isn't passed by the controller | Check controller's `render()` args; enable `strict_variables` in dev |
| "Unexpected token `endfor`" | A `{% for %}` without a matching `{% endfor %}` | Count your blocks; the profiler's source view highlights unmatched tags |
| Filter not found | Extension not registered, or typo | `php bin/console debug:twig` to list available filters |
| Template not found | Wrong path or bundle prefix missing | Ensure the file is in `templates/` or use `@BundleName/...` for bundle templates |
| Flash messages not showing | Session not started, or `app.flashes` not in context | Ensure session is enabled in `framework.yaml`; `app.flashes` is always available in Twig |
| Inheritance "block already defined" | Two child templates both override the same block, or a block is defined outside `extends` | Only the *innermost* template should define a block; check for duplicate `{% block %}` names |

##### 9.9.5 `twig:lint`

Since Symfony 7.x / Twig 3.x, there's a linter that catches syntax errors without rendering:

```bash
php bin/console lint:twig templates/
```

Add this to your CI pipeline; it fails fast on malformed Twig before a test suite even starts.

---

#### 9.10 Template Structure in the Invoicing App

Let's zoom out and look at the directory layout that emerges from everything above:

```
templates/
├── base.html.twig              # Global layout (nav, flashes, footer)
├── dashboard.html.twig         # extends base
├── _macros.html.twig           # Shared macros (invoice_summary, badge, pagination)
├── form/
│   └── theme.html.twig         # Form theme (extends form_div_layout)
├── invoice/
│   ├── base.html.twig          # extends base; adds invoice tabs
│   ├── list.html.twig          # extends invoice/base
│   ├── show.html.twig          # extends invoice/base; full detail view
│   ├── new.html.twig           # extends invoice/base; form
│   ├── edit.html.twig          # extends invoice/base; form
│   ├── _form.html.twig         # include'd by new/edit (the form itself)
│   └── _summary.html.twig      # include'd fragment (compact invoice card)
├── tenant/
│   ├── settings.html.twig
│   └── report.html.twig        # Potentially sandboxed (user-supplied template)
└── email/                      # (Ch. 16) Twig templates for transactional mail
    ├── invoice_sent.html.twig
    └── payment_reminder.html.twig
```

Conventions:

- **Underscore prefix** (`_macros`, `_form`, `_summary`) signals "not a page; include/embed only."
- **One `base` per section** (global `base`, `invoice/base`) keeps navigation context without deep inheritance chains.
- **Form themes live in `form/`** and are referenced globally; per-form overrides go in the form theme file, not scattered across page templates.
- **Email templates** are in their own directory and use a minimal base (no nav, no JS) because they render in a `<body>`-less context.

---

#### 9.11 Beyond HTML: Rendering Other Formats

Twig is HTML-centric but not HTML-only. The same engine can render:

- **JSON:** A `.json.twig` template with `autoescape: false` (or just use `|json_encode`). More practically, for APIs you'll use the Serializer (Ch. 19), not Twig.
- **CSV:** A `.csv.twig` template that loops and outputs comma-separated values. Useful for an "export invoices" endpoint.
- **Plain text / email:** `.txt.twig` for transactional emails (Ch. 16).
- **PDF:** Not directly, but you can render an HTML template and pipe it through a library like `Dompdf` or `wkhtmltopdf`. The template is still a standard `.html.twig`.

For the invoicing app, the "Export CSV" button on the invoice list uses:

```twig
{# templates/invoice/export.csv.twig #}
{% set separator = ',' %}
"Number","Issued","Due","Total","Status"
{% for invoice in invoices %}
"{{ invoice.number }}","{{ invoice.issuedAt|date('Y-m-d') }}","{{ invoice.dueAt|date('Y-m-d') }}","{{ invoice.total|number_format(2, '.', '') }}","{{ invoice.status }}"
{% endfor %}
```

```php
// In the controller
$response = $this->render('invoice/export.csv.twig', compact('invoices'));
return new Response($response, 200, [
    'Content-Type'        => 'text/csv',
    'Content-Disposition' => 'attachment; filename="invoices.csv"',
]);
```

> Note: for CSV, autoescaping with the `html` strategy will encode commas and quotes in a way that *breaks* CSV. For such templates, either disable autoescaping for that file (via a custom `FilesystemLoader` that marks it as safe) or handle quoting in a dedicated filter. In practice, for data export you often skip Twig entirely and stream a `fputcsv()` loop—Twig shines for *presentation*, not *data interchange*.

---

#### 9.12 Summary

| Concept | Key takeaway |
|---------|-------------|
| Syntax | `{{ }}` prints, `{% %}` controls flow, `{# #}` comments. Expressions are deliberately limited. |
| Inheritance | One base template defines blocks; children override. `parent()` extends rather than replaces. |
| include / embed | `include` for fragments; `embed` for "inherit from this template's blocks inline." |
| Macros | Named, parameterised markup blocks. Use `_self` or `import`. Be explicit about context. |
| Filters & functions | Pipe syntax for filters, call syntax for functions. Custom ones via `#[AsExtension]`. |
| Form themes | Override blocks (`form_widget`, `form_row`, etc.) to control form HTML globally or per-field. |
| Autoescaping | On by default for `html`/`js`/`css` contexts. `|raw` is an explicit opt-out—use rarely and deliberately. |
| Sandbox | Restricts which methods, properties, tags, and filters a template can use. Essential for untrusted templates. |
| Caching | Templates compile to PHP classes in `var/cache/{env}/twig/`. Warm in production. `strict_variables` catches typos. |
| Debugging | Web Profiler Twig panel, `{{ dump() }}`, `debug:twig`, `lint:twig`. |

---

#### Exercises

1. **Layout refactor.** Create a `templates/billing/base.html.twig` that extends `base.html.twig` and adds a left sidebar with links to *Invoices*, *Customers*, and *Settings*. Refactor `invoice/list.html.twig` and `invoice/show.html.twig` to extend it. Verify that the global flash messages still render.

2. **Macro for a data table.** Write a macro `data_table` that accepts a column header list, a row template (via `embed`), and a collection. Use it to render both the invoice list and the customer list. The macro should automatically add zebra striping and an "empty" row when the collection is empty.

3. **Custom filter `|time_ago`.** Implement a Twig filter that takes a `DateTimeInterface` and returns a human-readable relative time ("3 days ago", "2 weeks ago", "just now"). Register it via `#[AsExtension]`. Use it in the invoice list to display `issuedAt` in a secondary column.

4. **Form theme override.** Create a form theme that renders all form errors as an inline tooltip (a `<span class="tooltip">` positioned absolutely) rather than a `<ul>` below the field. Apply it to the invoice edit form and verify that validation errors appear on blur (you'll need a tiny JS snippet—don't worry, Ch. 14 covers AssetMapper and JS bundling in detail).

5. **Sandbox challenge.** Create a Twig template that a tenant can upload (stored as a DB string, rendered via a `Template` object implementing `Twig\LoaderInterface`). Enable the sandbox with a minimal allow-list. Try to make it render `{{ app.user.roles }}` and confirm that a `SecurityError` is thrown. Then extend the allow-list just enough to let the template display the tenant's name and invoice count.

6. **Template lint in CI.** Add a GitHub Actions step (or equivalent) that runs `php bin/console lint:twig templates/` on every pull request. Introduce a deliberate syntax error in a branch and verify the build fails with a helpful message.

7. **Performance check.** Using the Web Profiler, render the invoice list with 500 invoices. Note the total Twig render time. Then:
   - Add `{% for invoice in invoices|batch(50) %}` and wrap each batch in a `<tr>` with a `data-batch` attribute. Does the time change? Why?
   - Move the "latest 10 invoices" summary block into a separate `include` and compare the profiler output.
   - Clear the Twig cache and observe the first-request vs. cached-request timing.

8. **i18n dry run.** Wrap every user-facing string in `templates/invoice/list.html.twig` in `|trans` with a sensible message ID. Create a `translations/messages.fr.yaml` file with three translated strings. Switch the locale to `fr` (via the `Accept-Language` header or a query param) and verify the translations appear. (Full i18n is Ch. 27; this is just a warm-up.)

---

*Next chapter: [Chapter 10 — Forms](#). You'll build the invoice creation form with a dynamic line-item collection, custom field types, and validation integration—tying together the form theme work from §9.6 with the full `Form` component.*

I'll verify a couple of current Symfony form API details (factory method signatures and any recent deprecations) so the chapter's code is accurate for 7.4/8.x, then write it.



Let me confirm the exact signatures from the source.



Signatures confirmed (`create(type, data, options)`, `createBuilder(type, data, options)`). I have what I need. Here's Chapter 10.

---

### Chapter 10 — Forms

In Chapter 9 we turned PHP into HTML with Twig. But a web application is rarely only about *displaying* data — it's about *changing* it. And the most important way users change data in a browser is the humble `<form>`.

You could hand-roll every form by writing raw `<input>` tags, gluing together `$_POST` values, and writing your own validation and rendering. That works, and if you've built web apps for a while you've probably done exactly that. What you've also probably discovered is that the tedious part — keeping the rendered field in sync with the submitted value, wiring fields to object properties, re-displaying errors next to the right input, protecting against CSRF — is 80% boilerplate and 20% where bugs hide.

Symfony's **Form component** exists to delete that 80%. You describe a form as a structured, typed object graph; the framework renders it, binds submitted values back onto your model, validates the result, and hands you a clean object on the other side. This chapter walks through the whole of that machinery — from a one-field form to the invoice editor that will anchor the running project for the rest of Part III.

By the end of this chapter you will be able to:

- Build and render forms using the form factory and a custom form type;
- Choose the right built-in field type and understand the options that matter;
- Map form fields to entities and plain objects safely (and know *why* it's safe from mass assignment);
- Write your own reusable, composite field types;
- Grow and shrink forms at runtime with form events;
- Handle repeating data with collections (the invoice line items);
- Understand how validation and CSRF protection integrate, and where to switch them off.

---

#### 10.1 The problem forms solve

HTTP is stateless: a `POST` body is just a flat bag of strings. A browser form is the bridge between that flat bag and the structured, validated, typed objects your application thinks in. The Form component owns three jobs that otherwise leak into every controller:

1. **Rendering.** Given a form object, produce valid, consistent, accessible HTML — including re-populating fields with the user's previous input and marking the ones that failed validation.
2. **Data mapping.** Take the flat submitted bag and write it back onto an object (or a plain array), reading and writing the right properties, in the right order, with the right type conversions.
3. **Security and validation.** Reject requests that lack a valid CSRF token, and report field-level problems the user can actually act on.

The payoff is that a controller becomes almost declarative: *make the form, handle the request, check validity, persist.* Let's look at the moving parts before we write any code.

##### The object graph

A Symfony form is a tree. The **root** is a `FormInterface` (concretely a `Form`). It contains *children* — one `Form` per field — each of which has a *type*, *options*, and a *view* (the Twig-friendly representation used for rendering).

Three interfaces do most of the work, and you'll see all of them in every form you write:

| Type | Interface | Role |
|------|-----------|------|
| A form (root or field) | `FormInterface` | The built form: holds data, children, errors; `isSubmitted()`, `isValid()`, `getData()`. |
| A form builder | `FormBuilderInterface` | Fluent, mutable factory you use to *describe* the form: `add()`, `getForm()`. |
| A form type | a class extending `AbstractType` | A **named, reusable recipe** for a form or a field. This is the core of custom forms. |

The flow in every request is the same:

```
Request ──▶ FormBuilder ──▶ Form (built) ──▶ render in Twig
              ▲                   │
              │                   ▼
       FormType recipe      handleRequest() re-binds submitted
                            values, validates, updates data
```

`AbstractType` is the linchpin: it lets you encapsulate "a form for an invoice" (or "a money field", or "an address block") in one class that's testable in isolation, reusable everywhere, and renderable from Twig. Almost everything in this chapter is a variation on *build a type, build a form from it, render, handle, validate*.

---

#### 10.2 Your first form

Let's start with the smallest useful form: a "feedback" form with two fields and no backing object. We'll inject the **form factory** rather than relying on a controller helper, because the factory is the object you'll want to inject into services and to mock in tests (Chapter 22).

The factory's entry points all follow the same `(type, data, options)` shape:

```php
public function create(string $type = FormType::class, mixed $data = null, array $options = []): FormInterface;
public function createBuilder(string $type = FormType::class, mixed $data = null, array $options = []): FormBuilderInterface;
```

`createBuilder()` hands you the fluent `FormBuilderInterface` so you can assemble a form field-by-field inline; `create()` builds a form straight from a *type* (the class from §10.6 onward). For our first example we use the builder.

```php
// src/Controller/FeedbackController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\Form\Extension\Core\Type\EmailType;
use Symfony\Component\Form\Extension\Core\Type\SubmitType;
use Symfony\Component\Form\Extension\Core\Type\TextareaType;
use Symfony\Component\Form\FormFactoryInterface;
use Symfony\Component\Form\FormInterface;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;

final class FeedbackController extends AbstractController
{
    public function index(Request $request, FormFactoryInterface $formFactory): Response
    {
        $form = $formFactory->createBuilder()
            ->add('email', EmailType::class, ['label' => 'Your email'])
            ->add('message', TextareaType::class, ['label' => 'How can we help?'])
            ->add('send', SubmitType::class, ['label' => 'Send'])
            ->getForm();

        if ($request->isMethod('POST')) {
            $form->handleRequest($request);

            if ($form->isSubmitted() && $form->isValid()) {
                /** @var array $data shape: ['email' => string, 'message' => string] */
                $data = $form->getData();
                // ... dispatch a "send feedback" message (Ch. 17) ...

                return $this->redirectToRoute('feedback_thanks');
            }
        }

        return $this->render('feedback/index.html.twig', ['form' => $form]);
    }
}
```

The whole round trip in four lines: **build → `handleRequest` → `isValid` → `getData`**. `handleRequest()` does the heavy lifting — it reads the submitted payload, writes each value into the corresponding field (running that field's *transformers*, so strings become integers, dates, etc.), and marks the form submitted. `isValid()` is true only when the form was submitted **and** has no errors (see §10.11).

##### Rendering in Twig

The form is rendered with a small set of "theme" functions that expand into the actual HTML. The minimum is `form_start`, `form_widget`, and `form_end`:

```twig
{# templates/feedback/index.html.twig #}
{% extends 'base.html.twig' %}

{% block title %}Feedback{% endblock %}

{% block body %}
    <h1>Send us feedback</h1>

    {{ form_start(form) }}
        {{ form_widget(form) }}
    {{ form_end(form) }}
{% endblock %}
```

`form_widget(form)` renders *every* child, which is why the example above is so short. In real screens you'll almost always render fields individually so you can control layout — the next section shows both granular functions and the errors helpers you'll use constantly.

> **Why the `isMethod('POST')` guard?**
> `handleRequest()` is safe to call on a `GET` (it simply finds nothing to bind), but guarding with `isMethod('POST')` makes the intent explicit and avoids re-processing a form that was never submitted. It's a cheap habit worth keeping.

---

#### 10.3 Rendering fields, labels, and errors

For a single field, `form_widget(form.email)` only emits the `<input>`. A complete, labelled, accessible row uses three helpers:

```twig
{{ form_row(form.email) }}
```

`form_row()` is sugar for label + widget + errors + help. When you need to control each piece (and you usually do, to match your design system):

```twig
{{ form_label(form.message) }}
{{ form_widget(form.message) }}
{{ form_errors(form.message) }}
```

And to render errors for the **whole form** (e.g., form-level validation messages, not tied to one field), put a single call near the top:

```twig
{{ form_start(form) }}
    {{ form_errors(form) }}          {# non-field errors #}

    <div class="row">
        <div class="col">
            {{ form_label(form.email) }}
            {{ form_widget(form.email) }}
            {{ form_errors(form.email) }}
        </div>
    </div>

    <div class="row">
        {{ form_label(form.message) }}
        {{ form_widget(form.message) }}
        {{ form_errors(form.message) }}
    </div>

    <button type="submit">{{ form_widget(form.send) }}</button>
{{ form_end(form) }}
```

Two things to notice:

- **Errors appear where you render them.** If a field has a violation, `form_errors()` returns the message(s); otherwise it returns an empty string, so calling it unconditionally is the idiomatic pattern.
- **Failed fields are re-populated.** After an invalid submit, `form_widget(form.email)` renders the user's (invalid) value back into the input, because the form's data now *is* the submitted value. This is the "keep what they typed" behaviour people expect — and it comes for free.

Every widget also exposes CSS classes that make styling painless. `form_widget()` adds `form-control`-style hooks, and a field that has errors is rendered with the error state visible in the DOM (via the view's `invalid` flag, which form themes turn into a CSS class). You'll lean on these when you set up a form theme in §10.7.

---

#### 10.4 Built-in field types

The `Core` extension ships dozens of field types. You don't need to memorise them all, but you should know the workhorses and the non-obvious option each carries. Here's the working set for an application like ours:

| Type | Class | Produces / Notes |
|------|-------|------------------|
| Text | `TextType` | `<input type="text">`. The default for most fields. |
| Email | `EmailType` | `inputmode="email"`. Use `always_empty` to avoid pre-filling a saved address. |
| Password | `PasswordType` | `<input type="password">`. |
| Repeated | `RepeatedType` | Two matching password fields. The key type for registration/login. |
| Integer | `IntegerType` | Transforms `"42"` → `42`. Rejects non-numeric input. |
| Number | `NumberType` | `float`/`int` input; `scale` for decimals. |
| Money | `MoneyType` | Renders an amount + currency; `currency` option. |
| Percent | `PercentType` | Numeric with `%` affordance. |
| Textarea | `TextareaType` | Multi-line text. |
| Checkbox | `CheckboxType` | Boolean. |
| Choice | `ChoiceType` | `<select>`. `choices`, `multiple`, `expanded`, `placeholder`. |
| Entity | `EntityType` | `ChoiceType` bound to a Doctrine class. `class`, `choice_label`. |
| Date / Time / DateTime | `DateType`, `TimeType`, `DateTimeType` | `widget` (`single_text` or `choice`), `input` format. |
| Hidden | `HiddenType` | Non-editable value carried in the form. |
| Country / Language | `CountryType`, `LanguageType` | Curated choice lists. |
| File | `FileType` | `input` = `File` or `File[]`. |
| Submit / Reset / Button | `SubmitType`, `ResetType`, `ButtonType` | Controls. `SubmitType` is what turns a `<form>` into a `POST` that Symfony recognises as a submission. |

A few options deserve a call-out because they trip people up:

```php
use Symfony\Component\Form\Extension\Core\Type\DateTimeType;
use Symfony\Component\Form\Extension\Core\Type\MoneyType;
use Symfony\Bridge\Doctrine\Form\Type\EntityType;

// DateTime: choose the input format AND the widget
->add('issueDate', DateTimeType::class, [
    'widget' => 'single_text',              // one field, not 5
    'input'  => \DateTimeInterface::class,  // what the model property holds
])

// Money: pick the currency once, not per-row
->add('total', MoneyType::class, [
    'currency' => 'USD',
])

// Entity: name the class and how to label each option
->add('client', EntityType::class, [
    'class'        => Client::class,
    'choice_label' => 'name',               // a property, closure, or method
    'placeholder'  => '— Select a client —',
])
```

The `input` option is the most misunderstood. It tells the form what *type* the model property holds (`\DateTimeInterface::class`, `int::class`, `File::class`, or a format string like `'Y-m-d'`). Get it right and the round-trip conversion is invisible; get it wrong and you'll see a confusing type error at submit time. When mapping to a Doctrine entity, match `input` to the property's declared type.

> **`EntityType` and performance.** By default `EntityType` loads **every** row of the entity to build the choice list. Fine for a handful of options; disastrous for thousands. For large tables, restrict the list with a `query` option (a `QueryBuilder`), or use a search-as-you-type approach. We return to this in §10.14.

---

#### 10.5 The form builder in depth

`FormBuilderInterface` is a fluent API. The methods you'll actually use:

```php
$builder
    ->add('name', TextType::class, $options)      // add a field
    ->add('name', ['label' => '…'])               // options can be first-class
    ->remove('name')                              // remove a field
    ->rename('old', 'new')                        // move a field
    ->getForm();                                   // build and return a FormInterface
```

`add()`'s third argument is an **options array** — the same key/value pairs you passed in §10.2. Options are validated against a per-type definition (an `OptionsResolver`), so a typo'd option name fails *fast*, at build time, not at render time. That's a big part of why forms are safe to refactor: the framework tells you when an option doesn't belong.

##### Where options come from

Options are resolved in a predictable precedence order, which is worth internalising:

1. **Defaults** the type declares in `configureOptions()` (its own baseline).
2. **The parent's options** (each type can inherit and extend its parent's options).
3. **Per-field overrides** you pass in `add()`.

So a custom type sets sane defaults in `configureOptions()`, and callers override only what they need — you never repeat yourself across twenty call sites.

##### Themes are a builder option, too

The visual style is a **form theme** — a bundle of Twig blocks that decide how each widget renders. You set it globally (in config) or per form. We cover themes properly in §10.7; for now just know that `theme` is an option, not a separate mechanism.

---

#### 10.6 Mapping forms to objects

So far our form produced a plain array. Most of the time you want the other mode: **bind the form to an object** and let the framework read and write its properties. This is *data mapping*.

The mechanics:

- A form type declares a `data_class` — the class of the object the form represents.
- Each field maps to a property. The default *property path* is the **field name**; override it with `property_path` when they differ.
- On build, the form reads the object's properties to initialise fields. On submit, it writes submitted values back onto the object via the **PropertyAccess** component, running each field's transformers along the way.

Let's bind our feedback form to a small value object instead of an array — the same pattern you'll use everywhere.

```php
// src/Form/FeedbackForm.php  (a plain value object the form maps to)
namespace App\Form;

class FeedbackForm
{
    public function __construct(
        public string $email = '',
        public string $message = '',
    ) {
    }
}
```

```php
// src/Form/FeedbackType.php
namespace App\Form;

use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\EmailType;
use Symfony\Component\Form\Extension\Core\Type\SubmitType;
use Symfony\Component\Form\Extension\Core\Type\TextareaType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;

class FeedbackType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder
            ->add('email', EmailType::class, ['label' => 'Your email'])
            ->add('message', TextareaType::class, ['label' => 'How can we help?'])
            ->add('send', SubmitType::class, ['label' => 'Send']);
    }

    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefaults([
            'data_class' => FeedbackForm::class,
        ]);
    }
}
```

Now the controller is trivial: create the object, make the form *around it*, and on success the object is already populated.

```php
public function index(Request $request, FormFactoryInterface $formFactory): Response
{
    $feedback = new FeedbackForm();
    $form = $formFactory->create(FeedbackType::class, $feedback);

    if ($request->isMethod('POST')) {
        $form->handleRequest($request);

        if ($form->isSubmitted() && $form->isValid()) {
            // $feedback->email and $feedback->message are now set and typed.
            return $this->redirectToRoute('feedback_thanks');
        }
    }

    return $this->render('feedback/index.html.twig', ['form' => $form]);
}
```

Notice `$form->getData()` is redundant here — `$feedback` *is* the data, mutated in place. That in-place behaviour is exactly what you want when mapping to a Doctrine entity (more on the one caveat in §10.14).

##### Fields that don't map

Not every field corresponds to a property. Sometimes you need a field purely for the UI (a "current password" on an "update email" form, a hidden discriminator, a confirm checkbox). Mark such fields **unmapped**:

```php
->add('currentPassword', PasswordType::class, [
    'mapped' => false,          // never read from / written to the object
    'required' => false,
    'label'  => 'Current password',
])
```

`mapped => false` also means the field's value is *not* taken from the object when the form is first built — it starts empty. That's exactly right for a password field.

##### Why this is safe from mass assignment

This is the security win that's easy to overlook. When you `POST` to a normal controller with `$_POST`, any field the client sends is present in the bag — a determined user could add `role=ADMIN` or `tenantId=other-tenant` and have it applied. A Symfony form **does not do that**: the set of writable properties is precisely the fields you declared in `buildForm()`. Anything not declared is ignored, whether or not the client sent it. Data mapping is allowlist-based by construction, which is the single best reason to route object mutation through forms (or, for APIs, the Serializer with groups — Part V) rather than through raw request parameters.

> In our multi-tenant invoicing app this matters doubly: a form that maps to `Invoice` will only ever touch the fields you listed — never a `tenantId` you forgot to put in the list, and never a `status` you didn't expose.

---

#### 10.7 Form themes and rendering control

A **form theme** is a set of Twig blocks that define the HTML for each widget and each "compound" block. Symfony ships three you can point at directly:

```twig
{% form_theme form 'bootstrap_5_layout.html.twig' %}
{% form_theme form 'form_div_layout.html.twig' %}        {# neutral default #}
```

(Tailwind themes are available via the community `symfony/twig-bridge` integrations and third-party packages; pick whichever matches your stack.) Set one globally in `config/packages/twig.yaml` to apply it to every form:

```yaml
twig:
    form_themes:
        - 'bootstrap_5_layout.html.twig'
```

If you need to *override just one widget* — say, you want a custom `<input type="money">` — you write a small theme that overrides only the block you care about and keep the rest from the base theme:

```twig
{# templates/form/custom_theme.html.twig #}
{% extends 'bootstrap_5_layout.html.twig' %}

{% block amount_widget %}
    <div class="amount-field">
        $ {{ parent() }}
    </div>
{% endblock %}
```

```twig
{% form_theme form 'form/custom_theme.html.twig' %}
```

The block names follow a pattern (`_label`, `_widget`, `_row`, `_errors`) with a `compound` or plain variant. You rarely need to read them all, but when you're debugging "why does my field render like *this*?", grepping the theme for the block name is the fastest path. This is also how you get a consistent, on-brand look without ever hand-writing `<input>` tags again.

---

#### 10.8 Custom field types

`AbstractType` is where forms get powerful. A form type is a **reusable recipe** you can nest, parameterise, and test. There are two shapes you'll write constantly:

- **A *field* type**: a single, self-contained control (or a small group of controls) — e.g., "a line item", "an address", "a tax rate".
- **A *form* type**: the whole screen, composing field types — e.g., "the invoice editor".

Every custom type has up to three methods:

| Method | Purpose |
|--------|---------|
| `buildForm(FormBuilderInterface $builder, array $options)` | Add child fields and wire up form events. |
| `configureOptions(OptionsResolver $resolver)` | Declare the options this type accepts and their defaults. |
| `getBlockPrefix(): string` | The Twig prefix used to resolve theme blocks (defaults to a derived name). |

Let's build the **line item** type for our running project — a composite field that groups a description, a quantity, and a unit price, and maps to a `LineItem` object. It'll later be used as the entry type of a collection (§10.10).

```php
// src/Entity/LineItem.php
namespace App\Entity;

use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class LineItem
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    #[ORM\Column(length: 255)]
    private string $description = '';

    #[ORM\Column]
    private int $quantity = 1;

    #[ORM\Column(precision: 10, scale: 2)]
    private float $unitPrice = 0.0;

    public function __construct()
    {
    }

    // getters/setters and relationships omitted for brevity
}
```

```php
// src/Form/Type/LineItemType.php
namespace App\Form\Type;

use App\Entity\LineItem;
use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\IntegerType;
use Symfony\Component\Form\Extension\Core\Type\MoneyType;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;

class LineItemType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder
            ->add('description', TextType::class, [
                'label' => 'Description',
                'attr'  => ['placeholder' => 'What are you billing for?'],
            ])
            ->add('quantity', IntegerType::class, [
                'label' => 'Quantity',
            ])
            ->add('unitPrice', MoneyType::class, [
                'label'    => 'Unit price',
                'currency' => $options['currency'],
            ]);
    }

    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefaults([
            'data_class' => LineItem::class,
            'currency'   => 'USD',   // a custom option we just invented
        ]);
    }

    public function getBlockPrefix(): string
    {
        return 'line_item';
    }
}
```

Two things to notice:

1. **`currency` is a *custom* option.** Any key you declare in `configureOptions()` becomes a first-class, validated option for this type — callers pass `['currency' => 'EUR']` and get the same guarantees as built-in options. This is how you make types configurable without leaking raw option hashes around.
2. **`data_class` makes it a mapping type.** Because `LineItem`'s property names match the field names, no `property_path` is needed. The type is now a drop-in anywhere a "one line item" is required — including inside a collection.

`getBlockPrefix()` matters when you want a dedicated Twig theme block (e.g. `{% block line_item_row %}`) or a specific CSS class. If you don't override it, Symfony derives a name from the class (`line_item` here anyway, from `LineItemType` → snake_case minus `type`).

> **Testability payoff.** Because a type is just a class, you can unit-test it in isolation: build the form with a known `LineItem`, assert the children exist with the right types, submit a payload, assert the object was populated. No HTTP, no Twig, no database. We'll exercise this in Chapter 22.

---

#### 10.9 Dynamic forms with form events

Static forms can't handle "add a signature field only if the user is a lawyer". Symfony lets you mutate the form *as it processes a request* using **form events**. Three fire in this order during `handleRequest()`:

| Event | When | What `$event->getData()` holds |
|-------|------|-------------------------------|
| `PRE_SET_DATA` | Before the form's data is applied | The **object** (or array) being loaded — good for reacting to *existing* state. |
| `PRE_SUBMIT` | After the raw payload is read, before it's mapped | The **raw submitted array** — good for reacting to *what the user just sent*. |
| `SUBMIT` | After mapping onto the object | The **object**, now populated. |

The classic use is to add or remove fields based on a value. Consider an invoice: corporate clients must supply a purchase-order number, consumers don't. We add the `purchaseOrderNumber` field only when the submission carries a "corporate" flag.

```php
use Symfony\Component\Form\Extension\Core\Type\CheckboxType;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\FormEvent;
use Symfony\Component\Form\FormEvents;

public function buildForm(FormBuilderInterface $builder, array $options): void
{
    $builder
        ->add('clientType', ChoiceType::class, [
            'choices' => ['consumer' => 'Consumer', 'corporate' => 'Corporate'],
        ])
        ->add('isCorporate', CheckboxType::class, [
            'mapped' => false,      // a UI-only toggle
            'required' => false,
            'label' => 'Corporate client (requires PO number)',
        ]);

    // React to the submitted value: reveal the PO field when corporate.
    $builder->addEventListener(FormEvents::PRE_SUBMIT, function (FormEvent $event): void {
        $this->addCorporateFields($event->getForm(), $event->getData());
    });

    // And on the initial (non-submit) render, seed from the existing object.
    $builder->addEventListener(FormEvents::PRE_SET_DATA, function (FormEvent $event): void {
        $this->addCorporateFields($event->getForm(), $event->getData());
    });
}

/**
 * @param array|object|null $data
 */
private function addCorporateFields(FormInterface $form, array|object|null $data): void
{
    $isCorporate = is_array($data)
        ? (bool) ($data['isCorporate'] ?? false)
        : ($data instanceof Invoice && $data->getClientType() === 'corporate');

    if ($isCorporate) {
        $form->add('purchaseOrderNumber', TextType::class, [
            'label' => 'Purchase order number',
        ]);
    }
}
```

The reason you register the logic on **both** `PRE_SET_DATA` and `PRE_SUBMIT` is subtle but important: the field must exist both when the form is *first shown* (so a saved corporate invoice re-displays its PO) and when it's *submitted* (so the submitted PO is mapped). Register it once and the other path silently drops the field. Extracting the shared logic into one private method is the clean way to keep both paths in sync.

**A word on discipline.** Form events are the escape hatch, not the default. The more logic you cram into listeners, the harder the form becomes to reason about and test. Use them for *structure* changes (adding/removing fields) and for *derived* data; keep business rules in your domain and validators (§10.11, Ch. 11). A form whose `buildForm()` is mostly event listeners is a form in trouble.

---

#### 10.10 Collections: fields that grow

An invoice isn't a fixed number of line items — the user adds and removes them. This is the **collection** use case, and it's the single most instructive feature in the component, so let's do it properly.

`CollectionType` renders one *prototype* — a template of the entry type — and lets the front-end clone it. The options that matter:

```php
->add('lineItems', CollectionType::class, [
    'entry_type'    => LineItemType::class, // the type of each entry (§10.8)
    'allow_add'     => true,                // JS may add entries
    'allow_delete'  => true,                // JS may remove entries
    'prototype'     => true,                // expose one prototype for cloning
    'by_reference'  => false,               // ⚠ important with Doctrine
    'label'         => false,
])
```

**`by_reference => false`** is the one that bites people. With the default (`true`), the collection's entries are updated *in place* by reference; with Doctrine, that means the change set is sometimes not detected and your deletes/updates silently vanish. Setting it to `false` makes the collection rebuild its entries, which Doctrine sees as real changes. **Always set `by_reference => false` on a collection of entities.**

##### Rendering and the front-end

The prototype lets JavaScript clone a new row. In Twig, `form.widget.vars.prototype` exposes the prototype markup (with a `__name__` placeholder):

```twig
{{ form_start(form) }}
    {{ form_errors(form) }}

    {{ form_row(form.number) }}
    {{ form_row(form.client) }}
    {{ form_row(form.issueDate) }}
    {{ form_row(form.dueDate) }}

    <fieldset id="line-items">
        <legend>Line items</legend>

        {% for line in form.lineItems %}
            <div class="line-item-row" data-prototype>
                {{ form_row(line.description) }}
                {{ form_row(line.quantity) }}
                {{ form_row(line.unitPrice) }}
                <button type="button" class="btn-remove">Remove</button>
            </div>
        {% endfor %}

        <button type="button" id="add-line-item" class="btn-add">Add line item</button>
    </fieldset>

    {{ form_row(form.notes) }}
    <button type="submit">{{ form_widget(form.save) }}</button>
{{ form_end(form) }}
```

And the (vanilla) JS that makes add/remove work — Symfony just needs the submitted field names to line up with a collection index, which the form's naming strategy already produces:

```js
// assets/line_items.js
import { cloneElement, incrementNames } from './form_utils.js';

const container = document.getElementById('line-items');
const template = document.getElementById('line-item-prototype');
let index = container.querySelectorAll('.line-item-row').length;

document.getElementById('add-line-item').addEventListener('click', () => {
    const row = cloneElement(template);
    row.classList.remove('line-item-row');
    // rename __name__ placeholders to the next index
    row.innerHTML = row.innerHTML.replaceAll('__name__', index);
    container.insertAdjacentElement('beforeend', row);
    index++;
});

container.addEventListener('click', (e) => {
    if (e.target.matches('.btn-remove')) {
        e.target.closest('.line-item-row').remove();
    }
});
```

(The official docs ship a richer `form.js` with `incrementNames`/`cloneElement` helpers; you'll typically vendor that rather than hand-roll it. Asset bundling lives in Chapter 14.)

The key mental model: **you don't manage the array in PHP.** You submit named fields; `CollectionType` reassembles them into an `ArrayCollection` of `LineItem` objects on the parent, in submission order, honouring `allow_add`/`allow_delete`. Your controller never sees the string indices.

> **Deleting entities from a collection.** `allow_delete` removes an entry from the *collection*, but Doctrine still holds the entity. To actually delete removed rows, compare the original and submitted collections (or mark removed items for deletion) before `flush()`ing. We handle that in the running-project walkthrough in §10.13.

---

#### 10.11 Validation integration

Forms and the Validator (Chapter 11) are designed as partners. The relationship is worth stating precisely:

- **Form-level checks** come from *field options* — chiefly `required` (the field may not be empty) and the type's own transformation (e.g., `IntegerType` rejecting `"abc"`). These are enforced by the form itself.
- **Object-level checks** come from *constraints* on the model (`#[NotBlank]`, `#[Range]`, `#[Email]`, …). When the form maps to a constrained object, `isValid()` runs the validator over it and folds the resulting `ConstraintViolation`s into form errors.

So `isValid()` answers both "did the field accept a value of the right shape?" **and** "does the resulting object satisfy its constraints?" — and `form_errors()` renders the union.

```php
if ($form->isSubmitted() && $form->isValid()) {
    // safe to persist
} else {
    // render again; each field's form_errors() now carries messages
}
```

##### `required` vs `NotBlank`

These overlap and it's worth the distinction. `required => true` (a **form** option, default `true`) means "the user must enter *something*" — an empty string fails. `#[NotBlank]` (a **validator** constraint) means "the property must not be blank" — it runs against the *object*. For a mapped field you usually want *both*: `required` keeps the form honest at submit time, `NotBlank` keeps the model honest everywhere (including when it's mutated outside a form). For an optional field, set the form option `required => false` **and** drop the constraint — one without the other leaves a gap.

##### Groups

Constraints can be grouped (e.g., `Default`, `Update`, `Registration`). A form can request a specific group:

```php
$form = $formFactory->create(InvoiceType::class, $invoice, [
    'validation_groups' => ['Default', 'Submit'],
]);
```

This is how you apply "these rules only when saving from the UI" versus "these rules when loading". Chapter 11 goes deep; here it's enough to know the hook exists.

##### Turning validation off

Sometimes you deliberately don't want the validator run by a form — e.g., you'll validate in a different layer, or you're mapping a DTO whose constraints aren't relevant. Pass an empty group:

```php
$form = $formFactory->create(InvoiceType::class, $invoice, [
    'validation_groups' => false,   // form handles only its own field checks
]);
```

Or disable it for a single field with `mapped => false`. Use this sparingly and with a reason; the default (validate everything) is the safe one.

---

#### 10.12 CSRF protection

Every **root** form is CSRF-protected by default. When you render with `form_end()` (or `form_widget(form._token)`), Symfony embeds a hidden token:

```html
<input type="hidden" name="feedback[email]" value="...">
<input type="hidden" name="_token" value="9f2c...">
```

On submit, the framework verifies the token matches the session. A mismatch → a `403` (or, more precisely, a `CsrfTokenMismatchException` handled as a `403`). This is your defence against a malicious site tricking a logged-in user into submitting a form they never intended to fill. It's on **for free** — you do nothing to enable it, and you should do nothing to disable it for browser-facing forms.

You *do* disable it for **stateless** requests — APIs authenticated by JWT or API keys (Part V), webhooks (Chapter 18), or any flow with no session to anchor a token to:

```php
// For an API-style form with no session:
->add(..., [..., 'csrf_protection' => false])
```

or set the option at the form level. If you disable CSRF, make sure you've genuinely got *another* stateless authenticator in place — a form with neither a CSRF token nor a session-based auth is wide open.

> CSRF protection is scoped to **root** forms only. Nested/composite types don't carry a token; the enclosing root form's token covers the whole submission.

---

#### 10.13 Putting it together: the invoice editor

Time to assemble everything into the running project's invoice form: an entity-mapped form type, an `EntityType` for the client, a `CollectionType` of the `LineItemType` we built, a dynamic PO field, and a controller that persists it. This is the template for most data-entry screens in the app.

##### The type

```php
// src/Form/Type/InvoiceType.php
namespace App\Form\Type;

use App\Entity\Client;
use App\Entity\Invoice;
use Symfony\Bridge\Doctrine\Form\Type\EntityType;
use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\DateTimeType;
use Symfony\Component\Form\Extension\Core\Type\SubmitType;
use Symfony\Component\Form\Extension\Core\Type\TextareaType;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\Extension\Core\Type\ChoiceType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;

class InvoiceType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder
            ->add('number', TextType::class, ['label' => 'Invoice #'])
            ->add('client', EntityType::class, [
                'class'        => Client::class,
                'choice_label' => 'name',
                'placeholder'  => '— Select a client —',
            ])
            ->add('issueDate', DateTimeType::class, [
                'widget' => 'single_text',
                'input'  => \DateTimeInterface::class,
            ])
            ->add('dueDate', DateTimeType::class, [
                'widget' => 'single_text',
                'input'  => \DateTimeInterface::class,
            ])
            ->add('clientType', ChoiceType::class, [
                'label'   => 'Client type',
                'choices' => ['consumer' => 'Consumer', 'corporate' => 'Corporate'],
            ])
            ->add('lineItems', CollectionType::class, [
                'entry_type'   => LineItemType::class,
                'allow_add'    => true,
                'allow_delete' => true,
                'prototype'    => true,
                'by_reference' => false,
                'label'        => 'Line items',
            ])
            ->add('notes', TextareaType::class, [
                'label'    => 'Notes',
                'required' => false,
            ])
            ->add('save', SubmitType::class, ['label' => 'Save invoice']);

        // Dynamic PO field for corporate clients (§10.9)
        $addIfCorporate = function (FormBuilderInterface $form, array|object|null $data): void {
            $corporate = is_array($data)
                ? (($data['clientType'] ?? '') === 'corporate')
                : ($data instanceof Invoice && $data->getClientType() === 'corporate');
            if ($corporate) {
                $form->add('purchaseOrderNumber', TextType::class, [
                    'label'    => 'Purchase order #',
                    'required' => false,
                ]);
            }
        };

        $builder->addEventListener(FormEvents::PRE_SET_DATA, fn (FormEvent $e) => $addIfCorporate($e->getForm(), $e->getData()));
        $builder->addEventListener(FormEvents::PRE_SUBMIT, fn (FormEvent $e) => $addIfCorporate($e->getForm(), $e->getData()));
    }

    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefaults([
            'data_class' => Invoice::class,
        ]);
    }
}
```

##### The controller

```php
// src/Controller/InvoiceController.php
namespace App\Controller;

use App\Entity\Invoice;
use App\Form\Type\InvoiceType;
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\Form\FormFactoryInterface;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

final class InvoiceController extends AbstractController
{
    #[Route('/invoices/{id}/edit', name: 'invoice_edit', methods: ['GET', 'POST'])]
    public function edit(Invoice $invoice, Request $request,
                         FormFactoryInterface $formFactory,
                         EntityManagerInterface $em): Response
    {
        $form = $formFactory->create(InvoiceType::class, $invoice);

        if ($request->isMethod('POST')) {
            $form->handleRequest($request);

            if ($form->isSubmitted() && $form->isValid()) {
                // Persist any newly added line items (they arrive detached).
                foreach ($invoice->getLineItems() as $line) {
                    if (null === $line->getId()) {
                        $line->setInvoice($invoice);
                        $em->persist($line);
                    }
                }

                $em->flush();
                $this->addFlash('success', sprintf('Invoice %s updated.', $invoice->getNumber()));

                return $this->redirectToRoute('invoice_show', ['id' => $invoice->getId()]);
            }
        }

        return $this->render('invoice/edit.html.twig', ['form' => $form, 'invoice' => $invoice]);
    }
}
```

A few of the *why* behind the choices, so this reads as a pattern and not a snippet:

- **`by_reference => false` + persisting detached entries.** New line items are submitted as fresh `LineItem` objects; because the collection rebuilds them, they're not yet entities. We `persist()` any whose id is `null` before `flush()`. Removed entries (thanks to `allow_delete`) are simply absent from the collection — add explicit deletion logic there if your rows must be hard-deleted rather than orphaned.
- **`EntityType` for the client.** The client dropdown is bound to `Client::class` and labelled by `name`. In the multi-tenant app, scope that `query` option to the *current tenant's* clients so users can never select another tenant's client — a constraint the form's allowlist alone wouldn't enforce.
- **The dynamic PO field** works across both first-render and submit because the same closure is registered on both `PRE_SET_DATA` and `PRE_SUBMIT`.
- **Validation** runs over `Invoice` and its constraints (Chapter 11 will add things like `#[Assert\LessThanOrEqual]` on `dueDate` relative to `issueDate`); `isValid()` is the single gate before we trust and persist anything.

---

#### 10.14 Best practices and pitfalls

- **Route object mutation through forms (or the Serializer), never `$_POST`.** The allowlist mapping is your mass-assignment defence — don't bypass it. In a multi-tenant app, *additionally* scope `EntityType` queries and any cross-tenant reference to the current tenant.
- **Match `input` to the model's declared type.** Most "why is submit blowing up?" bugs are a wrong `input` (e.g., `DateTimeType` with a `string` property, or `MoneyType` with an `int` cent field).
- **`by_reference => false` on entity collections.** Non-negotiable with Doctrine.
- **Keep `buildForm()` declarative.** Events are for structure changes; business rules live in the domain and validators. If a listener is long, it's a service call in disguise.
- **Prefer `required` + a constraint over either alone.** They guard different layers (form submit vs. object integrity).
- **Watch `EntityType` list size.** Add a `query`/`QueryBuilder` for large tables, or a search-as-you-type pattern. Rendering 100,000 `<option>`s is not a feature.
- **Leave CSRF on** for any session-backed form; turn it off *only* for genuinely stateless flows, and make sure another authenticator covers them.
- **Don't compute totals in the form.** Line-item sums, taxes, currency conversion belong in the domain (§10.9 dynamic fields can *display* a computed total, but the number is computed by the model, not the form). This keeps the form a transport, not a calculator.
- **Make types configurable via `configureOptions`,** not by scattering option arrays — it's what keeps them DRY and testable.

---

#### 10.15 Exercises

Work in the running invoicing app. Each builds on the last.

1. **Edit client.** Create `ClientType` (a mapping type over `Client`) with `name` (required, `TextType`), `email` (`EmailType`), `billingEmail` (`EmailType`, not required), and `isTaxExempt` (`CheckboxType`). Wire an `edit` route that loads the client, handles the form, and persists on success. Confirm a failed submit re-renders with the entered values intact and errors under the failing fields.

2. **Mass-assignment check.** Temporarily add a `role` column to `Client`. *Without* adding `role` to the form, POST a request that includes `client[role]=ADMIN`. Confirm the value is ignored (the form never maps it). Then add the field to the form and confirm it now round-trips — and reflect on why this is the *desired* security behaviour.

3. **Repeated password + groups.** Add a "change password" screen as `PasswordChangeType` using `RepeatedType` (two `PasswordType` fields). Add `#[Assert\Length(min: 10)]` to the new password and apply the `Update` group only on this form. Confirm a short password is rejected with a clear message and that the constraint does *not* fire when the client is edited through the normal `ClientType`.

4. **Dynamic field.** In `InvoiceType`, add a `paymentTerms` (`ChoiceType`) field that appears only when `clientType` is `corporate` *or* a "custom terms" checkbox is ticked. Make sure it works on both first render (editing a saved invoice) and on submit. Add a unit test that builds the form with and without the condition and asserts the field's presence.

5. **Collection hardening.** In the invoice edit flow, make *deleted* line items actually disappear from the database (not just the collection). Add a test that submits an invoice with one fewer line item than was saved and asserts the removed row is gone after `flush()`.

6. **A reusable composite type.** Factor the client's billing details into a `BillingAddressType` (address line, city, postal code, `CountryType`). Use it inside `ClientType`. Then write a small custom form theme that renders the whole address as a two-column grid — overriding only the block(s) you need and inheriting the rest from your base theme.

7. **Performance.** Make the `client` `EntityType` load only the current tenant's clients via the `query` option (`QueryBuilder`). Verify in the Web Profiler (Chapter 23) that the query is scoped and that switching tenants changes the dropdown.

---

**Where we're headed.** We now have a full round trip: describe a form, render it, bind and validate it, and map the result onto the object graph. In Chapter 11 we turn the focus to the *other* half of `isValid()` — the Validator itself — constraints, groups, and custom validators that do the heavy lifting behind the form. Then in Chapter 12 we make sure *only the right users* can reach any of these forms at all.

### Chapter 11. Validation

By the end of this chapter, you will be able to declare and compose validation rules on your entities and DTOs, organize them into logical groups, build compound constraints that enforce business invariants, write your own reusable validators, and translate violation messages for your users.

The running invoicing app from Parts III–V gives us a natural domain: an `Invoice` must carry a unique number, a future due date, at least one line item whose quantities and prices are positive, and a computed total that matches the sum of its items. None of these rules is enforceable at the database level alone, and none belongs in a controller. This is the Validation component's territory.

---

#### 11.1 First Steps: Declaring a Constraint

The Validation component is enabled by default in every full-stack Symfony installation. If you installed via the framework bundle selector (`composer require symfony/skeleton`), confirm that the `symfony/validator` package is present in `composer.json`. It almost certainly is.

A **constraint** is a class that implements `Symfony\Component\Validator\Constraint`. You attach it to a class, a property, or a method using an attribute:

```php
// src/Entity/Invoice.php
use Symfony\Component\Validator\Constraints as Assert;

#[\Attribute(\Attribute::TARGET_CLASS | \Attribute::TARGET_PROPERTY)]
class Invoice
{
    #[Assert\NotBlank(message: 'An invoice number is required.')]
    #[Assert\Length(max: 32)]
    #[Assert\Regex(pattern: '/^INV-\d{4}-\d{5}$/')]
    private string $number;

    #[Assert\NotNull]
    #[Assert\Date]
    private \DateTimeImmutable $issueDate;

    // …
}
```

When Symfony asks the validator to check an `Invoice` instance, it inspects every constraint on the object graph, collects any `ConstraintViolation` objects, and returns them in a `ConstraintViolationList`. No exception is thrown; validation failures are data.

> **Convention in this book.** All constraints are expressed as PHP attributes. The XML and YAML equivalents remain available in `config/validation/` for legacy bundles that still ship them, but new code should prefer attributes for co-location with the class and IDE support.

---

#### 11.2 The Constraint Catalog

Symfony ships roughly 100 built-in constraints. Rather than list every one, the table below groups them by the problems they solve. Full signatures live in the official docs (see *Further Resources*, Appendix D).

| Category | Key constraints | Typical use |
|----------|----------------|-------------|
| **Presence** | `NotBlank`, `NotNull`, `Empty` | Required fields |
| **Strings** | `Length`, `Regex`, `Choice` (string), `Email`, `Url`, `Ip` | Format checks |
| **Numbers** | `GreaterThan`, `GreaterThanOrEqual`, `LessThan`, `LessThanOrEqual`, `Range`, `Number` | Numeric bounds |
| **Dates** | `Date`, `DateTime`, `Time`, `Count` (arrays) | Temporal fields |
| **Collections** | `Collection`, `Count`, `UniqueEntity`, `All`, `Each` | Nested / repeated fields |
| **Objects** | `Valid`, `CascadeValid` (now merged into `Valid`), `Type`, `IsInstance` | Nested entities |
| **Equality / Comparison** | `EqualTo`, `NotEqualTo`, `IdenticalTo` | Cross-field parity |
| **Financial / Data** | `IsTrue`, `IsFalse`, `Currency`, `Luhn`, `Country`, `Language`, `Locale` | Domain-specific formats |
| **Composite** | `Password`, `CardScheme` (Visa/MC/Amex), `ExpressionLanguage`, `DivisibleBy`, `Isbn`, `Issn`, `Luhn`, `NotCompromisedPassword` | Multi-rule checks |

##### 11.2.1 A Note on `Valid` and Cascading

When a property holds a reference to another object, the validator will **not** descend into it by default. The `#[Assert\Valid]` attribute (formerly `CascadeValid`) tells the validator to recursively validate the child:

```php
#[Assert\Valid]
#[Assert\Count(min: 1, minMessage: 'An invoice must have at least one line item.')]
private Collection $items;
```

Without `Valid`, a child `InvoiceItem` that carries its own `#[Assert\GreaterThan(0)]` on `quantity` would be silently skipped.

---

#### 11.3 Validation Groups

Groups let you apply different sets of rules in different contexts—think *registration* vs. *profile update*, or *draft* vs. *submitted*.

##### 11.3.1 Declaring Groups

Pass a string or array of group names to the `groups` option:

```php
use Symfony\Component\Validator\Constraints as Assert;

#[Assert\Groups(['default', 'invoice_submission'])]
class Invoice
{
    #[Assert\NotBlank(groups: ['default', 'invoice_submission'])]
    private string $number;

    // Only enforced when submitting a final invoice, not while editing a draft.
    #[Assert\NotBlank(groups: ['invoice_submission'])]
    #[Assert\GreaterThanOrEqual(new \DateTimeImmutable(), groups: ['invoice_submission'])]
    private \DateTimeImmutable $dueDate;

    #[Assert\Valid(groups: ['default', 'invoice_submission'])]
    #[Assert\Count(min: 1, groups: ['invoice_submission'])]
    private Collection $items;
}
```

The magic string `'default'` is the implicit group that every constraint without an explicit `groups` option belongs to. You always pass `'default'` explicitly when you validate with a specific group list; otherwise Symfony assumes you want only the `'default'` group.

##### 11.3.2 Validating a Specific Group

```php
use Symfony\Component\Validator\Validator\ValidatorInterface;

public function submit(
    Invoice $invoice,
    ValidatorInterface $validator,
): Response {
    $violations = $validator->validate($invoice, null, ['invoice_submission']);

    if (count($violations) > 0) {
        return new JsonResponse([
            'errors' => $violations->getIterator()->getArrayCopy()
                ->map(fn ($v) => ['property' => $v->getPropertyPath(), 'message' => $v->getMessage()]),
        ], Response::HTTP_UNPROCESSABLE_ENTITY);
    }

    // …persist and return 200
}
```

For form-based flows, Symfony's `FormValidator` automatically validates the group that matches the form's `validation_groups` option, so you rarely call the validator by hand in a classic HTML form.

##### 11.3.3 Group Inheritance with `#[Assert\GroupSequence]`

A *group sequence* validates groups in order and **stops at the first group that produces violations**:

```php
#[Assert\GroupSequence(['invoice_submitted', 'invoice_payment'])]
class Invoice
{
    #[Assert\NotBlank(groups: 'invoice_submitted')]
    private string $number;

    #[Assert\NotBlank(groups: 'invoice_payment')]
    private ?Payment $payment;
}
```

This is handy for multi-step wizards: step 1 must be valid before step 2's rules are even evaluated.

---

#### 11.4 Compound Constraints and Cross-Field Rules

Many business rules span more than one property. Symfony offers three progressively more powerful mechanisms.

##### 11.4.1 The `#[Assert\ExpressionLanguage]` Constraint

For a quick cross-field check, embed an ExpressionLanguage (EL) expression:

```php
#[\Attribute]
class Invoice
{
    // …
    #[Assert\ExpressionLanguage(
        expression: 'this.dueDate > this.issueDate',
        message: 'The due date must be after the issue date.',
    )]
    public function validateDateOrder(): void {}
}
```

EL expressions can reference any public property or getter on the object via `this`. The constraint must be placed on a *method* (often a zero-argument, no-op method) to keep the class clean.

##### 11.4.2 Compound Constraint via a Class-Level Validator

For rules that need multiple lines of logic, write a class-level constraint (Section 11.5). The `Invoice` entity would carry:

```php
#[Assert\TotalsMatch]   // our custom constraint
#[Assert\DueDateAfterIssue]
class Invoice
```

##### 11.4.3 The `Assert\Composite` Shortcut (Symfony 7.3+)

If your rule is "all of these constraints must pass *together*", the `Composite` constraint bundles them:

```php
#[Assert\Composite(
    constraints: [
        new Assert\NotBlank(),
        new Assert\Regex(pattern: '/^\d{2}-\d{3}$/'),
        new Assert\Length(min: 6, max: 6),
    ],
    errorOnInitialFailure: true,
)]
private string $taxId;
```

`Composite` is a structural tool; it does **not** evaluate cross-property logic. For that, use EL or a custom validator.

---

#### 11.5 Writing a Custom Constraint

This is the heart of the chapter. We will build two validators that the invoicing app genuinely needs:

1. **`TotalsMatch`** – the invoice's `total` must equal the sum of `quantity × unitPrice` across all items, within a 0.01 tolerance (floating-point safety).
2. **`UniqueInvoiceNumber`** – the invoice number must be unique *per tenant* (our multi-tenant SaaS constraint).

##### 11.5.1 The Constraint Class

```php
// src/Validator/Constraints/TotalsMatch.php
namespace App\Validator\Constraints;

use Symfony\Component\Validator\Constraint;

#[\Attribute(\Attribute::TARGET_CLASS)]
class TotalsMatch extends Constraint
{
    public string $message = 'The invoice total ({{ total }}) does not match the sum of its line items ({{ computed }}).';

    public function validatedBy(): string
    {
        return TotalsMatchValidator::class;
    }
}
```

Key rules:

- Extend `Symfony\Component\Validator\Constraint`.
- Annotate with `#[\Attribute(\Attribute::TARGET_CLASS)]` (or `TARGET_PROPERTY` / `TARGET_METHOD` as appropriate).
- Implement `validatedBy()` (and optionally `getRequiredOptions()`).
- Public properties on the constraint become *options* that users can override when applying the constraint.

##### 11.5.2 The Validator

```php
// src/Validator/TotalsMatchValidator.php
namespace App\Validator;

use App\Entity\Invoice;
use Symfony\Component\Validator\Constraint;
use Symfony\Component\Validator\ConstraintValidator;
use Symfony\Component\Validator\Exception\UnexpectedTypeException;

class TotalsMatchValidator extends ConstraintValidator
{
    public function validate(mixed $value, Constraint $constraint): void
    {
        if (!$constraint instanceof \App\Validator\Constraints\TotalsMatch) {
            throw new UnexpectedTypeException($constraint, TotalsMatch::class);
        }

        if (!$value instanceof Invoice) {
            return; // e.g. validating a child object; not our concern
        }

        $computed = $value->getItems()
            ->map(fn ($item) => $item->getQuantity() * $item->getUnitPrice())
            ->reduce(0.0, fn ($sum, $line) => $sum + $line);

        $tolerance = 0.01;

        if (abs($value->getTotal() - $computed) > $tolerance) {
            $this->context->buildViolation($constraint->message)
                ->setParameter('{{ total }}', number_format($value->getTotal(), 2))
                ->setParameter('{{ computed }}', number_format($computed, 2))
                ->atPath('total')
                ->addViolation();
        }
    }
}
```

The `ConstraintValidator` base class gives you:

- `$this->context` – a `ExecutionContextInterface` for building violations and resolving parameters.
- `$this->context->buildViolation(...)` – returns a `ViolationBuilder` with a fluent API.
- `atPath('property')` – attaches the violation to a specific property so form themes can highlight the right field.
- `setParameter('{{ token }}', $value)` – interpolates placeholders in the `message` template.

> **Tip.** Always check the constraint type and the validated value's type at the top of `validate()`. Symfony may pass a `null` value, a string, or a different object than you expect, especially with `Composite` or when the constraint is reused.

##### 11.5.3 The Multi-Tenant Unique Constraint

Uniqueness that depends on a *second* field (tenant) is the classic reason `#[Assert\UniqueEntity]` is not enough. Here is a production-grade custom validator:

```php
// src/Validator/Constraints/UniqueInvoiceNumber.php
namespace App\Validator\Constraints;

use Symfony\Component\Validator\Constraint;

#[\Attribute(\Attribute::TARGET_PROPERTY)]
class UniqueInvoiceNumber extends Constraint
{
    public string $message = 'An invoice with the number "{{ value }}" already exists in this workspace.';
    public string $ignoreValue = null; // e.g. current entity ID for update flows

    public function validatedBy(): string
    {
        return \App\Validator\UniqueInvoiceNumberValidator::class;
    }
}
```

```php
// src/Validator/UniqueInvoiceNumberValidator.php
namespace App\Validator;

use App\Entity\Invoice;
use App\Repository\InvoiceRepository;
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Component\Validator\Constraint;
use Symfony\Component\Validator\ConstraintValidator;
use Symfony\Component\Validator\Exception\UnexpectedTypeException;

class UniqueInvoiceNumberValidator extends ConstraintValidator
{
    public function __construct(
        private readonly EntityManagerInterface $em,
    ) {}

    public function validate(mixed $value, Constraint $constraint): void
    {
        if (!$constraint instanceof UniqueInvoiceNumber) {
            throw new UnexpectedTypeException($constraint, UniqueInvoiceNumber::class);
        }

        if (null === $value || '' === $value) {
            return; // Let NotBlank handle emptiness.
        }

        $object = $this->context->getObject();

        if (!$object instanceof Invoice) {
            return;
        }

        $tenant = $object->getTenant();

        $invoice = $this->em->getRepository(Invoice::class)
            ->findOneBy([
                'number' => $value,
                'tenant' => $tenant,
            ]);

        // Ignore the current entity when updating.
        if (null === $invoice || $invoice->getId() === $object->getId()) {
            return;
        }

        $this->context->buildViolation($constraint->message)
            ->setParameter('{{ value }}', $value)
            ->atPath('number')
            ->addViolation();
    }
}
```

Because the validator is a service (constructor-injected `EntityManagerInterface`), Symfony auto-configures it: no explicit service definition is needed as long as the `Validator` interface is not implemented. The container injects dependencies automatically.

##### 11.5.4 Applying the Custom Constraints

```php
// src/Entity/Invoice.php
use App\Validator\Constraints\TotalsMatch;
use App\Validator\Constraints\UniqueInvoiceNumber;
use Symfony\Component\Validator\Constraints as Assert;

#[TotalsMatch]
class Invoice
{
    #[Assert\NotBlank]
    #[Assert\Regex(pattern: '/^INV-\d{4}-\d{5}$/')]
    #[UniqueInvoiceNumber]
    private string $number;

    // …
}
```

---

#### 11.6 Constraint Violations: Reading and Presenting Errors

##### 11.6.1 The `ConstraintViolationList`

```php
$violations = $validator->validate($invoice);

if (count($violations) > 0) {
    foreach ($violations as $violation) {
        // $violation->getMessage()        → "The invoice total (150.00) does not match…"
        // $violation->getPropertyPath()   → "total" (or "items[0].quantity")
        // $violation->getInvalidValue()   → the raw PHP value
        // $violation->getParameters()     → ['{{total}}' => '150.00', …]
        // $violation->getCode()           → numeric or string code
    }
}
```

Nested paths use dot-notation: `items[3].unitPrice` means "the `unitPrice` of the fourth item".

##### 11.6.2 Violations in JSON APIs

For the invoicing app's REST endpoints (Chapter 19), map violations to a stable error shape:

```php
use Symfony\Component\Serializer\Normalizer\ConstraintViolationListNormalizer;

// In config/packages/framework.yaml
framework:
    serializer:
        default_context:
            callback: ~
            enable_max_depth: true

// Or handle manually:
private function violationArray(\Traversable $violations): array
{
    return array_map(
        static fn ($v) => [
            'property' => $v->getPropertyPath(),
            'message'  => $v->getMessage(),
            'code'     => $v->getCode(),
        ],
        iterator_to_array($violations)
    );
}
```

Symfony's `ConstraintViolationListNormalizer` (available when the Serializer component is present) does this for you and produces the Hydra error format when paired with API Platform (Chapter 20).

##### 11.6.3 Violations in HTML Forms

When you validate through a Symfony Form, violations are automatically mapped to the corresponding form field and rendered by the chosen form theme (`bootstrap_5`, `tailwind`, etc.). No extra work is required:

```twig
{{ form_row(invoiceForm.items) }}
{# Renders each item's field-level errors under that row #}
```

---

#### 11.7 Translating and Customizing Violation Messages

##### 11.7.1 The Translation Pipeline

Every violation message passes through the **Translation** component (Chapter 27) before reaching the user. The `message` string on a constraint is a *translation key* with parameter placeholders:

```php
#[Assert\NotBlank(message: 'invoice.number.not_blank')]
```

```yaml
# translations/invoice.en.yaml
invoice:
    number:
        not_blank: 'Please enter an invoice number.'
```

If the key is not found, the raw key string is displayed—useful during development. In production, always ship complete translation catalogs for every locale you support.

##### 11.7.2 Parameter Placeholders

Use `{{ param }}` in the message and `setParameter()` in the validator (or the constraint option) to interpolate:

```php
#[Assert\Length(
    min: 5,
    max: 10,
    minMessage: 'The code must be at least {{ limit }} characters.',
    maxMessage: 'The code must be at most {{ limit }} characters.',
)]
```

Built-in placeholders include `{{ value }}`, `{{ limit }}`, `{{ min }}`, `{{ max }}`, `{{ total }}`, `{{ each }}`, `{{ number }}`, and `{{ property }}`. Custom validators can inject any `{{ key }}` they choose.

##### 11.7.3 Overriding Messages Globally

If you need to change the wording of a built-in constraint across the whole app without editing every entity, configure a *message template override* in `config/packages/translation.yaml` or use the `validator.translation_domain` option. The most common approach is simply to set a `message` on the constraint where you apply it.

##### 11.7.4 Violation Codes

Every violation carries a numeric or string **code**. Built-in constraints use well-known integer codes (documented in the Constraint class constants). Custom validators can set a string code for programmatic handling:

```php
$this->context->buildViolation($constraint->message)
    ->setCode('INVOICE_TOTAL_MISMATCH')
    ->addViolation();
```

Clients or background jobs can then branch on the code without parsing human-readable text.

---

#### 11.8 Validation in Non-Form Contexts

Forms are the most visible consumer, but the validator is a general-purpose tool.

##### 11.8.1 Validating a Raw Array (e.g. API Payloads)

```php
use Symfony\Component\Validator\Constraints\Length;
use Symfony\Component\Validator\Constraints\NotBlank;
use Symfony\Component\Validator\Constraints\Collection;
use Symfony\Component\Validator\Constraints\Email;
use Symfony\Component\Validator\Constraints\Choice;

$constraints = new Collection([
    'fields' => [
        'email'    => new NotBlank(),
        'email'    => new Email(),
        'plan'     => new Choice(choices: ['free', 'pro', 'enterprise']),
        'fullName' => new Length(min: 2, max: 120),
    ],
    'allowExtraFields' => false,
    'allowExtraFieldsMessage' => 'Unknown field "{{ name }}" is not accepted.',
]);

$violations = $validator->validate($payload, $constraints);
```

This is the pattern used by the invoicing app's public webhook endpoint (Chapter 18) to validate incoming Stripe events before processing them.

##### 11.8.2 Validating with `#[Assert\All]` / `#[Assert\Each]`

For arrays of homogeneous values:

```php
#[Assert\All(new Assert\Regex(pattern: '/^\d{5}$/'))]
private array $zipCodes;

// Or, on a Collection property:
#[Assert\Each(new Assert\GreaterThan(0))]
private Collection $amounts;
```

##### 11.8.3 Validating in a Messenger Handler (Chapter 17)

A message handler that receives a `CreateInvoice` DTO can validate the payload before enqueuing side-effects:

```php
#[AsMessageHandler]
final class CreateInvoiceHandler
{
    public function __construct(
        private readonly ValidatorInterface $validator,
    ) {}

    public function __invoke(CreateInvoice $message): void
    {
        $violations = $this->validator->validate($message);
        if (count($violations) > 0) {
            throw new InvalidMessageException($violations);
        }
        // …
    }
}
```

`InvalidMessageException` is a Messenger-specific exception that causes the message to be routed to the dead-letter queue rather than retried.

---

#### 11.9 Configuration and Performance

##### 11.9.1 Metadata Cache

By default, Symfony caches the *metadata* (the list of constraints for each class) in the framework's cache pool. In production, this means constraints are parsed once and served from `var/cache/prod/`. Clear the cache after changing constraints:

```bash
php bin/console cache:clear
```

##### 11.9.2 Mapping Options in `config/packages/framework.yaml`

```yaml
framework:
    validation:
        email_validation_mode: html5        # html5 | html5-allow-free-named | strict
        enable_annotations: false           # attributes are the default in 7.4+
        validation_groups: ['default']      # default group for all validators
        not_compromised_password:
            enabled: true                   # check HaveIBeenPwned (requires network)
```

##### 11.9.3 Performance Considerations

- **Avoid `#[Assert\UniqueEntity]` in high-throughput write paths** when a database unique index suffices. The validator issues a SELECT per field; the DB index enforces the invariant at commit time with less overhead. Use the validator when you need a friendly, localized error message *before* the query round-trip.
- **Batch-validate collections.** `validate()` on a single object is O(1) in terms of DB hits for pure-format constraints. Custom validators that touch the database (like `UniqueInvoiceNumber`) scale linearly with the number of objects—validate them per-object or in a batch.
- **Keep expression-language expressions short.** EL is interpreted, not compiled. For complex logic, prefer a custom validator.

---

#### 11.10 Tying It Together: The `Invoice` Entity

Putting the chapter's techniques into the running invoicing app:

```php
// src/Entity/Invoice.php
namespace App\Entity;

use App\Validator\Constraints\TotalsMatch;
use App\Validator\Constraints\UniqueInvoiceNumber;
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;

#[ORM\Entity]
#[ORM\Table(name: 'invoices')]
#[TotalsMatch]
#[Assert\GroupSequence(['invoice_draft', 'invoice_final'])]
class Invoice
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    #[ORM\ManyToOne(targetEntity: Tenant::class, inversedBy: 'invoices')]
    #[ORM\JoinColumn(nullable: false)]
    private Tenant $tenant;

    #[ORM\Column(length: 32)]
    #[Assert\NotBlank(groups: ['invoice_draft', 'invoice_final'])]
    #[Assert\Regex(pattern: '/^INV-\d{4}-\d{5}$/')]
    #[UniqueInvoiceNumber]
    private string $number;

    #[ORM\Column(type: 'datetime_immutable')]
    #[Assert\NotNull(groups: ['invoice_draft', 'invoice_final'])]
    private \DateTimeImmutable $issueDate;

    #[ORM\Column(type: 'datetime_immutable', nullable: true)]
    #[Assert\NotNull(groups: ['invoice_final'])]
    #[Assert\GreaterThan(value: 'now', groups: ['invoice_final'],
                        message: 'The due date must be in the future.')]
    private ?\DateTimeImmutable $dueDate = null;

    #[ORM\OneToMany(mappedBy: 'invoice', targetEntity: InvoiceItem::class, cascade: ['persist', 'remove'])]
    #[Assert\Valid(groups: ['invoice_draft', 'invoice_final'])]
    #[Assert\Count(min: 1, groups: ['invoice_final'],
                  minMessage: 'An invoice must contain at least one line item.')]
    private Collection $items;

    #[ORM\Column(type: 'decimal', precision: 10, scale: 2)]
    #[Assert\GreaterThan(0, groups: ['invoice_draft', 'invoice_final'])]
    private string $total;

    // …getters / setters / addItem / removeItem
}
```

```php
// src/Entity/InvoiceItem.php
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;

#[ORM\Entity]
class InvoiceItem
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    #[ORM\ManyToOne(targetEntity: Invoice::class, inversedBy: 'items')]
    #[ORM\JoinColumn(nullable: false, onDelete: 'CASCADE')]
    private Invoice $invoice;

    #[ORM\Column(length: 512)]
    #[Assert\NotBlank]
    #[Assert\Length(max: 512)]
    private string $description;

    #[ORM\Column(type: 'integer')]
    #[Assert\GreaterThan(0, message: 'Quantity must be at least 1.')]
    private int $quantity;

    #[ORM\Column(type: 'decimal', precision: 10, scale: 2)]
    #[Assert\GreaterThanOrEqual(0)]
    private string $unitPrice;

    // …
}
```

A single call—`$validator->validate($invoice, null, ['invoice_final'])`—walks the entire object graph, checks format constraints, runs the custom `TotalsMatch` and `UniqueInvoiceNumber` validators, and returns a flat list of violations ready for the response.

---

#### 11.11 Common Pitfalls

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Forgetting `#[Assert\Valid]` on a relation | Child-entity constraints silently ignored | Add `#[Assert\Valid]` to the `OneToMany` / `ManyToOne` property |
| Validating with `'default'` group only | Group-specific constraints never run | Pass the correct group(s) to `validate()` |
| Using `NotBlank` on a nullable DB column without `#[Assert\Null]` alternative | `null` fails `NotBlank` even when the field is optional | Use `#[Assert\NotNull]` + `#[Assert\NotBlank]` only when truly required; leave the column nullable and omit the constraint otherwise |
| Returning an exception instead of collecting violations | HTTP 500 instead of 422 | Always check `count($violations) > 0` and return a structured error response |
| Forgetting to register the validator as a service (pre-7.0) | `No validator found for …` | In 7.4+, autoconfiguration handles this; for older code, tag the service with `validator.constraint_validator` |
| Using `{{ }}` in a message without `setParameter` | Literal `{{ value }}` shown to the user | Always call `->setParameter('{{ token }}', $value)` for custom placeholders |
| Validating a DTO in a Messenger handler *after* the handler has already started side-effects | Partial state changes on invalid input | Validate at the top of `__invoke()`, before any mutation |

---

#### 11.12 Exercises

1. **Constraint catalog warm-up.** Add the following constraints to the `Customer` entity from Part III: `fullName` must be 2–120 characters, `email` must be a valid RFC 5322 address, `taxId` must match the regex `/^\d{2}-\d{3}-\d{4}$/` and must be unique within the customer's tenant (write a `UniqueTaxId` constraint modeled on `UniqueInvoiceNumber`). Write a functional test that submits a customer form with a duplicate tax ID and asserts the specific violation message.

2. **Group sequence wizard.** The invoicing app has a three-step "New Invoice" wizard: (1) header (number, dates), (2) line items, (3) payment terms. Model the steps as a `GroupSequence` and write a controller that advances to the next step only when the current group passes validation. Return the violation list as JSON when a step fails.

3. **Custom validator with a service dependency.** Write a `ValidDiscount` constraint that ensures a `Discount` entity's percentage is between 0 and 100 **and** that the discount's effective date range does not overlap with any other active discount on the same `Tenant` (query the `DiscountRepository`). The validator must accept a `Tenant` via the entity's relation rather than a magic number. Include unit tests that mock the repository.

4. **Translating violations.** Create `translations/invoice.fr.yaml` and `translations/invoice.de.yaml` catalogs for every violation message in the `Invoice` and `InvoiceItem` entities. Write a feature test (or functional test) that sets the locale to `fr`, submits an invalid invoice, and asserts the French message appears in the JSON response.

5. **Array validation for a webhook.** The Stripe webhook endpoint (Chapter 18) receives a JSON payload. Build a `Collection` constraint that validates the top-level `type` is one of `invoice.paid`, `invoice.failed`, `invoice.created`, and that the nested `data.object` object contains at least `id`, `total_amount`, and `status`. Write a unit test with three fixtures: a valid payload, a missing required field, and an unknown `type`.

---

#### Summary

The Validation component is small in API surface but powerful in composition. Constraints are plain attributes; groups are strings; custom validators are a two-class pair. By the end of this chapter you should be comfortable:

- Choosing the right built-in constraint (or writing a custom one) for any business rule.
- Organizing constraints into groups and sequences for multi-step or role-specific flows.
- Reading `ConstraintViolation` objects and mapping them to your response format.
- Translating violation messages so your SaaS users see them in their preferred language.
- Integrating validation into forms, API endpoints, Messenger handlers, and webhook receivers.

In the next chapter we move to **Security**—the layer that decides *who* can create, read, update, and delete the very invoices we just learned to validate.

I'll research the current state of Symfony's security system to make sure the chapter is accurate for Symfony 7.4 LTS / 8.x, then write it in the book's style.



Let me fetch the official docs to ground the configuration details accurately.



Let me confirm the session fixation and full config structure from the official reference.



I have most details. Let me verify the exact `login_throttling` YAML structure to be precise.



I have everything I need, grounded in the current Symfony 7.4/8.x docs. Here is Chapter 12.

---

### Chapter 12 — Security

By the end of Part III our invoicing SaaS can render invoices and accept form input. That means it has *state that matters*: a tenant's invoices, its customers, its billing settings. The moment an application holds more than a public brochure, security stops being a "nice to have" and becomes the load-bearing wall of the architecture.

This chapter is the most "systems-level" chapter so far. We will do three things:

1. Build the **authentication** story — how a user proves who they are — using a custom form-login authenticator, the modern Passport-based API, and a real `User` entity for our multi-tenant app.
2. Build the **authorization** story — what an authenticated user may do — with roles, `access_control`, and (the more interesting tool) **voters**, which let us express rules like *"a member can edit their own tenant's unpaid invoices but never a paid one, and never another tenant's invoice at all."*
3. Harden the whole thing against the attacks that actually happen in production: **brute-force**, **session fixation**, **user enumeration**, and the "one password isn't enough" problem solved with **two-factor authentication**.

You should already have a `User` concept in mind, even if it's not built yet. We will build it here.

> **Conventions note.** Everything in the rest of this chapter uses **attribute-based routing** and **constructor promotion** (PHP 8.2+). Security *policy* — firewalls, roles, access rules — is always configured in `config/packages/security.yaml`; there is no attribute equivalent for firewalls, and you should not try to force one. Attributes are for *your* routes and controllers; YAML is for the security kernel.

---

#### 12.1 The mental model: authentication vs. authorization

These two words get blurred constantly, and the confusion leads to bugs. Keep them separate.

- **Authentication** answers *"who are you?"* It verifies a set of **credentials** (a password, an API key, a TOTP code) and, if they are valid, produces an **`AuthenticationToken`** that represents the authenticated principal.
- **Authorization** answers *"what are you allowed to do with that identity?"* It runs on a token that already exists and asks questions like *"does this user have `ROLE_ADMIN`?"* or *"can this user delete this invoice?"*

Everything in this chapter is one of those two questions. A useful way to picture the request is as a pipeline that the Security system hooks into:

```
            ┌─────────────────────────────────────────────────────────────┐
            │                       The request arrives                    │
            └───────────────────────────┬─────────────────────────────────┘
                                        ▼
                       ┌────────────────────────────────┐
                       │  Firewall(s) pick the listener │   ← picks which
                       │  chain applies to this URL     │     authenticator runs
                       └─────────────────┬──────────────┘
                                         ▼
              ┌──────────────────────────────────────────────────┐
              │  AUTHENTICATION                                  │
              │  supports() → authenticate() → validate → token  │
              └──────────────────────────────────────────────────┘
                                         │
                              (a token exists now)
                                         ▼
              ┌──────────────────────────────────────────────────┐
              │  AUTHORIZATION                                   │
              │  access_control rules, isGranted(), voters       │
              └──────────────────────────────────────────────────┘
                                         ▼
                       ┌────────────────────────────────┐
                       │  Controller runs (or 401/403)  │
                       └────────────────────────────────┘
```

Two small vocabulary items will appear constantly:

- **The firewall** is not a network device. It is *the named set of listeners and configuration that applies to a range of URLs*. You can have several firewalls; each request matches exactly one, and that one determines *how* the user is authenticated and *whether* they may even reach the resource.
- **The token** (an `AuthenticationToken`) is the object stored in the session that says *"a user is currently logged in, and here is who."* When there is no token (or an anonymous one), the user is **anonymous**.

With that in your head, let's look at the single file that ties it together.

---

#### 12.2 Anatomy of `security.yaml`

Almost all of your security *policy* lives in `config/packages/security.yaml`. It has a few top-level keys, each doing one job:

| Key | Job |
| --- | --- |
| `providers` | *Where do users come from?* Maps a named provider to a source (a Doctrine entity, memory, a custom class). |
| `firewalls` | *How is each URL range authenticated?* Defines the listener chain, the entry point (what a 401 does), and per-firewall extras like throttling and remember-me. |
| `access_control` | *Which roles may reach which paths?* Ordered, first-match-wins rules for authorization at the URL level. |
| `role_hierarchy` | Which roles *imply* other roles (e.g. `ROLE_ADMIN` implies `ROLE_BILLING`). |
| `password_hashers` | *How are passwords hashed?* Per user class. |

Here is a realistic starting point for our invoicing app:

```yaml
# config/packages/security.yaml
security:
    # Where do we load users from?
    password_hashers:
        App\Entity\User:
            algorithm: auto            # argon2id on modern systems; see §12.7

    # role_hierarchy: an admin is implicitly a billing manager and a member.
    role_hierarchy:
        ROLE_ADMIN: [ROLE_BILLING, ROLE_MEMBER]
        ROLE_BILLING: [ROLE_MEMBER]

    # access_control is checked in order; the FIRST match wins.
    # Being specific first, general last, is the #1 rule.
    access_control:
        - { path: ^/login, roles: PUBLIC_ACCESS }          # login page is public
        - { path: ^/register, roles: PUBLIC_ACCESS }
        - { path: ^/admin, roles: ROLE_ADMIN }             # admin console
        - { path: ^/billing, roles: ROLE_BILLING }         # billing area
        - { path: ^/, roles: ROLE_MEMBER }                 # everything else needs a member

    # ...
    providers:
        app_user_provider:
            entity:
                class: App\Entity\User
                property: email         # we look users up by their email

    # ...
    firewalls:
        dev:
            # Never secure the profiler, web debug toolbar, or static assets.
            pattern: ^/(_(profiler|wdt)|css|images|js)/
            security: false

        main:
            lazy: true                 # §12.3 — don't load the user until we must
            provider: app_user_provider
            form_login:                # §12.5 — we'll replace this with a custom
                login_path: app_login  # authenticator shortly
                check_path: app_login
                enable_csrf: true
            logout:
                path: app_logout
                target: app_login
            login_throttling:          # §12.12 — brute-force protection
                max_attempts: 5
                interval: '15 minutes'
```

A few things to internalize:

- **`access_control` is first-match-wins.** Put the most specific rules *first*. If `- { path: ^/, roles: ROLE_MEMBER }` came before `- { path: ^/admin, roles: ROLE_ADMIN }`, the admin rule would be unreachable and every request would be checked against `ROLE_MEMBER`. This ordering mistake is the single most common `security.yaml` bug.
- **`PUBLIC_ACCESS`** and **`IS_AUTHENTICATED_REMEMBERED`** are not real roles you must define; they are built-in tokens the security system understands.
- You can always inspect the *effective* configuration after everything has been merged:

```bash
$ php bin/console debug:firewall
$ php bin/console debug:firewall main --events   # see every listener in the chain
$ php bin/console debug:config security          # the fully-resolved config
```

`debug:firewall main --events` is, frankly, the most underused debugging tool in Symfony. When authentication "isn't working," it shows you exactly which listeners are on the chain and in what order.

---

#### 12.3 Firewalls in depth

A **firewall** is a named bucket of security behavior that applies to a set of URLs. Every incoming request is matched against the firewalls, and the *first* firewall whose `pattern` matches handles it.

```yaml
firewalls:
    main:
        pattern: ^/          # matches everything not claimed by an earlier firewall
        lazy: true
        provider: app_user_provider
        # ...
```

The `pattern` is a regular expression matched against the request path. A handful of patterns you will write constantly:

```yaml
# Static assets & debug tooling — never secure these
pattern: ^/(_(profiler|wdt)|css|images|js)/

# A public marketing site, but protect an /app dashboard
pattern: ^/app

# An API with a completely different (stateless) auth strategy
pattern: ^/api
```

##### `lazy` vs. `stateless`

Two firewall flags deserve attention because they change *when* work happens and *how* sessions behave:

- **`lazy: true`** — the firewall does *not* load the `User` entity from the database on every request. It only knows a user is logged in; it lazily fetches the full `User` the first time something (a voter, a template, a controller) actually needs it. This is a big win for apps where most of the session-carrying traffic does not touch the user. Enable it by default; disable it only if you need the user object on every single request.

- **`stateless: true`** — the firewall does **not** read or write the session at all. This is the correct setting for **API firewalls** authenticated by a bearer token or API key: each request carries its own credentials, and there is no per-user server-side session. (We build one of these in Part V; the concept lands here.)

```yaml
firewalls:
    api:
        pattern: ^/api
        stateless: true          # no session; credentials travel in each request
        json_login:              # Part V goes deep on this
            check_path: /api/login
        provider: app_user_provider
```

> **Tip.** A common multi-firewall layout for a SaaS is: a `dev` firewall (`security: false`) for assets and the profiler, a `main` (session-based) firewall for the browser app, and a `stateless` `api` firewall for the JSON endpoints. Each request matches exactly one; the ordering in the file decides the priority when patterns could overlap.

##### Restricting firewalls by host and method

`pattern` only looks at the path. If you need finer matching, a firewall can be restricted further by **host** and **HTTP method** (for example, "apply the API firewall only to `api.example.com` and only to non-`GET` requests"). The mechanism lives at the firewall level and lets one firewall serve a subdomain's API while another serves the main site on the same host.

##### The `dev` firewall and `security: false`

Setting `security: false` on a firewall means *"do nothing with security for these URLs"* — no authentication, no `access_control`. It exists so you can serve the web profiler, the debug toolbar, and static files without forcing every asset request through the authentication pipeline. Leave one of these in your app; it is not a cop-out, it is the intended design.

---

#### 12.4 Authenticators: the modern model

Since Symfony 5.3 the security system is built around **authenticators**. Each authenticator is a small class that knows how to:

1. decide **whether it applies** to the current request (`supports()`),
2. **extract credentials** and turn them into a **Passport** (`authenticate()`),
3. react to **success** (`onAuthenticationSuccess()`), and
4. react to **failure** (`onAuthenticationFailure()`).

The framework ships several you can use straight from YAML (`form_login`, `http_basic`, `http_digest`, `json_login`, `form_login_ldap`, `http_basic_ldap`, `access_token`, `login_link`, and `x509`). When none of them fit — and in our app we want a custom login experience — you write your own.

> **Historical note.** You may see the word **Guard** in older tutorials. The "security guard" system was the 2019–2021 way to do custom auth, and it is **removed** as of Symfony 7.0. Everything it did is now done with authenticators. If a blog post shows `GuardAuthenticatorInterface`, close the tab; it is a decade old.

##### Passports, badges, and credentials

The heart of the modern model is the **`Passport`**. Think of it as a document you hand to the security system that says *"here is the user, here is the credential to verify, and here are a few extra flags."*

A `Passport` is composed of:

- A **`UserBadge`** — *which* user (by a unique identifier like an email). The security system uses the configured **user provider** to load the actual `User` object from that identifier.
- **Credentials** — *what* to verify:
  - **`PasswordCredentials`** — a plaintext password, verified against the stored hash using the configured hasher.
  - **`CustomCredentials`** — a closure that does arbitrary verification (e.g. compare an API token).
  - Or, with a **`SelfValidatingPassport`**, *no* credentials at all (the badge alone is enough — typical for API keys where the "credential" is the lookup itself).
- Optional **badges** that change behavior:
  - **`RememberMeBadge`** — enable remember-me for this login.
  - **`PasswordUpgradeBadge`** — re-hash the stored password with a new algorithm if the old one is weaker (see §12.7).
  - **`CsrfTokenBadge`** — verify a CSRF token as part of this passport.

A minimal example of the pieces assembling:

```php
use Symfony\Component\Security\Http\Authenticator\Passport\Badge\UserBadge;
use Symfony\Component\Security\Http\Authenticator\Passport\Passport;
use Symfony\Component\Security\Http\Authenticator\Passport\SelfValidatingPassport;
use Symfony\Component\Security\Http\Authenticator\Passport\Credentials\PasswordCredentials;
use Symfony\Component\Security\Http\Authenticator\Passport\Credentials\CustomCredentials;

// Username + password (the user provider loads the User, the hasher checks the password):
return new Passport(
    new UserBadge($email),
    new PasswordCredentials($plaintextPassword)
);

// API key (no separate "credential" to check — the lookup *is* the check):
return new SelfValidatingPassport(new UserBadge($apiKey));
```

> **Security detail worth knowing.** The user identifier you pass to `UserBadge` is capped at 4096 characters. This is a deliberate guard against **session-storage-flooding** attacks where an attacker feeds a huge username to balloon the session. It is one of many small, boring defenses that are exactly what "secure by default" should mean.

##### The authenticator lifecycle

For a request that a firewall has claimed, Symfony runs each configured authenticator:

1. **`supports(Request $request): ?bool`** — quick, cheap. Return `false` and this authenticator is skipped entirely. This is how one firewall can host several authenticators (e.g. `json_login` and a custom API-key authenticator) without them trampling each other.
2. **`authenticate(Request $request): Passport`** — do the real work: read credentials, build the passport. If the credentials are *obviously* bad, throw a `CustomUserMessageAuthenticationException`.
3. **Validation** — the framework checks the passport (verifies the password, the CSRF badge, etc.). If it fails, an `AuthenticationException` is produced.
4. **`onAuthenticationSuccess(...)`** / **`onAuthenticationFailure(...)`** — your hook to return a redirect, a JSON response, or `null` to "let the request continue."

The two response methods have a subtle but important contract:

- Returning a **`Response`** (e.g. a redirect or a `401` JSON) **stops** the request and sends that response.
- Returning **`null`** means *"let the current request continue."* For a login *form*, that is what you want on failure — the controller re-renders the form with the error. For an *API*, you'd return a `401` JSON instead.

Two extra interfaces round out the picture:

- **`AuthenticationEntryPointInterface`** — defines the response sent to *start* authentication, i.e. what a user gets when they hit a protected page unauthenticated (normally a redirect to `/login`). Form-login already implements this; a custom authenticator should too if it's meant to trigger a login flow.
- **`InteractiveAuthenticatorInterface`** — marks an authenticator as one where the user *actively* logged in. Implementing it makes Symfony dispatch an **`InteractiveLoginEvent`**, which is the correct trigger for things like "log the login, refresh the last-login timestamp, or *require 2FA now*." (We use it in §12.11.)

> **Caution.** When an `AuthenticationException` reaches you, **never** render `$exception->getMessage()` to the user. That message can contain internal detail you do not want to expose. Render the safe pair instead: `$exception->getMessageKey()` plus `$exception->getMessageData()` (and translate them). If you need to control the message yourself, throw a `CustomUserMessageAuthenticationException`.

---

#### 12.5 Walkthrough: a custom form-login authenticator

This is the authenticator that powers the actual login of our invoicing app. We could keep the plain `form_login` from §12.2, but building it explicitly is (a) how you'd add custom behavior, and (b) the shape of nearly every real login.

Start with the generator, which scaffolds the class *and* wires the config:

```bash
$ php bin/console make:security:form
```

That creates `src/Security/EmailAuthenticator.php` and updates `security.yaml`. Here is the result, annotated:

```php
// src/Security/EmailAuthenticator.php
namespace App\Security;

use App\Repository\UserRepository;
use Symfony\Component\HttpFoundation\RedirectResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
use Symfony\Component\Security\Core\Exception\AuthenticationException;
use Symfony\Component\Security\Core\Exception\CustomUserMessageAuthenticationException;
use Symfony\Component\Security\Http\Authenticator\AbstractLoginFormAuthenticator;
use Symfony\Component\Security\Http\Authenticator\Passport\Badge\CsrfTokenBadge;
use Symfony\Component\Security\Http\Authenticator\Passport\Badge\RememberMeBadge;
use Symfony\Component\Security\Http\Authenticator\Passport\Badge\UserBadge;
use Symfony\Component\Security\Http\Authenticator\Passport\Passport;
use Symfony\Component\Security\Http\Authenticator\Passport\SelfValidatingPassport;
use Symfony\Component\Security\Http\Authenticator\Passport\Credentials\PasswordCredentials;
use Symfony\Component\Security\Http\EntryPoint\AuthenticationEntryPointInterface;
use Symfony\Component\Security\Http\Util\TargetPathTrait;

class EmailAuthenticator extends AbstractLoginFormAuthenticator
    implements AuthenticationEntryPointInterface
{
    use TargetPathTrait;   // gives us saveTargetPath()/getTargetPath()

    public function __construct(private readonly UserRepository $users) {}

    // 1) Only act on the /login route (this method is the "default supports()").
    public function getLoginUrl(): string
    {
        return $this->urlGenerator->generate('app_login');
    }

    // 2) Extract credentials from the POST and build a Passport.
    public function authenticate(Request $request): Passport
    {
        $email = (string) $request->request->get('email', '');
        $password = (string) $request->request->get('password', '');

        // Remember what they typed so we can prefill the field on failure.
        $request->getSession()->set('_security_last_email', $email);

        // The UserBadge carries the identifier; the provider loads the User.
        // The CsrfTokenBadge makes the CSRF check part of *this* passport.
        return new Passport(
            new UserBadge($email),
            new PasswordCredentials($password),
            [
                new CsrfTokenBadge('authenticate', $request->request->get('_csrf_token')),
                new RememberMeBadge(),
            ]
        );
    }

    // 3) On success: redirect to where they originally wanted to go.
    public function onAuthenticationSuccess(Request $request, TokenInterface $token, string $firewallName): ?Response
    {
        // TargetPathTrait remembered the "intended" URL when the 401 fired.
        if ($targetPath = $this->getTargetPath($request->getSession(), $firewallName)) {
            return new RedirectResponse($targetPath);
        }

        // Default landing spot for the app.
        return new RedirectResponse($this->urlGenerator->generate('dashboard'));
    }

    // 4) On failure: re-render the login form with the error, and clear the target.
    public function onAuthenticationFailure(Request $request, AuthenticationException $exception): Response
    {
        $request->getSession()->remove('_security_last_email');
        $request->getSession()->remove($this->targetPathParameter($firewallName));

        return $this->urlGenerator->generate('app_login', [
            // Never use getMessage() — see the Caution in §12.4.
            'error' => $exception->getMessageKey(),
        ]);
    }

    public function start(Request $request, ?AuthenticationException $authException = null): Response
    {
        // Called when an anonymous user hits a protected page: send them to /login.
        return new RedirectResponse($this->urlGenerator->generate('app_login'));
    }
}
```

The corresponding firewall now references the custom authenticator explicitly:

```yaml
# config/packages/security.yaml
firewalls:
    main:
        lazy: true
        provider: app_user_provider
        custom_authenticators:
            - App\Security\EmailAuthenticator   # ← ours
        remember_me:
            secret: '%kernel.secret%'
            lifetime: 604800   # 7 days
            name: REM
        logout:
            path: app_logout
            target: app_login
        login_throttling:
            max_attempts: 5
            interval: '15 minutes'
```

The login *route* and *template* are plain Symfony. Note the `form_login` fields — `email` and `password` — must match what the authenticator reads via `$request->request->get(...)`.

```php
// src/Controller/SecurityController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;
use Symfony\Component\Security\Http\Authentication\AuthenticationUtils;

class SecurityController extends AbstractController
{
    #[Route('/login', name: 'app_login')]
    public function login(AuthenticationUtils $authenticationUtils): Response
    {
        // If we're already logged in, send them home.
        if ($this->getUser()) {
            return $this->redirectToRoute('dashboard');
        }

        return $this->render('security/login.html.twig', [
            // Last entered email (if any) + the error from the last failed attempt.
            'last_email' => $authenticationUtils->getLastUsername(),
            'error'      => $authenticationUtils->getLastAuthenticationError(),
        ]);
    }

    #[Route('/logout', name: 'app_logout', methods: ['POST'])]
    public function logout(): void
    {
        // The firewall's logout listener handles everything.
        // (This method never actually runs.)
    }
}
```

```twig
{# templates/security/login.html.twig #}
<form method="post" action="{{ path('app_login') }}">
    {% if error %}
        <div class="alert alert-danger">{{ error.messageKey|trans(error.messageData, 'security') }}</div>
    {% endif %}

    <label for="email">Email</label>
    <input type="email" id="email" name="email" value="{{ last_email }}" required autofocus>

    <label for="password">Password</label>
    <input type="password" id="password" name="password" required>

    {# The CsrfTokenBadge in the authenticator expects this token id #}
    <input type="hidden" name="_csrf_token"
           value="{{ csrf_token('authenticate') }}">

    <button type="submit">Log in</button>
</form>
```

Three things to notice in that flow:

- **CSRF is enforced by the `CsrfTokenBadge`**, not by the template. The template merely *emits* a token; the badge *verifies* it as part of the passport. If the token is missing or wrong, authentication fails before any user is loaded.
- **The target path** is saved at the moment of the 401 (when an anonymous user tried to reach a protected page) and consumed by `TargetPathTrait` in `onAuthenticationSuccess`. That's why you can type `/admin/settings` while logged out and, after logging in, land exactly there.
- **`AuthenticationUtils`** is the friendlier, testable way to reach the "last username / last error" from the session inside a controller, rather than poking the session directly.

> **Tip.** For interactive logins like this one, also make the authenticator implement `InteractiveAuthenticatorInterface`. It costs one empty method and gives you the `InteractiveLoginEvent` for auditing and 2FA gating.

---

#### 12.6 User providers

The `UserBadge` carries a *string* (the email). Something has to turn that string into a real `User` object. That something is a **user provider**. You configure one in `security.yaml` under `providers`:

```yaml
security:
    providers:
        app_user_provider:
            entity:
                class: App\Entity\User
                property: email
```

The `entity` provider is a thin wrapper around a Doctrine `UserRepository`: it calls `findOneBy(['email' => $identifier])`. For most apps this is exactly right.

##### What a `User` must be

Your `User` class implements **`UserInterface`** (or, more usefully, extends the **`User`** abstract class which gives you a default `getRoles()`):

```php
// src/Entity/User.php
namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface;
use Symfony\Component\Security\Core\User\User as BaseUser;

#[ORM\Entity(repositoryClass: \App\Repository\UserRepository::class)]
class User extends BaseUser implements PasswordAuthenticatedUserInterface
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    #[ORM\Column(unique: true)]
    private ?string $email = null;

    // The *hashed* password — never store or log the plaintext.
    #[ORM\Column]
    private ?string $password = null;

    /** @var string[] A user's own roles, e.g. ['ROLE_ADMIN', 'ROLE_BILLING'] */
    #[ORM\Column]
    private array $roles = [];

    public function getUserIdentifier(): string
    {
        return $this->email;
    }

    // PasswordAuthenticatedUserInterface — required when using PasswordCredentials.
    public function getPassword(): ?string
    {
        return $this->password;
    }

    public function setPassword(string $password): void
    {
        $this->password = $password;
    }

    // Accessors for $id, $email, $roles omitted for brevity.
}
```

A couple of design decisions baked into `UserInterface` are worth stating out loud:

- **Roles are returned by `getRoles()`**, and they *must* be prefixed with `ROLE_` (see §12.8).
- **`getUserIdentifier()`** returns the unique string the provider keys on (here, the email).

Because our app is **multi-tenant**, note that the `User` is a *global* identity — a person with one email and a set of global roles. *Which tenant* they are acting on, and what they may do *inside a tenant*, is an **authorization** concern we solve with voters (§12.9), not by baking it into the user's role list. Keep authentication (the person) and tenancy (the context) cleanly separated; the moment you encode "user X belongs to tenant Y" into `getRoles()`, you will be editing roles to change data ownership and it will hurt.

##### Custom providers

If your users do not live in a single Doctrine entity (an LDAP directory, an external identity provider, a union of two tables), you implement **`UserProviderInterface`** yourself — it asks only for `loadUserByIdentifier()`, `refreshUser()`, and `supportsClass()`. Then register it:

```yaml
security:
    providers:
        my_custom:
            id: App\Security\CustomUserProvider   # a service implementing UserProviderInterface
```

---

#### 12.7 Password hashing

Passwords are never stored. What is stored is a one-way **hash** with a **salt**, and the salt is embedded in the stored string. Symfony's job is to (a) hash on registration, (b) verify on login, and (c) *upgrade* weak old hashes over time.

##### Configuring the hasher

The `password_hashers` key configures hashing **per user class**:

```yaml
# config/packages/security.yaml
security:
    password_hashers:
        App\Entity\User:
            algorithm: auto        # argon2id if available, bcrypt otherwise
```

`auto` picks **Argon2id** when the server supports it (the current OWASP-recommended choice) and falls back to bcrypt. You can pin an algorithm and tune its cost, but the defaults are already sensible:

```yaml
    password_hashers:
        App\Entity\User:
            algorithm: argon2id
            memory_cost: 65536     # 64 MB
            time_cost: 4
            # For bcrypt: cost: 12 (ignore the argon options)
```

> **Performance note.** These numbers are tuned to make each hash cost a few tens of milliseconds. That is *intentional* — you want brute-forcing a leaked hash to be slow. In your **test** environment you should drop the cost drastically (or disable it) so the test suite isn't paying that tax on every user you create. Put it in `config/packages/test/security.yaml`:
>
> ```yaml
> security:
>     password_hashers:
>         App\Entity\User:
>             algorithm: auto
>             cost: 4            # lowest for bcrypt
>             time_cost: 3       # lowest for argon
>             memory_cost: 10    # lowest for argon
> ```

##### Hashing in code (registration)

You hash with the **`PasswordHasherInterface`**, which the framework resolves for you. You do *not* call `password_hash()` directly — the hasher is what knows the algorithm and options from config, and it is what makes the test-environment override work.

```php
// src/Service/RegistrationService.php
namespace App\Service;

use App\Entity\User;
use Symfony\Component\PasswordHasher\Hasher\PasswordHasherInterface;
use Symfony\Component\Security\Core\Encoder\UserPasswordEncoderInterface; // (5.x name; see note)

class RegistrationService
{
    public function __construct(
        private readonly PasswordHasherInterface $passwordHasher,
        private readonly UserPasswordEncoderInterface $encoder, // or inject the hasher directly
    ) {}

    public function register(string $email, string $plainPassword): User
    {
        $user = new User();
        $user->setEmail($email);
        $user->setRoles(['ROLE_MEMBER']);   // sane default: a plain member

        // The single call that turns plaintext into a stored hash.
        $user->setPassword($this->passwordHasher->hashPassword($user, $plainPassword));

        return $user;
    }
}
```

> **Note on the API name.** You may see `UserPasswordEncoderInterface` (pre-6.0) or `PasswordHasherInterface` (6.0+). They do the same thing; `PasswordHasherInterface` is the current one. The methods are `hashPassword()` and `isPasswordValid()`.

##### Verifying and upgrading

Verification is a single boolean call — and it is what `PasswordCredentials` uses under the hood, so you rarely call it yourself on the login path:

```php
$ok = $this->passwordHasher->isPasswordValid($user, $submittedPlaintext);
```

**Password upgrading** matters the day you move from bcrypt to Argon2 (or from an old weak cost to a new one). Old users still have the old hash; you do not want to force them to reset. The pattern is: *if a login succeeds and the stored hash is not the best current algorithm, re-hash and save it.* `PasswordHasherInterface` exposes `needsRehash()`:

```php
// e.g. in a listener after successful interactive login, or a cron
if ($this->passwordHasher->needsRehash($user)) {
    $user->setPassword($this->passwordHasher->hashPassword($user, $knownPlaintext));
    $this->em->flush();
}
```

The Passport makes this automatic: add a **`PasswordUpgradeBadge`** (which needs the plaintext and a `PasswordUpgraderInterface`, e.g. your `UserRepository`) to the passport, and a successful login silently re-hashes the password. For most apps, a low-key "rehash on next login" listener is the pragmatic choice.

> **Security detail.** The `erase_credentials` firewall option you'll see in old configs *no longer does anything* — Symfony 8.0 removed the `eraseCredentials()` method from the user interface, and `erase_credentials` is deprecated (8.1) and slated for removal in 9.0. **Remove it from your config**; it only produces a deprecation warning now.

---

#### 12.8 Roles and access control

Roles are the coarse, fast form of authorization. They answer *"is this user at least this level?"* They are **not** designed for object-level rules (that's voters, next section).

##### The `ROLE_` prefix

In Symfony, a "role" is conventionally a string beginning with `ROLE_`. The prefix is a *convention the framework relies on* for its built-in checks:

```php
$this->isGranted('ROLE_ADMIN');      // does the user have ROLE_ADMIN (or imply it)?
$this->isGranted('ROLE_USER');       // any authenticated user has ROLE_USER automatically
```

An authenticated (non-anonymous) user **always** has `ROLE_USER` implicitly, even if `getRoles()` returns `[]`. Anonymous users have `ROLE_ANONYMOUS` (well, in modern Symfony the "is authenticated" check is what you test; `ROLE_ANONYMOUS` is legacy).

##### Role hierarchy

`role_hierarchy` lets one role imply others, so you don't have to hand every admin the full list:

```yaml
security:
    role_hierarchy:
        ROLE_ADMIN: [ROLE_BILLING, ROLE_MEMBER]
        ROLE_BILLING: [ROLE_MEMBER]
```

Under this, a user whose `getRoles()` returns `['ROLE_ADMIN']` *is* a billing manager and a member. `isGranted('ROLE_BILLING')` returns true for them.

##### `access_control` — URL-level authorization

`access_control` (from §12.2) is the URL-level gate. Re-examining it with the SaaS lens:

```yaml
access_control:
    - { path: ^/login, roles: PUBLIC_ACCESS }
    - { path: ^/register, roles: PUBLIC_ACCESS }
    - { path: ^/admin, roles: ROLE_ADMIN }
    - { path: ^/billing, roles: ROLE_BILLING }
    - { path: ^/, roles: ROLE_USER }
```

When a user hits `/billing`:
- if **anonymous** → the firewall's **entry point** fires → redirect to `/login` (a **401-ish** flow: *"prove who you are"*).
- if **authenticated but lacking `ROLE_BILLING`** → **403 Forbidden** (*"you are who you say, but you can't be here"*).

That 401-vs-403 distinction is the authentication/authorization split, expressed as HTTP.

##### Checking in code and templates

**In a controller**, via `AbstractController`:

```php
use Symfony\Component\Security\Core\Exception\AccessDeniedException;

public function editBilling(): Response
{
    // Throw 403 if not granted. Use a role, a voter attribute, or a full expression.
    $this->denyAccessUnlessGranted('ROLE_BILLING');

    // ...
}
```

**In Twig**, you have two helpers:

```twig
{# Roles and voter attributes #}
{% if is_granted('ROLE_ADMIN') %} ... {% endif %}

{# The current user (or null) #}
<h1>Hello {{ app.user.email }}</h1>
```

`is_granted()` in Twig accepts a role *or* a voter attribute (e.g. `is_granted('INVOICE_EDIT', invoice)`, where `INVOICE_EDIT` is an attribute your voter understands — §12.9).

> **Design rule of thumb.** Use `access_control` for *"who can reach this area."* Use `isGranted()`/`denyAccessUnlessGranted()` in controllers for *"can this user perform this action."* Reserve voters for anything that depends on the *specific object* being acted on.

---

#### 12.9 Voters: object-level authorization

Roles can say *"you may access `/billing`."* They cannot say *"you may delete **this** invoice, but only because it's yours, it's your tenant's, and it's unpaid."* That third, object-dependent kind of rule is exactly what **voters** are for.

A voter is a service that answers: *"for attribute **X**, against object **Y**, on behalf of the current user — **GRANT**, **DENY**, or **ABSTAIN**?"* Symfony ships a base `Voter` class and a `VoterInterface`. Voters are auto-registered (they implement the interface, so the container tags them) and are consulted by the **access decision manager** whenever you `isGranted('SOME_ATTRIBUTE', $object)` — in a controller, a voter, or Twig.

##### A real voter for our invoicing app

Let's encode a rule we'd actually ship:

> **`INVOICE_EDIT`** — a user may edit an invoice if **all** of these hold:
> 1. they are a member of the invoice's tenant, **and**
> 2. the invoice is not yet in a final/paid state, **and**
> 3. they are a billing manager or admin (plain members can *view* but not *edit*).

```php
// src/Security/InvoiceVoter.php
namespace App\Security;

use App\Entity\Invoice;
use App\Entity\Tenant;
use App\Entity\User;
use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
use Symfony\Component\Security\Core\Authorization\Voter\Voter;
use Symfony\Component\Security\Core\User\UserInterface;

class InvoiceVoter extends Voter
{
    protected function supports(string $attribute, mixed $subject): bool
    {
        // Claim only the attributes/subjects we understand, cheaply.
        // Returning false here means "I don't vote on this."
        return (INVOICE_EDIT === $attribute || INVOICE_DELETE === $attribute)
            && $subject instanceof Invoice;
    }

    protected function voteOnAttribute(string $attribute, mixed $subject, TokenInterface $token): bool
    {
        $invoice = $subject;               // we know it's an Invoice (see supports())
        /** @var User $user */
        $user = $token->getUser();

        // Rule 1 — tenancy: must be a member of this invoice's tenant.
        if (!$this->isMemberOf($user, $invoice->getTenant())) {
            return false;   // DENY
        }

        // Rule 2 — state: a finalized invoice is immutable.
        if ($invoice->isFinalized()) {
            return false;   // DENY
        }

        // Rule 3 — capability: editing requires the billing role.
        return match ($attribute) {
            INVOICE_EDIT   => $user->hasRole('ROLE_BILLING'),
            INVOICE_DELETE => $user->hasRole('ROLE_ADMIN'),
            default        => false,
        };
    }

    private function isMemberOf(UserInterface $user, Tenant $tenant): bool
    {
        // In a real app: a join table user_tenants, or User->getTenants().
        return in_array($tenant, $user->getTenants(), strict: true);
    }
}
```

Constants for the attribute strings keep them typosafe and grep-able:

```php
// Could live on the class, or in a dedicated constants class.
final class InvoiceVoter
{
    public const INVOICE_EDIT = 'INVOICE_EDIT';
    public const INVOICE_DELETE = 'INVOICE_DELETE';
    // ...
}
```

Now authorization reads like English, in any of the three places:

```php
// Controller
$this->denyAccessUnlessGranted(InvoiceVoter::INVOICE_EDIT, $invoice);

// Twig
{% if is_granted('INVOICE_EDIT', invoice) %}
    <a href="{{ path('invoice_edit', { id: invoice.id }) }}">Edit</a>
{% endif %}
```

##### How votes are combined: the access decision manager

When you `isGranted('INVOICE_EDIT', $invoice)`, one or more voters may vote. The **access decision manager** combines their votes into a single yes/no. The built-in strategies:

- **`affirmative`** (the default) — **one** GRANT means access; all-abstain means *deny*. Good default: a single voter saying "yes" is enough.
- **`unanimous`** — every voter that *votes* must GRANT (abstentions are ignored); any explicit DENY blocks. Stricter.
- **`consensus`** — count GRANTs vs DENYs; the majority wins.
- **`strict_affirmative`** — a GRANT wins, but if any voter explicitly DENY, that wins over abstain.

You configure it at the top level:

```yaml
security:
    access_decision_manager:
        strategy: affirmative        # default
        allow_if_all_abstain: false  # default: nobody voted → deny (safe)
```

`allow_if_all_abstain: false` is the safe default: if *no* voter knows about the attribute, the answer is **no**, not "silently yes." Keep it that way unless you have a very specific reason.

For full control you can swap in a **custom strategy** (`strategy_service`) or even replace the whole manager (`service`), both by implementing the corresponding interface — but reach for these only when the four built-ins genuinely can't express your policy.

##### A note on N+1 in voters

Voters run *per object*. In a list of 100 invoices where you call `is_granted('INVOICE_EDIT', $invoice)` for each row, a naive `isMemberOf()` that does a query will fire 100 queries — a textbook **N+1**, and Chapter 13 will give you the tools to *find* it. The fix is the same as anywhere in an app: **fetch what you need up front** (e.g. `JOIN` the tenant membership, or hydrate `$user->getTenants()` once) so the voter's checks are in-memory. Write voters to assume the object is already fully hydrated for the fields they touch, and make the repository responsible for hydrating those fields.

---

#### 12.10 The security context and the `Security` service

When you're *inside* a request, the "who is logged in" state is the **token** stored in the session. You reach it three ways:

- **`$this->getUser()`** in a controller (returns the `User` or `null`).
- **`app.user`** in Twig.
- Injecting the **`Security`** service for lower-level access (the `TokenStorage`, checking `isGranted`, reading roles).

```php
use Symfony\Component\Security\Core\Security;

class SomeService
{
    public function __construct(private readonly Security $security) {}

    public function currentUserId(): ?int
    {
        $user = $this->security->getUser();   // ?User
        return $user?->getId();
    }
}
```

Two habits save you pain:

1. **Always treat `getUser()` as nullable.** A controller can run for an anonymous user even on a page you *thought* was protected (a misordered `access_control` rule, a `security: false` firewall, a forgotten route).
2. **Prefer the `Security` service over poking `TokenStorage`** in services. It's the public, stable facade.

---

#### 12.11 Two-factor authentication

A password is **one factor**: something the user *knows*. A second factor is something they *have* (a phone, a TOTP app, a hardware key) or *are* (a fingerprint). For a SaaS that holds invoices and billing, "one password" is usually not enough for the admin and billing roles. This section builds a **TOTP** second factor the Symfony way.

> **Scope.** We will build the *integration* with Symfony's security model: how the second step fits into authenticators, when it's required, and how to gate it. The TOTP *math* (turning a shared secret + current time into a 6-digit code, and verifying it) is a small, well-specified algorithm. You can implement it yourself in ~40 lines (RFC 6238) or use a small vetted library. Don't hand-roll a *new* scheme; do use the standard one.

##### The flow

The clean way to bolt 2FA onto the existing form login is **two steps**, using what we already built:

1. **Step 1 — password.** Our `EmailAuthenticator` verifies the password *exactly as in §12.5*. If it succeeds **and** the user has 2FA enabled, we do **not** land them on the dashboard. Instead, `onAuthenticationSuccess()` redirects to a `/2fa` page, and we stash the *pending* user identifier in the session (the user is **not** fully authenticated yet).
2. **Step 2 — code.** A dedicated `/2fa` route renders a code form. A second, custom **`TwoFactorAuthenticator`** reads the TOTP code, verifies it against the secret stored for that user, and *then* completes authentication — this time for real.

The key insight: **the first authenticator succeeds but we intercept the redirect.** We keep the user in a "password-verified, awaiting code" state, and only the second authenticator produces the *trusted* token.

##### Storing the secret

Add to the `User` (or, cleaner, a `UserTwoFactor` entity):

```php
// Only for users who have enrolled in 2FA.
#[ORM\Column(nullable: true)]
private ?string $totpSecret = null;    // base32-encoded shared secret

#[ORM\Column]
private bool $totpEnabled = false;
```

Enrollment is a controller flow (not the login path): generate a secret, show a QR code (`otpauth://` URI) the user scans into Authenticator/1Password, have them type back a code to confirm, then persist the secret and set `totpEnabled = true`. (A `make:registration`-style endpoint; keep it behind `ROLE_USER`.)

##### Wiring step 1

Modify `EmailAuthenticator::onAuthenticationSuccess()` from §12.5:

```php
public function onAuthenticationSuccess(Request $request, TokenInterface $token, string $firewallName): ?Response
{
    /** @var User $user */
    $user = $token->getUser();

    // If this user requires a second factor, hold them at the 2FA gate.
    if ($user->getTotpEnabled()) {
        // We are NOT done yet — remember who is pending, don't send the real token onward.
        $request->getSession()->set('two_factor_uid', $user->getUserIdentifier());

        return new RedirectResponse($this->urlGenerator->generate('two_factor'));
    }

    // No 2FA: normal behavior.
    return new RedirectResponse($this->urlGenerator->generate('dashboard'));
}
```

The `/2fa` page and its route:

```php
#[Route('/2fa', name: 'two_factor', methods: ['GET', 'POST'])]
public function twoFactor(Request $request, EntityManagerInterface $em): Response
{
    $uid = $request->getSession()->get('two_factor_uid');
    if (null === $uid) {
        // Not mid-login; bounce to login.
        return $this->redirectToRoute('app_login');
    }

    if ($request->isMethod('POST')) {
        $code = (string) $request->request->get('code', '');
        // Verification + token completion happen in TwoFactorAuthenticator.
        // If the code is wrong, the authenticator returns a 401/redirect with error.
    }

    return $this->render('security/two_factor.html.twig');
}
```

##### The step-2 authenticator

This is a genuine **custom authenticator**, and it is where the "second step" really lives:

```php
// src/Security/TwoFactorAuthenticator.php
namespace App\Security;

use App\Entity\User;
use App\Repository\UserRepository;
use Symfony\Component\Csrf\CsrfTokenManagerInterface;
use Symfony\Component\HttpFoundation\RedirectResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
use Symfony\Component\Security\Core\Exception\AuthenticationException;
use Symfony\Component\Security\Core\Exception\CustomUserMessageAuthenticationException;
use Symfony\Component\Security\Http\Authenticator\AbstractAuthenticator;
use Symfony\Component\Security\Http\Authenticator\Passport\Badge\CsrfTokenBadge;
use Symfony\Component\Security\Http\Authenticator\Passport\Badge\UserBadge;
use Symfony\Component\Security\Http\Authenticator\Passport\SelfValidatingPassport;

class TwoFactorAuthenticator extends AbstractAuthenticator
{
    public function __construct(
        private readonly UserRepository $users,
        private readonly TOTPVerifier $totp,        // wraps the RFC 6238 verify()
        private readonly CsrfTokenManagerInterface $csrf,
    ) {}

    // Only act while the user is mid-2FA.
    public function supports(Request $request): ?bool
    {
        return $request->attributes->get('_route') === 'two_factor'
            && null !== $request->getSession()->get('two_factor_uid');
    }

    public function authenticate(Request $request): SelfValidatingPassport
    {
        $uid = $request->getSession()->get('two_factor_uid');

        $user = $this->users->findOneBy(['email' => $uid]);
        if (null === $user || !$user->getTotpEnabled()) {
            throw new CustomUserMessageAuthenticationException('two_factor.invalid');
        }

        $code = (string) $request->request->get('code', '');
        if (!$this->totp->verify($user->getTotpSecret(), $code)) {
            // Safe, translatable message. Never leak *why* it failed beyond "wrong code."
            throw new CustomUserMessageAuthenticationException('two_factor.invalid_code');
        }

        // Code accepted. Hand back the user with NO further credential to check —
        // the password was already verified in step 1.
        $badges = [];
        // If we want remember-me to survive the 2FA step, add a RememberMeBadge here.

        return new SelfValidatingPassport(
            new UserBadge($user->getUserIdentifier()),
            $badges
        );
    }

    public function onAuthenticationSuccess(Request $request, TokenInterface $token, string $firewallName): ?Response
    {
        // Done for real now.
        $request->getSession()->remove('two_factor_uid');
        return new RedirectResponse(/* dashboard */);
    }

    public function onAuthenticationFailure(Request $request, AuthenticationException $exception): Response
    {
        // Re-render /2fa with the error, keep them mid-2FA.
        return new RedirectResponse(/* two_factor with error */);
    }
}
```

Register it on the same `main` firewall:

```yaml
firewalls:
    main:
        # ...
        custom_authenticators:
            - App\Security\EmailAuthenticator
            - App\Security\TwoFactorAuthenticator
```

Both authenticators share the `main` firewall; `supports()` on each keeps them from firing at the wrong time. The `SelfValidatingPassport` in step 2 is deliberate: there is no *new* secret to verify against in step 2 beyond the TOTP code, which we've already checked — so the passport just asserts the user's identity.

> **Design choice.** Some teams instead require 2FA via a *separate firewall* or a dedicated bundle (there are well-maintained third-party 2FA bundles for exactly this). The first-party approach above is only ~150 lines and keeps you in full control; reach for a bundle when you need recovery codes, WebAuthn/passkeys, or push notifications as additional factors. Whichever route, the security *shape* is the same: **password first, factor second, trusted token only after both pass.**

---

#### 12.12 Brute-force protection: login throttling

An attacker can spray the login form with a huge dictionary of passwords. Even with strong hashing (which makes each *guess* slow on your side), a fast attacker with a good wordlist will eventually win — *unless* you cap the number of attempts per unit time. Symfony has this built in: **login throttling**.

Enable it per firewall (we already do, in §12.2):

```yaml
firewalls:
    main:
        login_throttling:
            max_attempts: 5          # attempts before throttling kicks in
            interval: '15 minutes'   # the window those attempts are counted over
```

With the defaults (`max_attempts: 5`, a 1-minute window) it's a gentle tripwire. For a SaaS login you almost certainly want something like the block above — **5 attempts per 15 minutes**.

##### How it works under the hood

Login throttling is a thin, opinionated layer on top of two things you've already seen in this book:

- **The RateLimiter component** — a fixed-window counter per key, stored in a cache. The `interval` and `max_attempts` come straight from this config.
- **The Lock component** — so a burst of simultaneous requests can't race past the counter.

Crucially, it maintains **two** counters:

1. **Per `username + IP`** — stops someone hammering *one specific account* they know exists.
2. **Per IP (global)** — stops one IP hammering *many accounts at once* (credential-stuffing across a list of known emails). The global limit is `max_attempts` higher per the design, so a single IP isn't cut off just because a family shares a connection while legitimately using several accounts.

When the cap is exceeded, the next attempt throws a **`TooManyLoginAttemptsAuthenticationException`** instead of even checking the password. Recognize it in `onAuthenticationFailure()` and give the user a clear, *different* message:

```php
use Symfony\Component\Security\Core\Exception\TooManyLoginAttemptsAuthenticationException;

public function onAuthenticationFailure(Request $request, AuthenticationException $exception): Response
{
    if ($exception instanceof TooManyLoginAttemptsAuthenticationException) {
        // "Too many attempts, try again later" — distinct from "bad credentials".
        return $this->urlGenerator->generate('app_login', ['error' => 'throttled']);
    }

    // ... normal bad-credentials path ...
}
```

> **Multi-tenant note.** For a SaaS, "per username + IP" is usually the right primary limiter (it protects a specific tenant's known admin email). If you run many tenants behind one IP (e.g. a managed hosting panel), the *global IP* limiter could be too aggressive; tune it, or key the limiter to include the tenant. The limiter keys are configurable, so you can add context if your topology demands it.

> **Caution.** Login throttling is a *deterrent*, not a *wall*. It makes automated credential-stuffing slow and noisy. The things that actually stop an account takeover are: strong hashing (§12.7), a 2FA factor (§12.11), and — if you can — **`expose_security_errors: none`** so the login form can't tell an attacker whether an email *exists* (user enumeration). A login endpoint that says "no such user" for 90% of a guessed list is a gift.

---

#### 12.13 Session security

A session is an authenticated user's *state on your server*, referenced by a **session ID in a cookie**. Two attacks dominate: **session fixation** and **session hijacking**.

##### Session fixation

**Session fixation**: the attacker gets a victim to use a session ID *the attacker already knows*. If your app keeps that same ID after the victim logs in, the attacker (who holds the same ID) now rides along as the victim.

The fix is to **change the session ID at the moment of authentication**. Symfony does this by default. The behavior is controlled by the **`session_fixation_strategy`** option (top-level in `security`):

```yaml
security:
    # Default is MIGRATE. The options:
    session_fixation_strategy: migrate
    #   - none: keep the session ID unchanged (INSECURE — do not use)
    #   - migrate: keep session *data*, but rotate the ID (default, recommended)
    #   - invalidate: rotate the ID AND discard session data (nuclear option)
```

- **`migrate`** (default): the session ID is regenerated on login, but the session's other attributes are preserved. This is the right default for web apps — you rotate the ID (defeating fixation) without losing what you stored.
- **`invalidate`**: rotates the ID and wipes everything. Use this if your session holds data that should *not* survive the login boundary, or if you'd rather keep sessions minimal.
- **`none`**: never. It exists for completeness; choosing it is an explicit, insecure decision.

##### Hardening the session *cookie*

The session cookie lives in your **`framework`** config (not `security`), and the flags matter:

```yaml
# config/packages/framework.yaml
framework:
    session:
        handler_id: ~                 # file-based by default; swap for a cache handler in prod
        cookie_secure: auto           # send the cookie over HTTPS only (auto = on behind TLS)
        cookie_samesite: lax          # restricts cross-site sending; 'lax' is a good default
        cookie_httponly: true         # never expose the session ID to JavaScript
        gc_maxlifetime: 3600          # how long an idle session may live
```

The flags, and what each defeats:

- **`cookie_secure: auto`** — the cookie only travels over HTTPS, so it can't be read off an unencrypted connection (network sniffing / hijacking).
- **`cookie_httponly: true`** — JavaScript can't read the cookie, so a stored-XSS bug can't exfiltrate the session ID.
- **`cookie_samesite: lax`** — the browser won't attach the cookie to cross-site *requests* (top-level navigations still work), blunting a whole class of **CSRF** and cross-site session-use attacks.
- **`gc_maxlifetime`** — idle sessions expire. A short max lifetime means a stolen ID is useful for only a short window.

##### Logout hygiene

When logging out, you want the session to actually be gone, and any lingering client-side state to be cleared. Configure it in the firewall:

```yaml
firewalls:
    main:
        logout:
            path: app_logout
            target: app_login
            invalidate_session: true      # default true: logging out kills the session
            delete_cookies:               # also remove any app cookies you set
                my_app_cookie: null
            clear_site_data:              # tell the browser to clear its data for your origin
                - cookies
                - storage
            enable_csrf: true             # logout is a state-changing action; protect it too
```

`invalidate_session` defaults to `true`, which means logging out on *any* firewall logs you out of *all* of them — usually what you want for a browser app. `enable_csrf: true` matters: a logout link is a state-changing action, and without CSRF protection a malicious page could log your user out just by loading in their browser.

---

#### 12.14 Putting it together: the invoicing app

Let's assemble what we've built into a coherent security posture for the SaaS, and say out loud what each layer is doing.

```yaml
# config/packages/security.yaml  (assembled)
security:
    expose_security_errors: none            # no user enumeration

    session_fixation_strategy: migrate      # rotate the session ID at login

    password_hashers:
        App\Entity\User:
            algorithm: auto

    role_hierarchy:
        ROLE_ADMIN: [ROLE_BILLING, ROLE_MEMBER]
        ROLE_BILLING: [ROLE_MEMBER]

    access_control:
        - { path: ^/login, roles: PUBLIC_ACCESS }
        - { path: ^/register, roles: PUBLIC_ACCESS }
        - { path: ^/2fa, roles: PUBLIC_ACCESS }      # the 2FA gate is reachable
        - { path: ^/admin, roles: ROLE_ADMIN }
        - { path: ^/billing, roles: ROLE_BILLING }
        - { path: ^/, roles: ROLE_USER }

    providers:
        app_user_provider:
            entity:
                class: App\Entity\User
                property: email

    firewalls:
        dev:
            pattern: ^/(_(profiler|wdt)|css|images|js)/
            security: false
        main:
            lazy: true
            provider: app_user_provider
            custom_authenticators:
                - App\Security\EmailAuthenticator      # password (step 1)
                - App\Security\TwoFactorAuthenticator  # TOTP (step 2)
            remember_me:
                secret: '%kernel.secret%'
                lifetime: 604800
                name: REM
            logout:
                path: app_logout
                target: app_login
                enable_csrf: true
                clear_site_data: [cookies, storage]
            login_throttling:
                max_attempts: 5
                interval: '15 minutes'
```

Now the authorization layers, in the order they'd fire for a request to `PATCH /invoices/42`:

1. **Firewall** `main` matches → the request is under the session-based firewall.
2. **`access_control`**: `^/` requires `ROLE_USER` → the user must be authenticated. (If not: entry point → `/login`.)
3. **Controller**: `denyAccessUnlessGranted(InvoiceVoter::INVOICE_EDIT, $invoice)`.
4. **`InvoiceVoter`**: checks tenancy → state → billing role (all in-memory on a hydrated invoice). **GRANT** or **403**.

Authentication (steps 2–3's *who*) is separate from the object-level policy (step 4's *is this specific invoice editable by this specific user*). That separation is what keeps the whole thing maintainable as the app grows.

> **The multi-tenant boundary, stated plainly.** Nothing in the *authentication* layer knows about tenants. Tenancy is enforced *twice*: in the `access_control`/controller boundary ("you must be a member of *this* tenant to be here") and in the voter ("you must be a member of *this* invoice's tenant"). Do both. Relying on the voter alone means every route is one forgotten `denyAccessUnlessGranted` away from a cross-tenant leak; relying on the URL boundary alone means a crafted request can touch another tenant's object. Defense in depth, applied to tenancy.

---

#### 12.15 A short security checklist

Defense in depth is just "more than one layer, each of which would stop the attack alone." Before this app ships, the checklist:

- [ ] **Passwords**: hashed with `auto`/Argon2id; cost lowered *only* in the `test` env; never logged, never in URLs, never in `getMessage()` output.
- [ ] **User enumeration**: `expose_security_errors: none`; login failures give the same generic message whether the email exists or the password is wrong.
- [ ] **Login form + logout**: both CSRF-protected (`CsrfTokenBadge` / `logout.enable_csrf`).
- [ ] **Brute force**: `login_throttling` on, tuned to your traffic; distinct "throttled" message.
- [ ] **2FA**: required for `ROLE_ADMIN`/`ROLE_BILLING`; password step and code step are separate; trusted token only after both.
- [ ] **Session**: `session_fixation_strategy: migrate`; cookie `secure`, `httponly`, `samesite: lax`; short `gc_maxlifetime`.
- [ ] **Access control**: specific-first ordering; `^/` catch-all *last*; every stateful route covered.
- [ ] **Voters**: object-level rules (tenancy + state + role); voters assume hydrated subjects; no per-row queries in a loop.
- [ ] **Roles**: `ROLE_` prefix; hierarchy used instead of hand-listing.

None of these is glamorous. Together they are the difference between "someone will find us in the news" and "we're fine."

---

#### 12.16 Summary

- **Authentication is "who"; authorization is "what may they do."** The firewall picks the authenticator chain per URL; the authenticator produces a token; `access_control`, `isGranted()`, and voters then decide what that token may touch.
- **Authenticators are small, testable classes.** `supports() → authenticate() → onAuthenticationSuccess()/onAuthenticationFailure()`, with the work described by a **Passport** (a `UserBadge`, credentials, and optional badges like `CsrfTokenBadge` or `RememberMeBadge`).
- **Write the login explicitly** (§12.5). It's the shape of every real login, and it's where you add CSRF, target paths, and the 2FA gate.
- **`UserInterface` + a `User` entity + an entity provider** is 95% of apps. Keep tenancy out of `getRoles()`; enforce it in voters.
- **Hash with the hasher, never `password_hash()` directly**; use `needsRehash()` / `PasswordUpgradeBadge` to move users to better algorithms over time.
- **Roles are coarse; voters are precise.** Voters express object-level policy (tenancy + state + capability) and are combined by the access decision manager.
- **Harden the boring bits**: login throttling, session fixation (`migrate`), session cookie flags, CSRF on logout, and no user enumeration.

The next chapter finally gives this security a database to defend.

---

#### 12.17 Exercises

**1. (Warm-up) Order the rules.** Rewrite the `access_control` in §12.2 so that `^/billing` also admits `ROLE_ADMIN` *without* changing the hierarchy. Then break it on purpose — put the `^/` catch-all first — and watch `debug:firewall` / a request to `/billing` as a member to see exactly where it stops. Describe what you observed.

**2. (User provider)** Add a **`lastLoginAt`** timestamp to `User`. On a *successful interactive* login, update it. Hint: implement `InteractiveAuthenticatorInterface` on `EmailAuthenticator`, listen for `InteractiveLoginEvent` in a subscriber, and persist. Verify the timestamp is *not* set for a `remember-me` (non-interactive) re-login.

**3. (Voter, the hard part)** Extend `InvoiceVoter` with an **`INVOICE_VIEW`** attribute: a member can view any invoice *in their own tenant*; an admin can view *any* invoice, *any* tenant. Add the corresponding `denyAccessUnlessGranted(INVOICE_VIEW, $invoice)` to the `invoice_show` controller and an `is_granted('INVOICE_VIEW', invoice)` guard in the template. Write **voter unit tests** covering: own-tenant member, other-tenant member, admin, and a finalized invoice. (No database needed — mock or construct the `Invoice` and `User`.)

**4. (Password upgrade)** Seed the test database with a user whose password is hashed with a *weak* algorithm (e.g. plain `plaintext` or a low-cost bcrypt via a second user class / `password_hashers` override in `test`). Add a "rehash on next login" subscriber using `needsRehash()`. Write a functional test that logs the user in once, re-logs in, and asserts the stored hash is now the `auto` algorithm.

**5. (Throttling)** With `max_attempts: 3` and a 5-minute `interval`, write a functional test that issues 3 failed logins and a 4th, asserting the 4th produces the `throttled` error **without** a password check. Then extend it: verify the *global IP* limiter still permits a *second* username from the same IP (i.e. per-account and per-IP limits are independent).

**6. (2FA end-to-end)** Complete the 2FA flow from §12.11. Enroll a user in TOTP (generate a secret + a code with a real TOTP library), POST the code to `/2fa`, and assert they land authenticated on the dashboard. Then assert that a *wrong* code does **not** authenticate them, and that hitting a protected route while "pending 2FA" (password OK, code not entered) sends them back to `/2fa`, not into the app.

**7. (Design discussion, no code)** Our app now has a `stateless` `api` firewall (Part V). Explain in a short paragraph: why `login_throttling` as configured here does *not* protect the API firewall, what the `username + IP` / `IP` limiter keys assume that breaks under a CDN or load balancer, and how you'd move brute-force protection to the API side (hint: a different key space — API key + IP — and possibly a different component).

**8. (Going further)** Research the four `access_decision_manager` strategies. In a SaaS where *some* voters are "policy" (security-critical) and some are "preference" (UX), the built-in strategies may not suffice. Sketch — in code or pseudocode — a **custom `AccessDecisionStrategyInterface`** that treats a DENY from any *policy* voter as an absolute veto, while letting *preference* voters use the default `affirmative` rule. What would you name the two voter "kinds," and how would a voter declare which kind it is?

I'll verify a few current framework facts first so the chapter is accurate for the Symfony 7.4 LTS / 8.x and PHP 8.2+ target, then write the chapter.



Let me confirm the Symfony 7.4 details and Doctrine version compatibility.



I have the key facts. Let me confirm one Doctrine 3 API detail that affects type-hints throughout the chapter.



Let me pull the ORM 3 upgrade notes to get the exact `EntityManager` typing and removed APIs right.



Let me confirm the `EntityManagerInterface` status in ORM 3 from the official upgrade notes.



Below is **Chapter 13**, written to match the book's conventions (attribute-based config, PHP 8.2+, the multi‑tenant SaaS invoicing running project) and current for the Symfony 7.4 LTS / Doctrine 3 target. It continues the running project where Chapter 12 left off (users and tenants already exist).

---

### Chapter 13. Doctrine ORM

By the end of Chapter 12 you have *people* in the system: a `User` entity, a `Tenant` that groups users into isolated workspaces, and a security layer that knows who is logged in. But an invoicing app isn't about logins. It's about *data that outlives a request* — companies, invoices, line items, totals. That's what this chapter is about.

Doctrine ORM is the persistence layer for Symfony. It lets you model your domain as ordinary PHP objects (entities), map them to relational tables without writing SQL for every read and write, and — critically for a multi‑tenant product — query that data safely and fast even as it grows into millions of rows.

This chapter builds the invoicing domain of the running project: the `Company`, `Product`, `Invoice`, and `LineItem` entities; the repositories that keep every query tenant‑scoped; and the migrations, fixtures, and performance habits that turn a prototype into something that can survive real usage.

> **About the versions in this chapter.** Symfony 7.4 LTS (released November 2025, PHP 8.2+, supported through 2029) is the baseline here. It works with both Doctrine ORM 2.19+ and 3.x; modern Symfony projects now ship **ORM 3**, which is what this chapter targets. The mapping, DQL, QueryBuilder, repository, migration, and fixture APIs shown here are identical on recent 2.x and 3.x, so the code is safe either way. Where 3.x matters, we'll say so.

#### 13.1 Installing and Configuring Doctrine

The fastest way in is the ORM pack, a metapackage that bundles the right versions of the database abstraction layer (DBAL), the ORM, and the two Symfony bundles that glue them to the framework:

```bash
composer require symfony/orm-pack
```

That installs `doctrine/dbal`, `doctrine/orm`, `doctrine/doctrine-bundle`, and `doctrine/doctrine-migrations-bundle`. Fixtures are separate (you only load them in dev and test), so add them when you get to 13.9:

```bash
composer require doctrine/doctrine-fixtures-bundle -W --dev
```

##### The database URL

Doctrine reads its connection from a `DATABASE_URL` parameter. Add it to your `.env` (the `#` in the password is URL‑encoded as `%23`; Symfony's `resolve:` processor also lets you reference other variables):

```env
# .env
DATABASE_URL="postgresql://app:%2Fpass%2F@127.0.0.1:5432/app?serverVersion=17&charset=utf8"
```

Swap the scheme for your driver — `mysql://` or `sqlite://` for local work. For the rest of the running project we assume PostgreSQL.

##### `config/packages/doctrine.yaml`

Symfony generates most of this for you. The parts worth understanding:

```yaml
# config/packages/doctrine.yaml
doctrine:
    dbal:
        url: '%env(resolve:DATABASE_URL)%'
        profiling_collect_backtrace: '%kernel.debug%'
    orm:
        auto_generate_proxy_classes: true
        naming_strategy: doctrine.orm.naming_strategy.underscore_number_aware
        auto_mapping: true
        mappings:
            App:
                type: attribute
                is_bundle: false
                dir: '%kernel.project_dir%/src/Entity'
                prefix: 'App\Entity'
                alias: App
```

A few decisions baked in here:

- **`type: attribute`** — entity metadata is read from PHP 8 attributes (`#[ORM\Entity]` and friends) on classes under `src/Entity`, not from XML/YAML or the old annotations. This is the book's convention throughout.
- **`prefix: 'App\Entity'`** and **`alias: App`** — together they let you write `FROM App\Invoice` in DQL and still map it to `App\Entity\Invoice`.
- **`auto_mapping: true`** — the bundle maps any class in the configured `dir`, so you don't have to list each entity.
- **`auto_generate_proxy_classes`** — lazy‑loading needs proxy classes. In dev we generate them on the fly; in `config/packages/prod/doctrine.yaml` we flip this to `false` and point at a prebuilt proxy directory so the compiler has nothing to do at runtime.

The migrations bundle gets its own file:

```yaml
# config/packages/doctrine_migrations.yaml
doctrine_migrations:
    migrations_paths:
        'DoctrineMigrations': '%kernel.project_dir%/migrations'
    enable_profiler: false
```

##### The entity manager and the Unit of Work

The central object is the **entity manager** — the thing you inject into controllers, services, and commands to load and save entities. In Symfony it's the service `doctrine.orm.default_entity_manager`. You type‑hint against the interface, exactly as the official Symfony 7.4 docs do:

```php
use Doctrine\ORM\EntityManagerInterface;

class InvoiceManager
{
    public function __construct(private EntityManagerInterface $em)
    {
    }
}
```

> **Tip:** Depending on the concrete `Doctrine\ORM\EntityManager` class also works and is common in ORM 3 codebases. If you want maximum portability across the persistence layer, `Doctrine\Persistence\ObjectManager` is the most abstract option — but it exposes fewer ORM‑specific methods. For an app this size, `EntityManagerInterface` is the right default.

The entity manager is a front door to two important ideas:

**The Unit of Work (UoW).** When you call `$em->persist($invoice)`, nothing is written to the database. Doctrine adds the object to an in‑memory *identity map* and, on the next `$em->flush()`, diffs that map against the database to produce a minimal set of `INSERT`, `UPDATE`, and `DELETE` statements. You can make a dozen changes in a request and flush once at the end:

```php
$company = $em->find(Company::class, 42);
$company->setName('ACME (renamed)');
$invoice->setTotalCents(19_000);
$em->flush(); // one transaction, the minimum SQL to reconcile both changes
```

**Lazy proxies.** When you load an `Invoice`, Doctrine doesn't eagerly fetch its related `LineItem` rows. It hands you a *proxy* — a stand‑in that issues a query the first time you actually touch the association. This is what makes object graphs cheap to load and, if you're not careful, what causes the N+1 problem we dissect in 13.10.

> **Note:** In ORM 2.x the entity argument to `flush()` used to let you flush a single object. **ORM 3 removed that** — `$em->flush($entity)` now throws. Always call `$em->flush()`. That's also what we do in this chapter.

#### 13.2 Entities and Attribute Mapping

Let's add the first new entity to the running project: a `Product`, a catalog item that a tenant can attach to invoices.

We'll recap the shape of the two entities from Chapter 12 so this is self‑contained. `Tenant` is the root of our multi‑tenant model; `User` hangs off it:

```php
// src/Entity/Tenant.php (introduced in Ch. 12, extended here)
use App\Repository\TenantRepository;
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity(repositoryClass: TenantRepository::class)]
class Tenant
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    #[ORM\Column(length: 120, unique: true)]
    private string $name;

    #[ORM\Column(length: 120, unique: true)]
    private string $slug;

    /** @var array<string, mixed> */
    #[ORM\Column(type: 'json', options: ['default' => '{}'])]
    private array $settings = [];

    /** @var Collection<int, User> */
    #[ORM\OneToMany(mappedBy: 'tenant', targetEntity: User::class)]
    private Collection $users;

    public function __construct()
    {
        $this->users = new ArrayCollection();
    }
    // getters and setters ...
}
```

And our new `Product`:

```php
// src/Entity/Product.php
use App\Repository\ProductRepository;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity(repositoryClass: ProductRepository::class)]
#[ORM\Index(columns: ['tenant_id'], name: 'idx_product_tenant')]
class Product
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    #[ORM\ManyToOne(inverseOwner: Tenant::class)]
    #[ORM\JoinColumn(nullable: false)]
    private ?Tenant $tenant = null;

    #[ORM\Column(length: 255)]
    private string $name;

    #[ORM\Column(length: 1024, nullable: true)]
    private ?string $description = null;

    #[ORM\Column]
    private int $unitPriceCents = 0;

    #[ORM\Column]
    private bool $active = true;

    #[ORM\Column]
    private \DateTimeImmutable $createdAt;

    public function __construct(string $name)
    {
        $this->name = $name;
        $this->createdAt = new \DateTimeImmutable();
    }

    public function getId(): ?int               { return $this->id; }
    public function getTenant(): ?Tenant         { return $this->tenant; }
    public function setTenant(Tenant $tenant): self
    {
        $this->tenant = $tenant;
        return $this;
    }
    public function getName(): string            { return $this->name; }
    public function getDescription(): ?string    { return $this->description; }
    public function getUnitPriceCents(): int     { return $this->unitPriceCents; }
    public function setUnitPriceCents(int $c): self
    {
        $this->unitPriceCents = $c;
        return $this;
    }
    public function isActive(): bool             { return $this->active; }
    public function setActive(bool $active): self
    {
        $this->active = $active;
        return $this;
    }
    public function getCreatedAt(): \DateTimeImmutable { return $this->createdAt; }
}
```

Walk through the attributes top to bottom, because each one is a mapping decision:

- **`#[ORM\Entity]`** marks the class as an entity. The optional `repositoryClass:` argument tells Doctrine which repository to return from `$em->getRepository()`.
- **`#[ORM\Id]` + `#[ORM\GeneratedValue]` + `#[ORM\Column]`** together define a surrogate primary key. `GeneratedValue` means "the database assigns the value" (typically an `AUTO_INCREMENT` / `IDENTITY` column). Note the property is typed `?int $id = null` — until it's persisted, there's no ID.
- **`#[ORM\Column(length: 255)]`** maps to a `VARCHAR(255)`. The `length` option is required for strings. Omit the `type` and Doctrine infers it from the PHP type: `string` → `string`, `int` → `integer`, `bool` → `boolean`, `\DateTimeImmutable` → `datetime_immutable`.
- **`nullable: true`** on `$description` — maps to a `NULL`‑able column. **In ORM 3 this matters more**, because ORM 3 no longer silently coerces a `NULL` into `0`/`''` for a non‑nullable scalar. If a column can be `NULL`, mark it `nullable: true` *and* make the property `?Type`.
- **`unique: true`** on a slug, **`#[ORM\Index(...)]`** on the class for the `tenant_id` column we query on constantly.
- **`inverseOwner: Tenant::class`** — this is a *many‑to‑one*: many products belong to one tenant. We'll unpack owning vs. inverse sides in 13.3.

##### The full column type menu

You don't need to memorize this, but it's the vocabulary of a mapping. The most useful ones for the running project:

| `type` | PHP type | SQL (PostgreSQL) | Notes |
|---|---|---|---|
| `integer` | `int` | `INT` | |
| `bigint` | `int` | `BIGINT` | Use for large counters / snowflake IDs |
| `smallint` | `int` | `SMALLINT` | |
| `float` / `decimal` | `float` | `REAL` / `NUMERIC` | `decimal` takes `precision`/`scale` |
| `boolean` | `bool` | `BOOLEAN` | |
| `string` | `string` | `VARCHAR` / `TEXT` | Add `length` or it may become `TEXT` |
| `text` | `string` | `TEXT` | For long content |
| `datetime_immutable` | `\DateTimeImmutable` | `TIMESTAMP` | We prefer immutable dates |
| `date_immutable` | `\DateTimeImmutable` | `DATE` | Calendar days (issue dates) |
| `json` | `array` | `JSON` / `JSONB` | For flexible settings blobs |
| `array` | `array` | serialized | Rarely what you want — prefer `json` |

```php
#[ORM\Column(type: 'date_immutable')]
private \DateTimeImmutable $issueDate;

#[ORM\Column(type: 'datetime_immutable')]
private \DateTimeImmutable $issuedAt;

#[ORM\Column(type: 'json')]
private array $settings = [];
```

> **Tip — store money as integers.** Never map money to a `float` or `decimal` and do arithmetic in PHP — you'll meet `19.00 - 0.01` becoming `18.990000000000002`. Model monetary amounts as **integer cents** (`int $totalCents`), do integer math (exact), and format only at the presentation edge. `Product::$unitPriceCents` and `Invoice::$totalCents` follow this rule.

##### Persisting and loading

The whole CRUD surface fits in one object. In a service (we keep business logic out of controllers and, for now, out of entities too):

```php
use App\Entity\Product;
use App\Entity\Tenant;
use Doctrine\ORM\EntityManagerInterface;

class CatalogManager
{
    public function __construct(private EntityManagerInterface $em)
    {
    }

    public function addProduct(Tenant $tenant, string $name, int $unitPriceCents): Product
    {
        $product = new Product($name);
        $product->setTenant($tenant);
        $product->setUnitPriceCents($unitPriceCents);

        $this->em->persist($product); // schedules the INSERT
        $this->em->flush();           // runs it, assigns $product->getId()

        return $product;
    }

    public function remove(Product $product): void
    {
        $this->em->remove($product);
        $this->em->flush();
    }
}
```

Loading is the inverse. `find()` consults the identity map first, so within a single request the same row always returns the *same* PHP object:

```php
$product = $this->em->find(Product::class, 123);
// or, when you have a repository:
$product = $em->getRepository(Product::class)->find(123);
```

`find()` returns `null` if there's no such ID, so guard the result. For "find by some condition" you'll almost always reach for a repository method (13.4) rather than a raw `find()`.

#### 13.3 Relationships and Fetch Strategies

Relationships are where object‑to‑table mapping earns its keep — and where most mistakes live. Let's build the invoicing core: a `Company` (a customer), an `Invoice`, and its `LineItem`s.

##### Owning side vs. inverse side

For any association between two entities, exactly **one side is the *owner***: it holds the foreign‑key column in the database. The other side is the *inverse*; it knows about the relationship but stores no FK. This is the single most important concept in the next section, so here's the rule of thumb:

- The side with `#[ORM\JoinColumn]` is the **owner**.
- The side with `mappedBy:` (on a `OneToMany`/`ManyToMany`) is the **inverse**.

A **many‑to‑one** is the simplest and the most common, and it's always the *owning* side. "Many invoices, each pointing at one company" is a many‑to‑one from the invoice's point of view — and the `company_id` column lives on the `invoices` table.

Let's define `Company` and `Invoice` together so you can see both sides of each arrow:

```php
// src/Entity/Company.php
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class Company
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    // many-to-one: OWNING side. company_id lives on the companies table.
    #[ORM\ManyToOne(inverseOwner: Tenant::class)]
    #[ORM\JoinColumn(nullable: false)]
    private ?Tenant $tenant = null;

    #[ORM\Column(length: 255)]
    private string $name;

    #[ORM\Column(length: 255)]
    private string $email;

    #[ORM\Column(nullable: true)]
    private ?string $billingAddress = null;

    // one-to-many: INVERSE side. No FK here; invoices.company_id points back.
    /** @var Collection<int, Invoice> */
    #[ORM\OneToMany(mappedBy: 'company', targetEntity: Invoice::class)]
    private Collection $invoices;

    public function __construct(string $name, string $email)
    {
        $this->name = $name;
        $this->email = $email;
        $this->invoices = new ArrayCollection();
    }
    // getters and setters ...
}
```

```php
// src/Entity/Invoice.php
use App\Repository\InvoiceRepository;
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity(repositoryClass: InvoiceRepository::class)]
#[ORM\Index(columns: ['tenant_id', 'issue_date'], name: 'idx_invoice_tenant_date')]
class Invoice
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    // many-to-one to Company (owning)
    #[ORM\ManyToOne(inverseOwner: Company::class)]
    #[ORM\JoinColumn(nullable: false)]
    private ?Company $company = null;

    // Denormalized many-to-one to Tenant (owning). We repeat the tenant on every
    // invoice so we can scope and isolate queries without joining through Company.
    #[ORM\ManyToOne(inverseOwner: Tenant::class)]
    #[ORM\JoinColumn(nullable: false)]
    private ?Tenant $tenant = null;

    #[ORM\Column(length: 32, unique: true)]
    private string $number; // INV-2026-000123

    #[ORM\Column(enumType: InvoiceStatus::class)]
    private InvoiceStatus $status = InvoiceStatus::Draft;

    #[ORM\Column(type: 'date_immutable')]
    private \DateTimeImmutable $issueDate;

    #[ORM\Column(type: 'date_immutable', nullable: true)]
    private ?\DateTimeImmutable $dueDate = null;

    #[ORM\Column]
    private int $totalCents = 0;

    // one-to-many to LineItem: INVERSE side. FK (line_item.invoice_id) is on the
    // LineItem. We cascade removals so deleting an invoice cleans up its lines.
    /** @var Collection<int, LineItem> */
    #[ORM\OneToMany(
        mappedBy: 'invoice',
        targetEntity: LineItem::class,
        cascade: ['remove'],
        orphanRemoval: true
    )]
    private Collection $lineItems;

    public function __construct()
    {
        $this->lineItems = new ArrayCollection();
    }

    public function addLineItem(LineItem $line): self
    {
        if (in_array($line, $this->lineItems, true)) {
            return $this;
        }
        $line->setInvoice($this);
        $this->lineItems[] = $line;
        return $this;
    }

    public function removeLineItem(LineItem $line): self
    {
        $this->lineItems->removeElement($line);
        return $this;
    }

    /** Recompute the cached total from the current lines. */
    public function recalculateTotal(): void
    {
        $total = 0;
        foreach ($this->lineItems as $line) {
            $total += $line->getLineTotalCents();
        }
        $this->totalCents = $total;
    }
    // getters and setters ...
}
```

Two things worth pausing on.

**We denormalized `tenant` onto `Invoice`.** In a multi‑tenant system, *every* business row should carry the tenant it belongs to, so you can (a) enforce isolation with a cheap `WHERE tenant_id = ?` and (b) run per‑tenant queries without walking through a parent. Yes, it duplicates data; that's the accepted trade for row‑level tenancy. You'll see this "tenant" FK on `Company`, `Product`, and `Invoice` throughout the project, and you'll see it drive the repository methods in 13.4.

**`cascade: ['remove']` + `orphanRemoval: true`** on `Invoice.lineItems` means: removing an `Invoice` removes its lines, and detaching a line from the collection marks it for removal. This is the natural lifecycle for invoice detail rows. We deliberately do *not* `cascade: ['persist']` on the inverse side — persisting is driven from the owning side and the service, and cascades are one of the most common sources of surprising writes. (We *do* put `cascade: ['persist']` on the `LineItem`'s own `invoice` association, shown next, so that persisting a newly built invoice-and-lines graph "just works" from the manager.)

Now the owning end of the invoice ↔ line relationship:

```php
// src/Entity/LineItem.php
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class LineItem
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    // many-to-one to Invoice (OWNING side). line_item.invoice_id lives here.
    #[ORM\ManyToOne(inverseOwner: Invoice::class, cascade: ['persist'])]
    #[ORM\JoinColumn(nullable: false, onDelete: 'CASCADE')]
    private ?Invoice $invoice = null;

    // Optional link to a catalog product (owning, nullable)
    #[ORM\ManyToOne(inverseOwner: Product::class)]
    #[ORM\JoinColumn(nullable: true)]
    private ?Product $product = null;

    #[ORM\Column(length: 512)]
    private string $description;

    #[ORM\Column]
    private int $quantity = 1;

    #[ORM\Column]
    private int $unitPriceCents;

    public function getLineTotalCents(): int
    {
        return $this->quantity * $this->unitPriceCents;
    }
    // getters and setters ...
}
```

The `onDelete: 'CASCADE'` is a database‑level safety net on top of the ORM‑level cascade/orphanRemoval: if a line is deleted by raw SQL or a bulk operation, the DB cleans up. Belt and suspenders, and cheap.

##### One‑to‑one and many‑to‑many

You'll meet these, so here's the shape:

**One‑to‑one.** Model it as a many‑to‑one where the FK is `unique: true`. Put the FK on the "many" side (the part that *has* one of the whole). For example, a `Company` having exactly one `Logo`:

```php
// src/Entity/Company.php (add)
#[ORM\OneToOne(mappedBy: 'company', cascade: ['all'])]
private ?Logo $logo = null;

// src/Entity/Logo.php (add)
#[ORM\Entity]
class Logo
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    #[ORM\OneToOne(inverseOwner: Company::class)]
    #[ORM\JoinColumn(nullable: false, unique: true)]
    private ?Company $company = null;
    // ...
}
```

**Many‑to‑many** uses a *join table* and, by default, is owned from the side declaring the `ManyToMany`. For a SaaS product we usually *avoid* raw many‑to‑many in favor of a real join entity — because join rows need their own data (e.g., a `ProductAssignment` with a per‑tenant price). If you ever need a plain M2M:

```php
#[ORM\ManyToMany]
#[ORM\JoinTable(name: 'product_tag',
    joinColumns:      [new ORM\JoinColumn(name: 'product_id', referencedColumnName: 'id')],
    inverseJoinColumns: [new ORM\JoinColumn(name: 'tag_id', referencedColumnName: 'id')]
)]
/** @var Collection<int, Tag> */
private Collection $tags;
```

For the invoicing domain we stay with one‑to‑many and many‑to‑one, which cover the real model.

##### Eager vs. lazy fetch

By default **every association is `lazy`**. That's a feature: loading an `Invoice` doesn't pull its lines until you touch them. You override this per‑association when a related object is needed *almost every time* you load the parent:

```php
#[ORM\ManyToOne(inverseOwner: Company::class, fetch: 'EAGER')]
#[ORM\JoinColumn(nullable: false)]
private ?Company $company = null;
```

`fetch: 'EAGER'` makes Doctrine join the company into the query that loads the invoice. Be conservative: eager‑loading *collections* (like `lineItems`) is a common way to accidentally load far more rows than you need. Usually the right move is to keep it lazy and fetch‑join *in the specific query* that needs it (see 13.10).

You can also flip the mode at runtime, typically once at boot in a bundle extension or compiler pass, for entities where the decision is global:

```php
$em->getClassMetadata(Invoice::class)
   ->setFetchMode('company', 'EAGER');
```

> **Note:** ORM 3 loads lazy associations through proxy objects. On PHP 8.4+, when you run ORM 3.6+, it can switch to native lazy‑loading objects instead of proxies — same semantics, fewer moving parts. Collections are lazy in every case. You don't have to do anything to benefit; it's a detail that shows up in your profiler flame graphs.

#### 13.4 Repositories

A repository is the gateway to one entity type. Doctrine gives you a default one for free; you subclass it to add the *named, intent‑revealing* queries your domain actually needs. For the running project, the `InvoiceRepository` is where our multi‑tenant rule becomes law: **no invoice query is complete until it's scoped to a tenant.**

##### The base repository

```php
// src/Repository/InvoiceRepository.php
use App\Entity\Invoice;
use App\Entity\InvoiceStatus;
use App\Entity\Tenant;
use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
use Doctrine\Persistence\Mapping\ClassMetadata;

/**
 * @extends ServiceEntityRepository<Invoice>
 */
class InvoiceRepository extends ServiceEntityRepository
{
    // In current DoctrineBundle the constructor takes a ClassMetadata.
    // If you don't need your own constructor arguments, you can omit it
    // entirely and inherit the parent's.
    public function __construct(ClassMetadata $metadata)
    {
        parent::__construct($metadata);
    }

    /**
     * Tenant‑scoped, paginated listing of invoices, optionally by status.
     *
     * @return list<Invoice>
     */
    public function findForTenant(
        Tenant $tenant,
        ?InvoiceStatus $status = null,
        int $limit = 50,
        int $offset = 0
    ): array {
        $qb = $this->createQueryBuilder('i')
            ->andWhere('i.tenant = :tenant')
            ->setParameter('tenant', $tenant)
            ->orderBy('i.issueDate', 'DESC')
            ->setMaxResults($limit)
            ->setFirstResult($offset);

        if (null !== $status) {
            $qb->andWhere('i.status = :status')
               ->setParameter('status', $status);
        }

        return $qb->getQuery()->getResult();
    }

    public function findOneByNumberAndTenant(string $number, Tenant $tenant): ?Invoice
    {
        return $this->createQueryBuilder('i')
            ->andWhere('i.number = :number')
            ->andWhere('i.tenant = :tenant')
            ->setParameter('number', $number)
            ->setParameter('tenant', $tenant)
            ->getQuery()
            ->getOneOrNullResult();
    }
}
```

Why this shape?

- **The `@extends ServiceEntityRepository<Invoice>` docblock** enables full static analysis (PHPStan/Psalm) — `find()`, `findOneBy()`, etc. return properly typed `Invoice`/`?Invoice`.
- **Every method takes a `Tenant`.** It's a deliberate, unmissable requirement: you literally can't call `findForTenant()` without naming a tenant. This is how you make an insecure query impossible to write, rather than relying on discipline.
- **`setMaxResults` + `setFirstResult`** is LIMIT/OFFSET pagination.

##### The built‑in finder methods

You inherit a small, handy toolkit and use it for the trivial cases:

```php
$repo->find($id);                 // by primary key
$repo->findAll();                 // every row — AVOID in app code, see 13.10
$repo->findBy(['tenant' => $t]);  // WHERE tenant_id = ?
$repo->findOneBy(['number' => 'INV-2026-0001']);
$repo->count(['status' => 'paid']);
```

Notice the last two take an **array of conditions** and, crucially, `findOneBy()` here does *not* include the tenant. That's exactly the footgun the named methods above exist to prevent. Reach for the explicit, tenant‑scoped methods; treat the inherited array finders as a dev/CLI convenience, not production logic.

##### Calling it from a service

```php
$invoices = $invoiceRepo->findForTenant($currentTenant, InvoiceStatus::Overdue);
```

Repositories return entities (or scalars), never domain decisions. If a method reads like a business rule ("the invoice is refundable"), that logic probably belongs in a value object or domain service, not the repository. Keep repositories as *data access*.

#### 13.5 DQL: The Entity Query Language

Sometimes you need a query that a repository method's fluent building can't express cleanly. DQL (Doctrine Query Language) is SQL‑like, but it operates on **entities and their fields**, not tables and columns. Doctrine translates it to real SQL for your driver.

```sql
SELECT i.number, i.status, i.totalCents, i.issueDate
FROM App\Entity\Invoice i
WHERE i.tenant = :tenant
  AND i.issueDate >= :since
ORDER BY i.issueDate DESC
```

You run DQL through the entity manager:

```php
use Doctrine\ORM\EntityManagerInterface;

$q = $em->createQuery(<<<DQL
    SELECT i
    FROM App\Entity\Invoice i
    WHERE i.tenant = :tenant AND i.status = :status
    ORDER BY i.issueDate DESC
DQL);
$q->setParameter('tenant', $tenant);
$q->setParameter('status', InvoiceStatus::Overdue);

$invoices = $q->getResult(); // list of App\Entity\Invoice
```

**Always use named parameters.** Interpolating values into the DQL string is both a SQL‑injection vector and a type error — DQL parameters are how Doctrine knows to bind a `Tenant` object (and resolve its ID) rather than a literal.

##### Aggregates and groups

DQL supports the usual aggregate functions: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`. A per‑status rollup for the tenant dashboard:

```php
$rows = $em->createQuery(<<<DQL
    SELECT i.status, COUNT(i) AS cnt, SUM(i.totalCents) AS total
    FROM App\Entity\Invoice i
    WHERE i.tenant = :tenant
    GROUP BY i.status
DQL)
    ->setParameter('tenant', $tenant)
    ->getResult(); // each row is an array: ['status' => ..., 'cnt' => ..., 'total' => ...]
```

When you only want a single number, use `getSingleScalarResult()` so you get a plain value back, not an array:

```php
$overdueCount = (int) $em->createQuery(
    'SELECT COUNT(i) FROM App\Entity\Invoice i WHERE i.tenant = :t AND i.status = :s'
)
    ->setParameter('t', $tenant)
    ->setParameter('s', InvoiceStatus::Overdue)
    ->getSingleScalarResult();
```

##### Subqueries and EXISTS

DQL supports subqueries in `WHERE`, and `EXISTS` — handy for "give me the companies that have at least one paid invoice":

```sql
SELECT c
FROM App\Entity\Company c
WHERE c.tenant = :tenant
  AND EXISTS (
      SELECT i FROM App\Entity\Invoice i
      WHERE i.company = c AND i.status = :paid
  )
```

##### Built‑in string and numeric functions

You can call a set of SQL functions directly in DQL without dropping to raw SQL: `LOWER`, `UPPER`, `CONCAT`, `LENGTH`, `SUBSTRING`, `TRIM`, `ABS`, `ROUND`, `MOD`, `LOCATE`, and more. A case‑insensitive search on the invoice number:

```sql
SELECT i FROM App\Entity\Invoice i
WHERE i.tenant = :tenant
  AND LOCATE(:needle, LOWER(i.number)) > 0
```

##### Hydration modes

`getResult()` hydrates **entity objects** (the default, `Query::HYDRATE_OBJECT`). For read‑only reporting where instantiating the full object graph is wasteful, hydrate to plain arrays or scalars:

```php
$rows = $q->getResult(Query::HYDRATE_ARRAY);   // list of associative arrays
```

For a single column of a single row, `getSingleScalarResult()` and `getSingleResult()` (a single entity) are your friends.

##### When DQL isn't enough

DQL is for *object* queries. When you need raw SQL, a vendor‑specific feature, or a bulk operation the UoW would fight you on, drop to **DBAL** through the connection:

```php
$conn = $em->getConnection();

// read
$sum = $conn->fetchOne(
    'SELECT COALESCE(SUM(total_cents), 0) FROM invoice WHERE tenant_id = ?',
    [$tenant->getId()]
);

// write (bulk, bypasses the identity map)
$conn->executeStatement(
    'UPDATE invoice SET status = ? WHERE tenant_id = ? AND issue_date < ?',
    ['overdue', $tenant->getId(), '2026-01-01']
);
```

> **Warning:** DBAL writes do **not** go through the Unit of Work or the identity map. If you update a row via DBAL, any in‑memory copy of that entity is now stale — clear or reload it (`$em->clear(Invoice::class)` or `refresh($entity)`). Use DBAL for bulk/edge cases, not for everyday CRUD.

#### 13.6 The QueryBuilder

The QueryBuilder is the fluent, method‑chaining way to build the same DQL you'd write by hand. It's the workhorse for repository methods and ad‑hoc, conditionally‑assembled queries.

```php
use Doctrine\ORM\QueryBuilder;

$qb = $em->createQueryBuilder();

$qb
    ->select('i')
    ->from(Invoice::class, 'i')
    ->andWhere('i.tenant = :tenant')
    ->andWhere('i.status IN (:statuses)')
    ->setParameter('tenant', $tenant)
    ->setParameter('statuses', [InvoiceStatus::Sent, InvoiceStatus::Overdue])
    ->orderBy('i.issueDate', 'DESC')
    ->setMaxResults(25);

$invoices = $qb->getQuery()->getResult();
```

Notes on the API:

- `from(Invoice::class, 'i')` — the **alias** (`i`) is what you reference everywhere after. In repository methods, `$this->createQueryBuilder('i')` skips the `from()` and starts with the alias ready.
- **`andWhere` vs `where`**: `where()` *replaces* any existing condition; `andWhere()` *adds* with an AND. When assembling conditions in a loop, prefer `andWhere()` so earlier conditions survive.
- **Parameters by name** keep the DQL clean and injection‑safe, exactly as in raw DQL.

##### Complex conditions with `expr()`

For ORs, grouped comparisons, and nested logic, use the expression builder so Doctrine generates the parentheses and SQL for you. (Note the naming: the historical `and()`/`or()` were renamed to **`andX()`/`orX()`** and the old names are gone in ORM 3.)

```php
use Doctrine\ORM\QueryBuilder;

$qb = $this->createQueryBuilder('i');
$expr = $qb->expr();

$qb->where($expr->andX(
    $expr->eq('i.tenant', ':tenant'),
    $expr->orX(
        $expr->contains('LOWER(i.number)', ':q'),
        $expr->gte('i.totalCents', ':min'),
        $expr->eq('i.status', ':status')
    )
))
->setParameter('tenant', $tenant)
->setParameter('q', mb_strtolower($search))
->setParameter('min', $minCents)
->setParameter('status', InvoiceStatus::Paid);
```

That assembles:

```
WHERE i.tenant = :tenant
  AND ( LOWER(i.number) LIKE '%:q%'
     OR i.total_cents >= :min
     OR i.status = :status )
```

##### DQL vs QueryBuilder vs DBAL

A quick decision guide:

| Situation | Use |
|---|---|
| Fixed, named query on one entity, reusable | **Repository method** (QueryBuilder or DQL inside) |
| Dynamically assembled WHERE (search/filter form) | **QueryBuilder** |
| One‑off, read‑only, no object graph needed, or aggregate | **DQL** with `HYDRATE_ARRAY` / scalar results |
| Raw SQL, vendor features, or bulk updates that bypass the UoW | **DBAL** |

There's no single "right" tool — repositories give you the *place* to keep queries; DQL and QueryBuilder are how you *write* them; DBAL is the escape hatch.

#### 13.7 Lifecycle Callbacks

Entities can hook into their own persistence lifecycle with attributes: `#[ORM\PrePersist]`, `#[ORM\PostPersist]`, `#[ORM\PreUpdate]`, `#[ORM\PostUpdate]`, `#[ORM\PreRemove]`, `#[ORM\PostRemove]`, and `#[ORM\PostLoad]`. The order for a save is:

```
PrePersist  →  (INSERT)  →  PostPersist        // first time only
PreUpdate   →  (UPDATE)  →  PostUpdate         // on subsequent changes
PreRemove   →  (DELETE)  →  PostRemove
PostLoad                                                    // every time it's loaded
```

The classic, appropriate use is **stamping timestamps** — small, stateless, no collaborators needed:

```php
// src/Entity/Invoice.php (add)
#[ORM\Column(type: 'datetime_immutable')]
private \DateTimeImmutable $createdAt;

#[ORM\Column(type: 'datetime_immutable', nullable: true)]
private ?\DateTimeImmutable $updatedAt = null;

#[ORM\PrePersist]
public function onPrePersist(): void
{
    $this->createdAt = new \DateTimeImmutable();
    $this->updatedAt = $this->createdAt;
}

#[ORM\PreUpdate]
public function onPreUpdate(): void
{
    $this->updatedAt = new \DateTimeImmutable();
}
```

Two cautions keep these safe:

**Callbacks receive no arguments.** You cannot write `onPrePersist(Tenant $t)` — the method must be `onPrePersist(): void` with zero parameters. That means you *can't* stamp a tenant from a callback; the tenant has to come from whatever context the callback's closure *can* see, which is nothing. Stamping the tenant belongs in the service that creates the invoice (the `InvoiceManager`), or in an event subscriber that has access to the current request's tenant. This is a frequent "gotcha," so here's the pattern we actually use:

```php
// src/Service/InvoiceManager.php
class InvoiceManager
{
    public function __construct(private EntityManagerInterface $em)
    {
    }

    public function createDraft(Tenant $tenant, Company $company, \DateTimeImmutable $issueDate): Invoice
    {
        $invoice = new Invoice();
        $invoice->setTenant($tenant);          // tenant set HERE, in context
        $invoice->setCompany($company);
        $invoice->setNumber($this->nextNumber($tenant));
        $invoice->setIssueDate($issueDate);

        $this->em->persist($invoice);
        $this->em->flush();

        return $invoice;
    }

    public function recalculate(Invoice $invoice): void
    {
        $invoice->recalculateTotal();          // derived state, kept explicit
        $this->em->flush();
    }
}
```

**Keep entities thin.** Callbacks that inject services, write to other tables, send emails, or make network calls turn your entity into a hidden coupling nightmare and make it hard to test. The rule: an entity callback may transform *its own* state (timestamps, a normalized slug, recomputing a field from its own data). The moment you need a collaborator, that logic is a **service** or an **event subscriber** (Chapter 7), triggered by the domain event your service raises — not by the entity.

> **Tip:** `PostLoad` is the one callback that fires on every read. It's occasionally used for denormalization, but it runs for *every* entity load in *every* query, so it's a performance trap. If you only need a value on one page, compute it in the query (a fetch‑join or a select), not in `PostLoad`.

#### 13.8 Migrations

Entities describe the *schema you want*. Migrations are the versioned, reviewable SQL that turns the database into that schema — and, just as importantly, that lets you roll it back.

Your schema changes whenever you add an entity, a column, a relationship, or an index. The workflow:

```bash
# 1. You've changed an entity (e.g., added Product, Company, Invoice, LineItem).

# 2. Validate the mapping is internally consistent (no bad FKs, orphaned mappedBy, etc.)
bin/console doctrine:schema:validate

# 3. Generate a migration from the diff between mapping and live schema
bin/console doctrine:migrations:diff

# 4. Review the generated file under migrations/ (always — see below)

# 5. Apply it
bin/console doctrine:migrations:migrate

# 6. Inspect state anytime
bin/console doctrine:migrations:status
```

`make:migration` gives you an *empty* migration when you want to write SQL by hand (a data backfill, a manual index, a raw `ALTER`):

```bash
bin/console make:migration
```

A generated migration looks like this:

```php
<?php

declare(strict_types=1);

namespace DoctrineMigrations;

use Doctrine\DBAL\Schema\Schema;
use Doctrine\Migrations\AbstractMigration;

final class Version20260312123456 extends AbstractMigration
{
    public function up(Schema $schema): void
    {
        // This method is called when migrating UP.
        $this->addSql('CREATE TABLE product (
            id SERIAL PRIMARY KEY,
            tenant_id INT NOT NULL,
            name VARCHAR(255) NOT NULL,
            description TEXT DEFAULT NULL,
            unit_price_cents INT NOT NULL,
            active BOOLEAN NOT NULL,
            created_at TIMESTAMP(0) WITHOUT TIME ZONE NOT NULL,
            INDEX idx_product_tenant (tenant_id)
        )');
        // ... company, invoice, line_item tables and their FKs ...
    }

    public function down(Schema $schema): void
    {
        // This method is called when rolling back the migration.
        $this->addSql('DROP TABLE line_item');
        $this->addSql('DROP TABLE invoice');
        $this->addSql('DROP TABLE company');
        $this->addSql('DROP TABLE product');
    }
}
```

Rolling back one step (or a specific version):

```bash
bin/console doctrine:migrations:rollback          # one step back
bin/console doctrine:migrations:rollback 20260312123456
```

##### Migration hygiene

- **Always review the generated SQL.** The diff is a *suggestion*. For existing tables it might propose destructive `ALTER` steps, drop data, or miss a `NOT NULL` you'd rather backfill first. Treat it as a starting point.
- **Write a real `down()`.** A `down()` that just does nothing means your rollback silently fails to restore state. At minimum, drop what `up()` created.
- **Big tables, separate data migrations.** On a table with millions of rows, a `NOT NULL` column with no default can fail or lock. Split it: (1) add the column nullable, (2) backfill in batches, (3) alter to `NOT NULL`. Each is its own small, testable migration.
- **Migrations are code — version them, commit them, and never edit an applied one.** If you need to change history, add a new migration; rewriting applied migrations breaks every other environment.
- **Don't run `doctrine:schema:update` in production.** It's a dev shortcut that issues unreviewed DDL. Migrations are the production path.

In CI, gate deploys on `bin/console doctrine:migrations:status` (no pending) and `doctrine:schema:validate` (clean mapping), then apply migrations in the release step before you switch traffic (more on zero‑downtime in Chapter 24).

#### 13.9 Fixtures

Fixtures are the seeded, repeatable data you load into a dev or test database: a couple of tenants, a few companies, some invoices. They make local development and functional tests deterministic.

Fixtures are **not** how production gets data — they're for `dev` and `test`. Load them with:

```bash
# wipe + reload (destructive — dev only)
bin/console doctrine:fixtures:load
```

A minimal fixture:

```php
<?php

namespace App\DataFixtures;

use App\Entity\Tenant;
use Doctrine\Bundle\FixturesBundle\Fixture;
use Doctrine\Persistence\ObjectManager;

class TenantFixtures extends Fixture
{
    public function load(ObjectManager $manager): void
    {
        $acme = new Tenant();
        $acme->setName('ACME Corp');
        $acme->setSlug('acme');

        $manager->persist($acme);
        $this->addReference('tenant.acme', $acme); // stash it for other fixtures

        $manager->flush();
    }
}
```

To order fixtures and let one reference another, implement `DependentFixtureInterface` and declare what you need:

```php
<?php

namespace App\DataFixtures;

use App\Entity\Company;
use App\Entity\Invoice;
use App\Entity\InvoiceStatus;
use App\Entity\LineItem;
use App\Entity\Product;
use Doctrine\Bundle\FixturesBundle\Fixture;
use Doctrine\Common\DataFixtures\DependentFixtureInterface;
use Doctrine\Persistence\ObjectManager;

class InvoicingFixtures extends Fixture implements DependentFixtureInterface
{
    public function load(ObjectManager $manager): void
    {
        $tenant = $this->getReference('tenant.acme'); // set by TenantFixtures

        $product = new Product('Monthly Hosting');
        $product->setTenant($tenant);
        $product->setUnitPriceCents(4_900);
        $manager->persist($product);

        $company = new Company('Globex', 'billing@globex.example');
        $company->setTenant($tenant);
        $manager->persist($company);

        $invoice = new Invoice();
        $invoice->setTenant($tenant);
        $invoice->setCompany($company);
        $invoice->setNumber('INV-2026-000001');
        $invoice->setStatus(InvoiceStatus::Sent);
        $invoice->setIssueDate(new \DateTimeImmutable('2026-09-01'));

        $line = new LineItem();
        $line->setInvoice($invoice);
        $line->setProduct($product);
        $line->setDescription('Monthly hosting — September');
        $line->setQuantity(1);
        $line->setUnitPriceCents(4_900);

        $invoice->addLineItem($line);
        $invoice->recalculateTotal();

        $manager->persist($invoice);
        $manager->persist($line);
        $manager->flush();
    }

    public function getDependencies(): array
    {
        return [TenantFixtures::class]; // run TenantFixtures first
    }
}
```

`addReference()` / `getReference()` are the fixture world's way of sharing objects across `load()` calls without hard‑coding IDs — the robust way to build a consistent seed graph.

##### Grouping and testing

Group fixtures so you can load subsets (a light "smoke" set vs. a heavy "load" set):

```php
class LoadFixtures extends Fixture implements GroupsFixtureInterface
{
    public function getGroups(): array
    {
        return ['load']; // heavier, opt‑in fixtures
    }
    // ...
}
```

```bash
bin/console doctrine:fixtures:load --group load
```

In **functional tests**, you typically combine fixtures with the framework's database reset so each test starts clean — that's covered properly in Chapter 22. For local dev, the plain `doctrine:fixtures:load` is enough.

#### 13.10 N+1 Detection and Query Performance

This is the section that separates a Doctrine app that's fast at 1k rows from one that crawls at 100k. Let's name the enemy first.

##### The N+1 problem

Imagine the invoices list page. You load the tenant's invoices (1 query). Then, for each invoice, you render its company name:

```php
$invoices = $repo->findForTenant($tenant);        // query #1 → N invoices
foreach ($invoices as $invoice) {
    $company = $invoice->getCompany()->getName(); // lazy proxy → 1 query EACH
}
```

That's **1 + N queries**: one for the list, then one per invoice to fetch its company. With 100 invoices, that's 101 round trips to the database. This is the N+1 problem, and lazy loading makes it *easy* to write and *hard* to notice until the data grows.

The same trap exists with collections: `foreach ($invoice->getLineItems() as $line)` issues a query per invoice if the collection hasn't been loaded.

##### Detecting it

You can't fix what you can't see. Three tools, in order of convenience:

**1. The Web Profiler (day‑to‑day).** Open any request in the dev environment's Profiler and look at the **Doctrine** panel. It lists *every* SQL query, the query count, and highlights the slow ones. A list page that fires 100+ queries is screaming N+1.

**2. `doctrine:query:sql` (CLI).** See exactly the SQL a piece of DQL produces, without running the app:

```bash
bin/console doctrine:query:sql "SELECT i FROM App\Entity\Invoice i JOIN FETCH i.company c WHERE i.tenant = 1"
```

**3. A PSR‑3 SQL logger.** For tests and long‑running commands, attach a logger to DBAL so each query is written somewhere you can count. In DBAL 4 logging is a middleware; configure it under `dbal: middlewares:` pointing at a logger service, and grep your logs for the query count on a request. (We wire the Profiler for dev; the logger is the headless equivalent.)

> **Rule of thumb:** if the query count on a page scales with the number of rows displayed, you have an N+1. It should be constant (a handful of queries), no matter how many rows you render.

##### The fixes

**1. Fetch‑join the association you need, in the query that needs it.** This is the primary tool. Add `JOIN FETCH` so the related rows come back in the *same* query as the parents:

```sql
SELECT i
FROM App\Entity\Invoice i
JOIN FETCH i.company c
WHERE i.tenant = :tenant
ORDER BY i.issueDate DESC
```

Now the list page is *one* query. For a collection:

```sql
SELECT i
FROM App\Entity\Invoice i
JOIN FETCH i.lineItems li
WHERE i.tenant = :tenant
```

In QueryBuilder the same is `->addSelect('li')->join('i.lineItems', 'li')`.

```php
return $this->createQueryBuilder('i')
    ->addSelect('li')
    ->join('i.lineItems', 'li')
    ->andWhere('i.tenant = :tenant')
    ->setParameter('tenant', $tenant)
    ->getQuery()
    ->getResult();
```

> **Warning — pagination + fetch‑join on a collection.** When you `setMaxResults()` a query that fetch‑joins a *collection*, Doctrine limits the number of **SQL rows**, not the number of **root entities** — an invoice with 5 lines occupies 5 rows, so you don't get the invoices you expect. Two fixes: (a) apply the limit to a subquery over the root entity, or (b) on ORM 3.1+ use `$query->getResultForPagination()`, which handles the count and offset correctly for fetch‑joined results. For paginated *lists* you usually fetch‑join only the scalar relation (the company) and load line items lazily on the detail page.

**2. Make it eager *only* when it's needed almost always.** If you load an `Invoice` and essentially always want its company, `fetch: 'EAGER'` on that association (13.3) removes the extra query by default. Don't do this on collections you usually don't need — you'll pay for the join on every single load.

**3. Select less, or hydrate less.** For a table of invoices you may not need the full object. Select the columns you render:

```php
$rows = $em->createQuery(
    'SELECT i.number, i.status, i.totalCents, i.issueDate FROM App\Entity\Invoice i WHERE i.tenant = :t'
)
    ->setParameter('t', $tenant)
    ->getResult(Query::HYDRATE_ARRAY); // plain arrays, no proxies
```

Or, if you need entities but only a few fields, restrict hydration with `setPartialObjects()` so Doctrine builds a *partial* object and you know touching anything else throws rather than silently querying.

**4. Index the columns you filter and sort.** We already put `#[ORM\Index]` on `tenant_id` and on the `(tenant_id, issue_date)` pair in `Invoice`. A query that scans the whole table to find a tenant's invoices will beat every fetch‑join trick you apply. Look at the Profiler's slowest queries and check the `EXPLAIN` for a seq scan; if there is one, you're missing an index.

**5. Bound memory in loops with `clear()`.** When a command or a batch job processes thousands of entities, the identity map grows without limit. Periodically flush and clear:

```php
$i = 0;
foreach ($ids as $id) {
    $invoice = $em->find(Invoice::class, $id);
    $this->process($invoice);
    if (++$i % 100 === 0) {
        $em->flush();
        $em->clear(); // releases the identity map so memory stays flat
    }
}
```

##### A performance checklist

Before you ship a data‑heavy page, run this list:

1. **Constant query count?** Open the Profiler on the page; confirm queries don't grow with row count.
2. **Fetch‑join what you always render**, especially scalar relations on list pages.
3. **Index FKs and filter/sort columns**; check `EXPLAIN` for full scans.
4. **Paginate** anything unbounded — never `findAll()` in app code.
5. **Hydrate arrays or partial objects** for read‑only reports.
6. **`clear()` in long loops** to keep the identity map bounded.
7. **Denormalize for the hot path** (like `totalCents` and `tenant` on `Invoice`) so the common query avoids a join or recompute.

Doctrine is fast when you're deliberate about *which* rows you ask for and *how many* times you ask. N+1 is the tax you pay for being sloppy about that; the tools above are how you pay it once and never again.

#### 13.11 Exercises

Work against the running project (a `dev` database with fixtures loaded). Assume `Tenant`, `User` (Ch. 12), and the entities from this chapter exist.

1. **Model a discount.** Add a `Discount` entity: many‑to‑one to `Tenant`, a `code` (unique per tenant), and a `percent` (1–100). Add a `getDiscountByCode(string $code)` method to a new `DiscountRepository` that is tenant‑scoped. Generate and review the migration; confirm `doctrine:schema:validate` passes.

2. **Apply it.** Add `Discount` (optional) to `Invoice`, then a `recalculateTotal()` that applies the discount's percent to the pre‑discount sum. Re‑seed fixtures with a discount and verify the stored `totalCents` is correct. Add a `PostPersist`/service test of your own to prove the total is recomputed, not manually set.

3. **Dashboard aggregates.** Write a repository method returning, for a tenant, the count and total cents of invoices grouped by status (the aggregate DQL from 13.5). Add a second method: "the number of companies with at least one paid invoice" using an `EXISTS` subquery.

4. **Fix an N+1.** Introduce a listing method that loads a tenant's invoices and, for each, reads the company name (deliberately reproducing the N+1). Confirm it in the Profiler. Then fix it with a `JOIN FETCH`, confirm the query count is now constant, and repeat the exercise for a page that also needs each invoice's line items.

5. **Pagination gotcha.** Take the line‑items fetch‑join from 13.10 and add `setMaxResults(5)`. Observe the surprising result set. Fix it using either the subquery‑limit approach or `getResultForPagination()`, and explain in a code comment which you chose and why.

6. **Tenant‑safe by construction.** Refactor the inherited `findBy([...])` calls (if any remain) in the project into explicit, tenant‑scoped repository methods. Grep the codebase for `->find(`, `->findAll()`, and `->findBy(` on invoice/company repositories and remove or justify each.

7. **Data migration.** Add a `notes` column (`text`, nullable) to `Invoice`. Split it into two migrations: one that adds it nullable, and one that backfills existing rows with an empty string. Write the `down()` for both.

8. **Lifecycle discipline.** Add `createdAt`/`updatedAt` to `Company` using `#[ORM\PrePersist]`/`#[ORM\PreUpdate]`. Then add a *service* (not a callback) that generates the `number` on new invoices from the tenant and the current year — and explain in a comment why the number generation belongs in the service and not a callback.

#### 13.12 Where this leaves us

You now have the full persistence spine of the running project: entities mapped with attributes, tenant‑scoped repositories, DQL and QueryBuilder for the queries those repositories run, migrations that evolve the schema, fixtures that seed it, and the performance habits that keep it fast.

We'll keep building on this. Chapter 14 turns to the *front end* — wiring the app's assets through AssetMapper and a build tool so the templates you've been writing actually have styles and scripts to serve. After that, Parts IV and V take these same entities and repositories out of the browser: console commands and Messenger jobs that *generate* invoices (Chapter 17), a REST and GraphQL API layer that *exposes* them (Chapters 19–21), and the testing and production hardening that makes all of it trustworthy (Part VI).

The single habit to carry forward from this chapter: **every query is a decision about how many rows you fetch and how many times you ask the database for them.** Get that right and Doctrine's object model is a gift; get it wrong and no amount of object elegance survives contact with a large table.

**Ch 14. Frontend Integration**
- AssetMapper: asset references, versioning, hot reload
- Integrating Vite/webpack; serving built assets in production

## Part IV — Beyond the Browser
**Ch 15. Console Commands**
- Command structure, arguments/options, interactive input
- Progress bars, output formatting, long-running tasks

**Ch 16. Email and Notifications**
- Mailer component: transports, templated messages, attachments
- Handling bounces and failures; testing mail locally (Mailpit)

**Ch 17. Asynchronous Processing with Messenger**
- Messages, handlers, routing; sync vs. async transports
- Retries, dead-letter queues, middleware, running workers

**Ch 18. Scheduling and Webhooks**
- The Scheduler component: cron-style tasks, locking
- The Webhook component: receiving and verifying external events

## Part V — APIs
**Ch 19. REST APIs with the Serializer**
- Serialization groups, normalizers, denormalizers
- Format negotiation, error formats, rate limiting

**Ch 20. API Platform**
- Declaring resources; OpenAPI/Swagger generation
- GraphQL, JSON:API, filtering and pagination

**Ch 21. API Authentication**
- Stateless authenticators, JWT, API keys, OAuth2 patterns

## Part VI — Quality and Production
**Ch 22. Testing**
- Unit vs. functional tests; `WebTestCase`, fixtures, mocking
- Browser testing with Panther; test environments and CI integration

**Ch 23. Debugging and Performance**
- VarDumper, Web Profiler, Blackfire
- Caching: cache pools, HTTP caching, invalidation strategies

**Ch 24. Deployment and Operations**
- Web server choices (PHP-FPM, FrankenPHP), Docker setup
- CI/CD pipelines, zero-downtime deploys, logging, monitoring, error tracking

## Part VII — Advanced Topics
**Ch 25. Workflows: Modeling State Machines**
- States, transitions, guards; visualizing workflows

**Ch 26. Specialized Components**
- UID (ULID/UUID), Lock, Semaphore, RateLimiter in practice

**Ch 27. Internationalization and Localization**
- Translation catalogs, ICU messages, locale negotiation

**Ch 28. Contributing to Symfony**
- Reading the codebase, writing tests for components, submitting PRs

## Appendices
- A: Cheat sheet (routing, DI, security, console commands)
- B: Component reference table with use cases
- C: Glossary
- D: Further resources (official docs, blog, community)

---

**Notes on currency:** The outline targets Symfony 7.4 LTS and the 8.x line (PHP 8.2+), using attribute-based configuration throughout and covering newer components (AssetMapper, Scheduler, Webhook) that have become standard in modern Symfony apps.
