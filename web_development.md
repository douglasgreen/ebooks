# A practical handbook for PHP 8.5 / Symfony and TypeScript / Vue engineers

**Identity, APIs, security, testing, and delivery — from local Docker to production GitLab CI.**

---

# Part I — Foundations

# Chapter 1. The Dual-Stack Mental Model

Most "full-stack" advice you'll find online assumes you'll pick one runtime and live inside it forever. Real systems don't work that way. This chapter lays out the mental model this book is built on: a **dual-stack architecture** where PHP and Node each do what they're best at, and neither pretends to be the other.

## Why This Shop Is Not "PHP *or* Node"

The question "should we use PHP or Node?" is usually the wrong question. It assumes you must choose a single runtime for everything. In a mature web product, different layers of the system have different jobs, and each job has a natural fit.

Here's how the responsibilities split in this book's architecture:

- **Symfony (PHP) owns the domain and persistence.** The business rules — what an order is, when an invoice can be refunded, how a subscription renews — live in a Symfony application backed by Doctrine and MySQL. This is the system of record. It's deliberate, versioned, tested, and boring in the best possible way.

- **Vue owns interaction.** Everything the user sees and touches — forms, dashboards, optimistic updates, drag-and-drop — is a Vue application. It doesn't know about MySQL. It doesn't know about Doctrine entities. It knows about the API, and that's it.

- **Node microframeworks own the glue.** Webhooks from Stripe, background workers processing a queue, a Backend-for-Frontend (BFF) that shapes responses for a mobile app, a quick internal admin tool — these are small, focused Express or Fastify services. They're cheap to write, cheap to replace, and cheap to throw away.

The key insight: **Node here is not a replacement for PHP. It's a complement to it.** You're not rewriting your Symfony app in Express. You're letting Symfony be the stable core while Node handles the messy edges of the system — the parts that change often, talk to third parties, or need to be built in an afternoon.

> **A note on the BFF:** A Backend-for-Frontend is a thin server layer built specifically for one frontend. Instead of the Vue app calling five different services and stitching the data together in the browser, the BFF makes those calls and returns one clean response. It keeps frontend code simple and moves integration complexity to the server, where it belongs.

## Request Lifetime

Before you can reason about any of this, you need a picture of what actually happens when a user clicks a button. Trace a typical request end to end:

```
Browser (Vue)
   │  HTTPS request
   ▼
NGINX (reverse proxy)
   │  routes by path / serves static assets
   ▼
PHP-FPM (Symfony)  or  Node (Express/Fastify)
   │  application logic
   ▼
MySQL / Redis / Queue
```

Let's walk through each hop:

1. **The browser** sends an HTTPS request. If it's asking for a page shell, CSS, or a compiled JS bundle, NGINX serves it straight from disk — no application code involved.

2. **NGINX** acts as traffic control. It terminates TLS, serves static files, and forwards dynamic requests to the right backend based on the URL. Requests to `/api/*` might go to PHP-FPM; requests to `/hooks/*` go to a Node service.

3. **PHP-FPM** receives the request and hands it to a Symfony worker. Symfony boots (with help from an opcache), runs routing, middleware, and your controllers, and returns a response. PHP-FPM's model is simple: one request, one worker, one clean lifecycle. No shared state between requests, which eliminates an entire class of bugs.

4. **Node services** run the same pattern for their slice of the work. A webhook endpoint receives a Stripe event, validates it, and pushes a job onto a queue. A worker process picks jobs off that queue and does the slow work — sending emails, generating PDFs, syncing with external APIs.

5. **MySQL** is the source of truth. **Redis** handles caching, sessions, and rate limiting — data that's fast to lose and fast to rebuild. The **queue** (often Redis-backed, sometimes something like RabbitMQ) decouples "accept the request" from "do the work," so a slow email send never blocks an HTTP response.

Memorize this path. When something breaks, your first debugging question is always: *which hop failed?* A 502 from NGINX means the upstream never answered. A 500 from Symfony means the domain layer threw. A job that never runs means the queue or worker failed. Knowing the lifetime turns "the site is broken" into a specific, answerable question.

## Bounded Contexts and the API as the Contract

The term **bounded context** comes from Domain-Driven Design (DDD). In plain terms: a bounded context is a boundary inside which a set of concepts means one specific thing.

"Order" means something different to the checkout team than it does to the shipping team. Inside the checkout context, an order is a cart that's been confirmed. Inside the fulfillment context, it's a box that needs a label. Both are valid. The trouble starts when one team's code reaches directly into the other team's database and assumes their definition.

In this architecture, the boundaries are physical, not just conceptual:

- The Symfony application owns its MySQL schema. Nobody else writes to those tables.
- Node services never run raw queries against the domain database. If they need data, they call the API or consume an event.
- The **HTTP API (and message formats) are the contract** between contexts. As long as the contract holds, each side can change its internals freely.

This is why the API deserves the same care as your database schema. Changing a field name in a JSON response can break three consumers. Treat API changes like schema migrations: version them, deprecate deliberately, and never make a breaking change silently.

## What "Full-Stack" Means Here

In this book, "full-stack developer" does **not** mean "equally expert in PHP, Node, Vue, MySQL, Redis, NGINX, and Docker." That person doesn't exist, and pretending otherwise produces shallow work everywhere.

What it means here is narrower and more useful:

- **You own features end to end.** When you build "export orders to CSV," you write the Vue button, the API endpoint, the query behind it, and the queue job that generates the file for large exports. You don't throw it over a wall.
- **You have working fluency in each layer**, with one or two layers where you're genuinely deep. Maybe you're a Symfony expert who can ship a competent Vue component and a small Node worker. That's full-stack.
- **You understand the seams.** You know what happens at the API boundary, how a request travels through NGINX, and why the queue exists. Most production incidents live in the seams between layers, not inside them.

The goal is **ownership plus literacy**: deep in your home layer, conversant everywhere else, and honest about the boundary between the two.

## When *Not* to Introduce a New Runtime or Store

Dual-stack is a discipline, not a buffet. Every additional runtime and data store adds real costs: deployment complexity, security patches, on-call knowledge, and one more thing that can fail at 3 a.m. Here are the questions to ask before adding anything new:

1. **Can the existing stack do this acceptably?** A Symfony console command run by cron handles most "background job" needs fine. You don't need a Node worker for a nightly report.

2. **Does this need to be a service, or is it a module?** If the new code shares your database, your auth, and your deployment, it's a module inside Symfony — not a microservice.

3. **What's the operational cost in a year?** A new runtime means a new Docker image, a new set of CVEs to patch, a new thing a teammate must learn to debug. Multiply that by every engineer who'll touch it.

4. **Is the problem actually about the runtime?** Teams often reach for Node because PHP feels slow — when the real problem is an N+1 query or a missing index. Fix the query first.

Concrete rules of thumb used throughout this book:

- **New runtime** only when there's a genuine ecosystem or concurrency need — streaming, real-time websockets at scale, or a library that exists only in Node.
- **New data store** (Redis, Elasticsearch, etc.) only when MySQL demonstrably cannot do the job, ideally with a benchmark or incident report to prove it.
- **New service** only when the team owning it is genuinely different, or the deployment lifecycle is genuinely different.

The best architecture is the one a small team can actually operate. Every piece you don't add is a piece that can't page you at night. Add things when the problem demands it — and no sooner.

---

# Chapter 2. Git as Daily Craft

Git is not just a tool you use at the end of your work — it's the record of every decision your team made. Treated casually, it becomes a junk drawer of `fix`, `wip`, and `asdf` commits that nobody can read. Treated as a craft, it becomes the most valuable documentation your project has. This chapter builds the habits that make the difference.

## Commits Are Reviewable Decisions, Not Save Points

The most common Git mistake is treating a commit like Ctrl+S: "I've been working for an hour, better commit." That produces commits with no shape, no purpose, and no reviewable content.

A better mental model: **a commit is a single decision, complete and explainable.** "Add validation to the refund endpoint." "Extract the invoice number generator into a service." "Upgrade Doctrine to fix the hydration bug." Each one does one thing, and each one leaves the codebase working.

This changes how you work day to day:

- **Stage deliberately.** Use `git add -p` to stage hunks piece by piece, so unrelated changes don't ride along in the same commit.
- **Commit when a thought is finished**, not when a timer goes off. If you're mid-thought and need to switch tasks, use a WIP branch or `git stash` — don't pollute shared history.
- **A commit should pass tests.** If commit #3 of 7 breaks the build, anyone who checks out that point is stuck. Squash or reorder before the branch merges.

Why does this matter? Because in six months, someone will run `git blame` on a confusing line of code and land on your commit. The commit message is the only thing that tells them *why*. A commit is a letter to that future reader. Write accordingly.

## Branching: Short-Lived, Protected, No Personal Empires

Keep the branch model boring. Boring branching is what makes continuous delivery possible.

The rules for this book's workflow:

- **The default branch (`main`) is protected.** Nobody pushes to it directly. Every change arrives via pull request, and every pull request requires review and a passing CI run.
- **Feature branches are short-lived.** Aim for hours to a couple of days, never weeks. A branch that lives for a month accumulates merge conflicts, drifts from `main`, and produces a review so large nobody reads it carefully.
- **No long-lived personal branches.** A "my-dev" branch that never merges back is a fork of the project. It hides work from review and guarantees a painful reconciliation later. If you need a personal sandbox, use a worktree (covered below) or a fork you rebase aggressively.
- **Branch off `main`, merge back to `main`.** No `develop` staging branch unless your release process genuinely requires one. Every extra long-lived branch is another place for work to get lost.

Small branches also mean small pull requests — and small pull requests get *real* reviews. A 4,000-line diff gets a rubber-stamp "LGTM." A 200-line diff gets actual scrutiny.

## History Hygiene: Rebase, Merge, Squash — and When Not to Rewrite

The rebase-versus-merge argument is mostly settled once you separate two contexts: **private history** (your unpushed branch) and **public history** (what others have built on).

The rule:

- **Rebase your own, unshared work freely.** While your feature branch exists only on your machine, rewrite it however you like. Reorder commits, squash the three "fix typo" commits into their parent, reword messages. This is exactly what history hygiene means.
- **Merge (or rebase with care) once the work is shared.** Once others have pulled your branch, rewriting it destroys *their* history. Don't do it.
- **Never rewrite `main`.** Not after a bad merge, not to "clean up," not ever. If something must be undone on a protected branch, do it with a forward change: `git revert` creates a new commit that undoes the old one. That's the honest record — "we did this, then we undid it" — not a lie that it never happened.

When to **squash**:

- Before merging, squash a branch's noisy history into one clean commit per logical change. Most platforms have a "Squash and merge" button — use it.
- Squash when the intermediate commits were exploration ("try approach A," "revert A," "try approach B"). Nobody benefits from archaeology of your dead ends.

When *not* to squash:

- When the intermediate commits are genuinely separate decisions that deserve separate entries in `git log` and separate revert points.
- When the branch is large and the intermediate commits each pass tests — the granular history is a feature, letting reviewers and future bisects work at finer resolution.

For integrating `main` into your feature branch, prefer `git rebase main` (keeps history linear) — but only while the branch is yours alone. If several people share the branch, use merge and accept the noise.

## Commit Messages That Survive Review and `git blame`

A good commit message answers two questions: **what** changed, and **why**. The diff already shows what — the message's real job is the why.

Use this structure:

```text
Summarize the change in 50 characters or fewer, imperative mood

Explain the problem this commit solves and why this approach was
chosen over the alternatives. Mention ticket IDs, link to discussion,
and note any behavior change visible to callers. Wrap at 72
characters so the message reads well in terminals and git log.
```

Conventions that pay off:

- **Imperative mood:** "Add rate limiting to login" not "Added rate limiting." Git itself uses this convention — `git merge` generates "Merge branch..." — and it reads consistently in `git log`.
- **Reference the ticket:** `Refs #482` or the Jira key. Future readers can find the full discussion.
- **Say why, not what:** "Increase token expiry to 30 minutes because mobile clients on flaky networks were logging users out mid-session" beats "Update config value" every time.
- **Mark breaking changes explicitly.** If a response field is removed, say which API version is affected.

The test for a good message: run `git blame` on any line, read the commit, and ask whether you'd understand the decision without opening the ticket. If not, the message failed.

## The Tools You Actually Need

Four Git features do most of the heavy lifting once you're past the basics. Learn them properly and you'll spend less time in Stack Overflow panic threads.

### `git bisect`

When a bug appears and nobody knows which commit introduced it, bisect runs a **binary search** over history:

```bash
git bisect start
git bisect bad                  # current commit has the bug
git bisect good v2.3.0          # this tag was fine
# Git checks out the middle commit. Test it, then:
git bisect bad                  # or: git bisect good
# Repeat — ~10 steps finds the culprit in 1,000 commits
git bisect reset
```

You can even automate the test:

```bash
git bisect run ./run-tests.sh
```

Bisect only works well when commits are small and each one builds — one more reason for the hygiene habits above.

### `git reflog`

The reflog is a record of everywhere your `HEAD` has been, kept locally for ~90 days. It's your undo button for "disasters" that aren't actually disasters:

```bash
git reflog
# a1b2c3d HEAD@{0}: reset: moving to HEAD~2
# e4f5g6h HEAD@{1}: commit: Add refund validation
git reset --hard e4f5g6h        # your "lost" commit is back
```

If you committed something and then reset it away, it's in the reflog. Internalize this and you'll stop fearing Git — almost nothing is truly lost.

### `git worktree`

Worktrees let you check out multiple branches into separate directories **from one repository**, without stashing or committing half-finished work:

```bash
git worktree add ../hotfix-dir hotfix/payment-bug
cd ../hotfix-dir
# fix the bug while your feature branch sits untouched in the main directory
```

This is the clean solution to the classic "I'm mid-refactor but production needs a fix now" problem. Each worktree gets its own checked-out files; they all share the same object database, so no duplicate disk bloat.

### Sparse Checkout

On large monorepos, sparse checkout limits your working directory to the directories you actually need:

```bash
git sparse-checkout init --cone
git sparse-checkout set apps/api packages/shared
```

You still have full history, but only relevant files on disk. Useful when a repo grows past the point where a full checkout is painful — most projects never need it, but knowing it exists prevents the "the repo is too big to work in" panic.

## `.gitignore`, LFS, and What Never Belongs in a Repo

### `.gitignore` as policy, not afterthought

A good `.gitignore` is written on day one, per stack. For this book's stack:

```gitignore
# Dependencies
/vendor/
/node_modules/

# Build output
/var/cache/
/var/log/
/public/build/
dist/

# Environment and secrets
.env
.env.*.local

# Editor and OS noise
.idea/
.vscode/
.DS_Store

# Local data
*.sql
*.dump
```

Two details matter:

- **`.env` files are ignored; `.env.example` is committed.** The example file documents which variables the app needs, with no real values.
- **Once a file is tracked, `.gitignore` won't untrack it.** If a secret was committed before being ignored, removing it from the working tree does nothing — it's still in history (see below).

### Git LFS

Git stores every version of every file, forever. That's perfect for source code and terrible for binaries — a few hundred design files or videos and your clone takes an hour. **Git Large File Storage (LFS)** replaces large binaries in history with pointer files, storing the actual content elsewhere:

```bash
git lfs install
git lfs track "*.psd" "*.mp4" "*.zip"
git add .gitattributes
```

Policy for this book: LFS is for *assets the app ships or tests against* — fonts, fixtures, media. It is **not** a place to hide things that don't belong in the repo at all.

### What never belongs in the repository

- **Secrets.** API keys, database passwords, signing keys, `.env` files with real credentials. Not even "temporarily." Secrets live in your environment/secret manager, and `.env.example` documents the shape.
- **`vendor/` and `node_modules/`.** Dependencies are defined by `composer.json` / `package.json` and lock files, and rebuilt on install. Committing them bloats every clone and hides dependency changes from review.
- **Dumps and logs.** Database dumps, `var/log`, uploaded user files. They're data, not code — and dumps often contain personal data, turning a convenience into a GDPR problem.
- **Build artifacts** that CI can regenerate (`dist/`, compiled assets).

**If a secret does get committed:** rotate the credential *first*. Deleting the file in a new commit does not remove it from history — anyone with repo access (or a leaked clone) can read it. After rotating, either rewrite history with `git filter-repo` (a coordinated, disruptive operation) or, if the repo will be made public, treat the secret as burned regardless.

## Code Review: A Security and Design Control, Not a Style Nitpick

If your reviews consist of "missing space before brace" and "LGTM," you're paying the cost of review without collecting the benefit. Review is the last checkpoint where a second pair of eyes sees the code before users do.

What a review should actually examine, in priority order:

1. **Security.** Is user input validated? Are queries parameterized? Is this endpoint authorized — not just authenticated? Is anything logged that shouldn't be? (This ties directly to Chapter 12's coverage of injection, OWASP's A03.)
2. **Correctness at the boundaries.** What happens with an empty list, a `null`, a concurrent request, a queue job that runs twice?
3. **Design and fit.** Does this change belong in Symfony or in the Vue layer? Does it respect the API contract from Chapter 1? Is it consistent with how neighboring code solves the same problem?
4. **Tests.** Do they test behavior, not implementation details? Would they catch a regression?
5. **Operational concerns.** New config? New dependency? Anything that needs a migration or a deploy-order note?

And what reviews should *not* be:

- **Style debates.** Settle formatting with automated tools — PHP-CS-Fixer, ESLint, Prettier — enforced in CI. If the linter can argue about it, humans shouldn't.
- **Gatekeeping.** Review exists to make the code and the reviewer's understanding better, not to demonstrate seniority. "Nit:" prefixes and blocking-vs-suggestion labels keep the signal clear.

Reviewer etiquette that keeps velocity high: review small PRs fast, ask questions instead of issuing verdicts when you're unsure, and distinguish "must fix" from "consider." Author etiquette: keep PRs small, describe the *why* in the description, and respond to every comment — even if the response is "won't fix, because."

## Signing, Protected Branches, and Supply-Chain Basics

Your code doesn't just need to be good — it needs to be *provably from where you think it came*. That's the supply chain: everything between "a developer wrote a line" and "it runs in production."

### Commit signing

A commit's author name is a string anyone can set. **Signed commits** attach a cryptographic signature (GPG or SSH key) that proves the commit came from a key you control:

```bash
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true
```

Platforms like GitHub then display a "Verified" badge. It's not perfect — it proves key possession, not intent — but it makes casual impersonation obvious and gives you a verifiable record of who actually pushed what.

### Protected branches

Branch protection is the enforcement layer for everything in this chapter:

- Direct pushes to `main` are blocked.
- Every merge requires at least one (ideally two) approving reviews.
- CI must pass — tests, linters, security scans — before merge is possible.
- History is protected: no force-pushes, no deletions.

Without protection, all the discipline above is voluntary. With it, the workflow is the path of least resistance.

### Supply-chain basics

Most modern code is mostly other people's code. Every dependency is a trust decision:

- **Pin and lock.** Commit `composer.lock` and `package-lock.json`. Builds must be reproducible — the same commit should always produce the same dependencies.
- **Review dependency updates.** Dependabot or Renovate can automate the PRs, but a human should glance at major version bumps. New dependencies deserve a quick look at maintenance activity, download numbers, and whether the package does what it claims — typosquatted packages ("reqeust" instead of "request") are a real and recurring attack.
- **Run automated scanning in CI.** `composer audit`, `npm audit`, and tools like Dependabot alerts catch known vulnerabilities in your dependency tree.
- **Use CI-generated artifacts, not laptops.** Production builds should come from your pipeline, where the process is scripted and logged — not from whichever developer's machine happened to deploy last.

This is the groundwork for OWASP **A03:2021 — Injection**, and for supply-chain risks more broadly: the theme is *verify inputs and verify provenance*. Untrusted input includes user form data, webhook payloads, third-party packages, and yes — the commits entering your protected branch.

---

# Chapter 3. The Web Platform You Are Actually Shipping

Frameworks come and go. jQuery, Angular, React, Vue — each generation of tooling sits on top of the same foundation: HTML, CSS, JavaScript, and the browser APIs beneath them. Developers who only learn the framework are always relearning their job every few years. Developers who know the platform can pick up any framework in a week.

This chapter covers the platform you're actually shipping to the user's browser — and where it's smarter to let TypeScript or your build tool do the work for you.

## HTML Is a Contract, Not a Template Leftover

In an SPA, it's tempting to treat HTML as scaffolding: one `<div id="app">` in `index.html`, everything else generated by Vue at runtime. That's a mistake with real costs — for accessibility, SEO, performance, and correctness.

Think of HTML as a **contract with the browser**. When you write `<button>`, you're promising the browser: this is clickable, focusable, activates on Enter and Space, announces itself to screen readers, and participates in form submission. Write `<div class="btn" @click="...">` instead, and you've broken that contract — now *you* have to reimplement all of it, and you'll miss something.

The rules that follow from treating HTML seriously:

- **Use semantic elements.** `<nav>`, `<main>`, `<article>`, `<section>`, `<header>`, `<footer>` give the page structure that assistive technology and search engines can navigate. A page of nested `<div>`s is structure-free.
- **Reach for native behavior before building it.** Native `<dialog>` gives you focus trapping, the Escape key, and backdrop styling for free. Native `<details>`/`<summary>` is a disclosure widget with zero JavaScript. Every line of JS you don't write to reimplement browser behavior is a line that can't have a bug.
- **Forms are contracts too.** `type="email"` triggers the right mobile keyboard and built-in validation. `required`, `min`, `pattern`, and `autocomplete` attributes work before any JavaScript loads. Vue's validation layer should *enhance* this, not replace it.
- **Label everything.** An `<input>` without an associated `<label>` (or `aria-label`) is invisible to screen readers. This is both an accessibility failure and, increasingly, a legal one.

One habit worth building immediately: view source on your production app and ask what a crawler or a screen reader sees on first paint. If the answer is "an empty div," you've made a tradeoff — make sure you made it deliberately (more on that under progressive enhancement).

## CSS: The Cascade Is a Feature

CSS gets blamed for problems caused by not understanding it. Four concepts explain most of modern CSS, and they're all worth knowing deeply because they determine what your Stylelint config will enforce later.

### The cascade and specificity

When two rules target the same element, the winner is decided by **specificity**, then by source order. Specificity is a three-part score: inline styles beat IDs, IDs beat classes, classes beat elements.

```css
p            { color: gray; }   /* specificity: 0,0,1 */
.text-muted  { color: #6c757d; } /* specificity: 0,1,0 — wins */
#sidebar p   { color: black; }   /* specificity: 1,0,1 — wins over both */
```

The practical rule: **keep specificity low and flat.** Prefer single-class selectors. Avoid IDs in stylesheets entirely — they're nearly impossible to override without another ID or `!important`. Once `!important` appears to fix a specificity war, it spreads, and the stylesheet becomes unmanageable. Stylelint will enforce exactly this later: no IDs, no `!important`, capped nesting depth.

### Custom properties (CSS variables)

Custom properties are values defined once and reused, with inheritance:

```css
:root {
  --color-danger: #dc3545;
  --space-unit: 8px;
}

.alert { border-left: 4px solid var(--color-danger); }
```

They're more than variables — they cascade, they're inherited, they work at runtime, and JavaScript can read and write them:

```js
el.style.setProperty('--color-danger', '#b02a37');
```

That makes them the natural bridge between your design tokens and dynamic state like theming. Dark mode becomes "redefine the custom properties on `[data-theme='dark']`" instead of duplicating every rule.

### Container queries

Media queries ask: *how big is the viewport?* Container queries ask: *how big is my container?* That's the question component authors actually care about — a card in a sidebar and the same card in a full-width layout need different rules, regardless of screen size.

```css
.card-wrapper { container-type: inline-size; }

@container (min-width: 400px) {
  .card { display: flex; gap: 16px; }
}
```

Container queries are what finally make components genuinely portable. A well-built component adapts to wherever it's dropped, which matters enormously when Vue components get reused across pages and products.

### Logical properties

`margin-left` assumes left-to-right text. The moment you ship Arabic or Hebrew, your layout mirrors — and every physical property points the wrong way. **Logical properties** describe direction relative to the flow of content instead:

```css
/* Instead of margin-left / padding-right / text-align: left */
margin-inline-start: 16px;
padding-inline-end: 24px;
text-align: start;
border-block-end: 1px solid var(--border);
```

The policy here is simple: use logical properties by default. It costs nothing today and makes internationalization free tomorrow. Stylelint will nudge you toward them.

## Modern ECMAScript: What to Learn, What to Delegate

JavaScript has stabilized dramatically. The language you write in 2025 is recognizably the language from five years ago, plus useful additions. Here's the split between what you should know cold and what you should hand off.

### Know cold: modules

ES modules (`import`/`export`) are the backbone of everything:

```js
// invoice-service.js
export function formatTotal(cents) {
  return new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' })
    .format(cents / 100);
}

// checkout.js
import { formatTotal } from './invoice-service.js';
```

Modules give you explicit dependencies, static analysis (which is what lets Vite tree-shake unused code), and top-level `await`. Understand the difference between default and named exports, and why circular imports are a design smell rather than a syntax problem.

### Know cold: iterators and async/await

**Iterators** power `for...of`, spread, destructuring, and `Array.from()`. Knowing that anything with a `[Symbol.iterator]` works with all of these — arrays, strings, Maps, Sets, NodeLists — means you stop converting things "just in case."

**`async`/`await`** is asynchronous code that reads synchronously:

```js
async function loadDashboard(userId) {
  const [orders, invoices] = await Promise.all([
    fetch(`/api/users/${userId}/orders`).then(r => r.json()),
    fetch(`/api/users/${userId}/invoices`).then(r => r.json()),
  ]);
  return { orders, invoices };
}
```

Two habits matter more than syntax:

- **Run independent awaits concurrently** with `Promise.all`. Sequentially awaiting independent requests multiplies latency for no reason.
- **Always handle rejection.** An unhandled promise rejection in an event handler fails silently. Wrap risky awaits in try/catch or attach `.catch()`.

### Useful additions worth knowing

- **`Intl` APIs**: dates, numbers, currency, pluralization, formatted correctly per locale — no library needed for most cases.
- **`Map` and `Set`**: proper keyed collections and deduplication (`new Set(items)`), with better semantics than object-as-map.
- **Optional chaining and nullish coalescing** (`user?.address?.city ?? 'Unknown'`): safe property access without `&&` pyramids.
- **Structured clone** (`structuredClone(obj)`): deep copy without the JSON round-trip hack.
- **Temporal** (arriving across browsers): a sane date/time API replacing the notoriously bad `Date`. Until it's universal, use a small library like Luxon or date-fns rather than raw `Date` arithmetic — timezone bugs from manual math are a classic production incident.

### What to leave to TypeScript

TypeScript is JavaScript plus a static type layer that's erased at build time. Don't think of it as a different language — think of it as the compiler catching mistakes before the browser does. Leave to it:

- **Types, interfaces, and generics.** Type your API responses once and let the compiler enforce the contract everywhere.
- **Enums, unions, and discriminated unions.** Modeling `"pending" | "paid" | "refunded"` as a union type makes impossible states unrepresentable.
- **Non-null assertions and type narrowing** — understand them so you know when the compiler is protecting you versus when you're overriding it.

The division of labor: **ECMAScript handles runtime behavior; TypeScript handles compile-time guarantees.** If you find yourself writing runtime checks purely to satisfy types, or fighting the type system to express something simple, step back — one of the two layers is being misused.

## Web APIs That Matter in an SPA

Vue gives you reactivity; the browser gives you capabilities. These are the Web APIs you'll reach for constantly in a single-page application.

### Fetch

The replacement for `XMLHttpRequest` and the foundation of every API call:

```js
const res = await fetch('/api/orders', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(payload),
});

if (!res.ok) {
  throw new Error(`Request failed: ${res.status}`);
}
const data = await res.json();
```

Three gotchas to internalize early:

- **Fetch does not reject on HTTP errors.** A 404 or 500 resolves normally — you must check `res.ok` yourself. Forgetting this is the single most common fetch bug.
- **Cookies need `credentials: 'include'`** for cross-origin requests; same-origin requests include them by default.
- **Abort requests** with an `AbortController` when the user navigates away or types a new search query — otherwise stale responses overwrite fresh ones.

### History API

An SPA changes views without page loads, which breaks the back button unless you maintain history yourself. The History API is how routers like Vue Router work under the hood:

```js
history.pushState({ orderId: 42 }, '', '/orders/42');
window.addEventListener('popstate', (event) => {
  // user hit Back — restore the view for event.state
});
```

You'll rarely call these directly — Vue Router wraps them — but understanding them explains why deep links, back-button behavior, and scroll restoration behave the way they do, and why router configuration is not optional polish.

### Storage

- **`localStorage`**: persistent key-value strings, ~5MB, synchronous. Good for theme preference, UI state. Bad for tokens in security-sensitive apps (readable by any script on the page — see Chapter 12).
- **`sessionStorage`**: same API, cleared when the tab closes. Good for wizard state.
- **IndexedDB**: asynchronous, structured, much larger. The right home for offline caches and larger datasets. Usually accessed through a wrapper library because its raw API is clunky.

Rule of thumb: `localStorage` for preferences, IndexedDB for data, cookies (httpOnly) for auth.

### Intersection Observer

Answers "is this element visible in the viewport?" without scroll-event thrashing. Powers infinite scroll, lazy-loaded images, and view tracking:

```js
const observer = new IntersectionObserver((entries) => {
  for (const entry of entries) {
    if (entry.isIntersecting) loadMoreItems();
  }
}, { rootMargin: '200px' });

observer.observe(document.querySelector('#sentinel'));
```

The `rootMargin` preloads slightly before the element scrolls into view, which is what makes lazy loading feel instant instead of janky.

### Clipboard

```js
await navigator.clipboard.writeText('INV-2024-0042');
const text = await navigator.clipboard.readText(); // requires permission
```

Requires a secure context (HTTPS) and, for reading, explicit permission. Perfect for "copy invoice number" buttons — and a good example of a feature that used to require a Flash hack and is now two lines.

### Credential Management

The WebAuthn / Credential Management API enables passkeys — biometric and hardware-key login that replaces passwords:

```js
const credential = await navigator.credentials.get({
  publicKey: serverGeneratedChallenge,
});
```

This is a server-side project as much as a frontend one (Symfony has growing passkey support), but the frontend half lives in this API. Even if you don't implement it yet, know it exists: passwordless is where authentication is heading, and retrofitting awareness of it later is harder than designing forms that accommodate it now.

## Progressive Enhancement vs. SPA-Only: When Each Is Honest

There's a long-running argument about whether every app should be a full SPA or whether sites should work without JavaScript. The honest answer: **it depends on what the page fundamentally is.**

**Progressive enhancement** means building core functionality with plain HTML (and server rendering), then layering JavaScript on top to improve the experience. The page works without JS; it works *better* with it.

**SPA-only** means the application cannot function without JavaScript. The server ships a shell; JS renders everything.

Neither is universally right. Ask: *what happens if JavaScript fails, is slow, or is blocked?*

Choose **progressive enhancement** when the task is fundamentally document-shaped:

- Content people search for — marketing pages, documentation, articles, product listings.
- Flows that must never fail — checkout steps, password reset, legal acceptance. A customer who can't pay because a JS bundle failed is lost revenue.
- Anything where first-paint speed and SEO are primary concerns.

Concretely, this often means Symfony serves fully rendered HTML, and Vue enhances specific islands: a richer date picker, live filtering, optimistic updates. The form submits normally if JS never loads.

Choose **SPA-only** when the product is genuinely an application:

- Dashboards, editors, admin tools — anything behind a login where SEO is irrelevant.
- Interfaces whose interaction model (drag-and-drop, real-time collaboration, canvas) can't degrade meaningfully.
- Tools where users are authenticated and invested — a broken bundle is a deploy incident, not a lost anonymous visitor.

What's *dishonest* is choosing SPA-only for a public content site because it's trendy, or forcing enhancement onto an internal admin tool nobody will ever visit without JS. Match the technique to the stakes. In this book's architecture, the public-facing surface leans enhanced server-rendered pages; the logged-in dashboard is a full Vue SPA. Both decisions are deliberate.

## Browser Support Policy and Vite Targets

"Supports all browsers" is not a policy — it's an inability to say no. A support policy is a written decision about which browsers you test, which you accept bugs from, and which you actively break.

A typical policy looks like this:

| Tier | Browsers | Commitment |
|------|----------|------------|
| Supported | Last 2 versions of Chrome, Edge, Firefox, Safari | Fully tested; bugs fixed |
| Functional | Older evergreen versions, Samsung Internet | Should work; best-effort fixes |
| Unsupported | IE11, very old Android WebView | No guarantee; may show an upgrade notice |

Two principles drive it:

- **Evergreen browsers auto-update**, so "last 2 versions" tracks reality automatically. The main exception to watch is Safari — iOS Safari updates are tied to OS releases, so users on old iPhones lag years behind. Check analytics for your actual audience before finalizing the list.
- **The policy is a build input, not just a QA checklist.** Which browsers you support determines how far back your JavaScript must compile.

### How this constrains Vite

Vite ships modern JavaScript by default and uses **esbuild** for transforms and **Browserslist** (via `@vitejs/plugin-legacy` when needed) to decide how far to transpile. Your `.browserslistrc` file is the dial:

```text
# .browserslistrc
defaults
not dead
not op_mini all
```

Tighten it and Vite emits less-transpiled, smaller bundles — native ES modules, no polyfills for features your audience already has. Loosen it (say, to include older Safari) and the legacy plugin injects polyfills and generates a second legacy bundle, increasing build time and payload for everyone.

The practical guidance:

- **Don't support browsers your analytics says you don't have.** Every extra year of browser support costs bundle size and build complexity forever.
- **Set the target explicitly** rather than accepting defaults blindly — defaults change, and silent changes to your output are how production surprises happen.
- **Revisit quarterly.** Dropping a dead browser is a one-line change and an immediate win; hoarding support for browsers nobody uses is pure tax.

---

# Chapter 4. Accessibility as a Feature Constraint

Accessibility is usually introduced as a compliance chore: run an audit, fix the red flags, ship. That framing guarantees failure, because it treats accessibility as something you *add* at the end rather than something you *build* from the start.

This chapter takes a different position: **accessibility is a feature constraint**, like security or performance. A checkout flow that a screen reader user cannot complete is as broken as one that throws a 500 error. The feature doesn't work. In this book's workflow, accessibility criteria appear in the ticket definition of done — not in a sprint-end audit nobody budgeted time for.

## WCAG 2.2 and POUR: The Map, Not the Territory

The **Web Content Accessibility Guidelines (WCAG)** are the international standard for digital accessibility. Version 2.2 is the current target, organized into three conformance levels: **A** (minimum), **AA** (the standard most laws and contracts reference), and **AAA** (aspirational). Aim for AA; treat AAA as selective goals where cheap.

WCAG's dozens of success criteria reduce to four principles you can memorize as **POUR**:

- **Perceivable** — users must be able to *sense* the content through sight, sound, or touch. Text alternatives for images, captions for video, sufficient contrast, content that works when zoomed to 200%.
- **Operable** — everything must be usable without a mouse. Full keyboard access, visible focus indicators, no time limits that punish slow users, no flashing that triggers seizures.
- **Understandable** — readable language, predictable behavior, clear labels, errors explained in text.
- **Robust** — content must work with assistive technology, now and in the future. Valid HTML, correct roles and names on custom widgets.

POUR is your review checklist. When evaluating any component, ask all four questions. Most accessibility failures are failures of Operable (keyboard) or Robust (broken semantics).

One orientation note: WCAG tells you *what* to achieve, not *how*. That's deliberate — techniques change as the platform evolves. Learn the principles deeply and the specific techniques follow naturally.

## Semantic HTML First; ARIA Only to Fill Gaps

Here is the rule that prevents 80% of accessibility bugs:

> **No ARIA is better than bad ARIA. Use native HTML first. Reach for ARIA only when HTML has no answer.**

Recall the contract idea from Chapter 3: `<button>` already announces itself as a button, receives keyboard focus, activates on Enter and Space, and exposes its label to screen readers. Build the same thing with `<div role="button" tabindex="0">` and you get *one* of those behaviors — the rest you must implement by hand, forever, in every place you use it.

The decision ladder:

1. **Is there a native element?** Use it. `<button>`, `<a href>`, `<input>`, `<select>`, `<details>`, `<dialog>` cover far more ground than developers assume.
2. **Native element exists but needs enhancement?** Keep the native element, add attributes. An `<input type="text">` with `aria-describedby` pointing at hint text.
3. **No native equivalent?** Build a custom component with correct ARIA roles, states, and full keyboard support — following the WAI-ARIA Authoring Practices patterns for that widget type.

When you do use ARIA, know what each attribute category does:

- **Roles** declare what something *is*: `role="tablist"`, `role="alert"`.
- **States and properties** declare what something *is like right now*: `aria-expanded="true"`, `aria-disabled`, `aria-current="page"`.
- **Names and relationships** connect things: `aria-label`, `aria-labelledby`, `aria-describedby`.

The classic failure mode is sprinkling ARIA on broken markup and calling it fixed. `role="button"` on a `<div>` with no keyboard handler is worse than nothing — screen reader users will try to activate it and fail silently.

## Keyboard Model, Focus Management, and Route Announcements

Keyboard access is the backbone of operability — and not just for blind users. Power users, people with motor disabilities who use switch devices, and anyone with a broken trackpad all navigate by keyboard.

### The keyboard model

The core interactions every interactive element must honor:

- **Tab / Shift+Tab** move focus forward and backward through the page in DOM order.
- **Enter** activates links and buttons.
- **Space** activates buttons and toggles checkboxes; scrolls otherwise.
- **Arrow keys** move within composite widgets — tabs, radio groups, menus, listboxes.

That last point matters for custom components: inside a tablist or menu, arrow keys should move between items while Tab moves *past* the whole widget. This "roving tabindex" pattern is documented in the ARIA Authoring Practices guide — copy it rather than inventing it.

### Focus management

Focus is invisible until it's the only thing a user has. Three rules:

- **Focus must always be visible.** Never write `outline: none` without providing an equally clear replacement (`:focus-visible` styles count). A keyboard user with no visible focus is lost.
- **Moving content must move focus.** When a modal opens, focus goes into it. When it closes, focus returns to the trigger. When a "skip to main content" link is activated, focus lands on `<main>`. When new content replaces old (search results load), focus should move to the new region or its heading.
- **Trap focus in modals.** Tabbing out of an open dialog into the dimmed background is disorienting and functionally broken. Native `<dialog>` does this for free — another argument for using it.

### SPA route changes

Here's a problem unique to SPAs: when Vue Router swaps views, the URL changes but **no page load occurs** — so screen readers announce nothing. A sighted user sees the new page; a screen reader user hears silence and may believe the app froze.

The fix is a **live region** that announces route changes:

```html
<div aria-live="polite" class="visually-hidden" ref="announcer">
  {{ announcement }}
</div>
```

```js
// In your router's afterEach hook:
router.afterEach((to) => {
  announcer.textContent = `Navigated to ${to.meta.title}`;
});
```

Vue Router doesn't do this automatically. Wire it once in your app shell and every route change becomes perceivable. Also set `document.title` per route — it's both an accessibility cue and basic usability.

## Forms, Errors, and Live Regions

Forms are where accessibility succeeds or fails most visibly, because forms are where users *must* succeed to finish their task.

### Labels and structure

- Every input gets a real `<label>` tied via `for`/`id` — placeholders are not labels (they vanish on input and have poor contrast).
- Group related fields with `<fieldset>` and `<legend>`: "Shipping address" wrapping street, city, zip.
- Mark required fields with the `required` attribute *and* communicate it in text — color alone or an asterisk alone isn't enough.

### Errors that actually help

An error message must survive three tests:

1. **It's programmatically associated.** Link the message to the field with `aria-describedby` (for hints) and `aria-invalid="true"` (for state), so screen readers announce them together.
2. **It's announced.** When validation fails after submit, move focus to the first invalid field, and/or summarize errors in a container marked `role="alert"` so the announcement happens immediately.
3. **It says how to fix it.** "Date of birth must be in MM/DD/YYYY format" — not "Invalid input."

```html
<label for="dob">Date of birth</label>
<input id="dob" type="text" aria-invalid="true"
       aria-describedby="dob-error dob-hint">
<p id="dob-hint">Format: MM/DD/YYYY</p>
<p id="dob-error" class="error">Enter your date of birth in MM/DD/YYYY format.</p>
```

### Live regions

A **live region** tells assistive technology "content here can change — announce changes." Two levels cover most needs:

- `role="alert"` / `aria-live="assertive"`: interrupt now. Form errors, session expiring.
- `aria-live="polite"`: wait for a pause, then announce. Search result counts ("23 results"), toast notifications, the route-change announcer above.

Critical gotcha: live regions only announce changes to elements **already present in the DOM**. If you insert the alert element at the same moment you fill it, many screen readers miss it. Render the live region up front (even empty) and update its text later.

## Color, Contrast, Motion, and `prefers-reduced-motion`

### Contrast

WCAG AA requires a contrast ratio of at least **4.5:1** for normal text and **3:1** for large text (18pt+ or 14pt bold) and for meaningful UI components like input borders and icons. Gray-on-gray placeholder text fails this constantly.

Check colors during design, not after build — tools like WebAIM's contrast checker take seconds. And never encode meaning in color alone: a red error border plus an icon plus text survives color blindness; red alone doesn't.

### Zoom and reflow

Users can legally need 200% zoom or 400% on mobile. Content must reflow into a single column without horizontal scrolling and without clipped or overlapping text. Test by zooming your own app — it's humbling and fast.

### Motion

Animations cause real harm: vestibular disorders make large moving animations genuinely nauseating, and flashing content can trigger seizures (keep flashes under 3 per second, always).

CSS gives you a media query for this:

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

For JavaScript-driven animation (Framer Motion, GSAP), check the equivalent matchMedia query before animating. The pattern: respect the OS-level setting everywhere, and keep essential feedback (loading spinners, focus movement) even when decorative motion is disabled.

## Testing: axe, Playwright Snapshots, Keyboard QA

Automated tooling catches roughly a third of accessibility issues — which makes it excellent for regression prevention and insufficient as proof of quality. Layer three kinds of testing.

### Automated scanning with axe

**axe-core** runs rules against rendered DOM and integrates directly into your test suite. With Playwright:

```js
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test('checkout page has no critical violations', async ({ page }) => {
  await page.goto('/checkout');
  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa', 'wcag22aa'])
    .analyze();
  expect(results.violations).toEqual([]);
});
```

Run this per page in CI. It catches missing alt text, low contrast, unlabeled inputs, and duplicate IDs — the mechanical stuff — permanently, for free.

### Accessibility snapshots

Playwright can serialize the **accessibility tree** — the structure screen readers actually consume — and snapshot it like visual regression testing:

```js
await expect(page.locator('body')).toMatchAriaSnapshot();
```

The snapshot shows whether your modal reads as a labeled dialog, whether your nav reads as navigation, whether your button has a name. When a refactor accidentally breaks semantics, the snapshot diff flags it in code review.

### Manual keyboard-only QA

Ten minutes per feature, no plugins required:

1. Unplug the mouse (metaphorically). Complete the entire task flow with Tab, Enter, Space, and arrows.
2. Watch the focus indicator — can you always tell where you are?
3. Open a modal: does focus enter it, stay in it, and return on close?
4. Submit an invalid form: is the error announced and focus moved?
5. Spot-check with a screen reader (VoiceOver on macOS comes free; NVDA on Windows) for one full pass. It feels awkward the first five times. Do it anyway.

This manual pass catches what axe can't: confusing focus order, missing announcements, and interactions that technically work but make no sense.

## Vue-Specific Pitfalls

Vue makes accessible components easy to build and easy to break. These are the traps that show up repeatedly.

### Custom components that destroy native semantics

The most common bug in Vue codebases: a reusable component renders a `<div>` where a native element belongs.

```vue
<!-- Bad: looks clickable, isn't a button -->
<div class="btn" @click="submit">{{ label }}</div>

<!-- Good: native semantics preserved -->
<button type="button" class="btn" @click="submit">{{ label }}</button>
```

Fix it structurally: make base components (`BaseButton`, `BaseInput`) wrap native elements, and audit third-party UI libraries before adopting them — many popular ones still ship div-based buttons. If a library fails the keyboard test, it fails.

Also watch props that override semantics: a component that accepts `href` should render `<a>`, not a click-handled `<button>`. Links navigate; buttons act. Conflating them breaks expectations for everyone, especially screen reader users listening for "link" versus "button."

### `RouterLink` vs. real buttons

This deserves its own callout because it's everywhere:

```vue
<!-- Wrong: navigation styled as a button -->
<RouterLink to="/invoices/new" class="btn btn-primary">New invoice</RouterLink>
<!-- ...is actually fine! RouterLink renders <a href="/invoices/new"> -->

<!-- Wrong: action disguised as a link -->
<RouterLink to="#" @click="deleteInvoice(invoice.id)">Delete</RouterLink>
```

The rule: **if it navigates, it's a link; if it performs an action, it's a button.** `RouterLink` rendering an anchor with a real `href` is exactly right for navigation — deep-linkable, middle-clickable, announced as a link. But "Delete," "Submit," "Add to cart" are actions: they belong on `<button>` elements, even when styled to look like links. Screen reader users hearing "link: Delete" will try to open it in a new tab.

### Other recurring Vue issues

- **`v-if` vs. `v-show` and focus loss:** removing the focused element with `v-if` drops focus to `<body>`. After conditional removals, explicitly restore focus somewhere sensible.
- **Transitions hiding content:** Vue transition wrappers can leave elements visually hidden but focusable, or animate content screen readers see instantly. Pair transitions with the reduced-motion query and manage focus around enter/leave hooks.
- **Dynamic lists:** when items reorder or filter, announce the change via a polite live region ("5 invoices shown") — silent reordering is invisible to non-visual users.
- **Icon-only buttons:** an icon button with no text needs `aria-label`. Audit these — they're the most commonly unnamed controls in any SPA.

## Accessibility Is an Acceptance Criterion, Not an Audit

Everything above collapses if accessibility lives in a sprint-end audit. Audits find problems after they're expensive to fix; acceptance criteria prevent them from being created.

What this means practically in your workflow:

- **Tickets carry accessibility criteria.** "User can complete refund flow using keyboard only" and "axe scan reports zero critical violations" sit next to functional requirements in the definition of done.
- **PRs include the manual checks.** The author notes "keyboard-tested, focus managed on modal close" alongside "tests passing."
- **CI enforces the floor.** axe runs in Playwright suites; violations fail the build. Humans spend their judgment on the things automation can't check.
- **Design reviews include contrast and focus states** before implementation starts — fixing contrast in Figma costs nothing; fixing it across a themed component library costs days.

The economics justify all of it: retrofitting accessibility onto a finished feature routinely costs multiples of building it correctly, and inaccessible features exclude real customers — roughly one in six people worldwide has a disability. Framed as a constraint, accessibility stops being charity or compliance theater and becomes what it actually is: **correctness for more of your users.**

---

# Part II — Identity, Authorization, and Security

# Chapter 5. OAuth 2.0 and OpenID Connect

Ask five developers to explain OAuth and you'll get five answers, at least three of which conflate authentication with authorization. That confusion causes real security holes — most famously, treating an access token as proof of who a user is.

This chapter fixes the vocabulary first, then walks through how the two layers actually work, how we implement them across the Symfony/Vue/Node stack, and how they fail in practice.

## Two Layers, Not Synonyms

**OAuth 2.0 is delegated authorization.** It answers one question: *"Can this client access this resource on behalf of this user?"* It was designed so you could let a photo-printing service read your Google Photos without giving it your Google password.

**OpenID Connect (OIDC) is authentication built on top of OAuth 2.0.** It answers a different question: *"Who is this user?"* OIDC adds a standard identity layer — an ID token containing verified claims about the user — so that "log in with X" flows have a well-defined protocol instead of everyone inventing their own.

The critical rule that follows:

> **Never derive identity from an access token.**

An access token says what a *client* may do at a *resource server*. Depending on the grant type, there may be no user involved at all (service-to-service tokens). Access tokens are opaque strings or JWTs whose contents are an implementation detail of the authorization server — not a spec-guaranteed identity document. When you need identity, use the **ID token** from OIDC, validated properly. Teams that skip this distinction end up trusting `sub` claims from tokens issued for entirely different purposes.

## Concepts

### The four roles

OAuth defines four parties. Keep them straight; every flow description below uses them:

- **Resource Owner** — the user (or system) that owns the data and grants access.
- **Client** — the application asking for access. Our Vue SPA, our Node webhook service.
- **Authorization Server / OpenID Provider (OP)** — issues tokens after authenticating the resource owner. Auth0, Keycloak, Okta, or your own Symfony-based server.
- **Resource Server** — the API holding the protected data. Our Symfony API, our internal Node services.

### The four tokens

- **Authorization code** — not really a token but a short-lived, single-use intermediate. The client exchanges it (with its credentials or PKCE verifier) for real tokens. It exists so secrets and tokens never travel through the browser's front channel.
- **Access token** — the credential presented to resource servers. Opaque string or JWT. Short-lived by design: minutes, not days.
- **Refresh token** — a long-lived credential used to obtain new access tokens without re-authenticating the user. High-value target; handle accordingly.
- **ID token** — the OIDC addition. A JWT containing claims about the authenticated user (`sub`, `email`, `iss`, `aud`, `exp`, `nonce`). This is the *only* token from which you may derive identity.

### Grants still acceptable

The OAuth spec has accumulated dead weight over fifteen years. Only two grants matter for new work:

- **Authorization Code + PKCE** — for anything involving a user: SPAs, mobile apps, traditional web apps. The user is redirected to the authorization server, approves, and the client receives a code which it exchanges for tokens. PKCE (explained below) makes this safe even for clients that can't hold secrets.
- **Client credentials** — for service-to-service calls with no user involved. A PHP worker pulling data from an internal API authenticates as *itself* using a client ID and secret, and receives an access token scoped to the service.

If someone proposes any other grant for a new integration, the answer is no.

### Grants that are dead

- **Implicit** — returned access tokens directly in the URL fragment because browsers couldn't do CORS properly in 2012. Tokens leak via browser history and referrer headers, and there's no refresh mechanism. Deprecated in OAuth 2.1. If a third-party SDK still uses it, that SDK is telling you something.
- **Resource Owner Password Credentials (ROPC)** — the app collects the user's actual password and trades it for tokens. Defeats the entire point of delegation, breaks SSO and MFA, and trains users to type their IdP password into your UI. Explicitly deprecated.

### The security mechanisms you must get right

- **PKCE ("pixy")** — Proof Key for Code Exchange. The client generates a random `code_verifier`, sends only its hash (`code_challenge`) in the authorization request, then proves possession of the original when exchanging the code. This prevents an attacker who intercepts the authorization code from redeeming it — essential for public clients (SPAs, mobile) that have no client secret, and now recommended for *all* clients.
- **Exact redirect URI matching** — the authorization server must compare the `redirect_uri` byte-for-byte against registered values. Prefix matching or wildcard matching lets attackers redirect codes to lookalike URLs.
- **`state`** — a random value sent in the authorization request and verified on callback. It binds the response to the request, defeating CSRF-style attacks where an attacker injects their own code into your flow.
- **`nonce`** — a random value included in the OIDC authentication request and checked against the ID token's `nonce` claim. Prevents token replay in authentication flows.
- **Sender-constrained tokens** — tokens bound to a specific client (via DPoP or mTLS), so a stolen token is useless to anyone else. Not yet universal, but the direction the ecosystem is moving; know the term for design discussions.

### OAuth 2.1: secure defaults made mandatory

**OAuth 2.1** isn't a new protocol — it's a consolidation. It takes 2.0, deletes the broken grants (Implicit, ROPC), mandates PKCE everywhere, requires exact redirect matching, and folds in years of BCPs (best current practices). Read it as: *"here's what OAuth 2.0 should have been if written today."* Everything in this chapter is compatible with both; writing 2.1-compliant code means you never have to think about the legacy paths.

## Implementation

### Symfony: IdP choice and firewall structure

First decision: **do you run your own authorization server?** Running one correctly is a serious undertaking — token lifecycle, rotation, revocation, compliance with evolving attack research. For most teams the honest answer is **use an external IdP** (Keycloak self-hosted, or Auth0/Okta managed) and make Symfony a resource server plus OIDC relying party.

If you must issue your own tokens, `league/oauth2-server` is the established PHP library — it implements Authorization Code + PKCE and client credentials correctly out of the box. Wrap it rather than reimplementing endpoints.

On the consuming side, Symfony's security component maps cleanly onto OAuth concepts:

```yaml
# config/packages/security.yaml
security:
    firewalls:
        api:
            pattern: ^/api
            stateless: true
            oauth2: true   # validates bearer tokens against the IdP
```

Key practices:

- **Stateless API firewall.** The API validates the access token on every request and stores nothing session-related. Sessions belong to the frontend/BFF layer.
- **Validate everything on the ID token**: signature against the OP's JWKS endpoint, `iss` matches your expected issuer exactly, `aud` contains your client ID, `exp` hasn't passed, `nonce` matches what you sent.
- **Clock skew.** Distributed servers disagree about time by seconds. Allow a small leeway (30–60 seconds) when checking `exp` and `nbf` — strict comparison causes intermittent, maddening auth failures right after deployment to a box with drifting clocks.
- **Token storage on the server side:** keep client secrets and refresh tokens in your secret manager or encrypted at rest — never in the database in plaintext, never in code, never in logs.

### Vue / SPA: Authorization Code + PKCE and the storage question

An SPA is a **public client**: any secret embedded in it is public. So it uses Authorization Code + PKCE with no client secret.

The flow, briefly: the SPA redirects the user to the IdP with `code_challenge` and `state`; the user authenticates there; the IdP redirects back to your exact registered redirect URI with a code; the SPA exchanges the code (plus `code_verifier`) for tokens.

Then comes the hard question: **where do the tokens live?**

- **In-memory + silent renew:** hold the access token in a JavaScript variable (never `localStorage`). On page reload, use a hidden iframe or refresh-token rotation against the IdP to silently re-establish a session. Pros: XSS can't exfiltrate a token that isn't persisted anywhere readable. Cons: more moving parts, and third-party cookie restrictions are making iframe-based silent renew unreliable.
- **httpOnly cookie via a BFF:** the SPA talks only to your own backend-for-frontend (that Node glue layer from Chapter 1); the BFF holds tokens server-side and sets an httpOnly, Secure, SameSite cookie for the browser. The browser never sees an access token at all. Pros: immune to token theft via XSS, works with strict cookie policies. Cons: you operate another hop, and CSRF protection becomes your responsibility (SameSite cookies plus anti-CSRF tokens).

For new builds in this book's architecture, **the BFF pattern is the default recommendation** — it converts a hard frontend problem into a routine backend one, and we already run Node services.

Either way: **never put tokens in `localStorage`.** Any XSS anywhere in your bundle graph reads them instantly.

### Node microservices validating JWTs

Your Express/Fastify services are resource servers. Validation is non-negotiable and has exactly five checks:

```js
import { jwtVerify, createRemoteJWKSet } from 'jose';

const JWKS = createRemoteJWKSet(new URL('https://idp.example.com/.well-known/jwks.json'));

async function requireAuth(req, res, next) {
  const token = req.headers.authorization?.replace('Bearer ', '');
  try {
    const { payload } = await jwtVerify(token, JWKS, {
      issuer: 'https://idp.example.com',   // exact match
      audience: 'invoices-api',             // intended for THIS api
      clockTolerance: 30,                   // skew leeway, seconds
    });
    req.auth = payload;
    next();
  } catch {
    res.status(401).json({ error: 'invalid_token' });
  }
}
```

1. **Signature** — verified against the issuer's published keys (JWKS), fetched and cached automatically.
2. **Issuer (`iss`)** — exact string match. Accepting any issuer means accepting tokens from any IdP, including an attacker's.
3. **Audience (`aud`)** — the token must name *this* API. A token minted for the billing API must not open the invoices API.
4. **Expiry (`exp`)** — with small clock tolerance.
5. **Algorithm** — pin it (`RS256`). Never accept `"alg": "none"`, and don't let the token choose the algorithm — the classic JWT downgrade attack.

Note what's *not* here: deriving the user's identity for authorization decisions beyond scopes. Scopes say what the client may do; if you need the user, that came through the ID token at login time, not from this access token.

### Machine-to-machine between PHP workers and internal APIs

Background workers and cron jobs have no user. They use **client credentials**:

```php
// Worker fetches its own token, caches until near expiry
$response = $http->post('https://idp.example.com/oauth/token', [
    'form_params' => [
        'grant_type' => 'client_credentials',
        'client_id' => $config['client_id'],
        'client_secret' => $config['client_secret'],
        'scope' => 'invoices:read',
    ],
]);
```

Practices that matter here:

- **Cache the token** until shortly before `exp` — fetching a fresh token per job hammers the IdP and adds latency to everything.
- **Scope narrowly per consumer.** The invoice PDF worker gets `invoices:read`, not a god-mode scope shared by every service.
- **One credential pair per service.** Shared credentials mean you can never revoke one misbehaving service without breaking them all, and audit logs become useless.

## Failure Modes

Knowing the happy path isn't enough. These are the attacks and mistakes that show up in real incident reports.

### Confused deputy

A client with legitimate authority is tricked into using it for an attacker's benefit. Classic case: your API accepts a resource identifier from the request and acts on it without checking the caller owns it — the attacker supplies *someone else's* invoice ID, and your authorized client dutifully processes it. Defense is unglamorous: **every request re-checks object ownership against the authenticated principal**, even though the client is trusted. Never trust the client to send only its own IDs.

### Token leakage via logs and URLs

Tokens end up in places they shouldn't:

- **URLs.** An access token in a query string lands in browser history, server access logs, proxy logs, and referrer headers. This is why the Implicit grant died and why tokens should appear only in `Authorization` headers or POST bodies.
- **Logs.** Debug logging that dumps full request objects captures bearer tokens verbatim. Scrub `authorization` headers before logging, and treat any log line containing a token as an incident requiring rotation.

### Refresh-token rotation mistakes

Refresh tokens are long-lived, so modern practice is **rotation**: each use returns a new refresh token and invalidates the old one. Done right, reuse of an already-rotated token signals theft — the IdP should revoke the whole token family. Common mistakes:

- **Not rotating at all**, so a stolen refresh token works indefinitely.
- **Rotating but not detecting reuse**, losing the theft signal entirely.
- **Grace-period abuse** — allowing old tokens too long during rotation "for reliability," widening the window stolen tokens work.

### Mix-up attacks

When a client supports multiple IdPs (say, Keycloak and a partner's OP), an attacker can manipulate the flow so the client sends the authorization code to the *wrong* token endpoint — leaking it. Defenses: bind the IdP selection into `state`, validate that the `iss` claim in responses matches the IdP you started the flow with, and use distinct redirect URIs per provider.

## Checklist for Reviewing Any New Client

Use this whenever a PR introduces a new OAuth client, integration, or grant. Every line is a yes/no — a single no blocks merge.

**Grant and flow**

- [ ] Uses Authorization Code + PKCE (user-facing) or client credentials (service-to-service) — nothing else
- [ ] No Implicit or ROPC anywhere, including third-party SDK configs
- [ ] Redirect URIs registered exactly; server compares byte-for-byte

**Request integrity**

- [ ] `state` generated with a CSPRNG, stored, and verified on callback
- [ ] `nonce` sent and verified against the ID token (OIDC flows)
- [ ] ID token fully validated: signature, `iss`, `aud`, `exp`, `nonce`

**Token handling**

- [ ] Access tokens kept in memory (SPA) or server-side (BFF/service)
- [ ] Nothing token-shaped in `localStorage`, sessionStorage, or URLs
- [ ] Logs scrubbed of authorization headers and tokens
- [ ] Refresh tokens rotated; reuse detection enabled where the IdP supports it
- [ ] Tokens cached appropriately; secrets in the secret manager, not code

**Validation (resource servers)**

- [ ] Signature verified against JWKS; algorithm pinned
- [ ] `iss` and `aud` exact-matched; expiry checked with bounded skew
- [ ] Object-level authorization checks ownership on every request (confused deputy)

**Operations**

- [ ] One credential set per client/service; scopes minimal
- [ ] Revocation path documented — how do we kill this client's access tomorrow?
- [ ] Clock sync monitored on hosts validating tokens

---

Authentication and authorization done this way becomes infrastructure you rarely think about — which is the point.

# Chapter 6. Authorization Models: RBAC, ABAC, ReBAC

Authentication answers "who are you?" Authorization answers "what may you do?" — and it's where most serious application security bugs live. Not because developers don't care, but because authorization is *design*, not configuration. Pick the wrong model early and every feature afterward fights it.

This chapter covers the three dominant models, maps them onto a concrete example we'll use throughout the book (a project board application), and shows exactly where enforcement belongs in our stack.

## Why Role-Only Designs Fall Short

OWASP's authorization guidance is blunt about this: role-based control alone doesn't scale to real applications. Roles describe *who someone is*; they say nothing about *which specific resource* they're acting on or *under what conditions*. "Admin" tells you nothing about whether Dana can archive board 4721 — only that admins generally can do admin-shaped things.

The practical consequence of role-only thinking is that every exception becomes a new role, and roles multiply until nobody can explain what any of them mean. Roles remain useful — as **one attribute among several**, not as the entire model.

## RBAC: Fast to Start, Then Role Soup

**Role-Based Access Control** assigns permissions to roles, and roles to users:

```
User → Role → Permission
Dana → board_admin → board.archive
```

RBAC's virtues are real:

- **Fast to build.** A `roles` table, a join table, one check in your controller.
- **Easy to reason about at small scale.** "What can an editor do?" has a lookup answer.
- **Familiar.** Every framework and CMS ships with it.

The failure mode arrives on schedule. Real requirements sound like: "Managers can edit boards in their own department." That's not a role — it's a relationship plus a scope. So you add `department_manager`. Then: "Contractors can view but not export." Add `contractor_viewer`. Then: "The manager who created the board can always edit it, even after leaving the department." Now you have overlapping roles whose interactions nobody fully understands.

This is **role soup**: dozens of roles, each encoding a different combination of attributes and relationships, with no way to answer "why can't Dana see this board?" without archaeology. The tell-tale sign is when granting access requires creating a bespoke role for one person.

## ABAC: Attributes All the Way Down

**Attribute-Based Access Control** makes decisions from attributes of three things:

- **Subject attributes** — who is asking? Department, clearance level, employment type, MFA status.
- **Resource attributes** — what are they acting on? Board owner, classification, department, creation date.
- **Environment attributes** — under what circumstances? Time of day, IP range, device posture, threat level.

A policy reads like a rule over those attributes:

> Allow if `subject.department == resource.department` AND `subject.clearance >= resource.classification` AND `environment.business_hours == true`

ABAC delivers genuine **least privilege** — access flows from properties, not from membership lists that rot. It also handles conditions RBAC can't express at all ("only during business hours," "only from managed devices").

The cost is **reasoning difficulty**. With many attributes, answering "why was this request denied?" requires evaluating the whole policy tree. Debugging becomes "which attribute had the unexpected value?" — and policies interact in ways that surprise even their authors. ABAC earns its complexity only when you genuinely need environmental or multi-dimensional conditions.

## ReBAC: Relationships, Not Flags

**Relationship-Based Access Control** models authorization as a graph of relationships between entities. Instead of asking "does Dana have the `board_editor` flag?", ReBAC asks:

- Is Dana the **owner of** board 4721?
- Is Dana a **member of** the organization that **owns** board 4721?
- Is Dana a **viewer on** a folder that **contains** board 4721?

Authorization becomes graph traversal: walk outward from the user through typed relationships until you reach the resource, then check whether the path grants the action. Google Zanzibar made this approach famous at planetary scale; open-source implementations like OpenFGA and SpiceDB bring it within reach of ordinary teams.

ReBAC shines exactly where RBAC breaks down:

- **Ownership** — "the creator can always edit" is a relationship, not a role.
- **Sharing** — "Dana shared this document with Carlos" is an edge in the graph.
- **Inheritance** — org → team → folder → board, with permissions flowing down the hierarchy.

The cost: you must model and store relationships explicitly, and traversal queries need indexing discipline. But note something important — **most business applications are mostly ReBAC problems wearing RBAC costumes.** Ownership, membership, sharing, hierarchy: these are relationships.

## Mapping the Project Board

Let's apply all three models to the book's running example: a Trello-like project board with organizations, teams, boards, and cards.

### Org membership is ReBAC

"Can Dana view board 4721?" decomposes into a graph question:

```
Dana —[member_of]→ Team Phoenix —[belongs_to]→ Org Acme —[owns]→ Board 4721
```

Store the edges (`memberships`, `team_org`, `org_boards`) and traverse them. This handles inheritance naturally: adding Dana to Team Phoenix grants access to everything that team's org owns, now and future. No per-board flags to maintain, no sync jobs to forget.

### "Admin" is an attribute

Org-level administration isn't a global role — it's an attribute of the *membership*: `org_membership.role = 'admin'`. The same human might be an admin in Acme and a plain member in Globex. Modeling admin as a **relationship attribute** keeps it scoped correctly, which a global `is_admin` flag never does.

### Business hours + device posture as ABAC — if needed

Suppose compliance requires that exports happen only during business hours from managed devices. That's pure environment/subject-attribute logic layered on top:

> Allow export if ReBAC grants read AND `env.hour ∈ [9, 18]` AND `device.managed == true`

Notice the layering: **ReBAC decides who has standing; ABAC decides whether conditions permit exercising it right now.** Don't reach for this unless a requirement actually demands it — each environment attribute adds debugging surface. Ship the ReBAC core first; add ABAC conditions when a real policy needs them.

### What we deliberately avoid

No global roles like `superuser` beyond break-glass operations, no per-resource boolean flags (`can_edit_board_4721`) — those are unqueryable and unauditable. If you find yourself adding a column named like a permission, stop: it's probably a missing edge in the graph.

## Enforcement Points: Where Checks Actually Live

Knowing the model is half the battle. The other half is knowing **where** to enforce it, because enforcement points differ wildly in strength.

From weakest to strongest:

1. **API gateway / edge** — coarse filtering: rate limits, IP rules, maybe "is this token valid for this service?" Never business authorization. Gateway rules drift out of sync with the domain instantly.
2. **Controller / route level** — "must be authenticated," "must have API scope X." Fine for coarse gates, useless for object decisions — the controller often doesn't know which board is involved yet.
3. **Domain service / voter** — the decision point: "may THIS actor perform THIS action on THIS object?" This is where ReBAC/ABAC evaluation belongs.
4. **Query filter** — the strongest and most neglected point. When listing resources, authorization must appear **in the SQL itself**, as a `WHERE` clause derived from the actor's relationships.

That last point deserves emphasis because it's where **IDOR** (Insecure Direct Object Reference) lives. Consider the classic bug:

```php
// VULNERABLE: fetches whatever ID the request mentions
$board = $boardRepository->find($request->get('id'));
$security->denyAccessUnlessGranted('EDIT', $board); // ...if $board exists
```

Two failures hide here. First, a nonexistent ID returns null and crashes differently than a forbidden one — leaking existence. Second, and worse, list endpoints rarely get even this check:

```php
// VULNERABLE: returns ALL boards, filtered by nothing
$boards = $boardRepository->findAll();
```

The fix is to make authorization part of data retrieval:

```php
// SAFE: the query itself is scoped by relationship
$qb = $boardRepository->createQueryBuilder('b')
    ->join('b.org', 'o')
    ->join('o.memberships', 'm')
    ->where('m.user = :user')
    ->setParameter('user', $actor);
```

Now unauthorized rows aren't rejected — they're *invisible*. There is no ID to reference directly. Rule of thumb: **every read path filters by relationship; every write path checks the decision explicitly.** Both, always.

## Symfony Voters vs. a Dedicated Policy Module

Symfony gives you two native tools, plus the option to go further.

### Voters

A **Voter** encapsulates a decision as a class:

```php
final class BoardVoter extends Voter
{
    protected function supports(string $attribute, mixed $subject): bool
    {
        return $attribute === 'EDIT' && $subject instanceof Board;
    }

    protected function voteOnAttribute(string $attribute, mixed $subject, TokenInterface $token): bool
    {
        $membership = $this->memberships->findFor($token->getUser(), $subject->getOrg());
        if (!$membership) {
            return false;
        }
        return $membership->isAdmin()
            || $subject->getOwnerId() === $token->getUserIdentifier();
    }
}
```

Called via `$security->isGranted('EDIT', $board)` or `#[IsGranted('EDIT', subject: 'board')]`. Voters are excellent up to moderate complexity: testable, injectable, composable.

### Security expressions

For simple declarative checks, expressions keep intent visible in routing config:

```yaml
controllers:
    resource: ../src/Controller/
    # e.g., #[IsGranted("is_granted('MEMBER', subject.getOrg())")]
```

Use them for shallow checks; anything with branching logic belongs in a Voter, not a YAML string.

### When to extract a dedicated policy module

When decisions start depending on external inputs (device posture, time), when multiple services need identical decisions, or when policy review becomes a compliance requirement, extract authorization into its own module — or adopt an external engine like OpenFGA. The interface stays simple regardless:

```php
interface PolicyEngine
{
    /** @return Decision with allow/deny plus an explanation */
    public function decide(Actor $actor, string $action, ResourceRef $resource, Context $ctx): Decision;
}
```

Returning a **decision with an explanation** pays off enormously in support tickets: "denied because your membership ended March 3" beats "403 Forbidden" for everyone involved. Start with Voters; refactor behind this interface when the pressure appears. Don't build the abstraction before the second consumer exists.

## Vue: UI Hiding Is UX, Not Authorization

Every SPA gets this wrong eventually: hiding the "Delete board" button because the current user lacks permission, and calling it done.

It isn't done. It isn't even authorization. The UI is running on the user's machine, in a bundle they can read and modify. Any client-side check is a suggestion. The real system boundary is your API.

So reframe it:

- **Client-side visibility = UX.** Hiding buttons users can't use reduces confusion and error noise. Do it — driven by permissions data the server already sent.
- **Server-side enforcement = authorization.** Every mutation endpoint independently verifies the decision. Always. Without exception. Even for actions the UI never exposes.

The corollary: the API must be complete without trusting the UI. If deleting a board requires calling an endpoint the UI hides, that endpoint still validates ownership server-side — because a crafted HTTP request costs an attacker nothing. Design reviews should ask of every mutation: *"what happens if this is called directly with curl?"* If the answer involves trusting the client, the design is wrong.

One practical pattern: return the user's effective permissions (or a compact capability set) with the session/bootstrap payload, so the Vue app renders accurately without extra requests — while treating that data as display-only.

## Testing Authorization: Matrix Tests

Authorization bugs are sneaky because they're about *combinations*: this actor, that resource, this action. Unit tests of voters catch some of it; what catches the rest is systematic coverage.

Build a **matrix test**: actors × resources × actions, asserting the expected outcome for every cell.

```php
public function authorizationMatrix(): array
{
    return [
        // actor              resource          action      expected
        'owner edits own board'      => ['dana',   'board-4721', 'EDIT',   true],
        'member edits org board'     => ['carlos', 'board-4721', 'EDIT',   true],
        'outsider edits org board'   => ['eve',    'board-4721', 'EDIT',   false],
        'member views other org'     => ['carlos', 'board-9000', 'VIEW',   false],
        'expired member edits'       => ['frank',  'board-4721', 'EDIT',   false],
        // ...
    ];
}
```

Each row runs the full stack — query filter included — against a seeded database. New edge cases become new rows, and the matrix doubles as living documentation of your policy.

### Horizontal privilege escalation

Vertical escalation is "user becomes admin." **Horizontal escalation** is subtler and far more common: a user accesses *another peer user's* resources at the same privilege level. Carlos reading Dana's board. Eve renaming Frank's card.

Test it explicitly and adversarially:

- For every endpoint taking an ID, assert that IDs belonging to *other tenants/users* return 404 (not 403 — existence leaks through 403s).
- Assert list endpoints contain zero foreign rows.
- Include sequential-ID probing in tests: `board-4721`, `board-4722` — auto-increment IDs make enumeration trivial, which is another argument for UUIDs on externally visible identifiers.

Also test the *negative space*: expired memberships, removed members mid-session, users deleted while holding active tokens. Authorization code rots quietly when relationships change underneath it.

---

Authorization done well is invisible: users see exactly what they should, attackers find no seams, and auditors get explanations instead of mysteries.

# Chapter 7. High-Level Web Security

Security advice usually comes in two useless forms: a poster of the OWASP Top 10 taped to a wall, and a 400-page tome nobody finishes. This chapter takes a third path — the Top 10 as an **engineering checklist** you walk through while building, with deep dives into the specific vulnerabilities our stack (Symfony, Vue, Node, MySQL) actually hits.

The current OWASP Top 10:2025, which we'll reference throughout:

1. Broken Access Control
2. Security Misconfiguration
3. Software Supply Chain Failures
4. Cryptographic Failures
5. Injection
6. Insecure Design
7. Authentication Failures
8. Software or Data Integrity Failures
9. Security Logging and Alerting Failures
10. Mishandling of Exceptional Conditions

Several of these got full chapters already — access control in Chapter 6, authentication in Chapter 5, supply chain basics in Chapter 2. Here we cover what remains and go deep on the browser-level mechanics every web developer must actually understand.

## Same-Origin Policy: What the Browser Actually Isolates

The **Same-Origin Policy (SOP)** is the most important security mechanism you never configured. It's the browser's rule that JavaScript running on one origin cannot read responses from another origin.

An origin is scheme + host + port. `https://app.example.com` and `https://api.example.com` are *different* origins. So are `http://` and `https://` of the same host.

What SOP isolates:

- **Reads.** Script from `evil.com` can *send* a request to `bank.com` (the request happens!) but cannot *read* the response. This is why CSRF is about state-changing requests, not data theft.
- **Storage.** `localStorage`, cookies (mostly), and IndexedDB are partitioned per origin.
- **DOM access.** A page can't script into an iframe from another origin.

What SOP does *not* isolate:

- **Sending requests.** Cross-origin requests fire freely; only reading responses is blocked. Cookies attached by the browser still travel.
- **Loading resources.** Images, scripts, stylesheets, and fonts load cross-origin all the time — `<script src>` is how CDNs work. SOP governs *reading back* the content, not fetching it.
- **Anything outside the browser.** Your server-to-server calls have no origin policy at all. SOP protects browsers, not backends.

Understanding this asymmetry explains half of web security: attacks exploit what SOP permits (sending), defenses extend what it blocks (reading).

## CORS: Explicit Cooperation Between Origins

**Cross-Origin Resource Sharing (CORS)** is how a server *opt-in relaxes* SOP for specific origins. It's not a firewall and not authentication — it's a set of response headers saying "I permit this."

### Simple vs. preflighted requests

Browsers split cross-origin requests into two categories:

- **Simple requests** — GET, HEAD, POST with basic content types (`text/plain`, `application/x-www-form-urlencoded`, `multipart/form-data`). These fire immediately; the browser just blocks reading the response unless headers permit.
- **Preflighted requests** — anything else: custom headers like `Authorization`, JSON content type, PUT/DELETE/PATCH. The browser first sends an OPTIONS request asking permission:

```http
OPTIONS /api/orders HTTP/1.1
Origin: https://app.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: authorization, content-type
```

The server answers:

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PATCH
Access-Control-Allow-Headers: authorization, content-type
Access-Control-Allow-Credentials: true
Vary: Origin
```

Only then does the real request fire. That's why unfamiliar CORS errors show up as mysterious failed OPTIONS calls.

### Credentialed requests and the rules that matter

When cookies or Authorization headers must accompany a cross-origin request, the server sets `Access-Control-Allow-Credentials: true` — and then two hard rules apply:

- **Never combine `Access-Control-Allow-Origin: *` with credentials.** The spec forbids it, and any framework that lets you do it is handing your users' cookies to arbitrary sites. If you find yourself wanting this, the design is wrong.
- **Use explicit allowlists**, echoed per-request against a configured list, with `Vary: Origin` so caches don't serve one origin's headers to another.

```php
// Symfony / NelmioCorsBundle style configuration
nelmio_cors:
    paths:
        '^/api/':
            allow_origin: ['https://app.example.com']
            allow_credentials: true
            allow_methods: ['GET', 'POST', 'PATCH', 'DELETE']
```

Also remember: CORS protects the *browser's user*, not your API. curl doesn't check CORS headers. An attacker who has stolen a token bypasses CORS entirely. CORS complements authz; it never replaces it.

## CSRF: Forging Requests the Browser Sends for You

Because SOP allows sending (but not reading) cross-origin requests, an attacker page can make *your browser* submit a state-changing request to your app — with your cookies attached automatically. That's **Cross-Site Request Forgery**: the victim's own credentials weaponized against them.

### Cookie semantics first

Modern cookie attributes eliminate most CSRF risk at the source:

- **`SameSite=Lax`** (the modern default): cookies are sent on top-level navigations but *not* on cross-site form posts, images, or fetches. Blocks the classic CSRF vector outright.
- **`SameSite=Strict`**: cookie never sent cross-site at all. Stronger, but breaks legitimate inbound links ("click this link to your dashboard" arrives logged out).
- **`Secure`**: cookie travels over HTTPS only.
- **`HttpOnly`**: JavaScript cannot read the cookie — mitigates XSS-based theft (not XSS itself).

Set these explicitly rather than trusting defaults:

```php
$cookie = Cookie::create('session')
    ->withSecure(true)
    ->withHttpOnly(true)
    ->withSameSite('Lax');
```

### Tokens when cookies aren't enough

For state-changing endpoints, layer tokens on top:

- **Synchronizer token:** the server embeds a random per-session token in each form; submissions must echo it back. The attacker's page can't read it (SOP blocks reading your pages).
- **Double-submit cookie:** the token lives in both a cookie and a request header; the server checks they match. Works well for SPAs because JS can read one copy to place in the header.

### SPA + bearer vs. cookie session

This choice determines whether CSRF is even in scope for your frontend:

- **Bearer token in memory** (Authorization header): the token isn't attached automatically by the browser, so there's nothing to forge. CSRF largely vanishes — but now token storage is your problem (Chapter 5's answer: BFF or in-memory, never localStorage).
- **Cookie session:** automatic attachment means CSRF protection is mandatory. SameSite plus synchronizer/double-submit tokens, always.

Pick one model deliberately. The worst position is accidentally both: cookie session *and* bearer tokens, protecting neither consistently.

## SQL Injection: Parameterization Is the Floor

Injection heads the classic list for good reason: it's trivially exploitable and catastrophic when hit. SQL injection happens when untrusted input becomes executable query structure:

```php
// NEVER: string interpolation into SQL
$q = "SELECT * FROM users WHERE email = '$email'";
// email = "' OR '1'='1" returns every user
// email = "'; DROP TABLE users; --" ends your evening badly
```

The floor — non-negotiable — is **parameterized queries**, where data and code travel separately:

```php
$stmt = $pdo->prepare('SELECT * FROM users WHERE email = ?');
$stmt->execute([$email]);
```

Doctrine parameterizes everything through DQL and the query builder by default. But here's the trap this book keeps returning to: **the ORM is not magic if you concatenate inside it.**

```php
// VULNERABLE despite being "ORM code"
$qb->where("u.email = '" . $email . "'");          // concatenated DQL
$em->createQuery("SELECT u FROM App\Entity\User u WHERE u.name LIKE '%" . $term . "%'");
```

DQL injection is SQL injection wearing a Doctrine badge. The rule survives translation to any layer:

```php
// Correct: bind parameters, always
$qb->where('u.email = :email')->setParameter('email', $email);
```

Two adjacent notes:

- **Ordering and column names can't be parameters** (they're identifiers, not values). Whitelist them against an allowed list instead of interpolating input.
- **Raw queries exist in every stack** — native SQL in Doctrine, `$wpdb` in WordPress plugins, string-built queries in Node. Audit them specifically; they're where parameterization discipline breaks down.

## XSS: Context-Aware Encoding

Cross-Site Scripting injects JavaScript into pages other users view. Unlike CSRF, XSS *defeats* SOP — injected script runs with your origin's full powers: reading DOM, calling APIs with the victim's session, exfiltrating localStorage.

The defense is **context-aware output encoding**: untrusted data must be escaped according to where it lands, because each context has different dangerous characters.

| Context | Danger | Encoding |
|---|---|---|
| HTML body | `<script>`, tag injection | HTML entity encode |
| Attribute value | breaking out of quotes | attribute encode |
| JavaScript string | `</script>`, quote escape | JS-safe serialization |
| URL parameter | `javascript:` schemes | URL encode + scheme allowlist |

Frameworks handle the common case automatically. **Vue escapes all interpolated content by default** — `{{ userInput }}` renders text, never markup. That default is why Vue apps are safer than jQuery-era ones, where manual escaping was everyone's job.

The exception is deliberate: **`v-html` renders raw HTML and bypasses all protection.**

```vue
<!-- Renders raw HTML — every use is a potential XSS -->
<div v-html="reportBody"></div>
```

Policy for this book: `v-html` is a **reviewed exception**, not a tool. Every occurrence needs a PR comment answering: where did this HTML come from? If it's user-generated, sanitize it server-side with an allowlist library (HTMLPurifier on PHP, sanitize-html on Node) before it ever reaches the template. If it's your own trusted markup, say why in the review. Unreviewed `v-html` fails CI review checklist — add it to the grep list alongside secrets.

Related traps worth naming: URLs from user input (`href="{{userUrl}}"` → `javascript:` payloads — validate schemes), and JSON embedded in `<script>` tags without proper encoding.

## SSRF: Outbound Fetching Is an Authorization Problem

Server-Side Request Forgery is listed under Broken Access Control in the 2025 Top 10, and the framing matters: your server becomes a confused deputy making network requests on an attacker's behalf.

The pattern: your app fetches a URL from user input — webhook targets, image imports, "check this link" features. Without controls, attackers point that fetch inward:

- `http://169.254.169.254/latest/meta-data/` — cloud metadata services holding instance credentials
- `http://localhost:6379/` — internal Redis
- `http://10.0.0.0/8` — anything on your private network

Treat outbound URL fetching like an authorization decision:

1. **Allowlist destinations** where possible — known hosts, known schemes (HTTPS only).
2. **Resolve DNS yourself and validate the IP** against blocklists: loopback, private ranges, link-local (that metadata address). Resolve *before* connecting, then connect to the validated IP — otherwise DNS rebinding defeats the check.
3. **Disable redirects or re-validate each hop** — a redirect is a second fetch decision.
4. **Run fetchers with no cloud metadata access** (IMDSv2 with token requirements, or network segmentation) so even a miss has nothing to steal.

If a feature requires fetching arbitrary URLs, that feature needs its own threat model — see the end of this chapter.

## Supply Chain: Lockfiles, Audits, Signed Artifacts

Chapter 2 covered the habits; here's the operational checklist. Modern applications are mostly dependencies, so dependency integrity *is* application integrity.

- **Commit lockfiles** (`composer.lock`, `package-lock.json`) and install with locked mode in CI. Builds must be reproducible — same commit, same bytes.
- **Automate vulnerability scanning:** `composer audit` for PHP, `npm audit` for Node, wired into CI so new CVEs fail builds or open tickets immediately.
- **Pin container images by digest**, not `:latest`. `node:latest` today and next month are different machines; digests make deployments auditable.
- **Sign CI artifacts** (Sigstore/cosign for containers, provenance attestations for builds) so production runs provably-built code.
- **Review new dependencies before adding them:** maintenance activity, download numbers, exact package spelling (typosquatting), and whether the functionality justifies the trust.

## Exceptional Conditions: Fail Closed, Leak Nothing

The last item on the 2025 list is quietly one of the most practical: security depends on handling failure correctly.

**Fail closed.** When an authorization check errors — IdP unreachable, database timeout mid-decision — deny the request. Code that catches exceptions and returns success-by-default turns every outage into an open door:

```php
try {
    $allowed = $policyEngine->decide(...);
} catch (\Throwable $e) {
    throw new AccessDeniedException(); // fail CLOSED, never fall through
}
```

**Leak nothing in errors.** Stack traces, file paths, SQL fragments, and framework versions belong in logs, not responses. Production error handlers return generic messages with a correlation ID; details stay server-side where debuggers live.

**Get status codes right.** Auth-related codes carry information, and using them carelessly leaks it:

- **401 Unauthorized** — "I don't know who you are." Missing/invalid credentials.
- **403 Forbidden** — "I know who you are; you may not do this." Use deliberately — it confirms the resource exists.
- **404 Not Found** — "no such thing." For object-level access checks, prefer 404 over 403: telling an attacker "this invoice exists but isn't yours" is enumeration assistance. Chapter 6's matrix tests assert exactly this.

## Using the Resources: Cheat Sheets and Threat Modeling

### OWASP Cheat Sheets as first lookup

The [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org) is the highest signal-per-minute security resource in software. Need to implement sessions, password storage, JWTs, CSRF, query parameterization, or file uploads? There's a cheat sheet written and reviewed by practitioners, updated as threats evolve.

Make it a team norm: **security questions get answered from cheat sheets first, blog posts second.** Blog posts age; the cheat sheet series has a maintenance process. Bookmark the index; start there.

### Threat modeling one endpoint before writing it

Full-application threat modeling is a project. Per-endpoint threat modeling is ten minutes, and it's the habit that compounds. Before writing a new endpoint, answer four questions (a lightweight STRIDE):

1. **What are we building?** One sentence: "POST /webhooks/stripe accepts payment events."
2. **What can go wrong?** Walk the categories: spoofing (is the signature verified?), tampering (is the payload validated?), repudiation (is it logged?), information disclosure (does the response leak?), denial of service (can one caller flood us?), elevation of privilege (does processing grant anything?).
3. **What will we do about it?** Signature verification, idempotency keys, rate limits — decided *now*, not discovered in review.
4. **Did we do a good job?** Revisit after incidents.

Write the answers in the PR description. Reviewers then verify decisions instead of guessing intent — and the endpoint's security reasoning becomes part of the record, right next to the commit messages from Chapter 2.

---

Security done this way stops being a phase and becomes a property of how you build.

---

# Part III — APIs, Data, and Change

# Chapter 8. API Design That Ages

APIs outlive the code around them. Frontends get rewritten, services get split and merged, but a published endpoint shape tends to persist for years — because someone, somewhere, built against it. Design decisions made casually on day one become permanent taxes on day one thousand.

This chapter covers the API conventions used throughout the book: REST done pragmatically, errors standardized per RFC 9457, pagination chosen deliberately, and OpenAPI as the single source of truth. The running example is our Project Board from Chapters 6 and 7.

## Stateless REST: What "REST" Still Means in Practice

REST has accumulated decades of academic baggage. Strip it to what actually matters for a web API:

- **Resources, not actions.** URLs name *things*, not operations: `POST /boards` creates a board; `DELETE /boards/42` removes it. If your URLs contain verbs (`/getBoards`, `/doArchive`), the design is drifting toward RPC, and you lose HTTP's uniform semantics for free.
- **HTTP verbs carry meaning:**

| Verb | Semantics | Idempotent? | Safe? |
|------|-----------|-------------|-------|
| GET | Read | Yes | Yes |
| PUT | Replace entirely | Yes | No |
| PATCH | Partial update | No (usually) | No |
| DELETE | Remove | Yes | No |
| POST | Create / process | **No** | No |

- **Idempotency** — repeating a request produces the same result. `DELETE /boards/42` called twice deletes once; the second call finds nothing to do and still returns success. This isn't pedantry: retries happen constantly (network blips, queue redelivery, user double-clicks). Idempotent verbs make retries harmless. POST is the exception, which is why it needs the idempotency-key treatment later in this chapter.
- **Statelessness** — every request carries everything needed to process it. No server-side session state that ties requests to a specific backend instance. This is what lets you scale horizontally, retry safely, and reason about failures. (Chapter 5's stateless API firewalls exist precisely for this.)
- **Status codes as the protocol.** 2xx success, 4xx caller's fault, 5xx your fault. Clients branch on these; don't return 200 with an error body.

What "REST" means in practice, then: resource-oriented URLs, correct verbs, correct status codes, stateless requests, and JSON bodies. Hypermedia purism is optional; consistency is not.

## Resource Modeling for the Project Board

Our board has organizations, teams, boards, lists, cards, and members. The modeling questions are always the same: what's a top-level resource, what's a sub-resource, and what's just a field?

**Top-level resources** — things addressed directly and having their own lifecycle:

```
/organizations
/organizations/{orgId}/members
/boards
/boards/{boardId}
/boards/{boardId}/lists
/boards/{boardId}/cards
/cards/{cardId}
```

**When to nest:** nest a sub-resource when it *cannot exist meaningfully outside its parent*. A list belongs to exactly one board — `/boards/{id}/lists` expresses that containment. Nesting also scopes authorization naturally: the route itself says "within this board," which Chapter 6's query filters then verify.

**When to flatten:** nest when the relationship is containment; flatten when the resource stands alone. A card is addressable as `/cards/{cardId}` as well as through its board — deep nesting (`/orgs/1/teams/2/boards/3/lists/4/cards/5`) creates brittle URLs where every ancestor ID must match, for no benefit. Rule of thumb: **one level of nesting communicates ownership; two levels communicate brittleness.**

**What's just a field:** don't create resources for things without independent lifecycle. Card assignment is `PATCH /cards/{id}` with `{"assigneeId": "..."}` — not `/cards/{id}/assignment`. Every resource you add is documentation, validation, and maintenance you now own.

## Error Responses: RFC 9457, Not Invented Shapes

Every API invents its own error format until someone standardizes it for them. Stop inventing. **RFC 9457** ("Problem Details for HTTP APIs") defines a JSON error format — and it supersedes the older RFC 7807, adding fields and media-type registration. Use it everywhere.

A problem details response:

```json
{
  "type": "https://api.example.com/errors/validation",
  "title": "Validation failed",
  "status": 422,
  "detail": "The request body contains invalid values.",
  "instance": "/boards/4721/cards",
  "errors": {
    "title": ["Title must not exceed 120 characters."],
    "dueDate": ["Due date must be in the future."]
  }
}
```

The standard fields:

- **`type`** — URI identifying the error *class*. It can resolve to human-readable docs; it must at least be a stable identifier clients can branch on.
- **`title`** — short, human-readable summary of the class. Same for every occurrence of the type.
- **`status`** — mirrors the HTTP status code. Redundant with the header, but useful when the body is logged or displayed without headers.
- **`detail`** — human-readable explanation of *this specific occurrence*.
- **`instance`** — the request path that produced the problem.
- **Extensions** — anything else you need. Field-level validation errors (above) are the most common extension; add them under your own names.

The payoff: clients write **one** error handler that branches on `type` and displays `detail`. No per-endpoint parsing, no `{"error": "nope"}` archaeology, no guessing whether the message lives in `message`, `error`, or `errors.0.msg`. Content type: `application/problem+json`.

### Symfony

Symfony maps exceptions to problem responses cleanly. With `symfony/serializer` handling the format:

```php
// src/EventListener/ProblemExceptionListener.php
final class ProblemExceptionListener implements EventSubscriberInterface
{
    public function onKernelException(ExceptionEvent $event): void
    {
        $e = $event->getThrowable();
        $status = $e instanceof HttpExceptionInterface ? $e->getStatusCode() : 500;

        $problem = [
            'type' => $this->typeFor($status),       // e.g. .../errors/validation
            'title' => Response::$statusTexts[$status],
            'status' => $status,
            'detail' => $status < 500 ? $e->getMessage() : 'Internal error.',
            'instance' => $event->getRequest()->getPathInfo(),
        ];

        if ($e instanceof ValidationFailedException) {
            $problem['errors'] = $this->fieldErrors($e);
        }

        $response = new JsonResponse($problem, $status, ['Content-Type' => 'application/problem+json']);
        $event->setResponse($response);
    }
}
```

Note the Chapter 7 discipline embedded here: 5xx responses get a generic `detail` — the real message goes to logs with a correlation ID, never to the client.

### Node / Hono

```ts
import { Hono } from 'hono';

const app = new Hono();

function problem(c: Context, status: number, type: string, detail: string, extra: object = {}) {
  return c.json(
    { type: `https://api.example.com/errors/${type}`, title: STATUS_TEXT[status],
      status, detail, instance: new URL(c.req.url).pathname, ...extra },
    status,
    { 'Content-Type': 'application/problem+json' },
  );
}

app.post('/webhooks/stripe', async (c) => {
  const sig = c.req.header('stripe-signature');
  if (!sig || !(await verifySignature(c.req.raw, sig))) {
    return problem(c, 401, 'invalid-signature', 'Webhook signature verification failed.');
  }
  // ...
});
```

Same shape, same discipline, shared contract. Write the helper once per service; never hand-roll a response inline.

## Pagination: Offset vs. Cursor

Every list endpoint needs pagination, and the choice between the two dominant schemes is a real architectural decision.

### Offset pagination

The familiar `?page=3&limit=20`, translating to SQL `LIMIT 20 OFFSET 40`:

- **Pros:** simple, supports random access and "jump to page 47," easy total counts.
- **Cons — and they compound:**
  - **Degrades with depth.** `OFFSET 100000` makes the database walk and discard 100,000 rows. Page 5,000 costs the same as page 1 in work, and returns nothing for it.
  - **Races on writes.** Insert or delete a row between page requests and items shift: the user sees a duplicate or misses a record entirely. With active feeds, offset pagination flickers.

### Cursor pagination

The cursor is an opaque pointer to a position — typically an encoded, sortable key like `created_at` + `id`:

```
GET /boards/4721/cards?limit=20
GET /boards/4721/cards?limit=20&cursor=eyJpZCI6IjQ3MjEtMTIzIn0
```

The query seeks directly: `WHERE (created_at, id) < (:cursor_created, :cursor_id) ORDER BY created_at DESC, id DESC LIMIT 20`. With an index on `(created_at, id)`, this costs the same on page 1 and page 5,000, and inserts/deletes don't shift results.

- **Pros:** stable, constant cost, race-resistant, naturally suits infinite scroll.
- **Cons:** no random access, no jump-to-page, and total counts become expensive (either cache them or drop them from the response).

### The policy

- **Default to cursor for any list that can grow** — feeds, activity logs, card lists, search results. In practice: almost everything.
- **Offset is allowed** for small, bounded, admin-style lists — a settings page with 30 rows, a dropdown of teams — where deep pages and races are structurally impossible.
- **Cursors are opaque.** Clients never parse them. That preserves your freedom to change the encoding (and lets you sign them to prevent tampering with sort keys).

Response envelope for cursors:

```json
{
  "data": [ /* cards */ ],
  "nextCursor": "eyJpZCI6IjQ3MjEtMTAzIn0",
  "hasMore": true
}
```

## Filtering, Sorting, Sparse Fieldsets

Keep these boring. Boring means: conventional syntax, documented in OpenAPI, validated server-side.

- **Filtering:** `?status=open&assignee=carlos`. Whitelist filterable fields (Chapter 6's query filters make this natural — a filter is only allowed if it maps to an indexed, authorized column). Reject unknown filters with a 400 problem response rather than ignoring them silently.
- **Sorting:** `?sort=-created_at,title`. Minus prefix for descending; comma-separated for multi-key. Same whitelist rule — sort columns must exist and be indexed, or every "sort by body text" request becomes a full scan.
- **Sparse fieldsets:** `?fields=title,dueDate` trims responses for bandwidth-sensitive clients. Worth having on large resources, worth skipping on small ones — implement it only when payloads actually hurt.
- **Everything is validated and documented.** The fastest way to make an API age badly is a query-string grammar that's different on every endpoint. One convention, applied everywhere, written down once.

## Versioning and Compatibility Rules

### URL vs. header versioning

- **URL versioning** (`/v2/boards`) — visible, cache-friendly, trivially routable, debuggable in logs and curl. Slightly impure REST (the version isn't really part of the resource), but its operational clarity wins.
- **Header versioning** (`API-Version: 2`) — cleaner URLs, but invisible in logs, easy to misconfigure, and harder to route.

**This book uses URL versioning.** The whole version appears in every URL, every log line, and every bug report. That legibility is worth the aesthetic complaint.

### Compatibility rules

Versioning is the *last* resort, not the workflow. The actual policy:

1. **Additive changes are not breaking.** New optional fields, new endpoints, new query parameters (with safe defaults) — ship freely, no version bump. Write clients that ignore unknown fields; make that a documented contract expectation.
2. **Breaking changes require a new major version.** Removing a field, renaming anything, changing semantics, tightening validation so previously-valid requests fail.
3. **Deprecate loudly and on a schedule.** Old versions get `Deprecation` and `Sunset` headers (RFC 8594), a documented removal date, and monitoring of remaining traffic. Kill a version only when its traffic is zero — you'll be surprised how long a "nobody uses this" version stays alive.
4. **Never break silently.** A field changing meaning in place is worse than removal, because clients depending on it keep "working" — wrongly.

## Idempotency Keys for Dangerous POSTs

POST creates things and triggers side effects — and networks retry. A payment POST that times out and gets retried must not charge twice. **Idempotency keys** solve this: the client sends a unique key per logical operation; the server deduplicates.

```
POST /payments
Idempotency-Key: 7f3c9a2e-4b1d-4e8a-9c2f-a1b2c3d4e5f6
```

Server behavior:

1. Key unseen → process normally, store the response against the key.
2. Key seen, same request body → return the **stored original response** (same status, same body). No double charge, no duplicate board.
3. Key seen, different body → reject with a 409/422 problem response; the client has a bug.

Store keys with a TTL (24 hours is typical) keyed to scope — per client, per endpoint — so keys can't be replayed across contexts. Stripe made this pattern mainstream; adopt it for any POST that moves money, sends email, or creates external side effects. Cheap insurance, trivial to implement, brutal to retrofit after the first double-charge incident.

## OpenAPI as the Contract

**OpenAPI** is a machine-readable description of your API: paths, operations, parameters, schemas, error responses. Treat it as the **source of truth**, and let everything else derive from it.

The workflow this book uses:

1. **Design the endpoint in OpenAPI first** — including the RFC 9457 error responses and pagination envelope.
2. **Review the spec** like code, because it is code: the contract reviewers approve is the spec, not one implementation.
3. **Generate and validate.** Symfony can validate requests/responses against the spec in tests; the Node services validate payloads at the boundary.
4. **Generate the Bruno collection from the spec.** **Bruno** is a git-friendly API client (files live in the repo, reviewable in PRs). Generating its collections *from* OpenAPI keeps manual collections honest — hand-maintained request collections drift from reality within weeks; generated ones can't. Never the other way around.

The alternative — code first, spec "updated later" — reliably produces specs that describe an API that existed six months ago. Spec-first has a cost (writing YAML before PHP), but it forces the design conversation while changing the design is still cheap.

## Worked Examples

### Symfony: cursor-paginated card list

```php
#[Route('/api/v1/boards/{boardId}/cards', methods: ['GET'])]
public function list(string $boardId, Request $request, Security $security): JsonResponse
{
    $limit = min($request->query->getInt('limit', 20), 100);
    $cursor = $request->query->get('cursor');

    $qb = $this->cards->createQueryBuilderForBoard($boardId, $security->getUser()); // authz in query
    $qb->setMaxResults($limit + 1);

    if ($cursor) {
        $c = Cursor::decode($cursor); // {createdAt, id}
        $qb->andWhere('(c.createdAt, c.id) < (:created, :id)')
           ->setParameter('created', $c->createdAt)
           ->setParameter('id', $c->id);
    }

    $rows = $qb->orderBy('c.createdAt', 'DESC')->addOrderBy('c.id', 'DESC')->getQuery()->getResult();
    $hasMore = count($rows) > $limit;
    $rows = array_slice($rows, 0, $limit);

    return $this->json([
        'data' => array_map(CardView::from(...), $rows),
        'nextCursor' => $hasMore ? Cursor::encode(end($rows)) : null,
        'hasMore' => $hasMore,
    ]);
}
```

Authorization lives in the query (`createQueryBuilderForBoard` applies Chapter 6's relationship filter), the cursor is opaque, and the limit is capped — three chapters of policy in fifteen lines.

### Node / Hono: problem+json with idempotency

```ts
app.post('/api/v1/boards/:boardId/cards', async (c) => {
  const idemKey = c.req.header('idempotency-key');
  if (!idemKey) {
    return problem(c, 400, 'missing-idempotency-key',
      'POST requests with side effects require an Idempotency-Key header.');
  }

  const cached = await idempotency.get(c.req.param('boardId'), idemKey);
  if (cached) {
    if (cached.bodyHash !== await hashBody(c.req.raw)) {
      return problem(c, 409, 'idempotency-key-reuse',
        'This key was used with a different request body.');
    }
    return c.body(cached.body, cached.status);
  }

  const card = await createCard(/* validated payload */);
  const response = c.json({ data: card }, 201);
  await idempotency.store(c.req.param('boardId'), idemKey, await hashBody(c.req.raw),
                          await response.clone().text(), 201);
  return response;
});
```

The key is required, reuse with a different body is rejected, and replays return the original response byte-for-byte.

---

A good API is boring in exactly the ways this chapter prescribes: predictable errors, stable pagination, documented parameters, honest versioning. Boring APIs are the ones other people can build against without calling you.

# Chapter 9. MySQL as It Exists Now

MySQL advice on the internet is a sediment of folklore. Half of it was written for 5.5 — a database from 2012 that lacked native JSON, modern character set defaults, and a decade of optimizer improvements. This chapter targets MySQL as it exists now (9.x-era documentation and behavior): native JSON types, sensible defaults, and an optimizer that rewards clean schema design.

The theme throughout: **MySQL is a relational database with a good JSON document type — not a document database with relational features bolted on.** Knowing which tool to reach for inside the database is the skill.

## Types You Should Actually Choose

Schema types are contracts with the storage engine. Choosing well is free; choosing badly costs forever.

### Integers and decimals

- **`INT UNSIGNED` / `BIGINT`** for counters and surrogate keys. Use `BIGINT UNSIGNED` for anything user-generated at scale — changing a column type on a billion-row table is not a fun week.
- **`DECIMAL(12,2)` for money. Always.** Floats and doubles round: `0.1 + 0.2 !== 0.3` in binary floating point, and "the invoice total is off by a cent" is a support ticket generator. DECIMAL stores exact values. Store minor units (cents) in integers if you prefer arithmetic simplicity — but never floats.
- **`BOOLEAN` is `TINYINT(1)`** in MySQL. Fine for flags; just remember comparisons are numeric.

### Datetimes: `DATETIME(6)` vs. `TIMESTAMP`

Both store date-and-time; they differ in ways that matter:

- **`TIMESTAMP`** stores UTC internally and converts on read/write using the session time zone. Range limited to 1970–2038 (the Y2K38 problem). Automatic initialization/update (`DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP`) works well.
- **`DATETIME(6)`** stores the literal wall-clock value with microsecond precision, no time zone conversion, no 2038 ceiling.

Policy for this book: **`DATETIME(6)` storing UTC, with the application responsible for time zone conversion.** Microsecond precision matters more than it seems — it makes ordering deterministic when multiple rows share a second (paired with an id tiebreaker, as in Chapter 8's cursors), and Doctrine maps it cleanly. Use `TIMESTAMP`'s auto-update convenience only when you've consciously accepted the range limit.

### Character sets

The old default `latin1` and the half-broken `utf8` (which is really `utf8mb3` — three bytes, no emoji, no rare CJK) are history. Modern MySQL defaults to **`utf8mb4`**, which stores full Unicode.

Set it explicitly anyway — old databases and old configs linger:

```sql
CREATE TABLE boards (
  ...
) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
```

The `0900_ai_ci` collation (Unicode 9.0, accent-insensitive, case-insensitive) is the modern default and correct for most text. One warning: `utf8mb4` indexes have a 767-byte limit under legacy row formats — irrelevant on modern `DYNAMIC` row format with `innodb_large_prefix` on (the default), but the source of a thousand stale Stack Overflow answers.

### Generated columns

A **generated column** computes its value from an expression, either virtually (computed on read) or stored (materialized on write):

```sql
ALTER TABLE invoices
  ADD COLUMN total_cents INT GENERATED ALWAYS AS (subtotal_cents + tax_cents) STORED;
```

Their killer feature: **you can index them.** Extract a value from JSON into a generated column, index it, and your document data becomes queryable like a column (more below).

## Indexes: The Difference Between Fast and Forensic

### Primary, secondary, composite

- **Primary key** — clustered in InnoDB: the table's rows are physically organized by it. Use a monotonically increasing key (`AUTO_INCREMENT` or UUIDv7/ULID) so inserts append; random UUIDv4 primary keys scatter writes across the B-tree and wreck buffer pool locality.
- **Secondary indexes** — every additional B-tree. Each one slows writes slightly and speeds specific reads enormously.
- **Composite indexes** — the workhorses. Column order follows the **leftmost prefix rule**: an index on `(org_id, status, created_at)` serves queries filtering `org_id`, `org_id + status`, or `org_id + status + created_at` — but *not* `status` alone.

Order composite indexes by: **equality columns first (most selective first), range column last.** The range column can only filter within the prefix before it — putting `created_at` between two equality columns orphans the third column.

```sql
-- Chapter 8's cursor pagination query wants exactly this:
ALTER TABLE cards ADD INDEX idx_board_cursor (board_id, created_at, id);
```

### Covering indexes

When an index contains *all* columns a query needs, the database never touches the table — the index alone answers the query ("Using index" in EXPLAIN). This is the cheapest large performance win in MySQL: extend an existing composite index with one extra column and a hot query drops from random row lookups to a sequential index scan.

### Prefix indexes

For long strings, index a prefix: `INDEX (email(20))`. Saves space, loses selectivity — a prefix that matches many distinct values poorly is nearly useless. Measure with `COUNT(DISTINCT LEFT(col, n))` before committing.

### When an index is a lie

An index that exists but isn't used — or worse, is used badly — is a lie you tell yourself:

- **Wrapping an indexed column in a function** kills it: `WHERE YEAR(created_at) = 2025` can't use the index; `WHERE created_at >= '2025-01-01'` can.
- **Leading wildcards** (`LIKE '%smith'`) can't use the B-tree. Trailing wildcards (`'smith%'`) can.
- **Type mismatches** — comparing an indexed `VARCHAR` to an integer forces implicit conversion and a full scan.
- **Low-selectivity single-column indexes** (`is_deleted`) — the optimizer rightly ignores them; a composite including them may still help.

An unused index is pure write tax. Audit with `sys.schema_unused_indexes` and drop the liars.

## Transactions, Isolation, Deadlocks, Locking Reads

### Isolation levels

MySQL/InnoDB defaults to **`REPEATABLE READ`**: within a transaction, repeated reads see the same snapshot, even as other transactions commit. It prevents dirty and non-repeatable reads; phantom rows are mostly handled by next-key locking.

You'll rarely change the default, but you must know what it implies: long transactions see an increasingly stale world, and "I read it at the start of my transaction" is not "it's still true now." For read-modify-write sequences, a snapshot read isn't enough — see below.

### Deadlocks

Two transactions each hold a lock the other needs. InnoDB detects and kills one (error 1213); the victim must retry. Deadlocks are normal, not exceptional — design for them:

- **Keep transactions short.** The longer a transaction holds locks, the wider the deadlock window.
- **Lock in a consistent order.** If every code path locks rows A→B, deadlock can't happen between them.
- **Retry on 1213.** Wrap transactional work in a retry-with-backoff. An unhandled deadlock error is a self-inflicted outage.

### `SELECT … FOR UPDATE`

The tool for read-modify-write correctness:

```sql
START TRANSACTION;
SELECT balance_cents FROM accounts WHERE id = 7 FOR UPDATE;
-- application computes new balance
UPDATE accounts SET balance_cents = ? WHERE id = 7;
COMMIT;
```

`FOR UPDATE` locks the selected rows until commit, so concurrent transactions queue instead of racing. Use it for any check-then-act sequence on money, inventory, or quota. The alternative — optimistic concurrency with version columns — suits low-contention writes; pick per case, but never ship a bare read-then-write on a financial path.

## Migrations as Reviewed Artifacts

Doctrine (PHP) and similar tools generate migrations; **humans still read the generated SQL before it ships.** A migration is code that runs against production data with no undo button — it deserves the same review rigor as any other change, plus more.

What review looks for:

- **Locking behavior.** `ALTER TABLE` on large tables can lock writes. Modern MySQL does most changes online, but some operations (changing column types, some index restructures) still rebuild. Know the difference; schedule rebuilds; consider `pt-online-schema-change` for the big ones.
- **Reversibility.** Every migration has a `down`. It may never run — but writing it forces you to understand what you're changing.
- **Backfills done safely.** Adding a `NOT NULL` column to a million-row table needs a default, and backfilling a million rows should happen in batches, not one transaction.
- **Deploy ordering.** Chapter 8's compatibility rules apply to schema too: ship additive changes first, deploy the code that uses them, remove old columns later. A migration that drops a column the running code still reads is an outage in a deploy script.

## JSON as a Data Type

MySQL's native `JSON` type validates on write, stores in an efficient binary format, and supports indexing through generated columns. It's genuinely useful — and genuinely easy to misuse.

### The mechanics

```sql
CREATE TABLE webhook_events (
  id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  provider    VARCHAR(32) NOT NULL,
  event_type  VARCHAR(64) NOT NULL,
  payload     JSON NOT NULL,
  received_at DATETIME(6) NOT NULL,
  -- Extract + index the fields we actually query:
  external_id VARCHAR(64) GENERATED ALWAYS AS (JSON_UNQUOTE(JSON_EXTRACT(payload, '$.id'))) STORED,
  INDEX idx_external (provider, external_id)
);
```

- **Validation:** invalid JSON is rejected at write time. The column cannot hold garbage.
- **`JSON_TABLE`:** turns a JSON array into rows for relational querying:

```sql
SELECT e.id, t.item_id, t.qty
FROM webhook_events e,
JSON_TABLE(e.payload, '$.line_items[*]'
  COLUMNS (item_id VARCHAR(32) PATH '$.sku', qty INT PATH '$.quantity')) AS t;
```

- **Partial updates:** `JSON_SET`, `JSON_REPLACE`, `JSON_REMOVE` modify paths in place — no read-modify-write of the whole document:

```sql
UPDATE webhook_events
SET payload = JSON_SET(payload, '$.status', 'processed')
WHERE id = 42;
```

- **Functional indexes:** MySQL 8.0.13+ lets you index expressions directly, or use the generated-column pattern above. Both make "query inside JSON" fast — but the fields you index should be few and stable.

### Legitimate uses

- **Genuinely schemaless payloads** — webhook bodies from Stripe, provider API responses you store verbatim for audit. The shape belongs to the vendor, changes on their schedule, and you query it rarely.
- **Feature-specific documents** — a form builder's saved form definition, a dashboard's widget layout. Self-contained, read-and-rendered-whole, owned by one feature.
- **Sparse optional attributes** — a product catalog where 200 attributes exist and each product uses 15. Modeling 200 columns is worse; an EAV table is much worse.

The common thread: **the JSON is written and read as a unit, by one feature, with the schema owned by something outside your control.**

### Abuse we will not ship

- **Using JSON to avoid modeling.** "We don't know the schema yet" is a reason to design carefully, not to defer design into a blob. If you're writing `WHERE JSON_EXTRACT(...) = ...` on day one, you have a schema — make it columns.
- **Querying JSON as if it were relational.** Joins across JSON paths, filtering large tables by JSON contents, aggregating over extracted fields — this is a relational workload in a document costume. It works until the table grows, then it's a forensic exercise in why nothing has an index.
- **Stuffing relations into documents "just for now."** An order document with 40 embedded line items becomes unqueryable ("which orders contain product X?"), unenforceable (no foreign keys inside JSON), and unfixable. The "just for now" becomes the permanent architecture.

The test: **if you need to query, join, constrain, or migrate the data relationally, it belongs in columns and tables.** JSON is for the rest.

## EXPLAIN, Slow Query Log, and How an ORM Hides a Table Scan

### EXPLAIN

Prefix any suspicious query with `EXPLAIN` (or `EXPLAIN ANALYZE` for actual execution stats):

```sql
EXPLAIN ANALYZE SELECT * FROM cards WHERE board_id = 4721 AND status = 'open' ORDER BY created_at DESC;
```

Read it for:

- **`type`:** `const`/`eq_ref` (great), `ref`/`range` (fine), `index` (scanning a whole index), `ALL` (full table scan — investigate).
- **`key` / `rows`:** which index was used, and how many rows the estimate covers. Estimates in the millions for a query returning 20 rows mean a missing or mis-ordered index.
- **`Extra`:** `Using index` (covering — good), `Using filesort` (sort not served by index — check ORDER BY against your composite index), `Using temporary` (usually bad).

### Slow query log

The slow query log records queries exceeding a threshold, with the plan:

```sql
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 0.2;      -- 200ms is a reasonable starting bar
SET GLOBAL log_queries_not_using_indexes = ON;
```

Pipe it into your observability stack and review weekly. The log finds the queries *users* feel, not just the ones you thought to test.

### How an ORM hides a table scan

Doctrine (and every ORM) adds distance between your intent and the SQL. The classic failures:

- **The N+1 query:** loading 50 cards then accessing `$card->getAssignee()` triggers 50 extra queries. Fix with joins or `Doctrine\ORM\Query::setFetchMode` / entity graph hints — and *detect* it in dev with `doctrine.dbal.logging` or tools that count queries per request. A page issuing 200 queries is a design bug wearing a performance costume.
- **Invisible full scans:** a DQL filter that looks innocent can compile to a non-indexable expression (functions on columns, JSON_EXTRACT without a functional index, `LIKE '%x'`).
- **Implicit lazy loading in templates:** Twig iterating a collection quietly triggers queries mid-render, where nobody profiles.

The discipline: **log the SQL in development, and read it.** Every ORM feature request should end with "and here's the query it produces" — if you can't say what SQL runs, you can't tune it, and you can't review it for the injection patterns from Chapter 7.

## Backup, Restore, and What "We Have Replicas" Does Not Mean

Backups are the least glamorous topic in this book and the one that decides whether an incident is an outage or a company-ending event.

### Backups

- **Full + incremental (binlog-based) backups**, tested on a schedule. Tools: `mysqldump` for small databases, physical backups (Percona XtraBackup) for real ones.
- **Encrypted, off-site, access-controlled.** A backup on the same server as the database protects against nothing — ransomware encrypts both. A backup readable by a compromised app credential is a data leak waiting to be found.
- **Point-in-time recovery (PITR):** full backup + binary logs lets you restore to any moment — essential when the failure is "someone ran the wrong DELETE at 14:32," not "the disk died."

### The test that matters

An unverified backup is a hope, not a backup. **Schedule restore drills:** restore last night's backup into a scratch instance, run the app against it, measure how long it took. The first drill always finds something — missing credentials, a schema dependency nobody documented, a 6-hour restore time you thought was 20 minutes. Your real RTO (recovery time objective) is what the drill measures, not what the runbook claims.

### What "we have replicas" does and does not mean

Replication is not backup, and conflating them is the classic infrastructure error:

- **What replicas give you:** read scaling, and fast failover when the primary *dies cleanly*.
- **What replicas do not protect against:**
  - **Replicated mistakes.** `DROP TABLE` on the primary replicates to every replica, instantly and faithfully. Only a backup (or delayed replica) saves you.
  - **Replication lag.** A replica may be seconds behind; "it committed" is not "every read sees it." Design reads that need consistency to hit the primary.
  - **Silent divergence.** A replica that stopped replicating looks healthy in your load balancer while serving stale data. Monitor `Seconds_Behind_Source` and replication status as production alerts, not dashboards you glance at.
  - **Corruption.** A corrupted primary happily replicates corruption.

The rule: **replicas are for availability and read scaling; backups are for recovery.** You need both, and neither substitutes for the other.

---

The data layer is where performance and correctness quietly live or die.

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
