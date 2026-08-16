# A practical handbook for PHP 8.5 / Symfony and TypeScript / Vue engineers

**Identity, APIs, security, testing, and delivery — from local Docker to production GitLab CI.**

---

## Positioning

**Audience.** Mid-level engineers joining a dual-stack shop, and experienced engineers who need a single source of truth for “how we build here.” Assumes comfort with HTTP and one language; does not assume both stacks.

**Promise.** After this book, a reader can design, implement, secure, test, debug, and ship a feature that spans a Symfony API, a Vue SPA, Redis, a queue, MySQL, NGINX, and GitLab CI — and can explain *why* each choice exists.

**Pedagogy.** Each chapter follows the same rhythm:

1. Concept and failure mode  
2. Standard or RFC (not folklore)  
3. PHP implementation  
4. TypeScript implementation  
5. Cross-stack contract  
6. Anti-patterns and review checklist  

**Running example.** A multi-tenant “Project Board” product: users, orgs, boards, cards, comments, webhooks, background jobs, feature flags, and an SPA. Every later chapter extends the same system.

**Version floor (as of August 2026).** PHP 8.5 (GA November 2025); Symfony 8.1 (current stable, PHP ≥ 8.4); Vue 3.5+ / Pinia / Vue Router 4; MySQL 9.x; Node LTS. Treat older folklore (especially MySQL 5.5-era habits) as historical, not current.[[1]](https://www.zend.com/resources/php-versions)

---

## Front Matter

- How this book maps to the existing frozen developer guidelines  
- Conventions: code samples, “do this / never this,” ADR callouts  
- Local prerequisites: Docker, Git, a modern browser, Node, PHP 8.5 CLI  
- How to run the companion repo (`docker compose up`)  
- Typographic conventions for paths, commands, and types  

---

# Part I — Foundations

## Chapter 1. The Dual-Stack Mental Model

- Why this shop is not “PHP *or* Node”: Symfony owns the domain and persistence; Vue owns interaction; Node microframeworks own glue (webhooks, workers, BFF, internal tools)  
- Request lifetime: browser → NGINX → PHP-FPM / Node → MySQL / Redis / queue  
- Bounded contexts and the API as the contract  
- What “full-stack” means here (you own the feature end to end, not every runtime equally)  
- When *not* to introduce a new runtime or store  

## Chapter 2. Git as Daily Craft

- Mental model: commits as reviewable decisions, not save points  
- Branching: short-lived feature branches, protected default, no long-lived personal mains  
- History hygiene: rebase vs merge, when to squash, when *not* to rewrite  
- Commit messages that survive code review and `git blame`  
- Bisect, reflog, worktrees, sparse checkout — the tools you actually need  
- `.gitignore`, LFS policy, and what never belongs in the repo (secrets, vendor, `node_modules`, dumps)  
- Code review as a security and design control, not a style nitpick  
- Signing, protected branches, and supply-chain basics (ties forward to OWASP A03)

## Chapter 3. The Web Platform You Are Actually Shipping

- HTML as a contract with the browser, not a Vue template leftover  
- CSS: cascade, specificity, custom properties, container queries, logical properties; what Stylelint will enforce later  
- Modern ECMAScript: modules, iterators, `async`/`await`, temporal/collection APIs you may actually use, and what to leave to TypeScript  
- Web APIs that matter in an SPA: Fetch, History, Storage, Intersection Observer, Clipboard, Credential Management  
- Progressive enhancement vs. “SPA-only” — when each is honest  
- Browser support policy and how it constrains Vite targets  

## Chapter 4. Accessibility as a Feature Constraint

- WCAG 2.2 orientation: POUR (Perceivable, Operable, Understandable, Robust)  
- Semantic HTML first; ARIA only to fill gaps  
- Keyboard model, focus management, and SPA route-change announcements  
- Forms, errors, and live regions  
- Color, contrast, motion, and `prefers-reduced-motion`  
- Testing: axe, Playwright accessibility snapshots, keyboard-only QA  
- Vue-specific pitfalls: custom components that destroy native semantics; `RouterLink` vs. real buttons  
- Accessibility is not a sprint-end audit — it is an acceptance criterion  

---

# Part II — Identity, Authorization, and Security

## Chapter 5. OAuth 2.0 and OpenID Connect

Treat these as two layers, not synonyms. OAuth 2.0 is delegated *authorization*. OpenID Connect is *authentication* built on OAuth. Do not derive identity from an access token.[[2]](https://openid.net/developers/how-connect-works/)

**Concepts**

- Roles: Resource Owner, Client, Authorization Server / OpenID Provider, Resource Server  
- Tokens: authorization code, access token, refresh token, ID token (JWT)  
- Grants that are still acceptable: Authorization Code + PKCE for public clients; client credentials for service-to-service  
- Grants that are dead: Implicit, Resource Owner Password Credentials  
- PKCE, exact redirect URI matching, `state` / `nonce`, sender-constrained tokens  
- OAuth 2.1 as the “secure defaults made mandatory” reading of 2.0  

**Implementation**

- Symfony: `league/oauth2-server` or an external IdP; security firewalls; storing tokens; clock skew  
- Vue / SPA: Authorization Code + PKCE; token storage (memory vs. httpOnly cookie BFF); silent renew  
- Node microservices validating JWTs (issuer, audience, signature, expiry)  
- Machine-to-machine between PHP workers and internal APIs  

**Failure modes**

- Confused deputy, token leakage via logs/URLs, refresh-token rotation mistakes, mix-up attacks  
- Checklist for reviewing any new client  

## Chapter 6. Authorization Models: RBAC, ABAC, ReBAC

OWASP still prefers attribute- and relationship-based control over role-only designs. Roles remain useful as *one* attribute, not the whole model.[[3]](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)

- RBAC: roles → permissions. Fast to start, explodes into role soup  
- ABAC: subject / resource / environment attributes. Least privilege, harder to reason about  
- ReBAC: “user *is owner of* board,” “user *is member of* org that *owns* board.” Graphs, not flags  
- Mapping the Project Board: org membership (ReBAC), “admin” as an attribute, “business hours + device posture” as ABAC if needed  
- Enforcement points: API gateway vs. controller vs. domain service vs. query filter (IDOR lives in the query)  
- Symfony Voters / security expressions vs. a dedicated policy module  
- Vue: UI hiding is *not* authorization; it is UX. Every mutation is re-checked server-side  
- Testing authorization: matrix tests (actor × resource × action), including horizontal privilege escalation  

## Chapter 7. High-Level Web Security

Walk the OWASP Top 10:2025 as an engineering checklist, not a poster. Current list: Broken Access Control; Security Misconfiguration; Software Supply Chain Failures; Cryptographic Failures; Injection; Insecure Design; Authentication Failures; Software or Data Integrity Failures; Security Logging and Alerting Failures; Mishandling of Exceptional Conditions.[[4]](https://owasp.org/Top10/2025/en/)

**Deep dives the stack actually hits**

- Same-Origin Policy and what the browser actually isolates  
- CORS: simple vs. preflight, credentialed requests, never `*` + cookies, explicit allowlists  
- CSRF: cookie semantics (`SameSite`, `Secure`, `HttpOnly`), double-submit / synchronizer tokens, SPA + bearer vs. cookie session  
- SQL injection: parameterized queries are the floor; ORM is not magic if you concatenate DQL/SQL  
- XSS: context-aware encoding; Vue’s default escaping and `v-html` as a reviewed exception  
- SSRF folded into Broken Access Control — treat outbound URL fetch as an authz problem  
- Supply chain: Composer / npm lockfiles, `composer audit` / `npm audit`, signed CI artifacts, pin images  
- Exceptional conditions: fail closed, don’t leak stack traces, don’t confuse 401/403/404 for authz  

**Resources the chapter teaches people to *use***

- OWASP Cheat Sheets as the first lookup, not blog posts  
- Threat modeling a single endpoint before writing it  

---

# Part III — APIs, Data, and Change

## Chapter 8. API Design That Ages

- Stateless RESTful design: resources, verbs, idempotency, what “REST” still means in practice  
- Resource modeling for the Project Board; when to nest, when to flatten  
- Error responses as `application/problem+json` per RFC 9457 (`type`, `title`, `status`, `detail`, `instance`, extensions for field errors) — stop inventing `{ "error": "nope" }` shapes. RFC 9457 superseded RFC 7807.[[5]](https://www.rfc-editor.org/info/rfc9457/)  
- Pagination:  
  - Offset: simple, random access, degrades as `OFFSET` grows, races on inserts/deletes  
  - Cursor: stable, seekable, no deep-page tax, no jump-to-page  
  - When each is allowed; default to cursor for any list that can grow  
- Filtering, sorting, sparse fieldsets — keep them boring and documented  
- Versioning policy (URL vs. header) and compatibility rules  
- Idempotency keys for POSTs that money or side effects can hit twice  
- OpenAPI as the contract; Bruno collections generated from it, not the other way around  
- Pagination and problem+json examples in both Symfony and a Node/Hono service  

## Chapter 9. MySQL as It Exists Now

Leave 5.5 folklore behind. Target current MySQL (9.x docs, native JSON type, modern optimizer).[[6]](https://dev.mysql.com/doc/en/json.html)

- Types you should actually choose: integers, decimals, datetimes (`DATETIME(6)` vs. `TIMESTAMP`), character sets (`utf8mb4`), generated columns  
- Indexes: primary, secondary, covering, composite order, prefix indexes, and when an index is a lie  
- Transactions, isolation levels, deadlocks, `SELECT … FOR UPDATE`  
- Migrations as reviewed artifacts (Doctrine / Prisma generate; humans still read the SQL)  
- **JSON as a data type:** storage, validation, `JSON_TABLE`, functional indexes, partial updates  
  - Legitimate uses: genuinely schemaless payloads, vendor blobs, feature-specific documents  
  - Abuse we will not ship: using JSON to avoid modeling; querying JSON as if it were a relational schema; stuffing relations into documents “just for now”  
- EXPLAIN, slow query log, and how an ORM hides a table scan  
- Backup, restore, and what “we have replicas” does and does not mean  

## Chapter 10. Feature Flags and Rolling Releases

Follow Fowler’s split: release vs. experiment vs. ops vs. permission flags.

- Flag types and lifespan (flags are inventory; unused flags are debt)  
- Evaluation points: Symfony voter / service vs. Vue computed vs. edge  
- Consistency: same actor, same flag, same answer across PHP, Node, and the SPA  
- Rolling release: canary, percentage, cohort, kill switch  
- Flags in tests: deterministic evaluation, no “flag-shaped ifs” leaking through the domain  
- Cleanup policy and who owns a flag after launch  
- Interaction with CI/CD (Chapter 22) and observability (Chapter 21)  

---

# Part IV — Platform

## Chapter 11. Docker and Docker Compose

- Images vs. containers vs. volumes vs. networks  
- Multi-stage builds for PHP-FPM and Node; non-root users; distroless where it pays  
- Compose for the whole Project Board: `php`, `nginx`, `mysql`, `redis`, `node-worker`, `mailhog`  
- Bind mounts vs. named volumes; file permissions between host and PHP  
- Healthchecks, depends_on with conditions, and why “it started” ≠ “it is ready”  
- Secrets: not in the image, not in git; Compose secrets / env files that never ship  
- What belongs in the image vs. runtime config  

## Chapter 12. NGINX as the Front Door

- Reverse proxy vs. static server vs. TLS terminator  
- PHP-FPM `fastcgi_pass`, `try_files`, and the front-controller pattern  
- SPA fallback (`try_files $uri /index.html`) and how it interacts with API routes  
- Gzip/brotli, caching headers, `X-Forwarded-*`, WebSocket upgrade for Vite HMR or queues dashboards  
- Rate limiting, request size, timeouts  
- Security headers: CSP, HSTS, `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy`  
- A reviewed `nginx.conf` for local and a production variant  

## Chapter 13. Redis as a KV Workhorse

- Data structures you will actually use: strings, hashes, lists, sets, sorted sets, streams  
- Cache aside vs. write-through; TTL and stampede  
- Sessions, rate-limit counters, feature-flag snapshots, distributed locks  
- Serialization and the “I cached a PHP object and now deploys explode” problem  
- Persistence: AOF vs. RDB, and when Redis is allowed to lose data  
- Key namespaces and eviction policy  
- What Redis is *not* (source of truth for money, long-term documents)

## Chapter 14. Message Queues

Two implementations, one vocabulary: producer, consumer, retry, dead letter, idempotency, at-least-once.

- **BullMQ (Node):** Redis-backed, jobs, queues, workers, repeatable jobs, priority, rate limits  
- **RabbitMQ (PHP):** exchanges, bindings, routing keys, ack/nack, prefetch, DLX  
- Choosing the broker by the owning runtime — do not invent a third  
- Poison messages, exponential backoff, and making handlers idempotent  
- Outbox pattern when a DB write and a publish must not diverge  
- Observability: queue depth, processing lag, retry storms  
- Project Board jobs: thumbnail generation, webhook delivery, search reindex, email  

## Chapter 15. ORMs Without Self-Harm

- **Doctrine (PHP):** entities, mappings (attributes), Unit of Work, identity map, lazy vs. extra-lazy vs. eager, DQL, QueryBuilder, second-level cache (usually no)  
- **Prisma (Node):** schema, client, relations, transactions, `prisma migrate` vs. `db push`  
- Type mappings as the source of schema truth; generated migrations still get a human review  
- N+1, partial hydration, and when to drop to SQL  
- Domain model vs. persistence model — when a 1:1 entity is fine, when it is not  
- Migrations in CI: expand/contract, lock safety, never edit a shipped migration  

---

# Part V — The PHP Stack

## Chapter 16. PHP 8.5 as a Language

Write *current* PHP, not 7.4 with types bolted on.

- Types: backed enums, intersection/union, `readonly`, property hooks / asymmetric visibility (as used with Symfony 8), never/`void`  
- PHP 8.5 highlights to actually adopt:  
  - Pipe operator `|>` for linear transforms  
  - `clone with` for immutable-style updates  
  - Native URI extension (RFC 3986 / WHATWG) — stop rolling URL parsers  
  - `array_first()` / `array_last()`, `#[\NoDiscard]`, always-on OPcache, fatal backtraces [[1]](https://www.zend.com/resources/php-versions)  
- Errors vs. exceptions; don’t swallow; don’t use exceptions for control flow  
- Autoloading, Composer classmap vs. PSR-4  
- Extensions we rely on: `pdo_mysql`, `redis`, `intl`, Xdebug (dev only)

## Chapter 17. Composer and Dependency Discipline

- `composer.json` vs. lockfile; `--no-dev` in production images  
- Semver, stability flags, why `^` is a contract  
- Private packages and path repositories  
- Scripts, plugins, and the trust boundary they create (A03)  
- `composer audit`, abandoned packages, and when to fork vs. replace  
- Autoload optimization in the Docker build  

## Chapter 18. Symfony, Practically

Use SymfonyCasts as the narrative overview; this chapter is the shop’s opinionated map.

- Application vs. bundle vs. component  
- Directory layout and what belongs in `src/`  
- HTTP Kernel, controllers as thin adapters, DTOs as input  
- Routing, validators, serializer (when to use it, when it becomes magic)  
- Dependency Injection: autowiring, attributes, env processors, when to write a compiler pass (almost never)  
- Security component: authenticators, firewalls, voters — wired to Chapters 5–6  
- Messenger: mapping queue ideas from Chapter 14 onto Symfony Messenger + RabbitMQ  
- Doctrine integration, migrations, fixtures  
- Console commands as first-class entry points (crons, one-shots, workers)  
- Config: `config/`, secrets, when `.env` is a lie  
- Events / EventDispatcher vs. Messenger vs. domain events  
- Forms: when the SPA owns the form and Symfony only sees JSON  
- Profiler in dev; why it never exists in prod images  

## Chapter 19. PHP Standards and Enforcement

- PSR map that matters: PSR-1, PSR-4, PSR-7/15/17 (HTTP), PSR-11 (container), PSR-12 as history  
- PER Coding Style as the living successor to PSR-12 — this is what php-cs-fixer targets now [[7]](https://www.php-fig.org/per/coding-style/)  
- Other PSRs you will touch: caching, logging, clock, clock? (PSR-20), container  
- **php-cs-fixer:** committed config, CI job, no style debate in review  
- **PHPStan:** strict rules, level 10 where the codebase allows, baseline as debt with an expiry, not a blanket  
- **Rector:** automated upgrades (PHP 8.4 → 8.5, Symfony rulesets); review the diff, don’t blind-apply  
- What is enforced locally (git hook optional) vs. what is a merge gate  

## Chapter 20. PHPUnit as a Design Tool

- Anatomy: test case, assertions, data providers, attributes  
- Test doubles: dummy, stub, fake, mock, spy — PHPUnit’s mock builder and when a hand-written fake is clearer  
- Isolated unit tests of domain services (no kernel)  
- Kernel / WebTestCase integration tests for HTTP + security + persistence  
- Database tests: transactions, fixtures, “don’t hit the real queue”  
- Testing voters, Messenger handlers, and console commands  
- Coverage as a spotlight, not a target number  
- What *not* to mock (the thing you are testing; the ORM in a way that tests the mock)

---

# Part VI — The TypeScript Stack

## Chapter 21. TypeScript and the Shared `tsconfig`

- Why TypeScript is the language, not a linter for JavaScript  
- Strictness we require: `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes` as applicable  
- The shared `tsconfig`: `base`, `app`, `node`, `vitest` project references  
- Types at the API boundary: generated from OpenAPI vs. hand-written zod/valibot parsers — *runtime* validation on the edge  
- `unknown` vs `any`; branded types for IDs  
- Module resolution, `paths`, and what breaks Vite vs. Node  
- When to write types, when to let inference work  

## Chapter 22. Node.js, npm, and Server Microframeworks

- Node as a runtime: event loop, worker threads (rarely), ESM  
- npm: lockfile, workspaces if we have them, `engines`, scripts, `npm ci` in CI  
- Choosing a microframework:  
  - **Express** — ubiquitous, middleware soup, know it to read it  
  - **Fastify** — schema-first, performance, plugins  
  - **Hono** — small, typed, great for workers and edge-shaped services  
- Where Node is allowed: BullMQ workers, BFF, webhooks, internal admin APIs — not a second monolith  
- Structured logging, graceful shutdown, health endpoints  

## Chapter 23. Vue: Components, State, and Routing

- Vue 3 mental model: SFCs, Composition API, `<script setup lang="ts">`  
- Reactivity: `ref` / `reactive` / `computed` / `watch`; what is not reactive  
- Component design: props, emits, slots, provide/inject; keep components boring  
- **Pinia:** stores as the application model, not a dumping ground; setup stores vs. options; store composition; persistence plugins when justified  
- **Vue Router:** `createWebHistory`, typed routes/params, navigation guards, scroll, lazy routes, view transitions  
- Data fetching: stores vs. composables vs. TanStack Query-style caches (pick one convention)  
- Forms and client-side validation that matches server problem+json  
- **Vue DevTools:** inspecting component tree, Pinia timelines, router, performance  
- When a component is too smart; when a store is a hidden service locator  

## Chapter 24. Vite as the Bundler and Dev Server

- Dev vs. build: native ESM, HMR, dependency pre-bundling  
- `vite.config` for the SPA: aliases matching `tsconfig`, env (`import.meta.env`), proxy to Symfony  
- Code splitting, dynamic import, and what actually ships  
- Asset handling, CSS modules / PostCSS, SVG strategy  
- Library mode if we publish a shared package  
- Source maps and how they connect to browser debugging (Chapter 27)

## Chapter 25. Frontend Standards Enforcement

- **ESLint:** flat config, Vue + TypeScript + a11y plugins, no conflicting Prematter rules  
- **Stylelint:** modern CSS, no old preprocessor habits unless we still have them  
- **Prettier:** formatting only; never an architectural debate  
- Shared configs in the monorepo / starter; CI as the source of truth  
- Ignore files and generated code  

## Chapter 26. Vitest, Bruno, and Playwright

- **Vitest:** unit and component tests; Vue Test Utils; the mocking guide used deliberately — `vi.fn`, module mocks, timers, `vi.spyOn`; prefer fakes at boundaries  
- What to unit-test in Vue (pure composables, stores, formatters) vs. what to leave to Playwright  
- **Bruno:** API collections next to the repo; environments; asserting problem+json and pagination; no Postman cloud as source of truth  
- **Playwright:** E2E against Compose; auth fixtures; network stubbing vs. full stack; a11y checks; visual only when it pays  
- Connecting this chapter to the test pyramid in Chapter 27  

---

# Part VII — Quality Across the Stack

## Chapter 27. The Test Pyramid in This Shop

Martin Fowler’s pyramid, made concrete.[[8]](https://owasp.org/www-project-top-ten/) *(pyramid concept; cite Fowler in-chapter)*

- Unit: domain rules, voters, Pinia stores, formatters — milliseconds, no I/O  
- Integration: HTTP + DB + authz; Symfony WebTestCase; Node + testcontainers/Redis  
- E2E: a few Playwright journeys that prove the contract (login, create board, flag-gated path)  
- Mocking policy:  
  - PHPUnit doubles for outbound (mail, billing, IdP)  
  - Vitest mocks for browser APIs and HTTP at the unit layer  
  - Never mock the code under test into passing  
- Test data builders, time (clock ports), randomness  
- CI time budget and how a slow suite becomes a culture problem  
- Flake protocol: quarantine with an owner, don’t retry-and-forget  

## Chapter 28. Step-Through Debugging

Reading logs is not debugging. This chapter is about stopping the process and looking.

- **Browser DevTools:** breakpoints, conditional / logpoints, call stack, scope, blackboxing, network, Performance, Memory; debugging Vue SFCs via source maps; Vue DevTools beside Chrome DevTools  
- **Node.js:** `node --inspect`, Chrome inspect, VS Code launch configs, debugging Vitest and a Fastify/Hono worker  
- **Xdebug:** step debug from PhpStorm / VS Code; `xdebug.mode=debug`; path mappings through Docker; debugging PHPUnit and Messenger consumers; why Xdebug is never in prod images  
- Debugging a request that crosses NGINX → Symfony → Redis → BullMQ → Vue  
- Reproduction first: failing test > debugger > guess  

## Chapter 29. Observability with ELK

- What belongs in a log line: request id, actor id, tenant, flag evaluation, duration — not payloads with PII  
- Correlation IDs from NGINX through PHP/Node to the SPA  
- Elasticsearch / Logstash / Kibana: indexes, structured JSON logs, useful dashboards  
- From “we have logs” to “we can answer ‘what happened to order X?’ in two minutes”  
- Alerting vs. dashboards; ties to OWASP A09 (logging and alerting failures)  
- Retention, redaction, and what never goes to ELK (tokens, passwords, card data)

---

# Part VIII — Delivery

## Chapter 30. GitLab CI/CD

- Pipeline as the only path to production  
- Stages: lint → typecheck / PHPStan → unit → integration → build images → E2E → deploy  
- Runners, caching (`composer`, `npm`, Docker layers), and why the cache is a hint  
- Jobs that must be merge gates vs. scheduled  
- Environments, review apps, and feature-flag-aware deploys  
- Image signing / SBOM if required; secret management (`CI` variables vs. a vault)  
- Branch pipelines vs. tag/release pipelines  
- Rollback: images are immutable; config/flags are the fast lever  
- Connecting to Chapter 10 (rolling releases) and Chapter 11 (the same Dockerfile locally and in CI)

## Chapter 31. Production Topology and Operations

- A reference diagram: CDN → NGINX → PHP-FPM / Node → MySQL primary/replica → Redis → RabbitMQ  
- 12-factor leftovers that still matter: config, disposability, logs as streams  
- Zero-downtime deploys and migration expand/contract  
- Backups, restore drills, and on-call notes that belong next to the code  
- Capacity: what we measure (p95, queue lag, slow queries) before we scale  

---

# Part IX — Putting a Feature Through the Machine

## Chapter 32. A Vertical Slice, Start to Finish

Walk one Project Board feature (e.g. “card assignments with email + webhook + flag”) through every layer:

1. Threat model and authz matrix (ReBAC: assignee, board members)  
2. OpenAPI + RFC 9457 errors + cursor pagination  
3. Doctrine migration + Prisma counterpart if a Node worker reads the same data  
4. Symfony endpoint, voter, Messenger message  
5. RabbitMQ / BullMQ workers  
6. Vue route, Pinia store, a11y, feature flag  
7. PHPUnit + Vitest + Bruno + Playwright  
8. Debug it once on purpose (Xdebug + DevTools)  
9. Pipeline, flag at 10%, watch ELK, roll to 100%, delete the flag  

This chapter is the exam. If a reader can do this unattended, they are productive here.

## Chapter 33. Review Checklists and ADRs

- PR checklist: security, a11y, tests at the right layer, migrations, flags, logs  
- When to write an ADR (new store, new grant type, new queue, JSON column)  
- Living handbook: how this book’s chapters map to repo `docs/` and CI templates  

---

# Back Matter

- Glossary (grant, voter, cursor, problem+json, ReBAC, …)  
- Annotated bibliography: RFCs, OWASP, Fowler, SymfonyCasts, official docs — primary sources first  
- Cheat sheets: Compose commands, Xdebug recipes, PHPStan baseline etiquette, Playwright auth fixture  
- Version log (so the handbook can track PHP 8.6 / Symfony 8.2 without rewriting narrative)  
- Index  

---

## Suggested Length and Sequencing

| Part | Chapters | Role |
|---|---|---|
| I Foundations | 1–4 | Onboarding week 1 |
| II Security | 5–7 | Before you touch auth or a public endpoint |
| III APIs & data | 8–10 | Before you add a table or a list endpoint |
| IV Platform | 11–15 | Before you run anything non-trivial locally |
| V PHP | 16–20 | Deep work on the API |
| VI TypeScript | 21–26 | Deep work on the SPA and workers |
| VII Quality | 27–29 | How we know it works |
| VIII Delivery | 30–31 | How it reaches users |
| IX Synthesis | 32–33 | Competence check |

Approximate book length if written fully: **450–550 pages**, or a handbook of ~35 long articles with the same headings. Chapter 32 can ship first as an internal “path to productivity” even before the rest is prose.

---

## Handbook Conversion Notes

When this becomes the permanent developer handbook:

- Split each chapter into a short *normative* page (rules, configs, checklists) and a longer *explanatory* page (why, history, worked example).  
- Normative pages are merge-gated: php-cs-fixer rules, `tsconfig`, NGINX snippets, OWASP review list.  
- Keep version floors in one place (Front Matter / Version log) so “current MySQL” and “PHP 8.5” do not rot independently.  
- The existing frozen guidelines become the first draft of the normative pages; this outline is the explanatory spine.

---

## What This Outline Deliberately Does Not Do

- It does not pick a third backend language or a second SPA framework.  
- It does not treat Redis or MySQL JSON as a substitute for a model.  
- It does not teach OAuth by pasting a “login with Google” tutorial.  
- It does not confuse UI role checks with authorization.  
- It does not freeze MySQL 5.5 knowledge as current practice.

That is the book: one contract (HTTP + identity + data), two implementation stacks, and the platform that makes both shippable.
