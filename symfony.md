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

**Ch 7. Events and Middleware**
- EventDispatcher: custom events, subscribers, priorities
- Kernel events for cross-cutting concerns; middleware-style patterns

**Ch 8. Bundles and Extension Points**
- Anatomy of a bundle; extending the framework with compiler passes
- When to write a bundle vs. a plain service

## Part III — Building Web Applications
**Ch 9. Twig Templating**
- Syntax, inheritance, macros, filters, functions
- Form themes, sandboxing, autoescaping, template caching and debugging

**Ch 10. Forms**
- Built-in field types, form builders, data mapping
- Custom field types, validation integration, CSRF protection

**Ch 11. Validation**
- Constraint catalog, groups, compound constraints
- Writing custom validators; translating violations

**Ch 12. Security**
- Firewalls and authenticators (custom authenticator walkthrough)
- Roles, voters, access control, password hashing
- Two-factor authentication, login throttling, session fixation

**Ch 13. Doctrine ORM**
- Entities, mappings via attributes, relationships and fetch strategies
- DQL, QueryBuilder, repositories, lifecycle callbacks
- Migrations, fixtures, N+1 detection, query performance

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
