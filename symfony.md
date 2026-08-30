# Book Outline: *Symfony in Depth: From First Request to Production*

**Working title:** *Symfony in Depth: From First Request to Production*
**Target audience:** Developers with solid PHP (8.2+) and basic web knowledge who want to build professional applications with Symfony 7.4 LTS / 8.x.
**Pedagogical approach:** Each chapter ends with exercises; a running project (a multi-tenant SaaS invoicing app) is built incrementally across Parts III–V, then hardened in Part VI.

---

## Front Matter
- Who this book is for / prerequisites (PHP 8.2+, Composer, basic HTTP & OOP)
- How the book is organized; conventions (code style, attribute vs. XML/YAML config)
- Setting up the environment: PHP, Symfony CLI, creating a project, directory layout, `.env` files

## Part I — Foundations
**Ch 1. The Symfony Ecosystem**
- History and design philosophy; components vs. bundles vs. framework
- Release model (minor every 6 months, LTS every 2 years), upgrade strategy
- Tour of the component library and where each fits

**Ch 2. HTTP Fundamentals with HttpFoundation**
- `Request` and `Response` objects; sessions, cookies, flash messages
- File uploads, client IP handling, request attributes vs. query parameters

**Ch 3. Routing**
- Route definitions via attributes; requirements, defaults, placeholders
- Redirects, route priorities, URL generation in templates and code
- Debugging routes with the console

**Ch 4. Controllers and the Request Lifecycle**
- Controller types (invokable, value objects, service controllers)
- The kernel event pipeline: from `request` to `response`
- Return types: responses, streams, redirects, JSON

## Part II — Core Architecture
**Ch 5. Dependency Injection**
- Service definitions, autowiring, autoconfiguration
- Constructor injection, service locator, decorators, aliases, tags
- Public vs. private services; container compilation and debugging

**Ch 6. Configuration System**
- `.env` hierarchy, parameters, per-environment overrides
- Bundle configuration trees and `config/` organization

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
