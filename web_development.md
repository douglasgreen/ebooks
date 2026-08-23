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

# Chapter 10. Feature Flags and Rolling Releases

Deploying code and releasing features used to be the same event — which meant every deploy was a gamble on everything in it. Feature flags break that coupling: **deploy the code dark, then turn it on deliberately**, for one user or everyone, with a switch that takes seconds instead of a rollback.

That power comes with a cost most teams discover too late: flags are inventory, and inventory accumulates. This chapter covers how to use flags well and how to stop them from burying you.

## Fowler's Split: Four Flag Types

Martin Fowler's taxonomy remains the clearest way to think about flags, because each type has a different owner, lifespan, and risk profile:

- **Release flags** — hide incomplete features from production while their code deploys continuously. Lifespan: days to a few weeks. Owner: the feature team. Removed when the feature ships fully.
- **Experiment flags** — A/B testing: route cohorts to variants and measure outcomes. Lifespan: the experiment's duration, then a decision (ship variant, remove flag). Owner: product/data.
- **Ops flags** — kill switches for operational control: disable an expensive search path when the database melts, turn off a flaky third-party integration. Lifespan: *long-lived by design*. Owner: operations/on-call.
- **Permission flags** — gate features per user or account tier ("premium users only"). These blur into entitlements; if they're permanent product behavior, consider whether they belong in your authorization model (Chapter 6) rather than a flag system.

The classification matters because it dictates cleanup policy. A release flag without a removal date is a bug in the ticket. An ops flag without monitoring is a landmine. Mixing types in one undifferentiated pile is how systems end up with 400 flags and nobody brave enough to delete any.

**Flags are inventory; unused flags are debt.** Every flag adds branches through your code that must be tested, reasoned about, and eventually removed. The chapter-end cleanup policy makes this concrete.

## Evaluation Points: Where Flags Get Checked

A flag can be evaluated at several layers, each with different tradeoffs:

- **Symfony voter / service** — server-side evaluation at request time. The authoritative point for anything security- or data-relevant. The flag service is injected like any dependency; decisions appear in logs alongside authorization decisions.
- **Vue computed** — client-side evaluation for UI presentation: hiding menu items, switching layouts. Fast and reactive, but remember Chapter 6's rule: this is UX, not enforcement. The API behind the hidden button still enforces independently.
- **Edge / CDN worker** — evaluation before the request reaches your origin. Useful for routing whole traffic cohorts, serving different static bundles, or blocking a broken feature before it consumes backend capacity. Highest latency win, least context.

The rule of thumb: **evaluate as close to the decision as possible, but never let a lower layer be the only check.** If a feature must not run, the Symfony service must refuse it — regardless of what the SPA decided to render.

## Consistency Across PHP, Node, and the SPA

Our stack evaluates flags in three places: Symfony controllers, Node services, and Vue components. Inconsistency here produces maddening bugs: the SPA shows the new checkout, but the PHP API rejects it because it evaluated the flag differently.

Consistency requires three things:

1. **One source of truth.** A single flag store (a service like LaunchDarkly/Unleash, or a self-hosted config table + Redis cache) that all three layers read. No parallel "frontend flags" JSON that drifts from the backend.
2. **One evaluation contract.** Same actor identity, same flag key, same targeting rules → same answer. That means the actor ID passed from the SPA must match the authenticated principal the backend sees. If the SPA evaluates flags for `user-123` while the API authenticates `user_123`, you have two systems.
3. **Bounded staleness.** Flag values propagate within a known window (poll interval, streaming update). Document it. When someone asks "why did half my session see the old UI?", the answer should be "the propagation window is 60 seconds," not a mystery.

Practical pattern: the backend resolves flags during bootstrap and ships the resolved set to the SPA in the initial payload (as with permissions in Chapter 6). The SPA uses those values for rendering; any mutation re-checks server-side. One evaluation, two consumers, no drift.

## Rolling Release: Canary, Percentage, Cohort, Kill Switch

Flags enable graduated delivery — shipping to increasing circles of confidence:

- **Canary:** enable for internal accounts first. The team eats its own cooking under production load. Cheap, fast signal.
- **Percentage:** enable for a random slice of traffic — 1%, 5%, 25%. Watch error rates, latency, and business metrics at each step. Random assignment keeps cohorts comparable; sticky assignment (hash of user ID) keeps individuals' experience consistent across requests.
- **Cohort:** enable for a defined group — beta opt-ins, one organization, free-tier accounts. Better than percentage when feedback loops matter more than statistical power; you can actually talk to the affected users.
- **Kill switch:** the ops flag that turns the feature off instantly when something breaks. Every risky feature ships with one, tested *before* launch — a kill switch you've never flipped is a hypothesis, not a control.

The discipline that ties these together: **define success and failure criteria before enabling.** "We'll watch error rates" is not criteria. "Roll to 25% only if p95 latency stays under 300ms and checkout conversion doesn't drop more than 2%" is. Without pre-committed thresholds, rollouts stall at 5% forever because nobody can prove it's safe to proceed — or worse, they proceed on vibes.

## Flags in Tests: Deterministic Evaluation, No Flag-Shaped Ifs

Flags interact badly with tests in two ways, both fixable with discipline.

First, **flaky tests**: if test outcomes depend on live flag state, your suite breaks whenever someone flips a flag. Fix: tests evaluate against a fixed, injected flag configuration. The flag service is just another dependency — mock it deterministically, and cover both branches explicitly where behavior differs.

Second, and deeper: **flag-shaped ifs leaking into the domain.** This is the failure mode where flags rot your architecture:

```php
// Domain method riddled with flag awareness
public function refund(Invoice $invoice): void
{
    if ($this->flags->isEnabled('new-refund-flow', $this->actor)) {
        // new logic
    } else {
        // old logic
    }
}
```

Now the domain layer depends on infrastructure, every test needs flag setup, and removing the flag later means surgery across the codebase. The fix is architectural: **keep flags at the edges.**

```php
// Composition root chooses the implementation; domain stays clean
$refunder = $flags->isEnabled('new-refund-flow')
    ? new V2Refunder($gateway)
    : new LegacyRefunder($gateway);
```

The domain interface (`Refunder`) never knows a flag exists. Both implementations exist as real classes with real tests. Removing the flag means deleting one class and one line of wiring — not excavating conditionals from forty files. Rule: **flags select between implementations at the boundary; they don't branch inside the core.**

## Cleanup Policy and Ownership

Every flag gets an owner and an expiration at creation time — recorded in the flag metadata itself:

- **Release flags:** owner is the feature team; expiry date set at creation (e.g., 30 days). CI runs a report of expired flags; expired flags block the release checklist until removed or explicitly extended.
- **Experiment flags:** expiry is the experiment end date. The decision (ship/remove) is a tracked task, not a memory.
- **Ops flags:** permanent by design, but reviewed quarterly — is the condition it guards still real? Is it monitored? Does anyone remember how to use it?
- **Permission flags:** audit annually against the entitlement model; migrate to proper authorization when they stabilize.

Removal is part of the feature's definition of done, not a someday task. The mechanical work is small if flags stayed at the edges: delete the wiring line, delete the dead implementation, delete the flag record. Teams that skip this accumulate hundreds of zombie flags until the flag system itself becomes untrustworthy — and once people stop believing flags reflect reality, they stop using them, and you've lost the capability entirely.

## Interaction with CI/CD and Observability

### With CI/CD (Chapter 22)

Flags change what "deploy" means, and pipelines should encode that:

- **Tests run against both flag states** for flagged paths — matrix the critical combinations rather than hoping.
- **Deploys are flag-neutral.** Code ships with the flag off; enabling is a separate, reversible action. Rollback of a bad feature becomes "flip the flag" (seconds) instead of "revert the deploy" (minutes, plus migration questions).
- **Flag changes get their own audit trail.** Who enabled what, when, for whom — logged like any production change, because a flag flip *is* a production change.
- **Schema migrations stay decoupled from flags.** Ship additive migrations first (Chapter 9), then the flagged code, then remove old paths after full rollout. Never make a flag flip require a simultaneous migration.

### With Observability (Chapter 21)

Flags without telemetry are superstition:

- **Tag metrics and traces with active flag states.** When latency spikes, the dashboard should show "checkout-v2 enabled for 25% of traffic" next to the graph — otherwise every incident starts with fifteen minutes of "did we change anything?"
- **Log flag evaluations** for significant decisions (feature enabled/denied for actor X), sampled sensibly.
- **Per-cohort dashboards during rollout:** error rate, latency, and conversion split by flag state. Divergence between cohorts is your earliest warning.
- **Alert on rollout health automatically:** if the 5% cohort's error rate exceeds baseline by the pre-committed threshold, page — don't wait for a human to notice a graph.

---

Feature flags, used with this discipline, convert releases from cliff-edge events into dial turns.

---

# Part IV — Platform

# Chapter 11. Docker and Docker Compose

"It works on my machine" is a symptom of an environment problem, not a developer problem. Docker solves it by making the machine a build artifact: a file that describes your runtime exactly, versioned in git, identical on every laptop and every server.

This chapter builds the mental model (images, containers, volumes, networks), then assembles the full Project Board stack — PHP, NGINX, MySQL, Redis, Node workers — into a Compose setup you'll use for the rest of the book.

## The Four Nouns

Everything in Docker reduces to four concepts. Get these straight and the rest is syntax:

- **Image** — an immutable template: filesystem layers plus metadata (entrypoint, env defaults). Built from a `Dockerfile`. Images are named and tagged (`php:8.3-fpm-alpine`) and stored in registries.
- **Container** — a running instance of an image. Start the same image ten times and you get ten isolated containers sharing nothing but the image layers. Containers are ephemeral by design: destroy and recreate them freely.
- **Volume** — persistent storage managed by Docker, living outside the container's lifecycle. Delete the container; the volume survives. This is how MySQL data outlives `docker compose down`.
- **Network** — an isolated virtual network containers join. On a user-defined network, containers reach each other **by service name**: your PHP container connects to host `mysql` port 3306, and DNS just works. No IP addresses, no port juggling between services.

The layering matters: images are built once and cached, containers are disposable, volumes persist data, networks provide names. Most beginner confusion comes from blurring image and container ("I deleted the container, why is my database empty?" — because the data was in the container's writable layer or you never made a volume).

## Multi-Stage Builds, Non-Root Users, Distroless

### Multi-stage builds

A production image should not contain compilers, package managers' caches, test suites, or dev dependencies. **Multi-stage builds** solve this: compile in one stage, copy only artifacts into a slim final stage.

PHP-FPM with Composer:

```dockerfile
# ---- Stage 1: vendor dependencies ----
FROM composer:2 AS vendor
WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install --no-dev --no-scripts --no-autoloader
COPY src/ src/
RUN composer dump-autoload --optimize

# ---- Stage 2: final runtime ----
FROM php:8.3-fpm-alpine
RUN docker-php-ext-install pdo_mysql opcache \
    && apk add --no-cache icu-libs
WORKDIR /app
COPY --from=vendor /app/vendor/ vendor/
COPY public/ public/
COPY src/ src/
COPY config/ config/

# Non-root user
RUN addgroup -S app && adduser -S app -G app
USER app

EXPOSE 9000
CMD ["php-fpm"]
```

Node worker:

```dockerfile
# ---- Stage 1: build ----
FROM node:22-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY tsconfig.json ./
COPY src/ src/
RUN npm run build && npm prune --omit=dev

# ---- Stage 2: runtime ----
FROM node:22-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/node_modules node_modules
COPY --from=build /app/dist dist
USER node
CMD ["node", "dist/worker.js"]
```

Key details:

- **`npm ci`, not `npm install`** — respects the lockfile exactly (Chapter 2's supply-chain rule).
- **Copy lockfiles before source.** Docker caches per-layer; copying `package.json` + lockfile first means dependency layers cache across code changes. Reversing the order rebuilds all dependencies on every edit.
- **`--no-dev` / `npm prune --omit=dev`** keeps PHPUnit, TypeScript, and friends out of production images — smaller attack surface, smaller pull size.

### Non-root users

Containers default to root, which means a container escape runs as root on the host. Both examples above switch to a dedicated unprivileged user (`USER app`, `USER node`). It costs two lines. Do it always.

### Distroless where it pays

**Distroless** images contain only your runtime — no shell, no package manager, no utilities. Nothing to exploit, nothing to patch. They pay off most for compiled/static runtimes (Go binaries especially). For PHP-FPM and Node, official slim/alpine images with a non-root user are the pragmatic choice; full distroless is awkward when your runtime expects a shell for entrypoint scripts. Know the concept, apply it where it fits.

## Compose for the Whole Project Board

Docker Compose describes a multi-container application in one YAML file:

```yaml
services:
  nginx:
    image: nginx:1.27-alpine
    ports:
      - "8080:80"
    volumes:
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      - ./public:/var/www/public:ro
    depends_on:
      php:
        condition: service_healthy

  php:
    build:
      context: .
      dockerfile: docker/php/Dockerfile
    volumes:
      - ./:/app
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD-SHELL", "php-fpm-healthcheck || kill -0 1"]
      interval: 5s
      timeout: 3s
      retries: 5

  mysql:
    image: mysql:9.0
    environment:
      MYSQL_DATABASE: projectboard
      MYSQL_USER: app
      MYSQL_PASSWORD_FILE: /run/secrets/db_password
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/db_root_password
    secrets:
      - db_password
      - db_root_password
    volumes:
      - mysql-data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-p$$(cat /run/secrets/db_root_password)"]
      interval: 5s
      timeout: 3s
      retries: 10

  redis:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - redis-data:/data

  node-worker:
    build:
      context: ./services/worker
    volumes:
      - ./services/worker/src:/app/src
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_started

  mailhog:
    image: mailhog/mailhog
    ports:
      - "8025:8025"

volumes:
  mysql-data:
  redis-data:

secrets:
  db_password:
    file: ./secrets/db_password.txt
  db_root_password:
    file: ./secrets/db_root_password.txt
```

One `docker compose up` brings up the entire system. Note what's *not* here: no installed local MySQL, no PHP version conflicts, no "works after reinstalling Node." New team members clone, run one command, and have the exact stack.

## Bind Mounts vs. Named Volumes

Two ways to give containers persistent or shared storage:

- **Bind mounts** map a host directory into the container (`./src:/app/src`). Edits on the host appear instantly in the container — essential for development, where you want live reload of PHP and hot rebuilds of Node without rebuilding images.
- **Named volumes** are Docker-managed storage (`mysql-data:/var/lib/mysql`). Docker owns location and lifecycle; performance is consistent; contents survive container recreation. Essential for databases.

The rule: **bind mounts for source code in development, named volumes for data everywhere.**

### File permissions between host and PHP

The classic Linux pain point: PHP-FPM inside the container runs as user `app` (uid 1000), but files bind-mounted from your host belong to *your* uid — which may be 1000 (fine) or something else entirely (permission denied everywhere).

Practical mitigations:

- **Match UIDs deliberately.** Set the container user's uid to match the typical host user (`adduser -u 1000`), or document the expected host uid.
- **On macOS and Windows (Docker Desktop), this mostly doesn't matter** — the file-sharing layer fakes permissions permissively. The problem is native Linux hosts.
- **Never chmod 777 as a fix.** It works, and it teaches everyone that permissions are decorative. Fix the uid mapping instead.
- **Cache directories** (`var/cache`, `var/log`) can be named volumes instead of bind mounts — the container writes there exclusively, so ownership stays internal.

## Healthchecks and Why "It Started" ≠ "It Is Ready"

A process starting and a service being ready are different events. MySQL takes seconds to initialize its data directory; during that window the process exists and connections fail. NGINX starts instantly but returns 502s until PHP-FPM accepts connections.

Compose's `healthcheck` defines readiness as a repeated probe: the example above pings MySQL every 5 seconds until it answers. Then `depends_on` with conditions makes startup ordering real:

```yaml
depends_on:
  mysql:
    condition: service_healthy   # wait until actually ready
redis:
  condition: service_started     # merely started is fine here
```

Plain `depends_on` (without conditions) only orders *start* — it does not wait for readiness. That's why half of all "connection refused" first-run experiences happen: the app container started before the database could accept connections.

Design principle beyond Compose: **every long-initializing service should expose a healthcheck**, and everything depending on it should gate on health, not existence. This habit carries directly into Kubernetes later, where readiness probes are mandatory infrastructure rather than a nice-to-have.

## Secrets: Not in the Image, Not in Git

Credentials end up in images and repos through carelessness, and both leak permanently (Chapter 7's supply-chain rules apply). The rules:

- **Never bake secrets into images.** An `ENV MYSQL_ROOT_PASSWORD=hunter2` line in a Dockerfile is readable by anyone who pulls the image — forever, in every layer history.
- **Never commit `.env` files with real values.** Commit `.env.example` with placeholder keys (Chapter 2). Add `.env` to `.gitignore`.
- **Use Compose secrets** for anything sensitive, as shown above: values live in files outside git, mounted at `/run/secrets/*` inside the container, never appearing in `docker inspect` output or process listings the way environment variables do.
- **For development convenience,** a committed `.env.example` plus a locally created `.env` (gitignored) holding dev-only passwords is acceptable — development credentials should still be *real enough* that nobody reuses production passwords out of habit.

The deeper habit: treat every credential as if it will leak eventually (Chapter 5's rotation discipline), and make leaking cheap to recover — which means secrets must be replaceable without rebuilding images or editing committed files.

---

With the full stack running locally in one command, we're equipped to develop against the real system.

# Chapter 12. NGINX as the Front Door

Every request to your application passes through NGINX first — which makes it simultaneously your router, your TLS guard, your static file server, and your first line of rate limiting. It's also the component teams most often configure by copy-paste, inheriting settings nobody understands.

This chapter builds the front door deliberately: what each role does, how PHP-FPM and SPA routing actually work through it, and a reviewed configuration you can defend line by line.

## Three Roles in One Process

NGINX wears three hats, and knowing which one is active at any moment clarifies every config decision:

- **Reverse proxy** — receives client requests and forwards them to backend services (PHP-FPM, Node). The client never talks to your application directly; NGINX decides who handles what.
- **Static server** — serves files from disk directly: compiled Vue bundles, images, fonts. This is dramatically faster than letting any application process touch them, and it's why Chapter 1's request lifetime has NGINX intercepting asset requests before they ever reach PHP.
- **TLS terminator** — decrypts HTTPS at the edge. Backends receive plain HTTP over the internal network; certificates live in exactly one place.

One process, all three roles, dispatched per request by `location` blocks.

## PHP-FPM: `fastcgi_pass`, `try_files`, and the Front Controller

### The front-controller pattern

Symfony (like most modern frameworks) routes *everything* through one entry point: `public/index.php`. There are no "pages" on disk — there's one PHP script that reads the URL and dispatches internally. NGINX's job is to make that happen without exposing anything else.

The canonical block:

```nginx
location / {
    try_files $uri $uri/ /index.php$is_args$args;
}

location ~ ^/index\.php(/|$) {
    fastcgi_pass php:9000;
    fastcgi_split_path_info ^(.+\.php)(/.*)$;
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
    fastcgi_param DOCUMENT_ROOT $realpath_root;
    internal;
}
```

Reading it:

- **`try_files $uri $uri/ /index.php`** — try the requested path as a literal file, then as a directory, then fall back to the front controller with original query args (`$is_args$args`) intact. This single directive implements pretty URLs.
- **`fastcgi_pass php:9000`** — forward to the FastCGI protocol, not HTTP. In Compose, `php` resolves via the network DNS from Chapter 11.
- **`internal;`** — this location can't be requested directly. Only internal rewrites (from `try_files`) reach it. Without it, clever URLs can sometimes probe the handler directly.
- **`$realpath_root`** — resolves symlinks before building `SCRIPT_FILENAME`. Combined with the next trick, this closes a classic exploit class.

### The security detail that matters

A naive config lets attackers execute *uploaded* or *cached* PHP files by requesting their paths — because any `.php` match gets passed to FPM. Two defenses:

1. Match only the known front controller (as above), never a broad `location ~ \.php$`.
2. Add `cgi.fix_pathinfo=0` in `php.ini`, so FPM doesn't guess at path resolution.

If your config contains `location ~ \.php$ { ... }` with no other constraints, stop and fix it today.

## SPA Fallback and Its Interaction with API Routes

Vue SPAs need the opposite fallback: since routing happens client-side, *any* unknown path should serve the app shell so the router can take over:

```nginx
location / {
    root /var/www/public;
    try_files $uri $uri/ /index.html;
}
```

But an app that serves both an SPA and an API must partition traffic carefully — otherwise `/api/orders` falls back to `index.html` and returns HTML where JSON was expected (a bug that produces baffling "Unexpected token '<'" errors in the browser console).

Structure it explicitly:

```nginx
# API goes to PHP
location /api/ {
    try_files $uri /index.php$is_args$args;
}

# Webhooks go to Node
location /hooks/ {
    proxy_pass http://node-hooks:3000;
    # ...
}

# Everything else: SPA assets + fallback
location / {
    try_files $uri $uri/ /index.html;
}
```

Rules of thumb:

- **API prefixes win by specificity** — declare them before the catch-all and keep the catch-all last.
- **Never let hashed asset paths collide with API paths.** Serving built assets under `/assets/` keeps the namespaces clean.
- **Cache immutable assets aggressively:** hashed filenames (`app.a3f9c2.js`) mean content never changes, so cache forever:

```nginx
location /assets/ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}
```

## Compression, Caching Headers, Forwarded Headers, WebSockets

### Gzip and brotli

Compress text responses — HTML, CSS, JS, JSON, SVG. Brotli compresses better than gzip but costs more CPU; common practice is brotli for pre-compressed static assets, gzip for dynamic responses:

```nginx
gzip on;
gzip_types text/plain text/css application/json application/javascript image/svg+xml;
gzip_min_length 1024;
gzip_vary on;   # tells caches the response varies by Accept-Encoding
```

Never compress already-compressed formats (images, video) — wasted CPU, zero gain.

### Caching headers

Chapter 3's browser policy lands here. Beyond immutable assets:

- **HTML/app shell:** short cache or none (`no-cache` lets the browser revalidate), so deploys propagate quickly.
- **API responses:** default to no-store unless deliberately cacheable; when cacheable, use explicit `Cache-Control` with `max-age` and consider `Vary` carefully.
- **Static media:** long max-age with content-hashed names.

### `X-Forwarded-*`

Behind a reverse proxy, your application sees NGINX's IP, not the client's. Forward the truth:

```nginx
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header Host $host;
```

Then tell Symfony to trust only NGINX (`framework.trusted_proxies` set to the internal network) — trusting these headers blindly lets clients spoof their own IP, which breaks rate limiting and audit logs. Same story in Node: Express needs `app.set('trust proxy', ...)`, scoped precisely.

### WebSocket upgrades

Vite's dev HMR and real-time dashboards speak WebSocket, which starts life as an HTTP request with an `Upgrade` header. Proxied connections need the upgrade explicitly forwarded:

```nginx
location /ws/ {
    proxy_pass http://node-realtime:3000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 3600s;   # idle sockets otherwise die at 60s
}
```

Without `proxy_http_version 1.1` and the Upgrade headers, the handshake fails; without the read timeout bump, long-lived sockets drop after a minute of silence.

## Rate Limiting, Request Size, Timeouts

These three settings are cheap insurance against accidental and deliberate overload:

```nginx
# Rate limit: 10 requests/sec per client IP, with a burst allowance
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

server {
    # Reject oversized bodies early — protects PHP too (post_max_size)
    client_max_body_size 10m;

    location /api/ {
        limit_req zone=api burst=20 nodelay;
        limit_req_status 429;

        proxy_connect_timeout 5s;
        proxy_send_timeout    30s;
        proxy_read_timeout    30s;
    }
}
```

- **Rate limiting** uses token buckets keyed by IP (or a header if you're behind another proxy). Return **429**, not 503, so well-behaved clients know to back off. Tune per endpoint class: webhooks from Stripe arrive in bursts and deserve different limits than login attempts — and login endpoints deserve *stricter* ones (brute-force defense).
- **Request size caps** blunt upload-based DoS. Set them at both layers (NGINX and PHP's `post_max_size`/`upload_max_filesize`) consistently.
- **Timeouts** prevent slow-loris-style resource exhaustion and stop hung backends from pinning workers. Connect/send/read timeouts should reflect realistic worst-case backend behavior — a queue-backed export endpoint may legitimately take minutes and belongs on its own location block with its own timeouts.

## Security Headers

Headers are your cheapest defense-in-depth. Set them once at the edge:

```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
add_header Content-Security-Policy "default-src 'self'; ..." always;
```

What each does:

- **HSTS** — forces HTTPS for future visits; browsers refuse plain HTTP after seeing it. Start with a modest `max-age`, raise it once confident. Include subdomains only when *all* subdomains are HTTPS-ready — HSTS is hard to undo for visitors who've cached it.
- **Content-Security-Policy** — the strongest anti-XSS control available: an allowlist of where scripts, styles, and connections may come from. A strict CSP (`default-src 'self'`, no `unsafe-inline`) blocks injected script even when an XSS slips past escaping. Roll out with `Content-Security-Policy-Report-Only` first, tighten iteratively, enforce when violations hit zero. For a Vue SPA with API calls, expect something like `default-src 'self'; connect-src 'self' https://api.example.com; img-src 'self' data:; style-src 'self'`.
- **`X-Content-Type-Options: nosniff`** — stops browsers from guessing content types (prevents uploaded "images" executing as scripts).
- **Referrer-Policy** — controls how much URL leaks to other sites via the Referer header; `strict-origin-when-cross-origin` is the sane default.
- **Permissions-Policy** — disables powerful browser APIs you don't use (camera, microphone), shrinking the attack surface if XSS occurs.

Two gotchas: `add_header` directives inside a `location` block *replace* inherited server-level ones (repeat what you need), and `always` ensures headers appear on error responses too — security headers on 404s matter as much as on 200s.

## Reviewed Configurations

### Local development (`docker/nginx/default.conf`)

Optimized for iteration: no TLS (the Compose network is trusted), verbose errors, Vite HMR proxied:

```nginx
server {
    listen 80;
    server_name localhost;
    root /var/www/public;
    index index.php index.html;

    # Symfony API
    location /api/ {
        try_files $uri /index.php$is_args$args;
    }

    location ~ ^/index\.php(/|$) {
        fastcgi_pass php:9000;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        fastcgi_param DOCUMENT_ROOT $realpath_root;
        internal;
    }

    # Vite dev server (HMR websocket included)
    location /vite-dev/ {
        proxy_pass http://vite:5173/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    # Built SPA assets
    location /assets/ {
        expires 1h;
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### Production variant

Adds TLS, security headers, compression, rate limits, and hides version details:

```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;      # force TLS
}

server {
    listen 443 ssl http2;
    server_name example.com;
    root /var/www/public;

    ssl_certificate     /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    # Security headers (repeated fully — add_header doesn't merge)
    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy "default-src 'self'; connect-src 'self'; img-src 'self' data:; style-src 'self'" always;

    gzip on;
    gzip_types text/css application/javascript application/json image/svg+xml;
    client_max_body_size 10m;
    server_tokens off;

    location /api/ {
        limit_req zone=api burst=20 nodelay;
        limit_req_status 429;
        try_files $uri /index.php$is_args$args;
    }

    location ~ ^/index\.php(/|$) {
        include fastcgi_params;
        fastcgi_pass php:9000;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        fastcgi_param DOCUMENT_ROOT $realpath_root;
        fastcgi_read_timeout 30s;
        internal;
    }

    location /assets/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    location = /index.html {
        add_header Cache-Control "no-cache";
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

Review checklist for any NGINX change, mirroring our PR discipline:

- [ ] No broad `\ .php$` matching; front controller only, marked `internal`
- [ ] `$realpath_root` used for `SCRIPT_FILENAME`
- [ ] Security headers present with `always`; CSP tested via Report-Only first
- [ ] Rate limits and body size caps appropriate per route class
- [ ] Trusted proxies configured on the application side — headers can't be spoofed
- [ ] Timeouts reflect actual backend behavior; long operations isolated
- [ ] Config validated (`nginx -t`) in CI before deploy

---

The front door now routes, protects, and accelerates everything behind it.

# Chapter 13. Redis as a KV Workhorse

Redis is in our stack for one reason: it's fast. Sub-millisecond reads from memory, atomic operations, and data structures that map directly onto the problems web apps actually have — caching, counting, queueing, locking.

It's also the component most likely to be misused, because "it's just a key-value store" invites treating it as a database. This chapter covers what Redis is genuinely good at, how to cache without shooting yourself during deploys, and — just as important — what Redis must never be.

## Data Structures You Will Actually Use

Redis isn't a flat key-value store; each value has a type with its own operations. Five of them cover nearly everything:

- **Strings** — the workhorse. Cached JSON blobs, counters (`INCR`), feature-flag snapshots, distributed locks (`SET key value NX EX 30`). If you only ever used strings, you'd still get most of Redis's value.
- **Hashes** — field-value maps within one key. Natural fit for an object snapshot: `HSET session:abc user_id 42 role admin`. Read or update single fields without serializing the whole blob.
- **Lists** — ordered sequences with push/pop at both ends. Simple queues (producer `LPUSH`, consumer `BRPOP`) and recent-activity feeds.
- **Sets** — unordered unique members with O(1) add/remove/contains. Tag membership, rate-limit key tracking, and set algebra (`SINTER`, `SDIFF`) for "users who liked A but not B."
- **Sorted sets** — sets where every member carries a score, ordered by it. The killer structure for leaderboards, time-series windows (`ZRANGEBYSCORE`), and priority queues.
- **Streams** — append-only logs with consumer groups: at-least-once delivery, acknowledgment, replay. When your queue needs more than list semantics — multiple consumers, retries, backlogs you can inspect — streams are the modern answer over lists.

Rule of thumb: pick the structure that matches the *access pattern*, not habit. Most "Redis is slow" stories are someone doing hundreds of string round-trips where one hash or pipeline would do.

## Cache Aside vs. Write-Through; TTL and Stampede

### The two patterns

**Cache aside** (the default): application checks Redis first; on miss, reads MySQL, writes the result to Redis with a TTL, returns it.

```php
$key = 'board:4721:view';
$cached = $redis->get($key);
if ($cached !== false) {
    return $cached;
}
$board = $boardRepository->findView($id);   // authoritative read
$redis->setex($key, 300, serialize($board)); // TTL 5 minutes
return $board;
```

**Write-through**: every write to the database also writes to the cache in the same operation. Cache is always fresh, but every write pays cache latency and failures couple the two systems.

Policy for this book: **cache aside everywhere**, plus explicit invalidation on write (delete the affected keys after committing). Write-through only for hot counters where staleness is unacceptable. Cache aside fails safe — a stale entry expires; write-through fails coupled.

### TTLs are mandatory

Every cached key gets a TTL. No exceptions. An unbounded cache is a slow-motion outage: memory fills, eviction starts evicting *something*, and what gets evicted is rarely what you'd choose. TTL sizing reflects tolerance for staleness — a board view might tolerate 5 minutes; a permissions snapshot maybe 60 seconds; a rate-limit counter uses its window length as its TTL.

### Stampede protection

The **cache stampede**: a hot key expires, and fifty concurrent requests all miss simultaneously, all hit MySQL with the same expensive query, all try to write back. Under load this can take the database down precisely when traffic peaks.

Defenses, in increasing sophistication:

1. **Jittered TTLs** — add random seconds so identical keys created together don't expire together.
2. **Single-flight lock** — the first miss acquires a short-lived lock (`SET lock:key ... NX EX 5`) and rebuilds; others wait briefly then re-read. One query instead of fifty.
3. **Stale-while-revalidate** — serve the expired value immediately while rebuilding in the background. Users never wait; freshness lags by one cycle.

For most teams, jitter plus a lock on known-hot keys covers it. Implement stale-while-revalidate when a specific key's rebuild cost justifies it.

## Sessions, Rate-Limit Counters, Flag Snapshots, Locks

Four standard jobs where Redis earns its keep in this stack:

**Sessions.** PHP's native handler stores sessions in Redis (`session.save_handler = redis`), making sessions work across multiple PHP-FPM nodes without sticky routing — required for stateless horizontal scaling. Use `HttpOnly`, `Secure`, `SameSite` cookies as in Chapter 7; Redis is just the storage.

**Rate-limit counters.** Atomic increments with expiry implement sliding-window limits cleanly:

```php
$key = sprintf('ratelimit:%s:%s', $userId, floor(time() / 60));
$count = $redis->incr($key);
if ($count === 1) {
    $redis->expire($key, 120);   // cover the window boundary
}
if ($count > 100) {
    throw new TooManyRequests();
}
```

`INCR` is atomic even under heavy concurrency — no read-modify-write races. For stricter precision, sorted-set sliding windows beat fixed buckets.

**Feature-flag snapshots.** Chapter 10's consistency requirement — same answer across PHP, Node, and Vue — lands here. Flags resolve server-side into a compact snapshot per actor, cached in Redis with a short TTL, shipped to the SPA in bootstrap. All three layers read the same snapshot; propagation delay equals the TTL.

**Distributed locks.** `SET resource:lock <token> NX EX 30` grants a lease that auto-expires — the safety net when a lock holder crashes. Rules that prevent classic bugs:

- Always set both NX (only if absent) and EX (expiry) in one atomic command.
- Store a random token and release via a Lua script that checks the token before deleting — otherwise process A can delete process B's lock after A's lease expired.
- Treat locks as *hints that reduce duplicate work*, not correctness guarantees. Anything truly critical (money) belongs in a database transaction with row locks (Chapter 9's `FOR UPDATE`), not a Redis lease.

## Serialization and the Deploy Explosion Problem

Here's a failure mode that hits almost every team once:

> You cache a serialized PHP object. You deploy a refactor that renames a class property. Every cached object now deserializes into garbage — or worse, silently loses fields — until TTLs expire. Production breaks for ten minutes after every deploy.

PHP's `serialize()` embeds class names and internal structure. It's fragile across refactors and unsafe with untrusted input (unserialization of attacker-controlled data is a documented RCE vector). Similar traps exist in every ecosystem.

The rules:

- **Cache stable, versioned representations — not live objects.** Serialize arrays of scalars, or better, the exact JSON shape your API returns. Plain arrays and JSON survive refactors that rename classes.
- **Version your cache keys:** `board:4721:v2:view`. Bump the version whenever the payload shape changes; old entries expire harmlessly while new ones populate. Deploys stop exploding because shape changes start new namespaces instead of corrupting old ones.
- **Never unserialize untrusted data.** If a value didn't come from your own code, treat it as hostile — use JSON.
- **Match serializers across services.** The Node worker reading a PHP-written cache entry needs an agreed format — JSON with documented field names, not PHP's proprietary format.

## Persistence: AOF vs. RDB, and Losing Data Gracefully

Redis offers two persistence mechanisms:

- **RDB** — point-in-time snapshots on a schedule (e.g., every 5 minutes if enough keys changed). Compact, fast to restore, but you lose everything since the last snapshot.
- **AOF** (Append Only File) — logs every write operation. With `appendfsync everysec`, you lose at most ~1 second of writes. Larger files, slower restarts.

Common production setup: **both** — RDB for compact backups, AOF for fine-grained durability.

But step back to the design question: **when is Redis allowed to lose data?** Answer it per use case, before configuring anything:

- **Cache entries:** losing them costs a latency spike while they repopulate. Totally acceptable — this is why cache-aside exists.
- **Rate-limit counters:** losing them resets limits briefly. Acceptable.
- **Queues holding real work:** losing a job means a customer's export never happens. *Not* acceptable — either enable AOF properly, acknowledge jobs only after durable processing downstream, or move the queue to something built for durability.
- **Anything financial or authoritative:** never in Redis at all (see below).

Configure persistence to match the strictest use case sharing the instance — or better, separate instances by durability requirement so a cache-only Redis can run fast and loose while the queue Redis runs careful.

## Key Namespaces and Eviction Policy

### Namespaces

Redis has no databases-within-databases worth using (the numbered `SELECT 0–15` scheme is deprecated thinking). Structure comes from **key naming conventions**:

```
cache:board:{id}:view:v2
cache:user:{id}:permissions
sess:{sessionId}
ratelimit:{userId}:{window}
lock:export:{jobId}
queue:exports
flags:snapshot:{actorId}
```

Patterns: `purpose:entity:id[:qualifier][:version]`. Colons are convention, not enforcement — the discipline is human. Good namespaces make `SCAN`-based inspection possible ("show me everything in `cache:`"), make bulk deletion sane, and make collisions between features impossible. Bad namespaces produce two features writing `user:123` with different meanings, discovered during an incident.

### Eviction policy

When memory fills, Redis evicts according to `maxmemory-policy`:

- **`noeviction`** — writes fail when full. Correct for queues and locks: dropping data silently is worse than erroring loudly.
- **`allkeys-lru` / `volatile-lru`** — evict least-recently-used keys, across all keys or only those with TTLs. Correct for pure caches.
- **`allkeys-lfu`** — evicts least-*frequently*-used; often better than LRU for mixed workloads.

The trap: one shared instance serving both a cache (wants LRU) and a queue (wants noeviction) can't have both. Either isolate by instance (preferred — Compose makes this cheap) or accept the compromise consciously and document it.

## What Redis Is Not

The chapter's most important section is the negative space:

- **Not a source of truth for money.** Balances, ledger entries, payment state — these live in MySQL with transactions and row locks. Redis can *cache* a balance view, but the database decides. A Redis flush must never be able to lose money.
- **Not a long-term document store.** No rich querying, no schema evolution story, no backup tooling comparable to a real database. If data matters beyond days, it belongs in MySQL (possibly with a JSON column — Chapter 9).
- **Not automatically durable.** Even with AOF, async replication and fsync windows mean "committed" is softer than in MySQL. Design accordingly: idempotent consumers, durable job storage, and the assumption that any given Redis instance may vanish.
- **Not a substitute for modeling.** "We'll put it in Redis and figure out the schema later" is Chapter 9's JSON abuse wearing a different coat.

The healthy mental model: **MySQL remembers; Redis accelerates and coordinates.** If deleting the entire Redis instance would cause anything worse than a temporary performance dip, some data is living in the wrong place.

---

# Chapter 14. Message Queues

Some work must not happen inside an HTTP request. Generating a thumbnail takes seconds; sending email can hang; a webhook delivery depends on someone else's uptime. If your controller does these inline, your users wait on them and your failures cascade. Message queues decouple "accept the request" from "do the work" — the request returns instantly, and the work happens reliably in the background.

This chapter covers two queue implementations — one per runtime in our stack — and the shared vocabulary that makes them interchangeable in your head.

## One Vocabulary, Two Implementations

Every queue system, whatever its branding, expresses the same concepts:

- **Producer** — publishes a message describing work to be done.
- **Consumer / worker** — picks up messages and executes them.
- **Retry** — failed jobs are attempted again, usually with delays.
- **Dead letter** — after retries are exhausted, messages go to a holding area for human inspection instead of vanishing or looping forever.
- **Idempotency** — because delivery is *at-least-once* (more below), handlers must tolerate processing the same message twice.
- **At-least-once semantics** — the fundamental contract: queues guarantee a message is delivered *at least* once, never *exactly* once. Network partitions between "worker finished" and "broker acknowledged" produce duplicates. Every design decision downstream flows from accepting this.

Internalize that last point now. Everything else in this chapter — idempotency keys, deduplication, careful acknowledgment — is engineering around at-least-once delivery.

## BullMQ (Node): Redis-Backed Jobs

**BullMQ** is the modern Node job queue built on Redis. Its model is direct: you create **queues**, add **jobs** (named payloads), and run **workers** that process them.

```ts
// Producer (anywhere in the Node services)
import { Queue } from 'bullmq';

const thumbnails = new Queue('thumbnails', { connection: redisConnection });

await thumbnails.add('generate', {
  cardId: 'card_4721',
  imageUrl: 'https://cdn.example.com/attachments/abc.png',
}, {
  attempts: 5,
  backoff: { type: 'exponential', delay: 2000 },
  removeOnComplete: 1000,
});
```

```ts
// Worker (separate process)
import { Worker } from 'bullmq';

new Worker('thumbnails', async (job) => {
  const { cardId, imageUrl } = job.data;
  await generateThumbnail(imageUrl);          // may throw
  await markThumbnailReady(cardId);
}, {
  connection: redisConnection,
  concurrency: 4,
  limiter: { max: 10, duration: 1000 },       // rate limit: 10 jobs/sec
});
```

What BullMQ gives you out of the box:

- **Retries with exponential backoff** (`attempts` + `backoff`) — configured per job.
- **Repeatable jobs** — cron-style scheduling ("run the nightly digest at 02:00") without external crontab.
- **Priority** — urgent jobs jump the line within a queue.
- **Rate limiting** — per-worker throughput caps, essential when the downstream API has quotas.
- **Delayed jobs** — "retry this in 30 minutes," "send this reminder tomorrow."
- **A real dashboard** (Bull Board) showing depth, failures, and active jobs.

The tradeoff to respect: BullMQ's durability is Redis durability (Chapter 13). With AOF enabled and sensible settings it's dependable for most business work — but it inherits every Redis caveat, including "the instance vanished." For jobs whose loss would be catastrophic, either enable AOF properly and monitor it, or use RabbitMQ.

## RabbitMQ (PHP): Exchanges, Bindings, Routing

**RabbitMQ** implements AMQP, a protocol with more machinery than BullMQ — and correspondingly more power for complex routing.

The core model:

- **Producers publish to exchanges**, never directly to queues.
- **Exchanges route messages to queues via bindings and routing keys.**
- **Consumers acknowledge** messages explicitly.

Exchange types determine routing behavior:

- **Direct** — exact routing-key match. `publish(exchange: 'jobs', key: 'email.welcome')` reaches queues bound to exactly that key.
- **Topic** — wildcard matching: binding `email.*` catches `email.welcome` and `email.digest`; `#.critical` catches anything ending in `.critical`.
- **Fanout** — broadcast to all bound queues. One message, many independent consumers.

In PHP, the mature client is `php-amqplib` (or Symfony Messenger's AMQP transport, which wraps it):

```php
// Publisher
$channel->basic_publish(
    new AMQPMessage(json_encode($payload), [
        'delivery_mode' => AMQPMessage::DELIVERY_MODE_PERSISTENT, // survive broker restart
        'message_id' => $idempotencyKey,
    ]),
    exchange: 'jobs',
    routing_key: 'search.reindex',
);

// Consumer
$channel->basic_qos(null, prefetchCount: 10, null);   // fair dispatch
$channel->basic_consume('search-reindex', callback: function (AMQPMessage $msg) {
    try {
        handleReindex(json_decode($msg->getBody(), true));
        $msg->ack();
    } catch (TransientException) {
        $msg->nack(requeue: true);                    // retry later
    } catch (PermanentException) {
        $msg->nack(requeue: false);                   // → dead letter exchange
    }
});
```

Key mechanics:

- **Ack/nack:** the broker redelivers any message left unacknowledged when a consumer dies — this *is* the at-least-once guarantee. Ack only after the work durably completes.
- **Prefetch:** how many unacked messages one consumer holds. Too high and one slow consumer hoards work while others idle; too low and throughput suffers. Start around 10 and tune under load.
- **DLX (dead letter exchange):** configure each queue with a dead-letter exchange so rejected messages land somewhere inspectable rather than being dropped. A queue without a DLX is a black hole wearing a retry costume.
- **Persistent messages + durable queues** together survive broker restarts; either alone does not.

## Choosing the Broker by Owning Runtime

Our stack runs PHP and Node side by side. Which broker where? The rule from Chapter 1 applies: **let the owning runtime pick, and don't invent a third option.**

- **Node-owned jobs → BullMQ.** Webhook deliveries, BFF-triggered background tasks, anything living in the Node services. The ecosystem fit is native, the dashboard is excellent, and Redis is already running.
- **PHP-owned jobs → RabbitMQ.** Symfony-side domain events — reindexing, notifications triggered by domain changes. Symfony Messenger speaks AMQP natively, and RabbitMQ's routing earns its complexity as the event backbone grows.

Two anti-patterns to refuse:

1. **A third broker "for neutrality."** Every additional broker is another thing to operate, monitor, patch, and learn during incidents. Two is already generous.
2. **Cross-runtime coupling through queue internals.** If a Node worker must consume PHP-produced messages, communicate through a documented JSON envelope on a shared queue — not by making Node speak AMQP topology designed for PHP, or worse, having PHP write BullMQ's internal Redis structures (they're private API and change between versions). Shared vocabulary, separate brokers, clean boundary.

## Poison Messages, Backoff, and Idempotent Handlers

### Poison messages

A **poison message** fails on every attempt — malformed payload, a bug triggered by specific data, a downstream API rejecting something permanently. Without handling, it either loops forever consuming capacity or crashes workers repeatedly. Both are outages caused by one bad record.

Defense is structural: distinguish failure types in handlers.

- **Transient failures** (network blip, timeout, lock contention) → retry with backoff.
- **Permanent failures** (validation error, unknown entity, bug) → dead-letter immediately. Retrying cannot help; humans must look.

Never let an exception type ambiguity decide — catch deliberately, classify explicitly.

### Exponential backoff

Retries should space themselves out: 2s, 4s, 8s, 16s... plus jitter. This protects struggling downstream systems from synchronized retry waves — the **retry storm**, where a recovering service gets hammered by everyone's queued retries simultaneously and falls over again. Backoff with jitter converts a stampede into a trickle. Cap total attempts (typically 5) before dead-lettering; infinite retries just relocate the outage.

### Idempotent handlers

Because of at-least-once delivery, assume every handler will eventually process the same message twice. Make that harmless:

- **Natural idempotency first.** "Set thumbnail path to X" is idempotent; "increment counter" is not. Prefer designs where repeating produces the same state.
- **Deduplication keys.** Store processed message IDs (with TTL) and skip known ones:

```php
if (!$redis->set("processed:{$msgId}", 1, ['nx', 'ex' => 86400])) {
    return; // already handled
}
```

- **Idempotency keys end-to-end.** Chapter 8's pattern applies here too: the message carries a key; the side effect (payment, email send) is keyed against it at the destination.

Test it: every handler's test suite includes "process the same message twice, assert no duplicate effect."

## The Outbox Pattern: When DB Write and Publish Must Not Diverge

Here's a subtle failure that bites every event-driven system: the handler writes to MySQL, then publishes an event — and crashes between the two. Now the database says one thing and the rest of the system believes another. Wrapping both in a transaction doesn't fix it: the publish isn't transactional, and committing the DB write while the publish fails still diverges.

The **outbox pattern** solves it by making the publish part of the same database transaction:

1. In the same MySQL transaction as the business write, insert an event row into an `outbox` table.
2. Commit. Write and event are now atomic — both exist or neither does.
3. A separate relay process reads unpublished outbox rows, publishes them to the broker, marks them published.

```sql
-- Same transaction as the business change:
INSERT INTO orders (...) VALUES (...);
INSERT INTO outbox (id, topic, payload, created_at)
VALUES (?, 'order.placed', ?, NOW(6));
```

Delivery becomes at-least-once again (relay crash after publish but before marking), so consumers stay idempotent — but divergence becomes impossible. The outbox table doubles as a durable audit log of everything the domain ever announced.

Cost: slight latency (events publish milliseconds later) and one more table. Worth it for any event whose loss breaks consistency. Skip it for fire-and-forget notifications where eventual loss is tolerable.

## Observability: Depth, Lag, Storms

Queues fail quietly — everything looks fine until the backlog is hours long. Watch three signals:

- **Queue depth** — messages waiting. Some baseline is normal; growth over minutes is not. Alert on sustained growth, not absolute numbers.
- **Processing lag** — age of the oldest pending message. More honest than depth: a deep-but-fast-moving queue is healthy; a shallow queue whose oldest message is 40 minutes old is broken. Consumers should expose "oldest unprocessed timestamp."
- **Retry rates and storms** — spikes in retry counts signal either a poisoned batch or a struggling dependency. Correlate with Chapter 10's flag telemetry and deploys: "retries tripled right after the 14:00 deploy" is the most valuable sentence in an incident.

Also track per-job success/failure counts, dead-letter arrivals (each one deserves a ticket), and consumer liveness (a crashed worker stops draining silently unless heartbeats are monitored).

## Project Board Jobs

Mapping the patterns onto our running example:

| Job | Runtime | Queue choice | Notes |
|---|---|---|---|
| Thumbnail generation | Node | BullMQ | CPU-bound image work; rate-limited; retries for transient storage errors |
| Webhook delivery | Node | BullMQ | Outbound HTTP with strict timeouts, exponential backoff, DLQ after 5 attempts; idempotency key per delivery |
| Search reindex | PHP | RabbitMQ | Triggered by domain events through the outbox; fanout to search indexer |
| Email | PHP | RabbitMQ | Template rendering + provider API; permanent failures (bad address) dead-letter immediately |

Each job follows the same checklist: classified error handling, capped retries with jittered backoff, idempotent handler, dead-letter destination, and metrics for depth and lag. Build the checklist once as a base class or shared helper; apply it everywhere.

---

Queues turn "it works" into "it works reliably under failure" — provided handlers respect at-least-once delivery and operators watch the lag.

# Chapter 15. ORMs Without Self-Harm

ORMs get blamed for a lot of damage they didn't cause. The tool doesn't create the N+1 query or the 40-second page load — the developer who treated the ORM as magic did. Used with understanding, an ORM eliminates the most error-prone code you'd otherwise write by hand (parameterization, hydration, change tracking) while staying out of the way when you need raw SQL.

This chapter covers the two ORMs in our stack — Doctrine for PHP, Prisma for Node — and the discipline that keeps them from harming you.

## Doctrine (PHP)

### Entities and mappings

A Doctrine **entity** is a PHP class mapped to a table. Modern mapping uses PHP attributes:

```php
#[ORM\Entity(repositoryClass: BoardRepository::class)]
#[ORM\Table(name: 'boards')]
class Board
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column(type: 'bigint', options: ['unsigned' => true])]
    private ?int $id = null;

    #[ORM\Column(length: 120)]
    private string $title;

    #[ORM\Column(type: 'datetime_immutable', precision: 6)]
    private \DateTimeImmutable $createdAt;

    #[ORM\ManyToOne(targetEntity: Organization::class, inversedBy: 'boards')]
    #[ORM\JoinColumn(nullable: false)]
    private Organization $organization;

    /** @var Collection<int, Card> */
    #[ORM\OneToMany(mappedBy: 'board', targetEntity: Card::class)]
    private Collection $cards;
}
```

The mapping is the contract between your object graph and Chapter 9's schema. Note `datetime_immutable` — prefer immutable value objects; mutable dates passed around by reference are a classic source of subtle bugs.

### Unit of Work and the identity map

Doctrine's core is the **Unit of Work**: it tracks every entity it has loaded, compares current state against the original snapshot at flush time, and generates exactly the SQL needed for changes.

```php
$board = $repository->find(4721);
$board->setTitle('Q3 Roadmap');
$em->flush();   // one UPDATE — no explicit save() call
```

The **identity map** guarantees that within a request, `find(4721)` twice returns *the same object instance*. This prevents duplicate-row confusion but creates two classic traps:

- **Long-lived EntityManagers** accumulate tracked entities until memory balloons. In web requests this is self-limiting (one manager per request); in long-running workers and consumers, call `$em->clear()` periodically.
- **Detached entities:** an entity loaded in one context and modified in another won't be flushed unless re-attached (`merge`). Symptom: "my changes silently don't save."

### Fetch modes: lazy, extra-lazy, eager

How associations load is where Doctrine performance lives or dies:

- **Lazy (default):** accessing `$board->getCards()` triggers a query on first access. Convenient, dangerous in loops — this is the N+1 engine.
- **Eager:** association loads with the parent via JOIN. Predictable, but eager-loading everything drags half your database into memory per query.
- **Extra-lazy:** collections support `count()` and slicing without hydrating all elements. Right choice for large collections where you mostly need counts and pages.

Policy: keep mappings lazy, then fetch deliberately per use case with query hints — never make eager global.

### DQL and QueryBuilder

**DQL** queries your object model, not tables:

```php
$dql = 'SELECT b, o FROM App\Entity\Board b JOIN b.organization o WHERE o.slug = :slug';
```

The **QueryBuilder** builds DQL programmatically — and as Chapter 7 warned, parameter binding remains mandatory:

```php
$qb = $em->createQueryBuilder()
    ->select('b')
    ->from(Board::class, 'b')
    ->where('b.organization = :org')
    ->setParameter('org', $org)
    ->orderBy('b.createdAt', 'DESC');
```

### Second-level cache: usually no

Doctrine offers a second-level cache that stores query/entity results across requests. It sounds appealing and mostly isn't: invalidation across application nodes is hard to get right, stale reads become mysterious bugs, and Redis-based caching at the application layer (Chapter 13) gives you explicit control over keys and TTLs. Default answer: **no second-level cache.** Use cache-aside where profiling proves the need.

## Prisma (Node)

### Schema-first design

Prisma inverts Doctrine's flow: you define a declarative schema file, and Prisma generates both the migration SQL and a fully typed client:

```prisma
model Board {
  id        BigInt   @id @default(autoincrement())
  title     String   @db.VarChar(120)
  createdAt DateTime @db.DateTime(6)
  orgId     BigInt
  org       Organization @relation(fields: [orgId], references: [id])
  cards     Card[]

  @@index([orgId, createdAt])
}
```

```ts
const boards = await prisma.board.findMany({
  where: { org: { slug: 'acme' } },
  include: { cards: { take: 20, orderBy: { createdAt: 'desc' } } },
});
// `boards` is fully typed — TypeScript knows every field
```

The generated client is Prisma's superpower: types flow from schema to every query result, so renaming a column becomes a compile error instead of a runtime surprise.

### Transactions

Prisma offers two forms:

```ts
// Sequential interactive transaction
await prisma.$transaction(async (tx) => {
  const board = await tx.board.findUnique({ where: { id } });
  await tx.card.updateMany({ where: { boardId: id }, data: { archived: true } });
});

// Batch of independent operations
await prisma.$transaction([
  prisma.auditLog.create({ data: { ... } }),
  prisma.board.update({ where: { id }, data: { status: 'archived' } }),
]);
```

Interactive transactions hold connections open — keep them short (Chapter 9's locking rules apply identically).

### `prisma migrate` vs. `db push`

Two commands, very different contracts:

- **`prisma migrate dev`** generates timestamped SQL migration files from schema changes — reviewable artifacts, applied in order, committed to git. **This is what production uses.**
- **`prisma db push`** syncs the database directly to the schema with no migration history. Fine for throwaway prototypes and local experiments; catastrophic as a team workflow because there's no record of what changed and no safe path between environments.

Rule: `db push` never touches anything that matters.

## Type Mappings as Source of Truth; Humans Still Review

Both tools can generate schema from types (Doctrine) or types from schema (Prisma). Either direction works — what matters is that **exactly one artifact is the source of truth**, and everyone knows which.

In this book: **the type/mapping definitions are truth.** Doctrine attributes and the Prisma schema describe reality; migrations are generated from them. This keeps types and schema from drifting — the failure mode where the entity says `string` and the column says `TEXT` and nobody notices until a truncation bug.

But generation doesn't remove human responsibility. Generated migrations still get reviewed like any other PR (Chapter 9's checklist): locking behavior, reversibility, backfill safety, deploy ordering. The generator doesn't know your table has 40 million rows; the reviewer must.

## N+1, Partial Hydration, and When to Drop to SQL

### N+1, concretely

```php
$cards = $cardRepository->findBy(['board' => $board]);   // 1 query
foreach ($cards as $card) {
    echo $card->getAssignee()->getName();                 // +1 query PER CARD
}
```

Fifty-one queries where one would do. Detection: log SQL in development (Chapter 9) and count queries per request; anything above a handful needs explaining. Fix: fetch joins.

```php
$qb->select('c', 'a')
   ->from(Card::class, 'c')
   ->leftJoin('c.assignee', 'a')
   ->where('c.board = :board');
```

One query, hydrated graph, done. In Prisma, the equivalent sin is querying in loops instead of using `include`/`select`.

### Partial hydration

Loading full entities when you need three columns wastes memory and bandwidth. For read-heavy views, select scalars:

```php
$qb->select('c.id', 'c.title', 'a.name AS assignee')
   ->from(Card::class, 'c')
   ->leftJoin('c.assignee', 'a');
```

You get arrays, not managed entities — which is exactly right for API responses and reports. Rule: **entities for mutation workflows, partial hydration for read views.**

### When to drop to SQL

ORMs are bad at some things, and pretending otherwise produces atrocities:

- **Reporting and analytics** — aggregations across many tables, window functions, CTEs. Write native SQL, map results to DTOs.
- **Bulk operations** — updating 100,000 rows through the Unit of Work loads all 100,000 into memory first. Use `UPDATE ... WHERE` statements directly.
- **Chapter 6's authorization filters** — expressible in QueryBuilder, but complex relationship traversal sometimes reads better as hand-written SQL with full control over indexes.

Doctrine supports this cleanly: native queries with ResultSetMapping, or DBAL directly. Dropping to SQL isn't defeat; it's using each tool for its strength. What's non-negotiable even in raw SQL: parameters stay bound (Chapter 7), and the query gets EXPLAINed before shipping (Chapter 9).

## Domain Model vs. Persistence Model

Does your business logic live on the same classes the ORM maps? Sometimes yes, sometimes it shouldn't.

**When 1:1 is fine:** CRUD-shaped domains where entities are mostly data with light behavior — our Board, Card, and List entities carry validation and small methods, and that's honest. Most line-of-business apps are this. Fighting for "purity" here adds mapping layers that translate without adding safety.

**When to split:** when persistence concerns start distorting the domain:

- Entities requiring constructor arguments the database can't provide (services, clocks).
- Value objects so rich that ORM mapping fights the design.
- Aggregate boundaries where loading one concept must not drag five related tables.
- Read models whose shape exists only for a specific screen — these were never domain objects anyway (use partial hydration / DTOs).

The pragmatic split: **rich domain objects for write paths, thin DTOs for read paths, shared persistence mapping only where the shapes coincide.** Don't build a separate persistence layer speculatively; extract it when the friction is real, not when a conference talk says so.

## Migrations in CI

Migrations are code that runs against production with no undo — they deserve pipeline enforcement, not just review attention.

### Expand/contract

Never deploy a migration and the code using it simultaneously. The **expand/contract** pattern sequences changes safely:

1. **Expand:** additive migration — new column (nullable or defaulted), new table, new index. Old code ignores it; nothing breaks.
2. **Deploy code** that writes to both old and new locations, reads from new with fallback.
3. **Backfill** existing rows in batches (not one giant transaction).
4. **Contract:** after full rollout and verification, a second migration removes the old column/table.

Each step deploys independently and rolls back independently. A rename becomes add-new → dual-write → backfill → switch-reads → drop-old, spread across several boring releases instead of one terrifying one.

### Lock safety

Schema changes take metadata locks; on busy tables, a blocking ALTER stalls every query behind it. CI should flag risky operations (type changes, long rebuilds) for explicit scheduling, and production migrations run with lock-wait timeouts configured so worst case fails fast rather than freezing the site.

### Never edit a shipped migration

Once a migration has run anywhere shared — a teammate's machine counts — it's history. Editing it desynchronizes everyone whose database already applied the original. Fixes come as *new* migrations. Enforce mechanically: CI checks that migration files are append-only (compare against the previous commit).

Also gate in CI: migrations run against a scratch database on every PR (proving they apply cleanly from zero), and the expand/contract ordering is verified by the deploy pipeline's staging pass.

---

---

# Part V — The PHP Stack

# Chapter 16. PHP 8.5 as a Language

A lot of "PHP code" in the wild is PHP 7.4 with type hints sprinkled on top — written before enums existed, before readonly properties, before the language grew a real type system. Modern PHP is a different language wearing the same name: typed, expressive, with functional conveniences that eliminate whole categories of boilerplate.

This chapter covers how we write PHP in this book: current syntax used deliberately, error handling done honestly, and the tooling underneath.

## Types: The Foundation

### Backed enums

Enums replace the stringly-typed constants scattered through most codebases:

```php
enum InvoiceStatus: string
{
    case Draft = 'draft';
    case Issued = 'issued';
    case Paid = 'paid';
    case Refunded = 'refunded';
}
```

What you gain: the type system now *proves* a status is one of four values — impossible states like `'pyad'` (typo) or `null` can't reach your domain logic. Backed enums give each case a storable value (`->value`) for database columns, and `from()`/`tryFrom()` for hydration. Doctrine maps them natively. Every "magic string" comparison in your codebase is a candidate for conversion.

### Union and intersection types

```php
function findUser(int|string $id): User|null { ... }        // union
function log(Auditable&JsonSerializable $entity): void { } // intersection
```

Unions express honest signatures ("accepts an ID or a slug"); intersections demand multiple capabilities at once. Combined with nullable shorthand (`?User`), most docblock type comments become redundant — delete them and let the language carry the truth.

### `readonly`

```php
final class Money
{
    public function __construct(
        public readonly int $amountCents,
        public readonly Currency $currency,
    ) {}
}
```

`readonly` properties are writable once (at construction) and immutable after. This makes value objects genuinely immutable without hand-written getters guarding setters — write the constructor promotion, get immutability free. Mark every property `readonly` unless you have a concrete reason not to; mutability should be a decision, not a default.

### Property hooks and asymmetric visibility

PHP 8.4+ lets properties define their own read/write behavior — no more boilerplate getter/setter pairs:

```php
class Invoice
{
    public string $customerName {
        get => strtoupper($this->customerName);
        set => trim($value) ?: throw new \InvalidArgumentException('Name required');
    }

    // Asymmetric visibility: publicly readable, privately writable
    public private(set) InvoiceStatus $status = InvoiceStatus::Draft;
}
```

**Asymmetric visibility** (`public private(set)`, `public protected(set)`) is the quiet workhorse here: anyone may *read* the status, but only the class itself may *change* it. That encodes "state transitions happen inside the aggregate" directly in the language — exactly what Chapter 6's domain boundaries want. Symfony 8's components lean on these features heavily; expect to see hooked properties throughout modern framework code.

### `never` and `void`

- **`void`** — the function returns nothing meaningful.
- **`never`** — the function *does not return at all*: it always throws or exits.

```php
function abort(int $status): never
{
    throw new HttpException($status);
}
```

`never` is documentation the static analyzer enforces: code after a `never` call is unreachable, and forgetting a `throw` inside becomes a caught type error. Use it on all guard/abort helpers.

## PHP 8.5 Highlights Worth Adopting

### The pipe operator `|>`

Pipes chain transformations left-to-right, replacing nested calls that read inside-out:

```php
// Before: read this from the inside out
$title = ucwords(mb_strtolower(trim($raw)));

// After: read it top to bottom
$title = $raw |> trim(...) |> mb_strtolower(...) |> ucwords(...);
```

Each stage takes one argument; first-class callable syntax (`trim(...)`) keeps it terse. Pipes shine for data-mapping pipelines — normalizing input, transforming API payloads — where the sequence of steps *is* the logic. Don't force them everywhere; use them when linear reading beats nesting.

### `clone with` for immutable updates

Updating an immutable object used to mean either mutable setters (defeating the point) or manual copy constructors. PHP 8.5 adds `clone with`:

```php
$updated = $invoice |> clone with (
    status: InvoiceStatus::Paid,
    paidAt: new \DateTimeImmutable(),
);
```

The original stays untouched; the clone carries the changes. Combined with `readonly` properties, this gives you record-style updates — the pattern Vue's reactivity and Redux made mainstream, now native. Ideal for DTOs, command objects, and event payloads.

### Native URI extension

Every project eventually rolls its own URL parser, and every hand-rolled parser has bugs — especially around IPv6 hosts, empty ports, and Unicode. PHP 8.5 ships a proper URI extension implementing both RFC 3986 and WHATWG URL semantics:

```php
$uri = Uri\Rfc3986\Uri::parse('https://example.com:443/path?q=1#frag');
$uri->getHost();     // 'example.com'
$uri->withPath('/v2/path');   // immutable — returns a new instance

Uri\WhatWg\Url::parse('https://ex ample.com'); // browser-grade leniency rules
```

The WHATWG flavor matches what browsers do (relevant for validating redirect URLs); RFC 3986 matches strict spec parsing. Either way: **stop writing `parse_url` wrapper classes.** Chapter 7's SSRF defenses also benefit — validated, structured URIs beat regex checks on raw strings.

### Small sharp tools

- **`array_first()` / `array_last()`** — the end of `$array[array_key_first($array)]` gymnastics. Null-safe on empty arrays.
- **`#[\NoDiscard]`** — mark functions whose return values must be consumed:

```php
#[\NoDiscard]
function validatePayment(Payment $p): ValidationResult { ... }

validatePayment($payment);        // compiler warning: result discarded
$result ??= validatePayment($payment);  // explicit discard silences it
```

Perfect for validation and check-style methods where ignoring the answer is always a bug.

- **Always-on OPcache** — PHP 8.5 makes OPcache permanently enabled; the old "did you turn it on?" support question dies. Verify production still tunes `opcache.memory_consumption` and enables JIT where profiling justifies it.
- **Fatal error backtraces** — uncaught fatals now print full stack traces in logs, ending the era of cryptic "Maximum execution time exceeded" with zero context. Better incident forensics for free.

## Errors vs. Exceptions

PHP has two failure channels, and confusing them causes real damage:

- **Errors** (many now converted to `\Error` exceptions): programming mistakes — type violations, undefined methods, out-of-memory. These indicate *bugs*. You generally should not catch them; let them crash loudly during development and surface in monitoring.
- **Exceptions**: anticipated, recoverable conditions — a payment declined, a lock timeout, invalid user input. Catch what you can meaningfully handle.

Two rules from Chapter 7 apply verbatim here:

**Don't swallow.** An empty catch block converts a loud failure into silent corruption:

```php
try {
    $this->gateway->charge($payment);
} catch (\Throwable $e) {
    // "handle later" = never. The charge state is now unknown.
}
```

If you catch, either handle concretely (retry, compensate, convert to a domain result) or rethrow with context. Log-and-rethrow is usually worse than either — it double-counts incidents.

**Don't use exceptions for control flow.** Exceptions are expensive to construct and semantically wrong for expected outcomes:

```php
// Bad: exceptions as goto
try {
    $user = $repo->find($id);
} catch (NotFoundException) {
    $user = null;
}

// Good: query methods return nullables; exceptions signal broken contracts
$user = $repo->find($id); // ?User
```

Reserve exceptions for situations that shouldn't happen given correct usage — and make "shouldn't happen" states impossible with types wherever possible (nullable returns, enums instead of magic strings). Fewer exceptions means fewer paths where swallowing matters.

## Autoloading: Composer Classmap vs. PSR-4

Composer autoloads your classes so files load on demand. Two strategies:

- **PSR-4** — namespace maps to directory; class `App\Domain\Board` lives at `src/Domain/Board.php`. Standard for application code: predictable, works with dev-time file changes without rebuilding anything.
- **Classmap** — Composer scans and builds an explicit class-to-file map. Faster lookups, but requires `composer dump-autoload` whenever classes move or appear.

Standard configuration:

```json
{
    "autoload": {
        "psr-4": { "App\\": "src/" }
    },
    "autoload-dev": {
        "psr-4": { "App\\Tests\\": "tests/" }
    }
}
```

Production optimization: `composer dump-autoload --optimize` (or `--classmap-authoritative` when nothing loads outside the map) bakes PSR-4 into a classmap — best of both worlds. Authoritative mode additionally skips filesystem checks for missing classes, saving microseconds per request that add up under FPM.

One discipline note: a class not found by the autoloader is almost always a namespace/directory mismatch. Fix the mapping, don't add `require` statements — those fossilize instantly.

## Extensions We Rely On

Our Docker images (Chapter 11) install a deliberate, minimal set:

- **`pdo_mysql`** — the database driver beneath Doctrine. Non-negotiable.
- **`redis`** — phpredis extension for sessions, cache, and locks (Chapter 13). The C extension dramatically outperforms pure-PHP clients under FPM concurrency.
- **`intl`** — internationalization: locale-aware formatting, collation, IDN handling. Required by many Symfony components and the right tool for any user-facing date/number/currency formatting (don't hand-roll it).
- **Xdebug — development only.** Step debugging, coverage, and profiling are invaluable locally; in production Xdebug is pure overhead and a potential information leak. Keep it out of production images entirely — separate dev requirements, enforced by the multi-stage build.

Everything else earns its place the same way dependencies do: added deliberately, reviewed, and removed when unused. A lean extension list is a smaller attack surface and faster image builds.

---

Modern PHP gives you a type system strong enough to make whole bug categories unrepresentable — if you write current code rather than muscle memory.

# Chapter 17. Composer and Dependency Discipline

Your application is a thin layer of your own code over a mountain of other people's code. Composer manages that mountain — and how you manage it determines whether dependency updates are routine maintenance or recurring emergencies.

This chapter covers the mechanics (`composer.json`, lockfiles, version constraints) and the judgment calls underneath them (trust, forking, replacement).

## `composer.json` vs. Lockfile — and `--no-dev` in Production

The two files answer different questions:

- **`composer.json`** declares *intent*: which packages you need, at which version ranges. It's hand-written and committed.
- **`composer.lock`** records *reality*: the exact versions resolved right now, with hashes. It's machine-written by Composer and committed too.

The lockfile is what makes builds reproducible. Without it, every install re-resolves versions against Packagist's current state — meaning two deploys a week apart can run different library versions with zero changes on your side. That's Chapter 7's supply-chain rule in practice:

```bash
# Development: resolve and update
composer update          # reads composer.json, writes composer.lock

# CI / production: install exactly what's locked
composer install         # reads composer.lock, ignores ranges
```

Rule: **`update` is a deliberate act that produces a reviewed diff; `install` is everything else.** If your deploy pipeline runs `composer update`, fix it today.

In production images, dev dependencies must not ship:

```dockerfile
RUN composer install --no-dev --optimize-autoloader --no-interaction --no-progress
```

PHPUnit, PHPStan, and friends are attack surface and image weight with no runtime value. The multi-stage build from Chapter 11 keeps them in the vendor stage only.

## Semver, Stability Flags, and Why `^` Is a Contract

### Semantic versioning

Most PHP packages follow semver: `MAJOR.MINOR.PATCH`.

- **PATCH** (2.3.1 → 2.3.2): bug fixes, no API change.
- **MINOR** (2.3 → 2.4): new features, backward-compatible.
- **MAJOR** (2.x → 3.0): breaking changes.

Semver is a promise, not a guarantee — maintainers sometimes break things in minors by accident. Treat it as a strong prior, verified by your test suite, not blind trust.

### Constraint operators

```json
"require": {
    "symfony/console": "^7.0",
    "monolog/monolog": "~3.5.0",
    "ext-redis": "*"
}
```

- **`^7.0`** — "compatible with 7.x": allows up to `<8.0`. This is the caret contract: minor updates flow automatically; majors never arrive uninvited.
- **`~3.5.0`** — tilde pins tighter: only patches (`>=3.5.0 <3.6.0`). Use when even minor updates scare you.
- **Exact pinning** (`"3.5.1"`) — rarely appropriate in `composer.json`; that's what the lockfile is for.

Why "`^` is a contract": it means you've accepted responsibility for testing minor updates. Your CI suite is what makes the contract safe — without tests, `^` just means "unvetted changes reach production silently." With tests, `^` gives you security fixes and improvements for free via routine `composer update` PRs.

### Stability flags

Packagist tags releases as stable, beta, alpha, RC, or dev. By default Composer installs stable only. When you genuinely need pre-release software:

```json
"require": {
    "some/package": "^2.0@beta"
},
"minimum-stability": "stable",
"prefer-stable": true
```

Keep `minimum-stability: stable` globally and opt into instability per-package (`@beta`). A project-wide unstable floor drags *every* dependency toward pre-releases — an accident waiting to compile.

## Private Packages and Path Repositories

Not all code lives on Packagist. Two mechanisms bring private code into Composer's world:

### Private Packagist / VCS repositories

Point Composer at a private git repo (or a hosted service like Private Packagist):

```json
"repositories": [
    {
        "type": "vcs",
        "url": "git@github.com:acme/acme-shared-kernel.git"
    }
]
```

Composer reads that repo's tags and branches as if it were any package. Auth happens via SSH keys or tokens — keep credentials in CI secrets, never in committed config.

### Path repositories

For monorepo-style development, map local directories as packages:

```json
"repositories": [
    {
        "type": "path",
        "url": "../packages/*",
        "options": { "symlink": true }
    }
]
```

Changes to the local package appear instantly (via symlink) during development, while CI can inline them into the build. Ideal for shared libraries developed alongside their consumers.

One caution for both: private packages are part of your supply chain too. They need version discipline, changelogs, and audit trails like anything public — arguably more, since nobody else is watching them.

## Scripts, Plugins, and the Trust Boundary

Here's the uncomfortable truth about Composer: **running `composer update` executes arbitrary code.**

Two mechanisms make this so:

- **Scripts** — lifecycle hooks in `composer.json` (`post-install-cmd`, `post-update-cmd`) that run shell commands or PHP callables.
- **Plugins** — packages that hook deeply into Composer itself, modifying behavior during install. A malicious plugin doesn't wait for a script trigger; it activates on installation.

This is why OWASP classifies dependency-manager compromise under injection (A03): a compromised upstream package doesn't need a vulnerability in your code — its install scripts *are* code execution on your build machines, with access to environment variables, credentials, and network.

Practical defenses:

- **Review scripts in `composer.json`** like any code. A `post-install-cmd` you didn't write is an incident.
- **Disable plugins you don't need:** `"config": { "allow-plugins": { "php-http/discovery": true } }` — the allowlist form ensures nothing else can self-enable.
- **Treat new transitive dependencies as review surface.** `composer update` diffs should be scanned for unexpected additions — a package gaining a new dependency deserves a glance at what that dependency does.
- **CI runs installs in isolated environments** without production credentials in scope. Build machines should never hold keys that matter beyond artifact signing.

None of this means paranoia about Composer itself — it means recognizing that the trust boundary isn't "my code vs. the internet," it's "every line of code my build executes."

## `composer audit`, Abandoned Packages, Fork vs. Replace

### Auditing

```bash
composer audit              # checks locked packages against known CVEs
composer audit --locked     # same, explicit
```

Wire it into CI (Chapter 7) so new advisories fail builds or open tickets immediately. Pair it with Dependabot/Renovate for update PRs — audit tells you what's broken; automation proposes the fix.

### Abandoned packages

When a maintainer archives a package, Composer flags it as abandoned during install. An abandoned package isn't broken — but it will never receive another security fix, and its bugs are now yours forever. Every abandoned flag demands a decision, made deliberately rather than ignored until the package rots.

### Fork vs. replace

When a dependency needs something upstream won't accept — a bugfix, a feature, a behavioral change — you have three options, in order of preference:

1. **Replace.** If the package is small and the need is small, absorb its functionality into your codebase. You own it now, fully visible, fully tested. Best when the package is a utility, not infrastructure.
2. **Fork.** Maintain a copy under your organization, load it via VCS repository, and rename the package (`"acme/forked-lib"` with a `replace` directive). Justified for substantial libraries where replacing means rewriting. Costs: you inherit maintenance forever, and drift from upstream grows until merging back becomes impossible. Set a review cadence ("check upstream quarterly") or the fork silently fossilizes.
3. **Patch upstream.** Submit a PR and temporarily pin around the issue. Slowest path, best outcome when it works — the fix lands for everyone and your diff disappears.

The decision hinges on one question: **who maintains this in eighteen months?** Replace means you do, knowingly. Fork means you do, with upstream guilt. Patch means they do, eventually. Choose consciously and write the choice down in the fork's README or the ticket.

## Autoload Optimization in the Docker Build

Chapter 11's Dockerfile showed the flag; here's why each piece matters:

```dockerfile
RUN composer install \
    --no-dev \
    --no-interaction \
    --no-progress \
    --prefer-dist \
    --optimize-autoloader \
    --classmap-authoritative \
    --apcu-autoloader
```

- **`--optimize-autoloader`** — converts PSR-4 lookups into a classmap: one array lookup instead of filesystem path resolution per class load.
- **`--classmap-authoritative`** — goes further: the classmap is *the* truth. Missing classes fail instantly instead of triggering filesystem scans. Fastest option; requires that nothing loads classes outside Composer's knowledge (true for well-built apps).
- **`--apcu-autoloader`** — caches resolution in APCu across requests; useful when classmap-authoritative isn't viable.
- **`--prefer-dist`** — downloads packaged archives instead of cloning repos: faster, smaller layers.

Do this in the final image stage only. During development you want plain PSR-4 so newly created classes work without rebuilding autoload maps.

Also verify in CI that the optimized autoloader resolves cleanly — `composer dump-autoload --optimize` failing silently in a cached Docker layer has ruined more than one Friday deploy.

---

Dependencies managed this way become boring: updates arrive as reviewed PRs, audits pass, and nothing surprises you at deploy time.

# Chapter 18. Symfony, Practically

Symfony is a large framework, and most confusion about it comes from mixing up its layers — treating components as bundles, or bundles as your application. This chapter is the shop's opinionated map: what each piece is for, what belongs where, and how the pieces wire into everything we've built so far (Chapters 5–6 security, Chapter 14 queues, Chapter 15 Doctrine).

For a narrative walkthrough of the framework itself, SymfonyCasts is the best resource available. This chapter assumes that context and focuses on how *we* use it.

## Application vs. Bundle vs. Component

Three distinct concepts, frequently conflated:

- **Components** are standalone libraries: `HttpFoundation` (request/response objects), `Routing`, `Validator`, `DependencyInjection`. Usable without the framework at all.
- **Bundles** are distribution packages for components — configuration, services, and integration glued together (e.g., `DoctrineBundle`). Bundles configure infrastructure; they're installed, not written.
- **Your application** is the code in `src/`: controllers, entities, services, voters. It is *not* a bundle.

The rule that follows: **you write application code; you install bundles; you almost never create new bundles.** Creating a bundle for your own app code was old-Symfony practice and produces indirection nobody can navigate. If you need to share code across projects, extract a plain Composer package (Chapter 17's path repositories) — not a bundle.

## Directory Layout and What Belongs in `src/`

The default layout is good; the discipline is keeping it honest:

```
src/
├── Controller/        # thin HTTP adapters only
├── Entity/            # Doctrine entities
├── Repository/        # query logic
├── Security/          # voters, authenticators
├── Service/           # domain and application services
├── Dto/               # input/output shapes
├── EventSubscriber/   # framework event hooks
├── Command/           # console entry points
└── Message/ + MessageHandler/   # Messenger
config/
├── packages/          # per-bundle config
├── routes.yaml
└── services.yaml      # DI wiring beyond autowiring
```

What belongs in `src/`: anything specific to this application. What doesn't: generic utilities that could serve any project (extract to a package), vendor configuration overrides (belong in `config/packages/`), and generated code (belongs wherever generation puts it).

The smell to watch for: `Controller/` accumulating business logic. Controllers are adapters — see next section.

## HTTP Kernel, Thin Controllers, DTOs

The **HTTP Kernel** is Symfony's request pipeline: request → routing → firewall → controller → response, with events fired at each stage (`kernel.request`, `kernel.controller`, `kernel.response`, `kernel.exception`) that listeners can intercept. You met one such listener in Chapter 8's problem+json exception handler.

**Controllers as thin adapters** means their entire job is:

1. Deserialize input into a DTO,
2. Call a service,
3. Serialize the result.

```php
#[Route('/api/v1/boards/{boardId}/cards', methods: ['POST'])]
public function create(
    string $boardId,
    Request $request,
    CreateCard $createCard,          // service = the actual use case
): JsonResponse {
    /** @var CreateCardInput $input */
    $input = $this->serializer->deserialize($request->getContent(), CreateCardInput::class, 'json');
    $errors = $this->validator->validate($input);
    if (count($errors) > 0) {
        throw new ValidationFailedException($input, $errors);   // → RFC 9457 via listener
    }

    $card = $createCard->execute($boardId, $input);

    return $this->json(CardView::from($card), Response::HTTP_CREATED);
}
```

No repository calls, no authorization decisions inline (those live in voters), no business rules. A controller you can read in ten seconds is doing its job.

**DTOs as input** deserve emphasis: never bind requests directly onto entities. An entity with a public `status` property bound from user JSON is a mass-assignment vulnerability waiting to happen. DTOs define exactly which fields the client may set; validation lives on them; mapping to domain happens explicitly in the service layer.

## Routing, Validators, Serializer

- **Routing:** attributes on controllers (`#[Route]`) keep route definitions beside handlers. Route parameters become method arguments; requirements (`requirements: ['boardId' => '[0-9a-f-]+']`) reject malformed input before your code runs.
- **Validator:** constraint attributes on DTOs (`#[Assert\NotBlank]`, `#[Assert\Length(max: 120)]`, custom constraints) plus `$validator->validate()`. Validation failures flow into the RFC 9457 error listener from Chapter 8 — field errors map directly into the `errors` extension.
- **Serializer:** converts between JSON and objects. Powerful, and easy to overuse: serialization groups (`#[Groups(['public', 'admin'])]`) sprinkled across entities turn "what does this endpoint return?" into an archaeology project across attribute annotations.

Opinionated policy: **the serializer handles input deserialization into DTOs and simple output views; complex API responses come from explicit view classes** (`CardView::from($card)`) where the shape is visible in one file. When you find yourself debugging which serialization group applies, you've crossed into magic — replace with explicit mapping.

## Dependency Injection

Symfony's DI container resolves your object graph automatically. The practices:

- **Autowiring by default:** type-hint dependencies in constructors; the container figures out the rest. Interface bindings for multiple implementations go in `services.yaml`.
- **Attributes for edge cases:** `#[Autowire(service: '...')]`, `#[AutowireParam(...)]`, or constructor injection of specific tagged services when autowiring can't guess.
- **Env processors** handle environment values with semantics: `%env(int:DB_POOL_SIZE)%`, `%env(bool:FEATURE_X)%`, `%env(resolve:DATABASE_URL)%`. They parse at compile time — no stringly-typed config at runtime.
- **Compiler passes: almost never.** A compiler pass rewrites the container at build time — powerful, opaque, and nearly always replaceable by service attributes, tags with autoconfiguration, or plain factory services. If you're writing one, first ask whether a tagged service locator would do. In years of Symfony work, legitimate compiler passes are rare enough to count on one hand.

The discipline underneath all of it: **constructor injection only, no service locators, no `$container->get()` in application code.** Dependencies visible in the constructor are testable and honest; hidden lookups are neither.

## Security Component: Authenticators, Firewalls, Voters

Wiring Chapters 5–6 into Symfony's model:

- **Firewalls** partition the app by path pattern: the stateless API firewall validates bearer tokens (Chapter 5); a web firewall might hold sessions for server-rendered pages. Each firewall has its own authenticator chain.
- **Authenticators** implement how credentials become a passport: JWT validation against the IdP's JWKS, API key lookup, form login. One per mechanism, registered under its firewall.
- **Voters** implement Chapter 6's authorization decisions: `BoardVoter` checking ReBAC relationships, called via `isGranted('EDIT', $board)` or `#[IsGranted]`. Object-level checks belong here; coarse authentication belongs in firewalls.

The division of labor stays clean: authenticators answer "who are you?", voters answer "may you do this?", and the query filters from Chapter 6 make unauthorized rows invisible regardless.

## Messenger: Chapter 14's Queues, Symfony-Native

Symfony **Messenger** maps our queue vocabulary onto PHP-native abstractions:

```php
// Message: a plain serializable object
final class ReindexBoard
{
    public function __construct(public readonly string $boardId) {}
}

// Handler: the consumer
final class ReindexBoardHandler implements MessageHandlerInterface
{
    public function __invoke(ReindexBoard $message): void { /* ... */ }
}
```

```yaml
# config/packages/messenger.yaml
framework:
    messenger:
        transports:
            async_amqp:
                dsn: '%env(RABBITMQ_DSN)%'
                retry_strategy:
                    max_retries: 5
                    delay_multiplier: 2
            failed: 'doctrine://default?queue_name=failed'
        routing:
            App\Message\ReindexBoard: async_amqp
```

Mapping from Chapter 14: messages are jobs, transports are brokers (AMQP transport backed by RabbitMQ), retries with backoff are configured per transport, and the `failed` transport is the dead-letter destination. Dispatching is `$bus->dispatch(new ReindexBoard($id))` — synchronous in tests (with the test transport), queued in production.

Combine with Chapter 14's outbox pattern: dispatch happens inside the Doctrine transaction via Messenger's Doctrine integration, or the outbox relay publishes afterward. Either way, handlers stay idempotent — Messenger redelivers on worker crash like any AMQP consumer.

## Doctrine Integration, Migrations, Fixtures

Chapter 15 covered Doctrine deeply; the Symfony-specific glue:

- **Migrations:** `bin/console doctrine:migrations:diff` generates from entity changes; humans review (always); CI verifies migrations apply cleanly from scratch and are append-only.
- **Fixtures:** development seed data via `Foundry` (the modern choice) or DoctrineFixturesBundle. Fixtures must be deterministic and fast — tests depend on them. Never let fixtures leak toward production data patterns (real emails, real-looking passwords people might reuse).
- **Entity manager scope:** one EM per request by default; long-running workers call `$em->clear()` periodically (Chapter 15's identity-map trap).

## Console Commands as First-Class Entry Points

Console commands aren't dev toys — they're production entry points alongside HTTP:

```php
#[AsCommand(name: 'app:invoices:retry-failed')]
final class RetryFailedInvoicesCommand extends Command
{
    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        // same services as controllers — same domain, same rules
    }
}
```

Uses in this stack: cron-driven maintenance (nightly digests, cleanup jobs), one-shot operational scripts (backfills, data repairs), and long-running workers (`messenger:consume`). Discipline: commands get the same treatment as endpoints — validation, authorization context where relevant, structured output, proper exit codes (0 success, nonzero failure so cron and orchestrators detect breakage). A command that always exits 0 is invisible to monitoring.

## Config: `config/`, Secrets, and When `.env` Is a Lie

Configuration layers, in resolution order: environment variables override `.env` files, which feed `config/packages/*.yaml`.

The honest framing: **`.env` files are a development convenience, not a configuration system.** In production, `.env` should not exist — values come from the environment (container orchestration, secret managers). Teams that ship `.env` to production end up with stale values that "work until someone restarts," and secrets sitting in deploy artifacts.

For genuinely sensitive values, Symfony's **secrets vault** (`APP_RUNTIME` secrets stored encrypted in the repo, decrypted at runtime with a master key held outside git) offers a middle ground for small teams. Larger setups use Vault/AWS Secrets Manager with env processors bridging them.

Rule of thumb: if a value differs between environments, it's config (env vars). If it's identical everywhere but sensitive, it's a secret (vault). If it's identical everywhere and harmless, it's code.

## Events vs. Messenger vs. Domain Events

Three mechanisms, three scopes — choosing wrong creates coupling bugs:

- **EventDispatcher (Symfony events):** in-process, synchronous, same-request side effects. Right for framework integration points (kernel events) and immediate reactions within one process. Listeners run inline — a slow listener slows the request.
- **Messenger:** cross-process, asynchronous, durable. Right for work that must survive the request: emails, indexing, webhooks (Chapter 14).
- **Domain events:** a design concept, not a tool — records of "something happened" emitted by the domain ("OrderPlaced"). They may be dispatched synchronously (in-process listeners) or persisted to the outbox and published via Messenger. The distinction matters because domain events describe *business facts* while framework events describe *technical moments*.

Decision rule: does the reaction need to happen before the response returns? Synchronous dispatcher. Must it survive a crash? Messenger/outbox. Is it describing business meaning? Model it as a domain event first, then choose delivery.

## Forms: When the SPA Owns the Form

Classic Symfony Forms (`FormBuilder`, CSRF tokens, server-rendered widgets) assume Symfony renders the HTML. In our architecture, Vue owns every form surface — so the Form component mostly stays unused, and that's fine.

What remains relevant:

- **CSRF:** SPA + bearer-token auth (Chapter 7) means forms don't need synchronizer tokens; cookie-session architectures do.
- **Validation:** still server-side, on DTOs, via the Validator component — the SPA's client-side validation is UX (Chapter 6's rule), never enforcement.
- **File uploads:** handled as explicit endpoints with size/type validation, not Form magic.

If you later add server-rendered pages (Chapter 3's progressive-enhancement tier), Forms earns its place there. Until then, skip it rather than half-use it.

## Profiler in Dev; Never in Prod

The **Web Profiler** toolbar is Symfony's best development feature: per-request timelines, database queries with duplicates flagged, cache hits, logged messages, security context. Use it constantly during development — it makes N+1 queries and slow kernel events visible without any setup.

And it must never exist in production images. The profiler exposes configuration, container contents, request data, and sometimes tokens — a full information-disclosure incident behind one URL. Multi-stage builds (Chapter 11) exclude it naturally: `symfony/profiler-pack` belongs in `require-dev`, and production installs run `--no-dev`. Verify in CI that `/ _profiler /` returns 404 against the production build.

---

With the backend framework mapped end to end, we turn to the frontend's equivalent: Vite, TypeScript, and the Vue build pipeline — configured with the same discipline.

# Chapter 19. PHP Standards and Enforcement

Code style debates are the most expensive conversations in software — hours of review time arguing about brace placement, repeated in every PR, forever. The fix isn't consensus; it's automation. Machines enforce style, humans review substance.

This chapter maps the standards landscape that matters, then configures the three tools that make enforcement automatic: php-cs-fixer, PHPStan, and Rector.

## The PSR Map That Matters

PHP-FIG's PSRs (PHP Standard Recommendations) split into categories worth different levels of your attention:

### Standards that shape how you write

- **PSR-1 (Basic Coding Standard):** the floor — files declare symbols or cause side effects, not both; classes use `StudlyCaps`; namespaces match directory structure. You'll never think about it because everything below implies it.
- **PSR-4 (Autoloading):** the namespace-to-directory convention from Chapter 16. Every Composer project runs on it.
- **PSR-12:** the comprehensive code-style standard — indentation, line length, method ordering. It's frozen and officially superseded; treat it as history. Its successor is where active development happens (next section).

### Interfaces you'll consume, not implement

These matter as contracts between packages:

- **PSR-7 / PSR-15 / PSR-17 (HTTP):** immutable Request/Response value objects (7), middleware signatures (15), and their factories (17). Symfony doesn't natively run on PSR-7, but bridges exist, and any HTTP library you pull into Node-adjacent PHP work will speak it. Recognize the shapes.
- **PSR-11 (Container):** `get($id)` — the interface behind every DI container. Relevant when writing reusable packages that shouldn't couple to Symfony's container specifically.
- **PSR-3 (Logging):** `LoggerInterface` with `debug/info/warning/error` methods. Symfony uses it natively; type-hint it rather than concrete loggers so tests can swap implementations freely.
- **PSR-6 / PSR-16 (Caching):** the full and simple caching interfaces. Symfony's cache component implements both; PSR-16 (`CacheInterface` with `get`/`set`) is what you should type-hint for plain key-value caching.
- **PSR-20 (Clock):** `now()` returning a `DateTimeImmutable`. Underrated and quietly important: injecting a clock instead of calling `new DateTime()` everywhere makes time-dependent logic testable (Chapter 14's backoff windows, Chapter 9's TTLs). If your service needs to know the time, inject `ClockInterface`.

The pattern across all of these: **type-hint interfaces, not implementations**, and your code stays portable across frameworks and testable by construction.

## PER Coding Style: The Living Standard

**PER (PHP Evolving Recommendations)** is FIG's new mechanism for standards that update over time. **PER Coding Style** is the living successor to the frozen PSR-12 — same spirit, actively maintained, incorporating modern syntax (enums, readonly properties, constructor promotion, attributes) that PSR-12 predates.

Practical implications:

- New projects target **PER Coding Style 2.x+**, not PSR-12.
- **php-cs-fixer targets PER now.** Its rule sets have moved; if your `.php-cs-fixer.php` still references PSR-12 rulesets explicitly, you're formatting for a retired standard.
- Expect incremental revisions — that's the point of PER. When a new minor drops, updating your fixer config is a one-line change plus a formatted diff, reviewed like any PR.

You don't need to read the spec end-to-end; the tool encodes it. But know which standard you're following, because "we follow PSR-12" and "we follow PER" produce different diffs on the same file.

## php-cs-fixer: Config Committed, CI Enforced, Debate Ended

**PHP-CS-Fixer** rewrites source files to conform to configured rules. The setup that ends style debates permanently:

```php
// .php-cs-fixer.php
return (new PhpCsFixer\Config())
    ->setRules([
        '@PER-CS' => true,
        '@Symfony' => true,
        'declare_strict_types' => true,
        'ordered_imports' => ['sort_algorithm' => 'alpha'],
        'final_class' => true,
    ])
    ->setFinder(
        PhpCsFixer\Finder::create()
            ->in([__DIR__ . '/src', __DIR__ . '/tests'])
    );
```

Three commitments make it work:

1. **Config committed to the repo.** Style lives in version control, identical for every contributor. No personal preferences applied via editor plugins fighting CI.
2. **CI job enforces it:** run in check mode (`--dry-run --diff`); violations fail the build. Formatting drift cannot reach `main`.
3. **Humans stop reviewing style.** Chapter 6's review guidance said this already; this tool makes it real. Any style comment in review gets one answer: "the fixer owns this."

Run `vendor/bin/php-cs-fixer fix` before committing (optionally via git hook — see the end of this chapter). The first run produces one large mechanical diff; land it alone, with no functional changes riding along.

## PHPStan: Strict Rules and Honest Baselines

**PHPStan** performs static analysis — reading your code without running it to find type errors, dead code, unreachable branches, and contract violations. It's the single highest-leverage quality tool in the PHP ecosystem.

Configuration:

```yaml
# phpstan.neon
parameters:
    level: max          # start here for new projects
    paths:
        - src
        - tests
```

Levels run 0–10. Level 10 demands exhaustive null-safety and full type coverage — genuinely achievable on a well-typed codebase, brutal on an inherited mess.

Policy for this book:

- **New projects start at `max` (level 10).** Writing to satisfy it costs little when done from the start and pays off immediately — half of runtime bugs are type errors PHPStan catches statically.
- **Inherited codebases climb gradually.** Start at whatever passes, then ratchet up per-module.
- **Docblocks carry types where inference can't** — array shapes (`@param array{title: string, due: \DateTimeImmutable}`) turn untyped arrays into checked structures. This is how level-10 analysis stays practical without heavy DTO ceremony.

### Baselines as debt with expiry

On existing code, PHPStan offers a **baseline**: a generated file listing every current violation, excluded from reporting, letting you enforce the rules only on *new* code.

```bash
vendor/bin/phpstan analyse --generate-baseline
```

Baselines are valuable and dangerous. Dangerous because they become permanent: nobody regenerates them, and 4,000 ignored errors fossilize into "how we work." The policy here: **a baseline is debt with an expiry date.** Commit it alongside a tracked ticket ("reduce baseline to zero by Q3"), review its size in quarterly hygiene work, and shrink it continuously — every touched file should leave the baseline smaller. A baseline that only ever grows means enforcement has quietly stopped.

## Rector: Automated Upgrades, Human-Reviewed

**Rector** performs automated refactoring at scale — migrating code across PHP versions and applying framework rulesets:

```bash
# Upgrade codebase from PHP 8.3 patterns to 8.5
vendor/bin/rector process src --set php85

# Symfony-specific upgrades
vendor/bin/rector process src --config rector-symfony.php
```

What it does well:

- **Language migrations:** replacing deprecated constructs, adopting new syntax (`array_first()`, property hooks) mechanically across thousands of lines.
- **Framework upgrades:** Symfony publishes Rector rulesets per version — much of the tedious deprecation-chasing becomes automated.

Two disciplines keep Rector safe:

1. **Review the diff; don't blind-apply.** Rector's transformations are usually correct and occasionally surprising. Read the generated diff like any PR — it's large but shallow, mostly mechanical repetition. Run the test suite after; tests exist precisely to validate bulk changes.
2. **Separate upgrade PRs from feature work.** A Rector pass touches hundreds of files; mixing it with features makes both unreviewable. Ship upgrades standalone.

Used this way, major-version upgrades stop being quarter-long projects and become scheduled maintenance — the difference between "we're stuck on PHP 8.2" and "we're always current."

## What's Local vs. What Gates Merges

Not every check needs to block every push. Split enforcement by cost and consequence:

| Check | Where | Why |
|---|---|---|
| php-cs-fixer fix | Local hook (optional), pre-commit | Fast feedback; saves a CI round-trip |
| PHPStan | **Merge gate** (required CI job) | Correctness issues must never reach main |
| Tests + coverage | **Merge gate** | Obvious |
| `composer audit` | **Merge gate** + scheduled weekly | New CVEs block merges; existing ones surface periodically |
| Rector | Scheduled / manual, separate PRs | Too disruptive for routine gating |
| Baseline shrinkage | Quarterly review | Hygiene cadence |

Notes on each layer:

- **Local hooks are convenience, never security.** Git hooks live in `.git/`, aren't cloned, and can be skipped with one flag. Use them (via a tool like CaptainHook with a committed config, installed by `composer install`) for fast feedback on style and quick static checks — but assume nothing about whether they ran. All *enforcement* happens in CI, which nobody can bypass.
- **Keep the merge gate fast.** A 20-minute pipeline trains people to bypass review. Parallelize jobs, cache vendor directories aggressively, and reserve heavyweight checks (full integration suites) for the final stage.
- **Scheduled scans complement gates.** Weekly full audits catch things that drifted in while nobody was looking — abandoned packages, newly published CVEs against locked versions.

The philosophy underneath: **automation handles conformance, humans handle judgment.** Style, types, and known vulnerabilities are machine-checkable — checking them manually wastes the scarcest resource in engineering, which is reviewer attention. Spend that attention on design, correctness, and the security reasoning from Chapter 7 that no linter can perform.

---

# Chapter 20. PHPUnit as a Design Tool

Tests verify behavior — but writing them does something else just as valuable: it *measures your design*. Code that's hard to test is telling you something. Hidden dependencies, god objects, entangled concerns — the test suite surfaces these before production does.

This chapter covers PHPUnit mechanics and then the judgment that separates useful suites from theater: which doubles to use, where the kernel belongs, and what never gets mocked.

## Anatomy: Test Case, Assertions, Providers, Attributes

A PHPUnit test lives in a class extending `TestCase`, one method per scenario:

```php
final class MoneyTest extends TestCase
{
    public function testAdditionKeepsCurrency(): void
    {
        $five = new Money(500, Currency::USD);
        $three = new Money(300, Currency::USD);

        $result = $five->add($three);

        $this->assertSame(800, $result->amountCents);
        $this->assertSame(Currency::USD, $result->currency);
    }
}
```

Core conventions worth internalizing:

- **One concept per test method**, named for the behavior: `test_refund_fails_after_90_days`, not `test_refund2`. The name is the first failure report readers see.
- **Arrange–Act–Assert** structure keeps tests scannable: build inputs, invoke, verify. If Arrange dwarfs everything, extract builders.
- **`assertSame` over `assertEquals`** by default — strict identity/type checking catches coercion bugs that loose equality hides.

### Data providers

When scenarios differ only in input/output, data providers eliminate copy-paste:

```php
#[DataProvider('invalidTitlesProvider')]
public function testRejectsInvalidTitles(string $title): void
{
    $this->expectException(ValidationFailed::class);
    new CardTitle($title);
}

public static function invalidTitlesProvider(): array
{
    return [
        'empty string' => [''],
        'too long' => [str_repeat('x', 121)],
        'whitespace only' => ['   '],
    ];
}
```

Each provider row reports as its own test case — you get combinatorial coverage with named failures, at zero duplication cost.

### Attributes

PHPUnit 10+ moved configuration from docblocks to attributes: `#[DataProvider]`, `#[TestWith]`, `#[Depends]`, `#[Group('slow')]`, `#[RequiresPhp('>= 8.5')]`. Use attributes everywhere; docblock metadata is legacy and invisible to static analysis (Chapter 19's PHPStan can't check what it can't see).

## Test Doubles: A Precise Vocabulary

"Mock" has become generic slang for every stand-in, and the vagueness causes real misuse. The precise terms:

- **Dummy** — fills a parameter list; never used. `new NullLogger()` as a required argument.
- **Stub** — returns canned answers to make code run. "The gateway returns success."
- **Fake** — a working lightweight implementation. In-memory repository standing in for Doctrine; array-backed cache standing in for Redis.
- **Mock** — verifies *interactions*: "the email sender was called exactly once with this address." Fails when the expected call doesn't happen.
- **Spy** — records interactions for post-hoc verification rather than failing upfront.

PHPUnit's builder creates stubs and mocks alike:

```php
$gateway = $this->createMock(PaymentGateway::class);
$gateway->method('charge')
    ->willReturn(new ChargeResult('ok'));

// Interaction verification:
$mailer = $this->createMock(MailerInterface::class);
$mailer->expects($this->once())
    ->method('send')
    ->with($this->callback(fn (EmailMessage $m) => $m->to === 'dana@example.com'));
```

### When a hand-written fake beats the mock builder

Mocks scale poorly: each test re-describes behavior, and suites drown in setup noise. For heavily-used interfaces, a hand-written fake pays for itself:

```php
final class InMemoryBoardRepository implements BoardRepository
{
    /** @var array<string, Board> */
    public array $boards = [];

    public function find(string $id): ?Board { return $this->boards[$id] ?? null; }
    public function save(Board $board): void { $this->boards[$board->id()] = $board; }
}
```

Rules of thumb:

- **Stubs for trivial cases**, inline via the mock builder.
- **Fakes for interfaces touched by many tests** — repositories, caches, clocks, gateways. Write once in `tests/Fake/`, reuse everywhere. They also serve as living documentation of the interface contract.
- **Mocks only when interaction itself is the requirement.** "An email was sent" is a genuine behavioral claim. "The repository was called twice" usually isn't — it asserts implementation details and breaks on harmless refactors.

## Isolated Unit Tests: No Kernel

Unit tests instantiate classes directly — no Symfony container, no booting, no I/O:

```php
public function testExpiredMembershipCannotEdit(): void
{
    $policy = new BoardPolicy(new FrozenClock(...));
    $decision = $policy->canEdit($actor, $board);

    self::assertFalse($decision->allowed());
}
```

These are the fastest, most stable tests you own — milliseconds each, runnable constantly. Design implications follow directly from wanting more of them:

- **Dependencies arrive through constructors as interfaces** (PSR interfaces from Chapter 19 help).
- **Time comes from an injected clock**, not `new DateTimeImmutable()`.
- **No static calls, no singletons, no hidden globals.**

When a class resists unit testing, don't reach for mocking heroics — refactor until construction is honest. The resistance was the design feedback.

## Kernel / WebTestCase Integration Tests

Below unit tests sit integration tests that boot the full container and exercise HTTP + security + persistence together:

```php
final class ArchiveBoardTest extends WebTestCase
{
    public function testMemberCanArchiveOwnOrgsBoard(): void
    {
        $client = static::createClient();
        $user = $this->loginAs('carlos');           // authenticates through real security

        $client->request('POST', '/api/v1/boards/4721/archive');

        self::assertResponseStatusCodeSame(200);
        self::assertJsonContains(['status' => 'archived']);
    }

    public function testOutsiderGetsNotFound(): void
    {
        $client = static::createClient();
        $this->loginAs('eve');

        $client->request('POST', '/api/v1/boards/4721/archive');

        // 404, not 403 — existence must not leak (Chapters 6 & 7)
        self::assertResponseStatusCodeSame(404);
    }
}
```

These tests exercise what units cannot: routing, serialization, the firewall chain, voters, Doctrine transactions, and event listeners (including Chapter 8's problem+json mapping). They're slower — reserve them for the seams, and keep business logic coverage in fast unit tests.

## Database Tests: Transactions, Fixtures, No Real Queue

Database-backed tests need three disciplines:

**Transactions.** Wrap each test in a transaction rolled back at teardown — every test starts from a clean slate at full speed. Caveat from Chapter 9: code under test that manages its own transactions (Messenger handlers) breaks this pattern; use database reset or per-test schema reload there.

**Fixtures.** Deterministic seed data built programmatically per test group — Foundry factories create exactly the entities a scenario needs:

```php
$board = BoardFactory::createOne(['organization' => OrgFactory::new()]);
```

Shared fixture files ("the big YAML dump") rot silently; factories colocated with tests stay alive.

**No real queue.** Integration tests must not publish to RabbitMQ or enqueue BullMQ jobs. Configure the test environment with Messenger's `test` transport (collects messages in memory for assertions) or fake queue bindings. A test suite that sends emails and enqueues webhooks is an incident generator, not a safety net.

## Testing Voters, Handlers, and Commands

Three Symfony-specific patterns:

**Voters:** test both the decision and the denial path against seeded relationships — this feeds Chapter 6's matrix tests directly:

```php
public function testAdminMembershipGrantsArchive(): void
{
    $voter = new BoardVoter($this->membershipRepo);
    $token = $this->tokenFor($this->orgAdmin);

    self::assertTrue($voter->voteOnAttribute('ARCHIVE', $this->board, $token));
}
```

**Messenger handlers:** inject fakes for side-effect ports (mailer, indexer), assert state changes and dispatched messages. Use the test transport to assert "handler dispatched FollowupEmail" rather than mocking the bus internals. And always include the idempotency test from Chapter 14: process the same message twice, assert no duplicate effect.

**Console commands:** `CommandTester` drives commands headlessly:

```php
$tester = new CommandTester($command);
$tester->execute([]);

self::assertStringContainsString('3 invoices retried', $tester->getDisplay());
self::assertSame(Command::SUCCESS, $tester->getStatusCode());
```

Assert the exit code — Chapter 18 made exit codes the monitoring contract; untested nonzero paths break cron alerting invisibly.

## Coverage as Spotlight, Not Target

Coverage measures which lines executed during tests. Two failure modes surround it:

- **Target-number theater:** mandate 80%, and engineers write assertion-free tests that execute lines without verifying anything. You get the number and lose the safety.
- **Dismissal:** "coverage is meaningless" — true as a target, false as a signal.

Useful stance: **coverage is a spotlight for finding dark corners.** Review it after features ship: uncovered branches in payment handling mean untested money paths — fix that. Uncovered lines in a DTO getter? Ignore it. Track *changed-code coverage* (did tests cover what this PR touched?) rather than a global percentage, and treat gaps in critical paths as review findings, not dashboard metrics.

## What Not to Mock

Two rules prevent most mock-induced suffering:

**Never mock the thing under test.** Partially mocking a class to isolate one method (`getMockBuilder` on the system under test) produces a test asserting nothing about real behavior. If a method needs isolation to be tested, extract it into a collaborator — design feedback again.

**Never mock in ways that only test the mock.** The classic:

```php
$repo->expects($this->once())
     ->method('find')
     ->willReturn($board);

$result = $service->renameBoard($repo, 4721, 'New name');
$this->assertEquals('New name', /* ...what the mock returned... */);
```

If assertions verify values the mock itself supplied, the test proves the wiring of your assumptions, not the system's behavior. When mocking an ORM repository leads here, stop: use the in-memory fake, or drop to an integration test with a real database (transaction-wrapped). Mocking Doctrine deeply is a known anti-pattern — you'd be testing Doctrine's API surface against your imagination.

The underlying principle: **mock at architectural boundaries you own** (gateways, mailers, external APIs), **never at seams inside your own design** (repositories, domain services) — those get fakes or real implementations.

---

A well-built suite makes refactoring fearless and upgrades mechanical — Rector passes from Chapter 19 land safely precisely because tests exist.

---

# Part VI — The TypeScript Stack

# Chapter 21. TypeScript and the Shared `tsconfig`

TypeScript is often introduced as "JavaScript with a linter for types" — something you can gradually ignore. That framing wastes the tool. TypeScript is a *language*: a compile-time type system that makes entire bug categories unrepresentable, exactly as PHP's type system does on our backend (Chapter 16).

This chapter covers how we configure it across the monorepo, how types meet the API boundary, and where inference should be trusted instead of hand-writing.

## Why TypeScript Is the Language, Not a Linter

A linter reports suspicious patterns; a type system *proves* properties about your code before it runs. The difference shows up in refactoring:

- Rename a field in an API response with proper typing, and every affected call site becomes a compile error — you fix them mechanically and ship confidently.
- Do the same in plain JavaScript, and the breakage surfaces at runtime, in production, for whichever user hits that code path first.

The practical consequences of treating TS as the language:

- **No implicit `any` survives review.** Untyped code isn't "flexible"; it's unchecked.
- **Types document intent** better than comments do, because the compiler enforces them.
- **The API contract (Chapter 8) extends into the frontend**: if the OpenAPI spec says `dueDate` is a string, the Vue app knows it at compile time.

The cost is real but front-loaded: stricter configuration means more upfront annotation. The payoff compounds — every later change gets compiler verification for free.

## Strictness We Require

Modern `tsconfig` strictness flags, and why each earns its place:

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

- **`strict: true`** — the umbrella: `strictNullChecks`, `noImplicitAny`, `strictFunctionTypes`, and friends. Non-negotiable; this is the baseline everyone means by "TypeScript."
- **`noUncheckedIndexedAccess`** — indexing an array or record returns `T | undefined`. Array access *can* fail; pretending otherwise hides the most common runtime crash in JavaScript. This flag forces honest handling:
  ```ts
  const card = cards[id];        // Card | undefined
  if (!card) return null;        // forced to decide
  ```
- **`exactOptionalPropertyTypes`** — distinguishes `{ name?: string }` from `{ name: string | undefined }`. Without it, assigning explicit `undefined` sneaks past optional-property checks; with it, "absent" and "present-but-null" are different states, which matters enormously when serializing to JSON APIs.
- **`noImplicitOverride` / `noFallthroughCasesInSwitch`** — cheap guards against classic footguns.

Apply these per project as applicable: the SPA wants all of them; Node services want them plus Node-specific lib settings. The shared config (next section) makes consistency automatic.

## The Shared `tsconfig`: Project References

Multiple projects in one repo — the Vue app, Node workers, test suites — need consistent compiler settings without copy-paste drift. TypeScript **project references** solve this with a layered structure:

```
tsconfig.base.json     # shared compilerOptions
tsconfig.json          # solution file referencing all projects
apps/
  web/tsconfig.json    # extends base, adds DOM libs
services/
  worker/tsconfig.json # extends base, adds Node libs
vitest.config.ts       # test project reference
```

```jsonc
// tsconfig.base.json — the single source of truth
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "skipLibCheck": true
  }
}
```

```jsonc
// apps/web/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": { "lib": ["ES2023", "DOM", "DOM.Iterable"] },
  "include": ["src"]
}
```

Benefits beyond DRY: references give the compiler a whole-project graph (faster incremental builds), enforce clean module boundaries between packages, and let each environment declare its own globals (`DOM` vs. Node) so browser APIs can't leak into workers and vice versa.

## Types at the API Boundary: Generated vs. Parsed

Here's the trap that catches every team: **compile-time types say nothing about runtime data.**

```ts
interface OrderResponse { total: number }

const res = await fetch('/api/orders/1');
const order: OrderResponse = await res.json();   // ← lie!
// Nothing verified that res.json() matches OrderResponse.
// If the API returned {total: "12.00"}, order.total.toFixed() explodes.
```

The cast compiles fine and protects nothing. Types describe what *should* arrive; only runtime validation confirms what *did*. Two complementary approaches:

### Generated types from OpenAPI

Chapter 8 made OpenAPI the contract; tools like `openapi-typescript` generate TS interfaces directly from the spec:

```bash
npx openapi-typescript ./openapi.yaml -o src/api/schema.d.ts
```

The generated types stay synchronized with the contract by construction — regenerate in CI whenever the spec changes, and drift becomes impossible. This is the default for response shapes.

### Runtime parsers at the edge

Generated interfaces still don't validate. For data crossing the trust boundary — API responses, WebSocket messages, anything user-influenced — pair types with schema parsers (**zod** or **valibot**) that validate at runtime *and* infer their static types:

```ts
import { z } from 'zod';

const CardSchema = z.object({
  id: z.string(),
  title: z.string().max(120),
  dueDate: z.string().datetime().nullable(),
});

type Card = z.infer<typeof CardSchema>;   // static type derived from parser

export async function fetchCard(id: string): Promise<Card> {
  const res = await fetch(`/api/v1/cards/${id}`);
  return CardSchema.parse(await res.json());   // throws ProblemDetails-shaped error on mismatch
}
```

Policy: **parsers at the edge, generated types inside.** Validate once where data enters the application; everywhere downstream trusts the inferred types. Hand-write schemas only for payloads not covered by the OpenAPI spec (WebSocket frames, third-party responses).

## `unknown` vs. `any`; Branded Types for IDs

### `unknown` over `any`

- **`any`** disables checking: any operation is allowed on it, errors propagate silently. It's a hole in the type system.
- **`unknown`** is the safe top type: "something, unknown what." You must narrow before use — via `typeof`, `instanceof`, or a parser.

```ts
function handleEvent(data: unknown) {
  // data.foo  → compile error. Good.
  const parsed = WebhookPayload.parse(data);   // narrow explicitly
}
```

Rule: `any` appears only where a library forces it, wrapped immediately in a parse boundary. Everything else is `unknown`.

### Branded types for IDs

Every ID in the system is a string — which lets you pass a `userId` where a `boardId` belongs, and the compiler shrugs. **Branded types** make IDs distinct:

```ts
type Brand<T, B extends string> = T & { readonly __brand: B };

export type UserId = Brand<string, 'UserId'>;
export type BoardId = Brand<string, 'BoardId'>;

declare function getUser(id: UserId): Promise<User>;

getUser(boardId);   // compile error — brands don't match
```

The brand exists only at compile time (zero runtime cost); the constructor functions that create branded values become the single validation point. Apply this wherever two identifiers share a representation — it converts a whole class of "wrong ID passed" bugs into compile errors.

## Module Resolution, `paths`, and What Breaks Vite vs. Node

Module resolution differs by runtime, and misconfiguration here produces "works in dev, breaks in build" mysteries:

- **Vite (the SPA)** uses esbuild/rollup semantics — set `"moduleResolution": "bundler"` in the base config. It resolves extensions flexibly and expects ESM.
- **Node services** run real CommonJS or ESM — `"moduleResolution": "node16"`/`"nodenext"` enforces Node's actual rules, including mandatory `.js` extensions in relative imports under ESM.

Using bundler resolution for Node code compiles fine and fails at runtime — the classic divergence.

**Path aliases** (`paths`) map logical imports to locations:

```jsonc
{
  "baseUrl": ".",
  "paths": { "@/*": ["src/*"] }
}
```

They keep imports stable during refactors — but each runtime needs matching configuration: Vite reads `resolve.alias` from `vite.config.ts`; Vitest mirrors it; tsconfig-paths or the bundler handles Node. An alias configured in tsconfig alone compiles cleanly and breaks the build. Rule: **every alias exists in three places or zero places** — tsconfig, the bundler config, and the test runner config, kept in sync by the shared setup.

Also mind `verbatimModuleSyntax`: it forces explicit `import type` for type-only imports, which keeps type imports out of runtime bundles — required for correct tree-shaking and isolated module transforms.

## When to Write Types, When to Let Inference Work

TypeScript's inference is excellent. Over-annotating is its own anti-pattern:

```ts
// Redundant — inference already knows
const cards: Card[] = [];
const total: number = prices.reduce((a, b) => a + b, 0);

// Valuable — public boundaries and complex shapes
export function createBoard(input: CreateBoardInput): Promise<Board> { ... }
```

Guidelines:

- **Annotate exported function signatures** — they're contracts other modules read; inference there leaks implementation details into hover-tips and error messages.
- **Annotate object literals returned from factories** when the shape is non-obvious, catching missing fields at the literal rather than at consumption.
- **Let locals infer.** `const x = f()` needs no annotation unless `f`'s return is too loose.
- **Prefer inference from parsers** (`z.infer<typeof Schema>`) over hand-maintained interfaces — one source of truth, always synchronized.
- **Never annotate to silence an error** with `as`. A cast that fixes a complaint usually marks a real modeling problem; investigate before suppressing.

The balance point: types live at **boundaries** (API edges, module exports, shared contracts), inference flows through **internals**. Written that way, annotations stay few, meaningful, and maintained — while the compiler does the rest.

---

# Chapter 22. Node.js, npm, and Server Microframeworks

Node occupies a specific, deliberately narrow role in our architecture (Chapter 1): glue services — webhook receivers, BFFs, queue workers, internal tools. This chapter covers the runtime knowledge those services need, how we manage packages, and how to choose between the three microframeworks you'll actually encounter.

The boundary matters as much as the tooling: **Node services here are small and single-purpose. The moment a Node service grows a database schema and domain rules of its own, it has become a second monolith — and that's a design failure, not a success.**

## Node as a Runtime

### The event loop

Node runs JavaScript on a **single thread** with an event loop: I/O operations (network, disk) hand work to the OS and register callbacks; the loop picks up completed events and runs their handlers one at a time. This is why Node handles thousands of concurrent connections cheaply — each idle connection costs almost nothing.

The corollary is equally important: **any CPU-bound work blocks everything.** A tight loop parsing a 50MB JSON payload freezes every other request on that process. Consequences for our services:

- Webhook receivers and BFFs do I/O-shaped work — perfect fit.
- Image processing, PDF generation, heavy crypto belong in **worker threads** or, more often in our stack, in BullMQ jobs processed by dedicated worker processes (Chapter 14) — keeping request-handling processes responsive.
- **Worker threads** exist for genuine parallelism (`worker_threads` module), but they're rarely needed here: separate *processes* are simpler to deploy and scale horizontally. Reach for threads when sharing state across a computation matters; otherwise prefer more containers.

### ESM

Modern Node runs ECMAScript modules natively. Our convention:

- `"type": "module"` in `package.json`.
- Explicit file extensions in relative imports under ESM (`import { x } from './util.js'`) — enforced by TypeScript's `node16`/`nodenext` resolution from Chapter 21.
- Top-level `await` works, which simplifies service bootstrap.

## npm: Lockfile, Workspaces, Engines, Scripts

Mostly the mirror of Chapter 17's Composer discipline:

- **`package-lock.json` is committed**, and CI/production install with **`npm ci`** — clean install exactly from the lockfile, faster and stricter than `npm install`. Never run `update` in a pipeline.
- **`engines` declares the runtime:**

```json
{
  "engines": { "node": ">=22", "npm": ">=10" }
}
```

With an `.npmrc` containing `engine-strict=true`, mismatched local environments fail loudly instead of subtly misbehaving. Docker images pin the major version to match (Chapter 11).

- **Workspaces** — if the repo holds multiple Node packages (shared types, several microservices), npm workspaces hoist dependencies and link local packages:

```json
{
  "workspaces": ["services/*", "packages/*"]
}
```

Use them when two or more Node projects share code; skip them for a single-service repo where they're ceremony.

- **Scripts** are the task runner: `dev`, `build`, `test`, `lint`. Keep them thin — orchestration belongs in CI configuration, not nested script chains nobody can trace.

Supply-chain rules from Chapters 2 and 7 apply unchanged: lockfiles reviewed in PRs, `npm audit` gating merges, new dependencies justified before adoption.

## Choosing a Microframework

Three frameworks cover the landscape. Know all three — you'll read Express code regardless of what you write.

### Express — ubiquitous, middleware soup

Express is the default answer in tutorials and legacy code. Its model is a chain of middleware functions, each receiving `(req, res, next)`:

```ts
app.use(cors());
app.use(express.json());
app.post('/hooks/stripe', authenticateStripe, handler);
```

Strengths: enormous ecosystem, every middleware ever written exists for it, every Node developer knows it. Weaknesses: no built-in validation or serialization schemas, error handling relies on convention (four-argument middleware), and untyped by modern standards. **Know Express to read it; don't start new services here without a reason.**

### Fastify — schema-first, fast, plugin-based

Fastify treats JSON Schema as a first-class citizen:

```ts
const app = Fastify({ logger: true });

app.post('/cards', {
  schema: {
    body: CreateCardSchema,        // validates input
    response: { 201: CardViewSchema },  // strips output
  },
}, async (request, reply) => {
  return createCard(request.body);
});
```

Schemas validate input, serialize output (fast — precompiled), generate documentation, and feed TypeScript inference via type providers. Structured logging ships by default (pino). Plugins encapsulate concerns cleanly. For our BFF and internal APIs — real HTTP services with payloads worth validating — **Fastify is the default choice.**

### Hono — small, typed, edge-shaped

Hono is minimal, TypeScript-native, and runs anywhere: Node, Bun, Deno, Cloudflare Workers, other edge runtimes. Same routing ergonomics, tiny footprint:

```ts
import { Hono } from 'hono';

const app = new Hono()
  .post('/webhooks/stripe', zValidator('json', StripeEventSchema), async (c) => {
    const event = c.req.valid('json');
    await enqueue(event);
    return c.json({ received: true });
  });
```

Its RPC-style typed client also makes it pleasant for internal APIs consumed by other TS code. Choose Hono when the service is small, hot-path, or potentially edge-deployed — webhook receivers fit perfectly. Choose Fastify when schema-driven validation and ecosystem plugins matter more than footprint.

Decision summary: **Fastify for full-featured HTTP services, Hono for lean ones, Express only when reading or extending existing code.**

## Where Node Is Allowed

Restating the boundary from Chapter 1 with concrete examples:

| Allowed | Not allowed |
|---|---|
| BullMQ workers (thumbnails, webhooks, email dispatch) | Business domain logic (refunds, billing rules) |
| BFF shaping responses for the SPA | Owning MySQL tables as system of record |
| Webhook receivers validating + forwarding | Reimplementing Symfony's API beside itself |
| Internal admin APIs (short-lived, low-stakes) | A "Node rewrite" of the backend |
| Real-time relay (websocket fan-out) | Long-lived domain models in Prisma |

The test for any proposed Node service: **does it transform, route, relay, or offload — or does it decide?** Transformation and relay are Node's job. Deciding (business rules, state transitions, money) belongs in Symfony, next to the rest of the domain. A Node service that starts accumulating its own entity models and business rules should trigger a conversation about folding it into the core instead.

## Structured Logging, Graceful Shutdown, Health Endpoints

Three operational requirements every Node service must meet from day one:

### Structured logging

Logs are machine-read first. Use `pino` (built into Fastify):

```ts
logger.info({ jobId, cardId, durationMs }, 'thumbnail generated');
```

JSON lines with contextual fields — not string interpolation. Correlation IDs propagate through every hop (NGINX forwards them; BFF passes them to workers via job payloads), making one user's journey traceable across PHP and Node. Never log tokens or payloads wholesale (Chapter 7's leakage rules).

### Graceful shutdown

Killed mid-job workers lose messages and corrupt in-flight work. Handle signals explicitly:

```ts
process.on('SIGTERM', async () => {
  logger.info('shutting down: stopping intake');
  server.close();                    // stop accepting new requests
  await worker.close();              // finish in-flight jobs
  await redis.quit();
  process.exit(0);
});
```

Sequence: stop intake → drain current work → close connections → exit. Container orchestrators give roughly 30 seconds between SIGTERM and SIGKILL; sizing job timeouts within that window keeps deployments boring.

### Health endpoints

Two endpoints with different meanings:

- **`/healthz` (liveness):** "is this process alive?" Returns 200 unless the process is fundamentally broken. Must not check dependencies — if Redis blips, you want the process restarted *never*, just degraded.
- **`/readyz` (readiness):** "can this instance serve traffic?" Checks what actually gates serving: Redis connection for a BFF, queue subscription for a worker. Failing readiness removes the instance from load balancing without restarting it.

This mirrors Chapter 11's healthcheck discipline: readiness gates traffic, liveness gates existence. Every service gets both, wired into Compose healthchecks now and orchestrator probes later.

---

Node, kept small and purposeful, gives the stack its flexibility at the edges without threatening the domain core.

# Chapter 23. Vue: Components, State, and Routing

Vue's learning curve is gentle, which is exactly why mid-sized Vue codebases rot so often — it's easy to ship working components, and easy to accumulate a thousand of them with no architecture. This chapter covers the patterns that keep a Vue app coherent at scale: the reactivity model, component design discipline, Pinia stores that model the application rather than hoard it, and routing/fetching conventions chosen once and applied everywhere.

## The Vue 3 Mental Model

### SFCs and `<script setup>`

A **Single-File Component** (SFC) co-locates template, logic, and styles:

```vue
<script setup lang="ts">
import { ref } from 'vue';
import type { Card } from '@/api/types';

const props = defineProps<{ card: Card }>();
const emit = defineEmits<{ archived: [cardId: string] }>();

const confirming = ref(false);

function archive() {
  emit('archived', props.card.id);
}
</script>

<template>
  <article class="card">
    <h3>{{ card.title }}</h3>
    <button v-if="!confirming" @click="confirming = true">Archive</button>
    <button v-else @click="archive">Confirm archive?</button>
  </article>
</template>
```

`<script setup lang="ts">` is the standard: top-level bindings are automatically exposed to the template, TypeScript is first-class, and there's no `export default` ceremony. Every example in this book uses it.

## Reactivity: What Updates and What Doesn't

Vue's reactivity tracks property access during render and re-runs effects when dependencies change. The primitives:

- **`ref(value)`** — reactive container for any value; access via `.value` in script (auto-unwrapped in templates). The default choice.
- **`reactive(obj)`** — deep-reactive proxy of an object; no `.value`, but loses reactivity when destructured or reassigned. Use for grouped state, prefer `ref` otherwise.
- **`computed(fn)`** — derived value, cached until dependencies change:

```ts
const openCards = computed(() =>
  cards.value.filter(c => c.status === 'open')
);
```

- **`watch(source, cb)`** — side effects on change: persistence, analytics, imperative DOM work.

The trap list — "what is not reactive":

- **Destructuring** a `reactive()` object severs the connection (`const { title } = state` stops tracking). Use `toRefs()` if you must destructure.
- **Replacing** a `ref`'s inner object wholesale can lose fine-grained tracking in edge cases; mutate properties instead.
- **Non-POJO values**: class instances with private fields, DOM nodes, functions held as values — reactivity proxies only plain objects/arrays/collections reliably.
- **Values captured outside tracked contexts**: reading a ref inside a plain function called from an event handler doesn't create tracking — only render functions, computeds, and watchers track.
- **Module-level mutable state outside refs** — shared across components but invisible to reactivity; put it in a store instead.

When something "doesn't update," check this list before blaming Vue.

## Component Design: Keep Components Boring

Good components are boring: clear inputs, clear outputs, no surprises.

- **Props down, emits up.** Data flows in via typed props; events flow out via typed emits. A component that mutates its props is a bug factory — copy to local state if editing is needed.
- **Slots for composition.** Default slots for content injection, named slots for layout components (`<template #header>`), scoped slots when children need to hand data back to the parent's template.
- **`provide`/`inject` sparingly.** It skips prop-drilling through deep trees, but creates invisible dependencies. Reserve it for genuine cross-cutting concerns provided at the app root (theme, i18n instance) — not for passing data two levels down, where props are clearer.
- **One responsibility per component.** If a component handles fetching, formatting, and modal orchestration, split it: presentational child, composable for logic, parent for orchestration.

A useful smell test: could you describe what a component does in one sentence using only its props and emits? If the sentence requires "and also it fetches... and also it watches the route...", it's doing too much.

## Pinia: Stores as Application Model

**Pinia** is Vue's official store library. The critical framing: **a store is the application's model — not a dumping ground.**

The failure mode is familiar: a `useAppStore` grows into 2,000 lines holding user state, UI flags, cart contents, and three unrelated caches, imported by everything. That's not state management; it's a global variable with extra steps.

### Setup stores vs. options stores

Two syntaxes, same capability:

```ts
// Setup store — our default
export const useBoardStore = defineStore('board', () => {
  const cards = ref<Card[]>([]);
  const loading = ref(false);

  const openCount = computed(() => cards.value.filter(c => c.status === 'open').length);

  async function load(boardId: string) {
    loading.value = true;
    try {
      cards.value = await api.listCards(boardId);
    } finally {
      loading.value = false;
    }
  }

  return { cards, loading, openCount, load };
});
```

Setup stores read like composables, get full TypeScript inference without ceremony, and compose naturally — prefer them. Options stores (`state/getters/actions`) remain fine for simple cases.

### Store composition and boundaries

Stores should be **small, domain-scoped, and composable**:

```ts
export const useAuthStore = defineStore('auth', () => {
  const user = ref<User | null>(null);
  const permissions = ref<PermissionSet | null>(null);   // Chapter 6 bootstrap payload
  // ...
});

// boardStore uses authStore internally
const auth = useAuthStore();
```

Boundaries: one store per domain concept (auth, boards, notifications), never per page. Cross-store dependencies flow one direction. And remember Chapter 6's rule: permissions in the store drive *rendering*; every mutation re-checks server-side.

### Persistence plugins — when justified

Pinia plugins can sync stores to `localStorage`. Justified for genuine preferences (theme, sidebar collapse) and wizard state. Never justified for API-derived data that belongs in memory and refreshes from the source — persisted caches go stale invisibly and become their own bug category.

## Vue Router

### History mode and typed routes

`createWebHistory` gives clean URLs backed by the History API (Chapter 3) — with NGINX's SPA fallback from Chapter 12 making deep links work. Hash-mode URLs are legacy; don't use them.

Typed routes come from unplugin-vue-router or manual typing of params:

```ts
const route = useRoute<'/boards/[boardId]'>();   // generated types
route.params.boardId;   // string, not string | undefined guesswork
```

### Navigation guards

Guards run before route resolution completes:

```ts
router.beforeEach(async (to) => {
  const auth = useAuthStore();
  if (to.meta.requiresAuth && !auth.user) {
    return { name: 'login', query: { redirect: to.fullPath } };
  }
});
```

Use guards for authentication gates and coarse authorization (redirecting users away from pages they can't access — UX, not enforcement, per Chapter 6). Keep guards fast: anything async here delays every navigation.

### Scroll behavior, lazy routes, view transitions

- **Scroll behavior:** restore position on back-navigation, scroll to top (or anchor) on forward:

```ts
scrollBehavior(to, from, savedPosition) {
  return savedPosition ?? { top: 0 };
}
```

- **Lazy routes:** code-split per route so the initial bundle stays small:

```ts
{ path: '/settings', component: () => import('@/pages/SettingsPage.vue') }
```

Every non-critical page ships lazily; the login-to-dashboard path loads eagerly.

- **View transitions:** wrap `<RouterView>` in `<RouterView v-slot="{ Component }"><Transition ...>` for animated route changes — respecting `prefers-reduced-motion` per Chapter 4.

## Data Fetching: Pick One Convention

Three viable patterns exist; teams fail by mixing all three:

1. **Stores own fetching** — actions call the API, hold results. Good for genuinely shared state (auth, current board).
2. **Composables own fetching** — `useCards(boardId)` returns reactive state scoped to a component tree. Good for local concerns.
3. **TanStack Query-style caches** — declarative caching keyed by query parameters, with deduplication, background refetch, and invalidation built in. Best fit for server-state-heavy SPAs.

**This book's convention:** TanStack Query (`@tanstack/vue-query`) for all server state — it eliminates the hand-rolled loading/error/refetch boilerplate every store-based approach reinvents badly:

```ts
const { data: cards, isLoading } = useQuery({
  queryKey: ['cards', boardId],
  queryFn: () => api.listCards(boardId),
});

const mutation = useMutation({
  mutationFn: api.archiveCard,
  onSuccess: () => queryClient.invalidateQueries({ queryKey: ['cards', boardId] }),
});
```

Pinia retains what isn't server state: session/auth, UI preferences, client-only state. The dividing line is crisp: **server cache lives in Query; application state lives in Pinia; nothing lives in both.**

## Forms and Validation Matching problem+json

Client-side validation is UX (Chapter 4 made forms accessible; Chapter 6 made it non-authoritative). The server remains truth — and its errors arrive as RFC 9457 problem+json (Chapter 8), whose `errors` extension maps field names to messages.

Build one convention that consumes them:

```ts
async function submit() {
  try {
    await api.createCard(payload);
  } catch (e) {
    if (isProblemDetails(e) && e.errors) {
      fieldErrors.value = e.errors;      // { title: ["Title must not exceed 120 characters."] }
      focusFirstInvalidField();
    } else {
      toast.error(e.detail ?? 'Something went wrong');
    }
  }
}
```

The form displays `fieldErrors.title[0]` beside the input, wired via `aria-describedby` and `aria-invalid` (Chapter 4's markup). Client-side rules mirror server constraints (max lengths, formats) for instant feedback — but the server's response always wins on conflict, because the server's rules evolve independently.

## Vue DevTools

Install the browser extension; it's the fastest way to understand any Vue app:

- **Component inspector:** live component tree, selected component's props/state, click-to-open-source.
- **Pinia panel:** every store's current state plus a timeline of mutations — invaluable when debugging "who changed this?"
- **Router panel:** matched routes, params, guard history.
- **Timeline/events:** emitted events and reactivity triggers, showing why a component re-rendered.
- **Performance:** component render timing to spot expensive subtrees.

DevTools only activates in development builds — another reason production builds stay lean.

## When a Component Is Too Smart; When a Store Is a Service Locator

Two architectural smells close out the chapter:

**Too-smart components** show recognizable symptoms: they fetch their own data deep in the tree, make decisions based on route state, talk to multiple stores, and render complex conditionals. Each symptom alone is survivable; together they mean the component is an application disguised as a leaf node. Fix by inversion: move data decisions upward (or into Query keys), pass down props, emit intent upward. A component receiving `card` and emitting `archive` needs no knowledge of routes, stores, or APIs.

**Stores as hidden service locators** appear when every component imports five stores directly and reaches into whatever it needs — coupling everywhere, testing requiring the whole graph. The fix mirrors backend DI discipline (Chapter 18): components depend on *props and composables*, composables may touch stores, stores depend on each other narrowly. If you find yourself writing `useAppStore().something.unrelated.to.this.component`, stop — that's `$container->get()` with different syntax.

---

# Chapter 24. Vite as the Bundler and Dev Server

Vite changed frontend tooling by refusing to bundle during development. Instead of rebuilding your entire app on every save, it serves source files directly over native ES modules and transforms only what the browser requests. The result: dev servers that start in milliseconds and hot updates that feel instant.

This chapter covers both sides of Vite — the dev server experience and the production build — configured for our SPA.

## Dev vs. Build: Two Different Machines

Vite operates in two modes with fundamentally different mechanics:

**Development:** no bundling. The browser requests `src/components/Card.vue` as a native ES module; Vite intercepts the request, compiles that one file with esbuild, and serves it. Change a file, and Vite pushes the update over a WebSocket — **Hot Module Replacement (HMR)** swaps the module in place, preserving component state where possible. No full-page reload, no rebuild wait.

**Production:** full bundling. Native ESM in dev would mean hundreds of HTTP requests for deeply nested imports (the "waterfall problem" that made unbundled serving impractical years ago), so `vite build` produces optimized, code-split, minified bundles via Rollup.

The consequence to internalize: **dev and prod are different pipelines.** Bugs that appear only in production builds (import-order issues, environment differences, minification edge cases) are real and expected — which is why CI builds and smoke-tests the production bundle (Chapter 27's debugging story depends on it).

### Dependency pre-bundling

One wrinkle: dependencies in `node_modules` are CommonJS or thousands of tiny ESM files. On first dev-server start, Vite **pre-bundles** them with esbuild into single ESM files, cached in `node_modules/.vite`. This is why the first `npm run dev` takes a few seconds and subsequent starts are instant — and why deleting `.vite` is the standard fix when dependency caching misbehaves (add a dependency, and Vite usually detects and re-runs automatically).

## `vite.config` for the SPA

The configuration file ties everything together:

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';
import path from 'node:path';

export default defineConfig({
  plugins: [vue()],

  resolve: {
    alias: {
      '@': path.resolve(__dirname, 'src'),   // must match tsconfig paths (Ch. 21)
    },
  },

  server: {
    proxy: {
      // API calls go to Symfony during development
      '/api': { target: 'http://localhost:8080', changeOrigin: true },
      '/hooks': { target: 'http://localhost:3001' },
      // WebSocket for HMR itself
      ws: true,
    },
  },

  build: {
    sourcemap: true,
    rollupOptions: {
      output: {
        manualChunks: { vendor: ['vue', 'vue-router', 'pinia'] },
      },
    },
  },
});
```

Key points:

- **Aliases match `tsconfig` exactly** — Chapter 21's "three places or zero places" rule in action. A mismatch compiles fine in the IDE and fails the build.
- **The dev proxy** solves CORS during development: the browser talks to Vite's origin, and Vite forwards `/api/*` to Symfony (or the NGINX container from Chapter 11). No CORS headers needed locally; production uses NGINX's same-origin setup instead.
- **`import.meta.env`** exposes environment variables prefixed with `VITE_`:

```ts
const apiBase = import.meta.env.VITE_API_BASE;   // string | undefined
```

Only `VITE_`-prefixed variables reach the client bundle — everything else stays server-side. Critical security note: **these values are baked into the shipped JavaScript.** Anything in a `VITE_` variable is public. Secrets never go here (Chapter 7); the SPA's secrets live in the BFF (Chapter 5).

## Code Splitting and What Actually Ships

Every route-level component should load lazily (Chapter 23):

```ts
{ path: '/settings', component: () => import('@/pages/SettingsPage.vue') }
```

Vite splits each dynamic import into its own chunk. The initial load ships only what the first screen needs; navigation fetches chunks on demand, cached by the browser afterward.

Verify what actually ships — don't guess:

```bash
npx vite build
npx vite-bundle-visualizer   # treemap of every chunk and dependency
```

Watch for:

- **A bloated entry chunk** — usually a heavyweight library imported at startup (charting, editor, date libraries). Move behind dynamic imports at point of use.
- **Duplicate dependencies** — two versions of the same library from mismatched semver ranges; deduplicate.
- **The vendor chunk question:** `manualChunks` grouping framework code into one stable chunk improves long-term caching (app code changes don't invalidate the framework chunk), but can hurt if the vendor chunk grows larger than needed. Measure both ways.

Budget the result: set a hard ceiling on initial JS (e.g., 200KB gzipped) in CI, failing builds that exceed it. Bundle size only ratchets upward without a gate.

## Assets, CSS, and SVG

**Static assets** in `public/` ship verbatim at the root path. **Imported assets** (`import logo from '@/assets/logo.svg'`) are processed: small ones inline as data URIs, larger ones emitted as hashed files with the URL returned — giving you Chapter 12's immutable caching for free.

**CSS:** import plain CSS, or use CSS Modules (`.module.css`) for locally-scoped class names:

```vue
<style module>
.card { /* becomes a hashed, collision-free class */ }
</style>
```

PostCSS plugins (autoprefixer, nesting) configure via `postcss.config.js` — Vite applies them automatically. Custom properties (Chapter 3) carry design tokens; CSS Modules prevent the specificity wars that global stylesheets invite.

**SVG strategy:** three tiers, chosen per use:

- **Inline components** (via `vite-svg-loader`) when CSS must style the SVG — icons that inherit `currentColor`.
- **Asset imports** for static illustrations — cached, hashed, simple.
- **Sprites** for large icon sets — one request, `<use>` references.

Pick per case; the anti-pattern is mixing all three for the same icon family with no rule for which applies.

## Library Mode

When the monorepo publishes a shared package — the design system, typed API client — Vite's **library mode** builds it:

```ts
export default defineConfig({
  build: {
    lib: {
      entry: 'src/index.ts',
      formats: ['es'],          // ESM for modern consumers
      fileName: 'design-system',
    },
    rollupOptions: {
      external: ['vue'],        // peer deps stay external
    },
  },
});
```

Two details matter: **externals** (Vue stays external — the consuming app provides it, preventing duplicate Vue instances, which break reactivity) and **type declarations** (generate `.d.ts` alongside, usually via `vue-tsc`). The package then flows through npm workspaces (Chapter 22) with path repositories for local development (Chapter 17).

## Source Maps and Debugging

Production builds minify: `CardView.a3f9c2.js` contains no recognizable names. **Source maps** map minified output back to original source — the bridge between a production stack trace and the code you wrote.

```ts
build: {
  sourcemap: true,   // or 'hidden' — generate but don't reference
}
```

Modes:

- **`true`** — maps ship alongside bundles, referenced via `//# sourceMappingURL`. Simplest; also exposes original source to anyone. For internal tools, fine.
- **`hidden`** — maps are generated but not referenced in the bundle; upload them privately to your error tracker (Sentry-style). Users get no access; your tooling gets full traces. **This is the production default for public-facing apps.**
- **`false`** — no maps. Debugging production becomes archaeology. Avoid.

With maps configured, a minified stack trace from an error report resolves to exact original lines and files — the difference between "something in chunk 4 broke" and "`CardList.vue:47` called `formatTotal` on undefined." Chapter 27 builds the full browser-debugging workflow on top of this foundation.

---

The build pipeline now ships lean, debuggable bundles.

# Chapter 25. Frontend Standards Enforcement

Chapter 19 established the principle for PHP: machines enforce conformance, humans review judgment. This chapter applies the same discipline to the frontend with three tools — each with a sharply bounded job.

The boundaries matter because these tools overlap confusingly: ESLint can format code, Prettier can restructure it, and both can fight each other line by line. Clear division of labor ends that war before it starts.

## The Division of Labor

| Tool | Owns | Never touches |
|---|---|---|
| **Prettier** | Whitespace, formatting | Any rule about *what* code does |
| **ESLint** | Correctness, patterns, accessibility | Formatting |
| **Stylelint** | CSS conventions and quality | JavaScript |

One rule follows from this table: **ESLint's stylistic rules are disabled.** The `eslint-config-prettier` package turns off every ESLint rule that would conflict with Prettier. Without it, you get the classic pathology — ESLint demands one style, Prettier reformats to another, and every save produces a diff war. With it, Prettier formats and ESLint analyzes, and they never meet.

## ESLint: Flat Config

Modern ESLint uses **flat config** (`eslint.config.js` or `.mjs`) — an array of config objects applied in order, replacing the legacy cascading `.eslintrc` files:

```js
// eslint.config.mjs
import js from '@eslint/js';
import tseslint from 'typescript-eslint';
import pluginVue from 'eslint-plugin-vue';
import vueParser from 'vue-eslint-parser';
import a11y from 'eslint-plugin-vuejs-accessibility';
import prettier from 'eslint-config-prettier';

export default [
  js.configs.recommended,
  ...tseslint.configs.strictTypeChecked,
  prettier,

  {
    files: ['**/*.vue'],
    languageOptions: {
      parser: vueParser,
      parserOptions: { parser: tseslint.parser },
    },
    plugins: { vue: pluginVue, 'vuejs-accessibility': a11y },
    rules: {
      ...pluginVue.configs['flat/strongly-recommended'].rules,
      ...a11y.configs.recommended.rules,
      'no-console': ['error', { allow: ['warn', 'error'] }],
    },
  },

  // Node services get Node-specific globals
  {
    files: ['services/**/*.{ts,mts}'],
    languageOptions: { globals: globalThis.nodeGlobals ?? {} },
  },
];
```

What each layer contributes:

- **TypeScript strict-typechecked rules** catch unsafe patterns statically (`no-floating-promises`, `no-misused-promises`) — floating promises alone justify the setup, since an unawaited async call is a silent production failure.
- **Vue plugin rules** enforce SFC best practices: component naming, prop definition requirements, template correctness.
- **The accessibility plugin** automates part of Chapter 4's floor: missing `alt` attributes, interactive elements without keyboard handlers, ARIA misuse. It catches the mechanical violations; manual keyboard QA still covers the rest.
- **`no-console`** keeps debug logging out of shipped bundles — structured logging goes through your logger, not `console.log`.

"Prematter" conflicts (the legacy pre-flat-config era of mixed `.eslintrc` + overrides) simply don't exist in flat config: one file, evaluated top to bottom, later entries overriding earlier ones for their matched files. One source of truth, no inheritance archaeology.

## Stylelint

**Stylelint** lints CSS — including the CSS inside Vue SFC `<style>` blocks. Its value is preventing stylesheet entropy:

```js
// stylelint.config.js
export default {
  extends: ['stylelint-config-standard'],
  rules: {
    'selector-max-id': 0,                    // no IDs in selectors (Ch. 3)
    'declaration-no-important': true,        // no !important wars
    'max-nesting-depth': 3,                  // cap nesting
    'custom-property-pattern': '^([a-z][a-z0-9]*)(-[a-z0-9]+)*$',
    'property-no-vendor-prefix': true,       // let autoprefixer handle prefixes
    'color-no-hex': [true, { severity: 'warning' }],   // nudge toward tokens
  },
};
```

Each rule maps to Chapter 3's architecture:

- **No ID selectors and no `!important`** keep specificity flat, so styles remain overridable and predictable.
- **Nesting caps** prevent the deeply nested preprocessor habits (Sass-era four-level indentation) that produce unmaintainable selectors.
- **Vendor prefix bans** push prefixing to PostCSS tooling where it belongs.
- **Custom property patterns** keep design tokens consistent — and nudging hex colors toward tokens (as warnings) gradually migrates hardcoded colors into the theme system.

If the project still carries Sass, Stylelint handles SCSS via its postcss-scss syntax — but per Chapter 3's stack, plain modern CSS plus custom properties makes most preprocessing unnecessary. New code shouldn't introduce preprocessor features; migration happens opportunistically as files are touched.

## Prettier: Formatting Only

**Prettier** is opinionated by design: few options, strong defaults, no debate tolerated. Its entire job is ending discussions about semicolons and quote style by deciding them once.

```json
// .prettierrc
{
  "singleQuote": true,
  "semi": true,
  "printWidth": 100,
  "trailingComma": "all",
  "plugins": ["prettier-plugin-organize-imports"]
}
```

Two disciplines keep Prettier in its lane:

1. **Never encode architectural preferences in formatter options.** Prettier decides how code *looks*, never how it's *structured*. If a team argument surfaces ("should composables live near components?"), that's a convention decision documented in the repo — not a Prettier option.
2. **Run it on everything, mechanically.** Format-on-save locally, CI check mode (`prettier --check`) as gate. Like php-cs-fixer, land any large initial reformat as a standalone PR containing nothing else.

## Shared Configs in the Monorepo

Copy-pasted lint configs across services drift within months. Centralize instead — npm workspaces (Chapter 22) make shared packages trivial:

```
packages/
└── eslint-config/       # @acme/eslint-config
    └── index.mjs        # the flat config from above
packages/
└── stylelint-config/
packages/
└── tsconfig/            # Chapter 21's base configs
```

Every app and service then consumes one versioned package:

```js
// apps/web/eslint.config.mjs
import acme from '@acme/eslint-config';
export default [...acme.vue, ...acme.typescript];
```

Upgrading standards becomes a single PR bumping one package, with every project's diff visible at once. Pair this with a **starter template** for new services that wires all of it up from minute zero — new services should inherit enforcement, not assemble it.

## CI as the Source of Truth

Chapter 19's table applies verbatim here:

| Check | Where |
|---|---|
| Prettier check | Merge gate (fast fail first) |
| ESLint (incl. type-checked rules) | Merge gate |
| Stylelint | Merge gate |
| Accessibility plugin | Merge gate (with manual QA per Ch. 4) |
| Bundle size budget | Merge gate |

Local hooks (via husky + lint-staged, committed through the repo) give fast feedback on changed files only — but as always, they're convenience, not enforcement. Anyone can skip a hook; nobody skips CI. Keep lint-staged scoped to touched files for speed; run the full suite in CI.

## Ignore Files and Generated Code

Linters must not waste cycles — or worse, flag — generated artifacts. Maintain explicit ignores:

```js
// In eslint.config.mjs
{ ignores: ['dist/**', 'coverage/**', 'node_modules/**', 'src/api/schema.d.ts', '**/*.generated.ts'] }
```

```gitignore
# .prettierignore
dist/
coverage/
src/api/schema.d.ts
pnpm-lock.yaml
package-lock.json
```

The critical entry: **generated code is ignored everywhere.** Chapter 21's OpenAPI-generated types (`schema.d.ts`), any codegen output — none of it should be formatted or linted, because regenerating would fight the tools and diffs would be noise. Convention: generated files carry a recognizable suffix or live under a recognized directory, and every tool config excludes that pattern in one place.

Also exclude vendored third-party snippets and test fixtures containing intentionally broken code. An ignore list that grows unexamined is its own smell — review it when a "why isn't this file checked?" question arises, and prefer narrow, justified exclusions.

---

With both stacks now under automated enforcement, quality stops depending on reviewer vigilance for anything mechanical.

# Chapter 26. Vitest, Bruno, and Playwright

Frontend testing fails in two familiar ways: suites of shallow component tests that break on every refactor while catching nothing real, and a handful of slow E2E tests that everyone distrusts. This chapter builds the middle path — three tools, each owning the layer where it's strongest:

- **Vitest** for units and component logic — fast, run constantly.
- **Bruno** for API contract testing — collections versioned beside the code.
- **Playwright** for end-to-end flows against the full Compose stack — few, deliberate, high-value.

## Vitest

**Vitest** is Vite-native test running: same config and transforms as your build, so tests execute exactly what ships.

### Unit and component tests with Vue Test Utils

```ts
// composables/useCardFilter.test.ts
import { describe, it, expect } from 'vitest';
import { useCardFilter } from './useCardFilter';

describe('useCardFilter', () => {
  it('filters by status case-insensitively', () => {
    const { filtered } = useCardFilter([
      { id: '1', title: 'A', status: 'open' },
      { id: '2', title: 'B', status: 'CLOSED' },
    ]);

    expect(filtered.value({ status: 'open' })).toHaveLength(1);
  });
});
```

Component tests mount components with **Vue Test Utils**, asserting behavior through rendered output and emitted events:

```ts
import { mount } from '@vue/test-utils';
import CardItem from '@/components/CardItem.vue';

it('emits archived when confirmed', async () => {
  const wrapper = mount(CardItem, { props: { card: testCard } });

  await wrapper.find('[data-test="archive"]').trigger('click');
  await wrapper.find('[data-test="confirm"]').trigger('click');

  expect(wrapper.emitted('archived')).toEqual([[card.id]]);
});
```

Note `data-test` selectors rather than CSS classes or tag structure — tests that survive restyling are tests that keep running.

### The mocking guide, used deliberately

Vitest's mocking tools are powerful and easy to overuse. Each has a specific job:

- **`vi.fn()`** — creates a bare spy function; assert calls or supply return values. For injected dependencies passed as props/params.
- **`vi.spyOn(obj, 'method')`** — wraps an *existing* method, keeping its behavior unless overridden. Ideal for verifying an event fired without replacing the object.
- **Module mocks (`vi.mock`)** — replaces an entire module for every importer in the file. The nuclear option; reserve it for modules with side effects you can't inject around (browser APIs, third-party SDKs).
- **Fake timers (`vi.useFakeTimers()`)** — controls `setTimeout`, debounce, retry backoff deterministically instead of sleeping in tests.
- **Boundary fakes** — hand-written implementations at architectural seams (Chapter 20's pattern): a fake API client backed by an array beats mocking its module in twenty files.

The rule from Chapter 20 applies verbatim: mock at boundaries you own; never mock so deeply that the test verifies your assumptions about the mock. If a component test requires mocking three modules to render, the component is doing too much — move data access up into composables that can receive fakes.

## What to Unit-Test in Vue vs. Leave to Playwright

The split follows value per millisecond of test runtime:

**Unit-test in Vitest:**

- **Pure composables** — filtering, formatting, derived state. Fast, exhaustive, edge-case friendly.
- **Stores** — actions mutate state correctly, computed values derive properly. Pinia stores test cleanly with `setActivePinia(createPinia())`.
- **Formatters and validators** — currency display, date handling, problem+json parsing (feed it fixture payloads).
- **Component behavior contracts** — emits, conditional rendering driven by props. A handful per component, testing the contract rather than the markup.
- **API client parsers** — zod schemas reject malformed responses (fixture-driven).

**Leave to Playwright:**

- Multi-page flows (login → board → archive → notification).
- Anything depending on routing, real network timing, or browser quirks (focus, scroll, viewport).
- Visual layout questions — component tests assert presence and events, not pixel arrangement.
- Accessibility interactions across pages (keyboard traps, focus restoration on modal close) — these depend on real DOM sequencing that jsdom approximates poorly.

The smell indicating misplacement: unit tests full of mocks simulating "what the server would do" — that's E2E territory. Conversely, Playwright suites asserting arithmetic in formatters are burning minutes on what milliseconds cover.

## Bruno: API Collections as Code

**Bruno** stores API requests as plain text files in the repository — reviewable in PRs, diffable, no vendor cloud holding the source of truth.

```
bruno/
├── bruno.json
├── environments/
│   ├── local.bru          # baseUrl: http://localhost:8080
│   └── staging.bru
├── auth/
│   ├── login.bru
├── boards/
│   ├── list-cards.bru     # cursor pagination assertions
│   ├── create-card.bru    # problem+json validation error case
│   └── archive-board.bru
```

Per Chapter 8, these collections are **generated from the OpenAPI spec**, not maintained by hand — regeneration keeps them synchronized with the contract, while hand-edited collections drift within weeks.

Environments swap variables without touching requests:

```text
vars {
  baseUrl: http://localhost:8080
  testUser: carlos@example.com
}
```

### Asserting problem+json and pagination

Bruno supports scripted assertions — this is where it becomes a contract test suite, not a curl wrapper:

```js
// create-card.bru — invalid payload must yield RFC 9457 shape
test("validation errors use problem+json", function() {
  expect(res.getStatus()).to.equal(422);
  const body = res.getBody();
  expect(body.type).to.equal("https://api.example.com/errors/validation");
  expect(body.errors.title).to.be.an("array");
});

// list-cards.bru — cursor envelope contract
test("cursor pagination returns stable envelope", function() {
  const body = res.getBody();
  expect(body).to.have.property("nextCursor");
  expect(body).to.have.property("hasMore");
  expect(body.data).to.have.lengthOf.at.most(20);
});
```

These assertions encode Chapter 8's decisions where clients will feel them. Run collections in CI against the Compose stack — a breaking API change fails before any frontend code notices.

And the policy point stated plainly: **no Postman cloud as source of truth.** Vendor-hosted collections invisible to code review, inaccessible to CI without paid tiers, drifting silently — they're institutional memory held hostage. Collections live in git or they don't exist.

## Playwright

**Playwright** drives real browsers end-to-end. Its superpowers: auto-waiting (no sleep-and-pray), multiple browser engines, network interception, and trace artifacts on failure.

### E2E against Compose

Tests run against the full stack from Chapter 11 — real NGINX, PHP, MySQL, Redis — via `webServer` config:

```ts
// playwright.config.ts
export default defineConfig({
  webServer: {
    command: 'docker compose up --wait',
    url: 'http://localhost:8080',
    reuseExistingServer: !process.env.CI,
  },
  use: { baseURL: 'http://localhost:8080' },
});
```

`--wait` respects healthchecks, so tests start only when every service is genuinely ready (Chapter 11's readiness discipline).

### Auth fixtures

Logging in through the UI in every test wastes minutes and couples tests to the login page. Use fixtures that establish sessions directly:

```ts
export const test = base.extend<{ boardPage: Page }>({
  boardPage: async ({ page }, use) => {
    // authenticate via API, seed storage state — not via UI clicks
    await request.post('/api/v1/auth/token', { data: credentials });
    await page.addInitScript(/* inject session */);
    await use(page);
  },
});
```

One UI-level login test proves login works; everything else authenticates through the fast path. Storage-state reuse across tests keeps suites minutes instead of tens of minutes.

### Network stubbing vs. full stack

Playwright can intercept routes:

```ts
await page.route('**/api/v1/cards*', route =>
  route.fulfill({ json: fixtureCards }));
```

Use stubbing sparingly and deliberately:

- **Full stack** is the default for critical paths — checkout, archive, refund. These tests exist to catch integration failures; stubbing the API neuters them.
- **Stub** for scenarios the backend can't easily produce: error states (500s), latency simulation, third-party failures (Stripe declining), edge-case payloads. Stubbing *failures* is often more valuable than stubbing successes — the unhappy paths are exactly what manual testing skips.

Rule of thumb: stub the *third parties* and the *error cases*; never stub your own API on happy paths.

### Accessibility checks and visual testing

Wire Chapter 4's axe scans into E2E (the snippet appears there); run them per page in the same passes that exercise functionality — zero marginal setup cost after the first.

**Visual regression** (screenshot comparison) pays off narrowly: design-system components, marketing surfaces, email templates. Elsewhere it generates false-positive churn on every font rendering update. Policy: visual snapshots only for surfaces where pixels are the product, with explicit review ownership for baseline updates.

### Keeping E2E honest

E2E suites rot through growth. Guardrails:

- Cap the count deliberately (~20–40 flows covering money paths, auth, core CRUD).
- Every test asserts business outcomes ("board shows archived"), not implementation details.
- Failures produce traces (`trace: 'on-first-retry'`) and screenshots automatically — debugging without artifacts kills trust in the suite.
- Flaky tests get fixed or quarantined within days; a suite people rerun-until-green provides negative value.

## Connecting to the Test Pyramid

Fowler's test pyramid orders layers by count and cost: many fast units at the base, fewer integration tests in the middle, a handful of E2E at the top. Chapter 27 formalizes the strategy across both stacks; here's how today's tools slot in:

| Layer | Tool | Count | Runtime |
|---|---|---|---|
| Units (PHP + TS) | PHPUnit, Vitest | Thousands | Seconds |
| Integration | WebTestCase, component tests | Hundreds | Minutes |
| Contract | Bruno collections | Dozens | Minutes |
| End-to-end | Playwright | ~20–40 | Tens of minutes |

The economic logic: push coverage down wherever possible. A bug caught by a Vitest unit test costs seconds to diagnose; the same bug caught by Playwright costs a pipeline run plus artifact forensics. E2E earns its expense only for cross-layer truths no lower layer can verify — which is why the pyramid narrows instead of inverting into the "ice cream cone" anti-pattern of hundreds of brittle UI tests atop nothing.

---

# Part VII — Quality Across the Stack

# Chapter 27. The Test Pyramid in This Shop

Martin Fowler's test pyramid is the shape, not the plan. The concept — many cheap tests at the base, few expensive ones at the top — is easy to state and easy to violate. Teams invert it into the "ice cream cone" (hundreds of slow UI tests over a thin unit layer) not by deciding to, but by adding E2E tests whenever unit testing feels hard.

This chapter makes the pyramid concrete for our stack: what lives at each layer, where mocks are allowed, how test data and time work, and the two cultural mechanisms — CI budgets and flake protocol — that keep the pyramid from collapsing.

## Layer 1: Unit Tests

Milliseconds each, no I/O, run on every save:

- **PHP:** domain rules (`Money`, `InvoicePolicy`), voters against seeded relationship objects, DTO validation logic, formatters.
- **TypeScript/Vitest:** pure composables, Pinia store transitions, formatters, zod schema rejection cases.

The defining property isn't speed — it's **isolation**. A unit test constructs its world entirely in memory: no kernel boot, no database, no network. That's what makes thousands of them affordable, and what makes them the right home for exhaustive edge-case coverage (every branch of a validator, every boundary of a date range).

Design feedback applies as in Chapter 20: code that can't be tested here is telling you to inject dependencies, split responsibilities, or move I/O to boundaries.

## Layer 2: Integration Tests

Real infrastructure at the seams — HTTP, persistence, authorization — without full browser overhead:

- **Symfony `WebTestCase`:** boots the kernel, issues real requests through routing → firewall → controller → Doctrine. Chapter 20's patterns apply; these tests verify the framework wiring units can't see.
- **Node services with testcontainers:** spin up throwaway Redis/MySQL containers per test session via Testcontainers. A BullMQ worker test runs against real Redis semantics instead of a mock that guesses at BRPOP blocking behavior.
- **Contract tests:** Bruno collections against the Compose stack (Chapter 26).

Integration tests answer questions units structurally cannot: does the voter fire behind this route? Does the query filter actually exclude other tenants' rows? Does the problem+json listener catch exceptions thrown three layers deep? Keep them in the hundreds, targeted at seams — every integration test should name the seam it protects.

## Layer 3: End-to-End Journeys

A deliberately small set of Playwright flows proving the *contract between layers* — the promises the whole system makes to users:

- **Login works** end to end (IdP → BFF cookie → SPA bootstrap).
- **Create board → create card → archive board** — core CRUD across NGINX, PHP, MySQL, back through Vue.
- **A flag-gated path** — the feature flag flips behavior correctly across backend enforcement and frontend rendering (Chapter 10's consistency guarantee, verified).
- One money-path journey if billing exists.

Roughly 20–40 journeys total. Each asserts business outcomes ("board shows archived"), never DOM minutiae. When someone proposes E2E test number fifty, the correct question is: which lower layer failed to cover this?

## Mocking Policy

Chapter 20 established the vocabulary; here's the policy per layer.

### PHPUnit doubles for outbound dependencies

At integration and unit layers, everything *leaving* your trust boundary gets doubled:

- **Mail** — fake mailer collecting sent messages; assert contents when relevant.
- **Billing/gateways** — fake gateway with scripted outcomes (success, decline, timeout).
- **IdP** — fake token verifier or locally signed test tokens rather than live IdP calls. Integration tests may use a real Keycloak container when auth itself is under test; otherwise stub.

Outbound doubles keep suites hermetic: no external service can fail your build, rate-limit your CI, or leak test data into production systems.

### Vitest mocks at the unit layer

Browser APIs and network get mocked *at the unit layer only*: `vi.stubGlobal('fetch', ...)`, fake timers for debounce, module mocks for SDKs with side effects. Component tests receive fakes via props/provide rather than reaching into modules wherever injection is possible.

### Never mock the code under test into passing

The rule that keeps all of this honest, stated as an anti-pattern checklist:

```php
// The service under test has its repository mocked to return
// exactly the entity it will then be asserted on:
$repo->method('find')->willReturn($board);
$result = $service->rename($repo, 4721, 'New');
$this->assertEquals('New', $result->getTitle());
```

This test passes regardless of whether `rename()` contains logic or returns a hardcoded string — it verifies the author's assumptions about the mock, not the system. Symptoms of the disease: assertions checking values the mocks supplied, tests that survive deletion of the feature they claim to cover, mocks configured with behaviors the real dependency doesn't have.

Corrective actions, in preference order: use a hand-written fake with real (simple) behavior; drop to an integration test with transaction-wrapped real infrastructure; or recognize that the class needs restructuring so the interesting logic is separable from the I/O.

## Test Data Builders, Clock Ports, Randomness

Three utilities remove most test-suite friction:

### Builders

Fixture construction is where tests rot. **Builders** centralize it:

```php
final class BoardBuilder
{
    private Organization $org;
    private string $title = 'Test Board';

    public static function aBoard(): self { return new self(); }
    public function titled(string $t): self { $this->title = $t; return $this; }
    public function in(Organization $org): self { $this->org = $org; return $this; }
    public function build(): Board { /* ... */ }
}

BoardBuilder::aBoard()->titled('Q3')->in($acme)->build();
```

Defaults encode valid entities; overrides express only what the scenario cares about. When the schema changes (a new required column), one builder updates instead of four hundred fixtures. Foundry factories serve the same role on the Doctrine side; TypeScript builders mirror it for API payloads.

### Time via clock ports

Never call `new DateTimeImmutable()` inside domain code. Inject a clock (PSR-20 from Chapter 19):

```php
public function __construct(private ClockInterface $clock) {}

public function isExpired(Subscription $s): bool
{
    return $s->endsAt < $this->clock->now();
}
```

Tests freeze time deterministically: `new FrozenClock(new DateTimeImmutable('2025-06-01 12:00:00'))`. Expiry logic, retry windows, TTL math — all become deterministic assertions instead of sleep-and-hope. Vitest's fake timers provide the equivalent on the frontend; the shared principle is **time is an injected dependency, not ambient state.**

### Randomness

Randomness in tests creates flakes wearing a disguise. Rules:

- Seed anything random that affects assertions (`mt_srand(42)`, seeded faker instances) so failures reproduce.
- IDs and slugs come from builders, not random generators, unless uniqueness itself is under test.
- Property-based testing (Eris for PHP, fast-check for TS) *is* welcome randomness — generated inputs with shrunk counterexamples — but it's deliberate, reported deterministically, and belongs at the unit layer.

## CI Time Budget

Pipeline duration is a cultural variable disguised as a technical one:

- **Under ~10 minutes:** developers watch results, fix breaks immediately, trust stays high.
- **15–25 minutes:** context-switching begins; engineers start batching PRs to amortize wait time; merge velocity drops.
- **Over 30 minutes:** the suite becomes background noise. People stop watching, broken mains linger, and "it was green when I started" becomes standard vocabulary. At this point slow tests cause more bugs than they catch.

Manage the budget actively:

- **Parallelize by layer:** unit jobs run everywhere on every push; integration on PRs; E2E on merge to main plus nightly full runs.
- **Cache aggressively** — vendor directories, Playwright browsers, Docker layers, pre-built test databases (restore a snapshot instead of running all migrations).
- **Fail fast, fail first:** lint/typecheck stages complete before heavyweight stages launch; no point running a 12-minute E2E suite after a type error.
- **Measure per-job duration** and investigate regressions like any performance bug — a suite that grows 10% monthly hits the culture threshold within a year.

When the budget is genuinely exceeded by necessary coverage, the answer is splitting pipelines (fast gate for PRs, thorough gate for merges), never deleting coverage silently.

## Flake Protocol

A **flaky test** fails intermittently without product changes. Flakes are more corrosive than outright failures: teams learn to rerun until green, and once rerunning-until-green is normal, real failures hide inside the noise.

Protocol, applied mechanically:

1. **Quarantine immediately.** On second unexplained failure, mark the test quarantined (label/skip-group). Quarantined tests run separately, don't block merges, and appear on a visible dashboard. Quarantine is not deletion — the test still executes and reports.
2. **Assign an owner.** Every quarantine names the person investigating. Unowned quarantines accumulate forever; ownership forces triage.
3. **Root-cause within a sprint.** Common culprits: shared mutable state (fix with builders/factories), real timing races (fix the production race — the test found a bug!), order dependence between tests (fix isolation), environment drift (pin versions), genuine nondeterminism in the code under test.
4. **Fix or delete — never retry-and-forget.** Automatic retries mask symptoms. If a test needs retries to pass, it's measuring nothing reliable; either make it deterministic or remove it. Retries are permitted only as a transition while a quarantine ticket is open, capped and tracked.

The cultural frame matters: **a flake is a bug report, not bad luck.** Half of all flakes, investigated honestly, reveal real concurrency bugs or missing isolation — the suite doing its job uncomfortably. Teams that treat flakes as noise train themselves to ignore intermittent production failures too, because the skill being practiced is identical.

---

With the pyramid built and protected, quality becomes a property of the pipeline rather than of individual heroics.

# Chapter 28. Step-Through Debugging

Logging tells you what *happened*; a debugger shows you what *is*. When a bug resists log-based investigation — state-dependent failures, race conditions, "works in dev, not in prod" — reading output stops being enough. You need to freeze the process mid-flight and inspect it.

This chapter covers the three debuggers you'll live in — browser DevTools, Node's inspector, and Xdebug for PHP — plus the discipline that makes them effective instead of exploratory.

## Reproduction First

Before any tool: **can you reproduce the bug on demand?** Debugging without reproduction is guessing with extra steps.

The hierarchy of reproduction, best first:

1. **A failing test.** Reproduce the bug as code. It runs in milliseconds, never regresses, documents the bug permanently, and defines done ("the test passes"). Chapter 27's builders make this cheap.
2. **The debugger against a reproducible local case.** Full inspection, no permanent artifact.
3. **Guessing.** Change something, redeploy, hope. This isn't a strategy; it's how outages extend themselves.

Invest the first minutes of any bug session in shrinking the reproduction. A bug that reproduces in one click is nearly solved; a bug that reproduces "sometimes, after using the app for a while" needs that variability isolated before tools help. Once reproduction exists, the debuggers below turn inspection into answers.

## Browser DevTools

Open with F12 (or Cmd+Opt+I). The panels matter in rough order of daily use:

### Sources panel: breakpoints and stepping

Set a breakpoint by clicking a line number; execution pauses there with full access to:

- **Scope** — every variable in reach at that moment, including closures.
- **Call stack** — how execution arrived; click any frame to jump to its scope.
- **Watch expressions** — pinned values tracked across steps.

Stepping commands: step over (next line), step into (descend into calls), step out (finish current function). Most bugs die between setting one breakpoint and taking five steps.

**Conditional breakpoints** pause only when an expression holds — essential when a loop iterates 10,000 times but fails on one item:

```
// Right-click a breakpoint → Edit breakpoint:
card.id === 'card_4721'
```

**Logpoints** are breakpoints that log instead of pausing — `console.log` without editing source. Scatter them freely; remove nothing afterward.

### Blackboxing framework noise

When stepping through a stack trace, you don't want to walk Vue internals frame by frame. **Blackbox** vendor files (Settings → Ignore List patterns like `/node_modules/`, `vue.*.js`) so the debugger skips through them, stopping only in your code.

### Debugging Vue SFCs via source maps

Production bundles are minified, so enable source maps in DevTools (Settings → Sources → Enable JavaScript source maps). With Vite's maps configured (Chapter 24), breakpoints set in original `.vue`/`.ts` files resolve correctly even against deployed builds. In development this is automatic — your SFC sources appear directly in the Sources tree under `webpack://`-style origins or served paths.

### Network, Performance, Memory

- **Network tab:** filter to XHR/fetch, inspect request/response headers, payloads, timing waterfalls. The first stop for any API-shaped bug — often revealing that the frontend sent the wrong thing, or received the wrong thing, before any debugger opens.
- **Performance tab:** record interactions, read flame charts of scripting/rendering costs. Long tasks blocking the main thread show up here immediately — the visual form of Chapter 22's event-loop warning.
- **Memory tab:** heap snapshots before/after an action reveal leaks (detached DOM nodes, growing arrays in closures). Use when a tab "gets slower over time"; ignore otherwise.

### Beside Chrome DevTools: Vue DevTools

Run both simultaneously — they answer different questions. Chrome DevTools shows runtime truth (network, memory, JS state); Vue DevTools shows the framework's model: component tree, props/state per component, Pinia store mutations on a timeline, router history. When data renders wrong, check Vue DevTools first (is the component receiving what you think?); when data arrives wrong, switch to Network and backend debuggers.

## Node.js Debugging

### `--inspect` and the Chrome connection

Start any Node process with the inspector flag:

```bash
node --inspect dist/server.js        # listens on 127.0.0.1:9229
node --inspect-brk dist/server.js    # pauses on first line
```

Chrome exposes it at `chrome://inspect` → inspect. You get the full Sources experience — breakpoints, scopes, watch — against server-side code. For containers, publish the port (`9229:9229` in Compose) and connect to `host.docker.internal:9229`.

### VS Code launch configs

VS Code attaches natively with a committed launch configuration:

```json
// .vscode/launch.json
{
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug worker",
      "runtimeExecutable": "npm",
      "runtimeArgs": ["run", "dev"],
      "cwd": "${workspaceFolder}/services/worker",
      "skipFiles": ["<node_internals>/**", "**/node_modules/**"]
    }
  ]
}
```

Breakpoints in the editor, variables inline, no context switching — this is the daily driver for Node work.

### Debugging tests and workers

- **Vitest:** run a single file through the inspector — either via a launch config wrapping `vitest run path/to/file.test.ts --no-file-parallelism`, or Vitest's built-in `--inspect` passthrough. Breakpoints inside test *and* tested code both hit.
- **BullMQ workers:** attach to the worker process exactly as above. Set a breakpoint inside the processor; enqueue a job from another terminal; watch the job arrive in your paused process with full payload visibility. Far more informative than log lines when a job misbehaves.
- **Fastify/Hono services:** attach to the dev server process; requests from curl or Bruno land in your breakpoints. Pair with Bruno collections (Chapter 26) as scripted stimulus.

## Xdebug: PHP Step Debugging

### Setup and modes

Xdebug is PHP's debugger, controlled by `xdebug.mode`. Development images enable debug mode:

```ini
# docker/php/xdebug.ini (dev image only)
zend_extension=xdebug.so
xdebug.mode=develop,debug
xdebug.client_host=host.docker.internal   # reaches your IDE from inside Compose
xdebug.client_port=9003
xdebug.start_with_request=yes             # or trigger via cookie/header
```

In PhpStorm/VS Code: install nothing server-side beyond the extension; configure path mappings — **which container directory corresponds to which host directory** — and IDE listening on port 9003. Path mapping is where Docker setups break: the IDE must translate `/app/src/Controller.php` (container) back to `./src/Controller.php` (your machine) or breakpoints silently never bind. Verify the mapping matches Compose's bind mount exactly.

With `start_with_request=yes`, every request triggers a debug session; better for shared workflows, use the trigger form (`xdebug.start_with_request=trigger`) plus a browser extension or `XDEBUG_TRIGGER` header, stepping in only when wanted.

### Debugging PHPUnit and Messenger consumers

- **PHPUnit:** run tests through the IDE's debugger, or CLI with the trigger env var. Breakpoints inside both test and production code hit; step through domain logic with real Doctrine entities materialized. This is often faster than writing another unit test just to inspect intermediate state.
- **Messenger consumers:** attach to the long-running consumer process (`messenger:consume`) with Xdebug active, drop a breakpoint in a handler, dispatch a message from a shell or Bruno collection, and inspect the message payload and entity graph mid-flight. Remember the consumer is a persistent process — changes to code require restart (Chapter 15's identity-map caveats apply doubly while paused).
- **Long-running caution:** a paused breakpoint holds database connections and potentially row locks open. Never leave a breakpoint parked against a transactional flow longer than needed; MySQL's `innodb_lock_wait_timeout` will eventually error everything behind you.

### Why Xdebug never ships in production

Three reasons, all absolute: performance overhead even when idle, information disclosure if reachable (it can expose source and execute evaluation), and Chapter 11's principle that prod images contain only what prod runs. It lives in the dev stage of the multi-stage build, period.

## Debugging a Cross-Stack Request

Here's the workflow applied end-to-end — a card archive button that sometimes silently does nothing:

1. **Reproduce locally** via Compose (Chapter 11): click the button; confirm the failure occurs.
2. **Network tab first.** Was the request sent? What status came back?
   - No request → frontend bug: breakpoint in Vue DevTools / browser Sources at the submit handler. Step into the composable calling the API client.
   - Request present, response wrong → backend bug: continue below.
3. **NGINX hop:** check access logs for the request — did it reach NGINX, which upstream did it route to, what upstream status returned? A 502 here means the PHP container wasn't ready (Chapter 12's healthcheck story); a clean pass-through means move inward.
4. **Symfony hop:** Xdebug breakpoint at the controller. Inspect the deserialized DTO — validation errors? Then the voter — is `isGranted('ARCHIVE', $board)` returning false unexpectedly? Step into the voter with the real membership graph loaded.
5. **Data layer:** if the controller ran but state didn't change, check Doctrine's logged SQL (Chapter 9) — did the UPDATE execute? Transaction rolled back somewhere?
6. **Queue hop:** the archive triggers a reindex job. BullMQ dashboard shows the job stuck in `failed` with an error trace. Attach Node inspector to the worker, breakpoint in the handler, replay the job, find the Redis serialization mismatch from a recent deploy.
7. **Frontend confirmation:** after fixing, verify in the UI with Vue DevTools watching the Pinia store update and the query cache invalidating.

Total elapsed time with these tools: typically minutes. Without them — log archaeology across four systems — the same bug eats an afternoon. The skill being practiced is less about any single tool than about **knowing which layer to suspect next**, which is exactly Chapter 1's request-lifetime mental model turned into muscle memory.

---

Debugging closes individual incidents; observability prevents their recurrence and catches what reproduction can't.

# Chapter 29. Observability with ELK

When something breaks in production at 2 a.m., the difference between a ten-minute fix and an all-nighter is almost always the same thing: **can you find the relevant logs?** "We have logs" — gigabytes of unstructured text across a dozen services — is not observability. Being able to answer *"what exactly happened to order 4721?"* in two minutes is.

This chapter builds that capability with the ELK stack (Elasticsearch, Logstash, Kibana) and the logging discipline that makes it work.

## What Belongs in a Log Line

Every log line is structured JSON with consistent fields. The core set:

```json
{
  "@timestamp": "2025-06-01T14:32:07.412Z",
  "level": "info",
  "service": "api-php",
  "request_id": "req_01J8XK3M9P",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "actor_id": "usr_8f2k1",
  "tenant_id": "org_acme",
  "event": "card.archived",
  "board_id": "brd_4721",
  "duration_ms": 43,
  "flag_evals": {"archive_v2": true}
}
```

What earns a place in every line:

- **Request ID** — unique per HTTP request, propagated everywhere it flows.
- **Actor ID** — who triggered this (authenticated principal; `system` for jobs).
- **Tenant ID** — which organization. In multi-tenant systems this is your fastest filter during incidents ("is this affecting everyone or just Acme?").
- **Duration** — how long the operation took. Latency regressions surface from logs alone when duration is universal.
- **Event name** — a stable identifier (`card.archived`, `payment.failed`), not freeform prose. Events make dashboards and alerts possible; prose makes them impossible.
- **Flag evaluations** — Chapter 10's requirement: when behavior changed, the log proves whether a flag was involved.

And just as important, what does *not* belong:

- **Full request/response payloads** — they contain PII, tokens, card data (see redaction below). Log identifiers and *references* to payloads, not payloads.
- **Stack traces on every error line** — one structured field pointing to an error ID; full traces go to your error tracker, not duplicated into every log store query.
- **Debug noise at info level** — level discipline keeps signal extractable.

## Correlation IDs from NGINX Through PHP/Node to the SPA

Correlation is the whole game. One user click traverses NGINX → PHP → Redis → queue → Node worker → email provider. Without shared IDs, those are seven unrelated log streams; with them, they're one story.

The propagation chain:

```
Browser
  │  X-Request-ID: req_01J8XK3M9P   (generated by SPA or first hop)
  ▼
NGINX        → validates/generates if absent, forwards header,
               writes it to its access log
  ▼
PHP/Symfony  → reads header into request context;
               every Monolog line carries it automatically
  │            enqueues job with metadata {request_id, actor_id}
  ▼
Node worker  → BullMQ job payload carries the IDs;
               worker logger includes them on every line
```

Implementation details that make it stick:

- **Generate once, honor always:** NGINX generates `$request_id` if absent; backends never invent their own for the same request.
- **Automatic injection:** Symfony's Monolog processor reads the request stack and stamps every record. Node's pino child-loggers bind the IDs at request start so no handler forgets them.
- **Jobs carry lineage:** queue messages include the originating request ID (and actor), so background failures link back to the triggering action. Chapter 14's handlers log with these fields by default.
- **Return it to callers:** include the request ID in problem+json responses (`extensions`) and error pages. A user's screenshot containing the ID becomes instantly actionable support input.
- **Trace IDs for deeper spans:** where distributed tracing exists, `trace_id` complements `request_id` — same propagation path, richer structure.

Test the chain deliberately: trigger a request through Bruno (Chapter 26), then search Kibana for its ID. If lines don't appear across every hop, correlation is broken somewhere — fix it before you need it in an incident.

## Elasticsearch / Logstash / Kibana

### The pipeline

- **Logstash** (or lighter alternatives like Filebeat/Vector) collects JSON log output from containers and files, enriches (adds environment/host metadata), and ships onward.
- **Elasticsearch** stores and indexes documents, making field-level queries fast across terabytes.
- **Kibana** is the query UI and dashboard builder.

### Index strategy

Don't dump everything into one giant index:

- **Index per service per time period:** `logs-api-php-2025.06.01`, `logs-worker-thumbnails-2025.06.01`. Time-based indices enable cheap retention (delete old indices outright) and scoped queries.
- **Consistent mappings:** identical field names and types across services (`actor_id` is always keyword, never sometimes-text) — otherwise cross-service queries break silently. Enforce via a shared logging library config per language.
- **ILM (Index Lifecycle Management):** hot indices (queryable, replicated) age into warm (cheaper storage), then delete. Policy-as-code beats manual cleanup.

### Structured JSON as the entry ticket

Everything upstream of this chapter's value assumes **structured logs**. PHP: Monolog with a JSON formatter plus processors (request context, memory). Node: pino (Chapter 22). Both emit one JSON object per line — parseable without regexes, aggregatable by any field.

If your pipeline contains a Grok pattern parsing human-written prose, that's technical debt accruing interest daily. Fix the producers instead.

### Dashboards worth building early

- **Error rate by service over time** — the first chart opened in every incident.
- **p95 latency by endpoint** (from duration_ms aggregation).
- **Top exception types** — grouped by event/error class, not raw messages.
- **Queue depth/lag events** from workers (Chapter 14's signals).
- **Auth failures by actor/IP** — security visibility from day one (feeds the A09 tie-in below).

Dashboards answer recurring questions; build them when a question gets asked twice.

## From "We Have Logs" to Two-Minute Answers

The maturity test: pick any recent business transaction — order 4721 — and answer, within two minutes, without asking anyone:

> What happened to it? Which steps succeeded? Where did it fail, and why?

Achieving this requires three habits beyond tooling:

1. **Log state transitions, not noise.** Every meaningful domain change (`order.placed`, `payment.captured`, `export.completed`) emits one info line carrying entity ID and outcome. Business questions then map directly onto log queries.
2. **Log decisions, especially denials.** Authorization denials (Chapter 6), flag-driven branch selections (Chapter 10), validation rejections — these explain "why didn't it happen?" which is half of all incident questions.
3. **Practice the drill.** Run the two-minute test quarterly against real transactions. Every failure of the drill reveals either a missing log point or broken correlation — both cheap now, expensive during the actual outage.

Teams that pass this drill stop fearing production. Teams that don't, ship changes blind and debug by redeploying.

## Alerting vs. Dashboards

Dashboards are pull — someone must look. Alerts are push — someone must act. They serve different needs and fail differently:

- **Alert on symptoms users feel:** elevated 5xx rate, p95 latency breach, queue lag exceeding threshold, dead-letter arrivals, certificate expiry. Each alert maps to a runbook action.
- **Dashboard for exploration:** traffic patterns, cache hit rates, deployment markers. Watched during rollouts (Chapter 10), consulted during investigation.
- **Never alert on everything.** Alert fatigue is fatal: after the twentieth non-actionable page, engineers mute the channel, and the one alert that mattered dies unheard. Rule: **every alert must be actionable** — something a human should do differently right now. If the response is "hmm, interesting," it belongs on a dashboard.
- **Tie alerts to SLOs where possible:** error-budget burn rates beat static thresholds — they catch slow degradation that fixed thresholds miss until too late.

This is OWASP **A09: Security Logging and Alerting Failures** made operational. The Top 10 item isn't about having logs — it's about logs being *usable*: sufficient detail to detect attacks, retained long enough to investigate, monitored actively enough that breaches are noticed. Concretely for us: authentication failures, authorization denials, token validation errors, and admin actions are logged as first-class events (not buried in generic error streams), alerted on anomalous rates, and retained under policy. An intrusion attempt visible only in a dashboard nobody opens is A09 failing exactly as designed.

## Retention, Redaction, and What Never Goes to ELK

Log stores accumulate sensitive data with terrifying efficiency, because developers keep accidentally logging it. Three defenses:

### Redaction at the source

Scrub before emission, not in the pipeline:

```php
// Monolog processor: drop known-sensitive keys recursively
$redacted = preg_replace('/(authorization|password|token|card_number)/i', '[REDACTED]', $record);
```

```ts
// pino redact option — declarative paths
const logger = pino({ redact: { paths: ['req.headers.authorization', '*.token', '*.password'], censor: '[REDACTED]' } });
```

Pipeline-level redaction is a safety net, not the primary defense — by the time data reaches Logstash it has already been written to disk wherever the app runs.

### What never goes to ELK, categorically

- **Tokens and credentials** — bearer tokens, API keys, session IDs. Chapter 7's leakage rules apply doubly: a log store is a credential database with a search engine attached.
- **Passwords** — even hashed ones have no business in logs.
- **Cardholder data** — PANs, CVVs. PCI scope expands to every system touching card data; one logged card number drags your entire ELK cluster into compliance scope. Log last-four digits and provider references only.
- **Bulk PII** — names, emails, addresses belong in the database, referenced by ID in logs. Full payload logging of user objects is the most common accidental violation.
- **Health data or anything under legal retention restrictions** — know your regulatory exposure before choosing what to index.

### Retention policy

Retention balances investigation needs against cost and compliance:

- **Hot (30 days):** fully searchable, standard investigations.
- **Warm (90 days):** cheaper storage, slower retrieval — enough for most forensic timelines.
- **Cold/archive (1 year):** compressed snapshots restored on demand — driven by compliance requirements, not debugging habit.

Longer retention multiplies both cost and breach blast radius: a compromised log store exposes *everything ever logged*, including all those secrets that slipped past redaction before the policy existed. Shorter windows limit that damage. Choose deliberately, document it, and automate enforcement — a retention policy nobody enforces is a wish, not a control.

---

Logs tell you what happened; metrics and traces complete the picture.

---

# Part VIII — Delivery

# Chapter 30. GitLab CI/CD

Every rule in this book — review gates, test pyramids, signing discipline, expand/contract migrations — converges on one artifact: the CI/CD pipeline. It's the automated expression of your engineering standards. If the pipeline is the *only* path to production, then the pipeline's rigor *is* your production rigor.

This chapter assembles that pipeline in GitLab CI for our stack.

## The Pipeline Is the Only Path to Production

First principle, stated absolutely: **no human pushes code to production. Not via laptop, not via SSH, not "just this once."** Every deployment is a tagged, signed artifact produced by a pipeline run whose logs prove what was tested and by whom.

Why so strict? Because exceptions are how standards die. One "quick manual deploy during an incident" teaches everyone that the pipeline is optional under pressure — and pressure is exactly when discipline matters most. Chapter 7's supply-chain reasoning applies: an untracked deployment is unverified provenance, and provenance you can't verify is provenance an attacker can forge.

The corollary: if deploying through the pipeline is slow or painful, engineers will route around it. Pipeline speed isn't convenience; it's what makes the single-path rule sustainable (Chapter 27's time-budget logic applies here too).

## Stages

The full pipeline, ordered from cheap-and-broad to expensive-and-narrow:

```yaml
stages:
  - lint
  - typecheck
  - unit
  - integration
  - build
  - e2e
  - deploy
```

1. **Lint** — php-cs-fixer check, ESLint, Prettier, Stylelint. Seconds each; fail first, fail fast.
2. **Typecheck / PHPStan** — `tsc --noEmit`, PHPStan at configured level. Catches whole bug classes before anything runs.
3. **Unit** — PHPUnit and Vitest suites in parallel. Thousands of tests, seconds to a minute.
4. **Integration** — WebTestCase against MySQL service containers; Node tests against Testcontainers; Bruno contract collections against a Compose stack.
5. **Build images** — multi-stage Docker builds (Chapter 11) pushed to the registry with immutable tags.
6. **E2E** — Playwright against the built images running in Compose. Tests what actually ships, not what was on disk.
7. **Deploy** — promotion of the exact tested artifacts to the environment.

Ordering rationale: every stage is cheaper than everything after it. A typo costs seconds at lint; letting it reach E2E wastes twenty minutes of runner time and muddies failure signals. Each stage answers one question — *is the code well-formed? does it behave in isolation? do the pieces fit together? does the shipped artifact work end-to-end?* — and stops the pipeline when its answer is no.

## Runners, Caching, and Why the Cache Is a Hint

GitLab **runners** execute jobs. Self-hosted runners with Docker executor are the norm for this stack; scale them horizontally since parallel stages multiply concurrent demand.

Caching makes the difference between a 25-minute and a 5-minute pipeline:

```yaml
.php-cache: &php-cache
  cache:
    key:
      files:
        - composer.lock
    paths:
      - vendor/

unit-php:
  <<: *php-cache
  script:
    - composer install --prefer-dist
    - vendor/bin/phpunit
```

- **Composer/npm caches keyed on lockfile hash** — dependency layers rebuild only when dependencies change (the same layer-caching logic as Chapter 11's Dockerfiles).
- **Docker layer caching:** build with BuildKit against the registry (`--cache-from` previous images) so unchanged stages reuse cached layers.
- **Playwright browser binaries** cached once per version.

The critical caveat: **the cache is a hint, not a guarantee.** GitLab may evict entries, distribute keys unevenly across runners, or serve stale content after lockfile edge cases. Pipelines must be *correct without cache* — merely slower. Never encode correctness assumptions into cache behavior ("it works because node_modules persisted"), because the day the cache misses, you'll discover which jobs were secretly depending on residue. Validate periodically with cleared caches; treat a cold-cache run as the source of truth for timing.

## Merge Gates vs. Scheduled Jobs

Not every check belongs on every push. Split by cost and consequence:

| Job | Trigger | Rationale |
|---|---|---|
| Lint, typecheck, unit | **Merge gate** — every MR | Fast enough to never skip |
| Integration | **Merge gate** | Seams break silently |
| Image build + E2E | Merge gate (or pre-merge nightly) | Proves the shipped artifact |
| `composer audit` / `npm audit` | **Merge gate** + scheduled weekly | New CVEs block; drift surfaces |
| Rector upgrade pass | Scheduled / manual PRs | Too disruptive inline |
| Full regression suite | Nightly scheduled | Broader matrix than merge-time |
| Dependency update MRs | Scheduled (Renovate/Dependabot) | Automation proposes, humans approve |

Protected branches enforce the gates: required pipelines must succeed before merge, per Chapter 2. Scheduled jobs run off-branch against `main`, catching things that drifted in — newly published CVEs, flaky-test emergence, certificate expiries.

## Environments, Review Apps, Flag-Aware Deploys

### Environments

GitLab **environments** model deployment targets: `staging`, `production`. Each tracks what version deployed when, with a history viewable per environment — the audit trail for "what's live?" answered authoritatively rather than from memory.

```yaml
deploy-staging:
  stage: deploy
  environment:
    name: staging
    url: https://staging.example.com
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

Staging deploys automatically on merge to `main`; production deploys on tags or manual approval gates, per your release cadence.

### Review apps

For each merge request, spin up an isolated environment running the MR's images:

```yaml
review:
  stage: deploy
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    auto_stop_in: 3 days
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

Reviewers click a live URL instead of imagining behavior from diffs — disproportionately valuable for UI changes. Cost control via `auto_stop_in`; database strategy via seeded fixtures or shared anonymized snapshots, never production data.

### Feature-flag-aware deploys

Chapter 10's decoupling lands here: **deploys ship dark.** Code reaches production with flags off; enabling is a flag operation, independent of the pipeline. This gives three levers with different speeds:

- Rollback = revert deploy (minutes).
- Disable feature = flip flag (seconds).
- Kill switch = ops flag (instant).

The pipeline stays responsible for shipping safe code; flags stay responsible for exposure. Never couple them — a deploy script that flips flags mid-deploy couples the two failure modes into one incident.

## Signing, SBOM, and Secret Management

### Image signing and SBOM

Where supply-chain requirements demand it (Chapter 7):

- **Sign images** with Sigstore/cosign in the build job; production clusters verify signatures before pulling. Unsigned images can't reach prod even if the registry leaks.
- **Generate an SBOM** (Software Bill of Materials) per image — Syft or Docker Scout — and store it alongside. When the next log4j-style CVE drops, "which of our services contain X?" becomes a query instead of a week.

Both belong in the pipeline, not on laptops: artifacts signed by CI carry verifiable provenance; artifacts signed by developers carry unverifiable claims.

### Secrets: CI variables vs. vault

Two tiers:

- **GitLab CI variables (masked, protected)** — fine for non-critical values: registry credentials scoped to CI, staging secrets. Marked protected, they exist only on protected branches/tags; masked hides them in job logs.
- **External secret managers (Vault, cloud KMS)** — for production-grade credentials: short-lived dynamic credentials issued per job beat long-lived static variables, and revocation doesn't require touching GitLab.

Rules regardless of tier: no secrets in `.gitlab-ci.yml` itself (it's committed), no secrets echoed into logs (masking helps but isn't proof), and production deploy credentials grant nothing except deployment — least privilege per Chapter 5's scoping discipline.

## Branch Pipelines vs. Tag Pipelines

Two distinct pipeline types with different contracts:

**Branch/MR pipelines** answer "is this change good?" — full quality stages, deploy nowhere (except review apps), optimized for speed. They run constantly and must stay fast.

**Tag/release pipelines** answer "ship this version" — build from the tagged commit, sign, promote through environments, tag Docker images immutably (`app:1.42.0`). Tags are immutable by policy: rebuilding a tag produces a new artifact and violates reproducibility (Chapter 17's locked-build principle). If a release needs changes, cut a new tag.

```yaml
build-release:
  rules:
    - if: '$CI_COMMIT_TAG'
```

The distinction keeps everyday velocity high while giving releases their ceremony — and their auditability: every production version maps to exactly one tag, one commit, one signed artifact.

## Rollback: Immutable Images, Fast Levers

Rollbacks work only when deployments are **immutable and versioned**. Images tagged by digest (not `latest`) mean rolling back is re-pointing traffic at the previously deployed tag — the old image still exists, byte-identical, in the registry.

The rollback playbook, fastest lever first:

1. **Ops flag / kill switch** (seconds) — disable the misbehaving feature. First response to almost every bad deploy.
2. **Config change** (seconds–minutes) — revert a bad config value via env vars; no redeploy needed.
3. **Image rollback** (minutes) — redeploy the previous immutable tag. Works because schema migrations followed expand/contract (Chapter 15): the old code runs fine against additive-only schema changes.
4. **Database point-in-time restore** (last resort) — Chapter 9's PITR, used when data itself was corrupted, not merely code.

Note what makes step 3 reliable: the discipline that migrations are backward-compatible within a release window. A rollback that breaks against the current schema isn't a rollback — it's a second outage.

## Tying It Together

Three threads close the loop across this book:

- **Chapter 10:** deploys ship dark; flags control exposure; the pipeline never flips user-facing behavior implicitly.
- **Chapter 11:** the *same* multi-stage Dockerfile builds locally in Compose and in CI — identical artifacts everywhere, eliminating "works on my machine" structurally rather than rhetorically.
- **Chapters 26–29:** what the pipeline tests (Bruno, Playwright) is what observability watches in production (ELK, alerts) — the same contracts verified pre-deploy are monitored post-deploy.

---

# Chapter 31. Production Topology and Operations

Everything so far — architecture, testing, pipelines — assumed a production environment worth deploying to. This chapter draws that environment explicitly, revisits the operational principles from the 12-Factor App that still earn their keep, and covers the mechanics of zero-downtime deploys, backups, and knowing *when* to scale rather than guessing.

## The Reference Topology

The full production picture for our stack:

```
                    ┌─────────────┐
   Users ──────────▶│     CDN     │  static assets, TLS edge,
                    └──────┬──────┘  DDoS absorption
                           ▼
                    ┌─────────────┐
                    │    NGINX    │  reverse proxy, rate limits,
                    │  (2+ nodes) │  security headers
                    └──┬───────┬──┘
             /api/*    │       │   /hooks/*
                       ▼       ▼
              ┌─────────────┐ ┌─────────────────┐
              │ PHP-FPM ×N  │ │ Node services ×N │
              │  (Symfony)  │ │ BFF · webhooks · │
              │             │ │ BullMQ workers   │
              └──────┬──────┘ └────────┬────────┘
                     │                 │
        ┌────────────┼─────────────────┼───────────┐
        ▼            ▼                 ▼           ▼
 ┌────────────┐ ┌───────────┐   ┌────────────┐ ┌──────────┐
 │   MySQL    │ │   Redis   │   │  RabbitMQ  │ │  Object  │
 │ primary +  │ │ (cache/   │   │  (PHP      │ │ storage  │
 │  replica   │ │ sessions) │   │ consumers) │ │ (files)  │
 └────────────┘ └───────────┘   └────────────┘ └──────────┘
```

Reading it with the rules from earlier chapters:

- **The CDN terminates** static traffic and TLS edge concerns; NGINX never serves assets that belong at the edge.
- **NGINX and all app tiers scale horizontally** — stateless by design (12-factor below), so N identical instances behind load balancing just work.
- **MySQL is the only stateful tier** — one primary for writes, replicas for read scaling (with Chapter 9's caveats about what replication does *not* protect against).
- **Redis** holds cache/sessions/rate limits — disposable by policy (Chapter 13).
- **RabbitMQ** decouples PHP producers from consumers.
- **Object storage** holds uploads and generated files; containers hold no state.

Every arrow here was designed in earlier chapters. Operations is largely about keeping these promises intact under change.

## 12-Factor Leftovers That Still Matter

The Twelve-Factor App is fifteen years old; most of it is now absorbed into default practice (declarative dependency manifests, parity between environments). Three factors remain actively load-bearing:

**Config in the environment (Factor III).** Configuration varies across deploys; code doesn't. We've enforced this throughout: env vars via Compose and CI (Chapter 30), secrets from vaults (Chapters 5, 17), `.env` as dev-only convenience (Chapter 18). The operational payoff: **the same image runs in staging and production**, differing only by injected config. If an image needs rebuilding to change behavior, configuration has leaked into code.

**Disposability (Factor IX).** Processes start fast and shut down gracefully. Our stack honors this deliberately: PHP-FPM boots quickly with warm opcache, Node workers drain on SIGTERM (Chapter 22), healthchecks gate readiness (Chapter 11). Disposability is what makes horizontal scaling, rolling deploys, and crash recovery boring instead of dramatic.

**Logs as event streams (Factor XI).** A process writes logs to stdout without caring where they go; the environment routes them (Chapter 29's ELK pipeline). Applications that write their own log files to local disks create state on disposable machines — and lose visibility the moment a container dies. Stream to stdout; collect centrally.

One factor deserves explicit retirement: **dev/prod parity (Factor X)** said "use the same backing services." Modern reality is more nuanced — local MailHog vs. production SES, single MySQL vs. primary-plus-replicas — and Chapter 26's test strategy accounts for those seams deliberately rather than pretending they don't exist.

## Zero-Downtime Deploys

Users should never see a deploy. Achieving that requires coordinating three layers:

**Load balancer draining.** New instances join after passing readiness checks; old instances receive SIGTERM and finish in-flight requests before removal (grace periods sized to your slowest legitimate request). NGINX upstream changes or orchestrator rolling updates handle this mechanically — provided readiness gates are real, not "process started."

**Session continuity.** Stateless app tiers make this trivial: sessions live in Redis (Chapter 13), tokens are self-contained (Chapter 5), so any instance can serve any request. Sticky sessions would couple users to dying instances — we don't have them, by design.

**Database compatibility during the rollout window.** The hard part. During a rolling deploy, old and new application versions run *simultaneously* against one schema. Every migration must therefore be safe for both versions — which is exactly **expand/contract** (Chapter 15):

1. Expand: additive schema change ships first; old code ignores it.
2. Deploy: new code reads/writes both representations as needed.
3. Backfill: existing rows migrate in batches.
4. Contract: after full rollout, drop the old shape.

A migration that renames a column in one deploy breaks whichever version runs second — expand/contract makes such migrations impossible by construction. The rule of thumb: **any schema change must survive being rolled back mid-rollout**, which also keeps Chapter 30's rollback lever functional.

## Backups, Restore Drills, On-Call Notes

### Backups (recap, made operational)

Chapter 9 established the requirements; operations executes them:

- Nightly physical backups plus binlog retention for point-in-time recovery.
- Encrypted, replicated off-site, access-controlled separately from production credentials.
- **Restore drills monthly:** restore last night's backup into scratch infrastructure, boot the app against it, verify data integrity, record elapsed time. The drill measures your real RTO — the number you'll quote during the actual incident. First drills always find gaps (missing credentials, undocumented dependencies); finding them in a drill costs an afternoon, finding them mid-outage costs the business.

### Redis/RabbitMQ durability posture

Match persistence to consequence (Chapters 13–14): AOF on for queues holding real work, RDB-only acceptable for pure caches. Document per-instance what may be lost and why that's acceptable — this document prevents both over-engineering and silent data loss.

### Runbooks next to the code

Operational knowledge rots in wikis nobody updates. Keep runbooks **in the repository, beside the services they describe**:

```
docs/runbooks/
├── mysql-failover.md        # promote replica, repoint writers, verify lag=0
├── queue-backlog.md         # scale consumers, inspect DLQ, replay procedure
├── redis-flush-recovery.md  # expected degradation, cache-warm sequence
└── rollback-deploy.md       # the Chapter 30 playbook, concretely
```

Each runbook states symptoms, diagnosis commands, remediation steps, and verification — written *during calm* by whoever knows, reviewed like code. During an incident at 3 a.m., the runbook author's memory is unavailable; the file in git isn't. Tie each alert (Chapter 29) to its runbook link in the notification itself.

## Capacity: Measure Before You Scale

Scaling decisions made without measurement produce either waste (over-provisioned) or outages (under-provisioned). Four signals tell you when and where:

**p95/p99 latency per endpoint.** Aggregate averages hide tail pain; the 95th percentile shows what real users experience. Watch trends, not absolutes: p95 climbing steadily at stable traffic means growing inefficiency (missing index, cache hit-rate decline) — fix that *before* adding hardware, because hardware multiplies whatever efficiency you already have.

**Queue lag and depth.** From Chapter 14: oldest-message age and sustained backlog growth. Lag growing while consumer count is flat means throughput ceiling reached — scale consumers, but first check whether handler efficiency regressed (a slow downstream API call added last sprint can halve effective capacity).

**Slow query counts.** The slow query log (Chapter 9) trending upward at constant traffic signals data growth outpacing indexes. This is the earliest warning that the database tier needs attention — usually an index, occasionally read-replica offloading, rarely raw horsepower.

**Connection saturation.** MySQL `max_connections` headroom, Redis client counts, RabbitMQ channel usage. Exhaustion here fails suddenly and completely — set alerts well before limits (80% warnings).

The decision framework, in order:

1. **Fix efficiency first.** An N+1 query fixed beats a database upgrade bought. Profile (EXPLAIN, flame charts) before provisioning.
2. **Scale the bottlenecked tier horizontally** if stateless — app tiers, workers, consumers. This is why disposability matters.
3. **Scale stateful tiers deliberately** — read replicas when read load justifies replication lag tradeoffs; vertical sizing when working sets exceed memory.
4. **Load-test toward known ceilings.** Synthetic load against staging reveals breaking points before Black Friday does. Test the *current* architecture's limit annually at minimum.

Capacity planning is a habit of measurement, not a quarterly spreadsheet exercise. The dashboards from Chapters 29–30 already collect every signal above — the operational discipline is reviewing them on a schedule instead of discovering them during an incident.

---

# Part IX — Putting a Feature Through the Machine

# Chapter 32. A Vertical Slice, Start to Finish

Every chapter so far taught a layer in isolation. Real work doesn't happen in layers — it happens as **vertical slices**: one feature threading through threat model, contract, schema, backend, queue, frontend, tests, and rollout. This chapter walks one complete slice end to end.

The feature: **card assignments**. Users assign each other to cards; assignees get an email and an external webhook fires; rollout is flag-gated. Nothing here is new — every step cites the chapter that established it. That's the point: this is the exam. If you can execute this unattended, you're productive on this stack.

## Step 1: Threat Model and Authz Matrix

Before any code (Chapters 6–7): ten minutes of endpoint-level STRIDE, written into the PR description.

**What are we building?** "Assign a user to a card; notify them by email; fire an org-configured webhook."

**What can go wrong?**

- *Elevation of privilege:* Can I assign myself to another org's card? Can I assign arbitrary users to spam them?
- *Information disclosure:* Does assignment reveal users outside my org?
- *Repudiation:* Do we record who assigned whom?
- *DoS:* Can one actor trigger unbounded webhook fan-out?

**Decisions:** Assignment requires `ASSIGN` permission on the specific card (ReBAC). Assignable users are limited to board members. Webhook delivery is rate-limited per org. Email/webhook sends carry idempotency keys.

The authorization matrix, which becomes test rows later:

| Actor | Card | Action | Expected |
|---|---|---|---|
| Board member | own org | assign member | ✅ |
| Board member | other org | assign | ❌ 404 |
| Non-member | own org's board | assign | ❌ 404 |
| Assignee | — | self-assign | ❌ (must be done by others) |

## Step 2: OpenAPI, problem+json, Pagination

Contract first (Chapter 8). The spec fragment:

```yaml
paths:
  /boards/{boardId}/cards/{cardId}/assignments:
    post:
      operationId: createAssignment
      parameters:
        - { name: boardId, in: path, required: true, schema: { type: string } }
        - { name: cardId, in: path, required: true, schema: { type: string } }
      requestBody:
        content:
          application/json:
            schema:
              type: object
              required: [userId]
              properties:
                userId: { type: string }
      responses:
        "201": { $ref: "#/components/schemas/Assignment" }
        "404": { $ref: "#/components/responses/Problem" }
        "422": { $ref: "#/components/responses/ProblemValidation" }

  /boards/{boardId}/cards/{cardId}/assignments:
    get:
      parameters:
        - { name: cursor, in: query, schema: { type: string } }
        - { name: limit, in: query, schema: { type: integer, default: 20 } }
```

Notes against our standards: errors reference the shared RFC 9457 components (`type`, `title`, `status`, `detail`, field `errors`); the list endpoint uses cursor pagination (assignments grow); the spec is reviewed before implementation. Bruno collections regenerate from this spec.

## Step 3: Doctrine Migration (+ Node Read Path)

Schema (Chapter 9):

```sql
CREATE TABLE card_assignments (
  id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  card_id     BIGINT UNSIGNED NOT NULL,
  user_id     BIGINT UNSIGNED NOT NULL,
  assigned_by BIGINT UNSIGNED NOT NULL,
  created_at  DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  UNIQUE KEY uq_card_user (card_id, user_id),
  KEY idx_card_cursor (card_id, created_at, id),
  CONSTRAINT fk_ca_card FOREIGN KEY (card_id) REFERENCES cards(id),
  CONSTRAINT fk_ca_user FOREIGN KEY (user_id) REFERENCES users(id)
) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
```

Doctrine attributes declare the entity (Chapter 15); `doctrine:migrations:diff` generates this SQL; a human reviews it — additive only, indexed for the pagination query, unique constraint enforcing "one assignment per user per card."

Does a Node worker read this data? Our webhook worker needs assignment details — but per Chapter 14's boundary rule, it does **not** query MySQL directly. It receives everything in the message payload. No Prisma counterpart needed; if one were, it would be generated from the shared schema and reviewed identically.

## Step 4: Symfony Endpoint, Voter, Messenger Message

**Voter first** (Chapter 6) — the decision lives where Chapter 32's matrix says:

```php
// In CardVoter::voteOnAttribute()
case 'ASSIGN':
    $membership = $this->memberships->findFor($user, $subject->getBoard()->getOrganization());
    return $membership !== null;   // board members only
```

**DTO input** (Chapter 18):

```php
final class CreateAssignmentInput
{
    public function __construct(
        #[Assert\NotBlank] #[Assert\Uuid]
        public readonly string $userId,
    ) {}
}
```

**Controller stays thin:**

```php
#[Route('/api/v1/boards/{boardId}/cards/{cardId}/assignments', methods: ['POST'])]
public function create(string $boardId, string $cardId, Request $request,
                       Security $security, AssignCardUser $assign): JsonResponse
{
    $input = $this->serializer->deserialize($request->getContent(), CreateAssignmentInput::class, 'json');
    $errors = $this->validator->validate($input);
    if (count($errors)) {
        throw new ValidationFailedException($input, $errors);
    }

    $card = $this->cards->findByBoardOrFail($boardId, $cardId);   // 404 semantics
    $security->denyAccessUnlessGranted('ASSIGN', $card);

    $assignment = $assign->execute($card, $input->userId, $security->getUser());

    return $this->json(AssignmentView::from($assignment), Response::HTTP_CREATED);
}
```

**The service + outbox** (Chapter 14): `AssignCardUser` runs in one transaction — insert assignment, insert outbox row:

```php
public function execute(Card $card, string $userId, User $by): Assignment
{
    return $this->em->wrapInTransaction(function () use ($card, $userId, $by) {
        $assignment = Assignment::create($card, $this->users->findOrFail($userId), $by);
        $this->em->persist($assignment);
        $this->outbox->add(new CardAssigned(
            cardId: (string) $card->id(),
            assigneeId: $userId,
            assignedBy: (string) $by->id(),
            idempotencyKey: 'assign-' . $card->id() . '-' . $userId,
        ));
        return $assignment;
    });
}
```

The outbox relay publishes to RabbitMQ; write and event cannot diverge.

## Step 5: Consumers

**PHP consumer (RabbitMQ via Messenger)** — email (Chapters 14, 18):

```php
final class SendAssignmentEmailHandler implements MessageHandlerInterface
{
    public function __invoke(CardAssigned $msg): void
    {
        // dedupe by idempotency key (at-least-once delivery)
        if (!$this->dedupe->claim($msg->idempotencyKey . ':email')) {
            return;
        }
        $email = $this->renderer->render('assignment', /* ... */);
        $this->mailer->send($email);   // fake in tests; transient failures retry via transport config
    }
}
```

**Node worker (BullMQ)** — webhook delivery (Chapter 22): consumes from its Redis queue fed by a small bridge, or the outbox relay fans out to both brokers. Either way: strict timeouts, exponential backoff with jitter, dead-letter after five attempts, per-org rate limiting, idempotency key honored at delivery time.

Both handlers follow the checklist: classified errors, capped retries, dedupe, DLQ destination, depth/lag metrics.

## Step 6: Vue Route, Store, A11y, Flag

**API client** — types generated from the OpenAPI spec, runtime parsing at the edge (Chapter 21):

```ts
const AssignmentSchema = z.object({
  id: z.string(),
  user: z.object({ id: z.string(), name: z.string() }),
  createdAt: z.string().datetime(),
});
```

**Pinia store** holds client-only state; server state lives in TanStack Query (Chapter 23):

```ts
const mutation = useMutation({
  mutationFn: api.assignCard,
  onSuccess: (_, vars) => {
    queryClient.invalidateQueries({ queryKey: ['cards', vars.boardId] });
    toast.success('Member assigned');
  },
  onError: (e) => showProblemDetails(e),   // RFC 9457 → fieldErrors / toast
});
```

**Component:** native `<select>` over assignable members (semantic HTML, Chapter 3), labels wired (Chapter 4), error messages announced via the form pattern, keyboard-tested. Emits intent upward; no route/store knowledge inside.

**Feature flag** (Chapter 10): `card_assignments_v1` evaluated server-side during bootstrap and shipped to the SPA. When off, the UI hides the control — UX only; the API enforces independently via... actually here the flag gates the *feature*, so the Symfony service checks the flag too, keeping both layers consistent. The wiring selects between implementations at the composition root; no flag-shaped ifs in domain code.

## Step 7: Tests at Every Layer

**PHPUnit unit** (Chapter 20): voter matrix from Step 1 as data-provider rows; service transaction behavior with fakes.

```php
#[DataProvider('authzMatrix')]
public function testAssignmentAuthz(string $actor, string $board, bool $allowed): void { /* ... */ }
```

**Integration (WebTestCase):** POST creates assignment + outbox row atomically; duplicate returns 422 problem+json; non-member gets 404 (not 403 — Chapter 7).

**Messenger handler test:** process `CardAssigned` twice, assert exactly one email (idempotency verified).

**Vitest:** store transitions, problem-details parser against fixture payloads, component emits.

**Bruno:** collection asserts 201 shape, 422 problem+json fields, cursor envelope on GET (Chapter 26).

**Playwright:** one journey — login, open board, assign member via select, assert assignment visible, verify email arrived (MailHog in Compose). Runs against the full stack with the flag enabled.

## Step 8: Debug It Once, On Purpose

Deliberately break something and trace it across the whole stack (Chapter 28) — this is rehearsal, not waste:

1. Make the webhook worker throw on payload validation.
2. Click assign in the browser. Watch Network tab: 201 returns fine — frontend is innocent.
3. BullMQ dashboard: job in `failed` with stack trace. Attach Node inspector to the worker, breakpoint in the handler, replay.
4. Inspect payload: the bridge serialized PHP's `AssignmentView` with camelCase mismatch.
5. Fix the mapping, add a Bruno assertion pinning the wire format, re-run the journey green.

Twenty minutes spent once builds the cross-layer reflex that saves hours later.

## Step 9: Pipeline and Rollout

Commit sequence follows the book's rules: contract PR → migration PR (append-only check passes, CI applies from scratch) → implementation PRs referencing the spec → tests land with code (Chapter 27's changed-code coverage).

Pipeline stages run (Chapter 30): lint → PHPStan/tsc → units → integration → images → Playwright E2E → deploy to staging dark. Review app lets product click the real thing behind the flag.

Then the rollout ladder (Chapter 10):

1. **Canary:** enable for internal accounts. Watch ELK: `card.assignment.created` events flowing, email/webhook failure rates flat, p95 unchanged.
2. **10% cohort**, pre-committed criteria: error rate within baseline +0.5%, no dead-letter growth, assignment conversion healthy. Dashboards split by flag state.
3. **100%.** Criteria held for a week.
4. **Delete the flag.** Remove the composition-root branch, delete the legacy path, drop the flag record. Definition-of-done includes cleanup (Chapter 10) — the slice isn't finished until the flag is gone and the diff shows it.

---

That's the whole book in one walk: threat model to deleted flag, every layer doing the job its chapter defined. Run this slice yourself, unattended, against the Compose stack. If every step feels familiar rather than novel — if you know *which* chapter to reread when something surprises you — then this book has done its job, and the rest of your career on this stack is practice.

# Chapter 33. Review Checklists and ADRs

Everything in this book eventually compresses into two artifacts: **checklists** that run on every change, and **decision records** that explain why the system looks the way it does. Checklists enforce consistency; ADRs preserve reasoning. Teams without them re-litigate settled questions every quarter and forget *why* constraints exist until someone "simplifies" their way into an outage.

This closing chapter shows how to package the book's practices so they survive team turnover.

## The PR Checklist

The checklist lives in a PR template (`.gitlab/merge_request_templates/default.md`) — visible to every author and reviewer, checked per merge. It condenses Chapters 4–10, 15, 18–20 into one page:

```markdown
## Security
- [ ] Input validated server-side; no trust in client checks
- [ ] Object-level authz: ownership verified in query filters AND voters
- [ ] No new secrets in code; `.env.example` updated if config added
- [ ] Outbound URLs allowlisted; no user-controlled fetch targets unchecked
- [ ] Logs contain identifiers, not payloads/tokens

## Accessibility
- [ ] Keyboard-only pass completed; focus managed on modals/route changes
- [ ] axe scan clean (CI enforces); labels/ARIA correct on custom controls
- [ ] `prefers-reduced-motion` respected for any new animation

## Tests at the right layer
- [ ] Domain logic covered by unit tests; seams by integration tests
- [ ] Authorization matrix rows added for new actor × resource × action combos
- [ ] Idempotency tested for any new queue handler

## Data & migrations
- [ ] Migration is additive or expand/contract; append-only (no edits to shipped files)
- [ ] Index matches query shape (EXPLAIN attached for new hot queries)
- [ ] Backfills batched; rollback path documented

## Flags & rollout
- [ ] New flag has owner + expiry recorded; evaluated at composition root only
- [ ] Kill switch exists if this can misbehave in production

## Observability
- [ ] Correlation IDs propagate through any new hop
- [ ] New state transitions emit structured events
- [ ] Alert thresholds defined if this adds load-sensitive behavior
```

Two usage rules keep checklists honest:

1. **Checkboxes are answered, not assumed.** An empty box is a review comment. Authors self-check before requesting review; reviewers verify spot-checkable items rather than trusting ticks.
2. **The checklist is a floor, not a substitute for thinking.** Chapter 6's design judgment ("does this belong in Symfony or Vue?") isn't checkbox-able — the checklist handles what's mechanical so review attention stays on what's not.

Keep the template under one screen. A 40-item checklist gets skimmed into uselessness; prune items once CI automates them permanently.

## When to Write an ADR

An **Architecture Decision Record** captures one significant decision: context, options considered, choice made, consequences accepted. Format is secondary; brevity is mandatory. One decision, one short file:

```markdown
# ADR-014: Use RabbitMQ for PHP-side domain events

Date: 2025-06 | Status: Accepted

## Context
Domain events from Symfony need durable delivery with routing.
BullMQ covers Node jobs but PHP producers shouldn't write BullMQ's
private Redis structures (Ch. 14 boundary rule).

## Options
1. BullMQ from PHP via raw Redis structures — rejected: private API
2. RabbitMQ + Messenger AMQP transport — chosen
3. Single broker (RabbitMQ) for everything — deferred until Node
   side justifies migration cost

## Consequences
Two brokers to operate. Bridge service needed for cross-runtime
events. Revisit if event volume doubles.
```

**Write an ADR when a decision is expensive to reverse or expensive to re-explain.** Concrete triggers from this book's material:

- **New data store or instance class** (Chapter 9, 13) — every Redis/MySQL/Elasticsearch addition, including "we split cache onto its own Redis."
- **New grant type or auth mechanism** (Chapter 5) — adding any OAuth flow beyond the two sanctioned ones, adopting passkeys, changing token storage strategy.
- **New queue or consumer topology** (Chapter 14) — new brokers, cross-runtime bridges, outbox adoption decisions.
- **JSON columns** (Chapter 9) — each legitimate use deserves its justification written down, because abuse starts as unexamined convenience.
- **API breaking changes / version bumps** (Chapter 8).
- **Framework/runtime upgrades with architectural impact** (Chapter 16, 19) — "moving to PHP 8.5 property hooks across entities."
- **Flag lifecycle exceptions** (Chapter 10) — any flag granted permanent status.

**Don't write ADRs for** reversible choices, style matters (that's Chapter 19/25 tooling), or anything already covered by an existing ADR — link instead.

Store them in `docs/adr/NNNN-title.md`, numbered, immutable once accepted (supersede with a new ADR rather than editing history — git discipline applied to decisions). Review new ADRs like code: they're the highest-leverage documents in the repo.

The compounding value shows up years later. When someone proposes "why don't we let Node query MySQL directly?", ADR-014 answers with the original reasoning instead of a meeting. Decisions with recorded rationale get revisited when circumstances change; decisions with forgotten rationale get relitigated forever or cargo-culted blindly — both worse.

## Living Handbook: Book → Repo Mapping

A book read once is training; a handbook wired into daily workflow is infrastructure. This book maps directly onto repository structure so the knowledge lives where work happens:

```
docs/
├── adr/                    # decision records (this chapter)
├── runbooks/               # operational procedures (Ch. 31)
├── api/
│   └── openapi.yaml        # the contract (Ch. 8) — Bruno generates from it
├── security/
│   ├── threat-modeling.md  # the four-question template (Ch. 7)
│   └── oauth-checklist.md  # new-client review list (Ch. 5)
├── frontend/
│   ├── a11y-qa-script.md   # manual keyboard/screen-reader passes (Ch. 4)
│   └── conventions.md      # fetch/state/routing rules (Ch. 23–24)
└── backend/
    └── authorization.md    # ReBAC modeling guide (Ch. 6)

.gitlab/
├── merge_request_templates/default.md    # the PR checklist
└── ci/templates/           # shared pipeline includes (Ch. 30)

docker-compose.yml          # local parity (Ch. 11)
.php-cs-fixer.php · phpstan.neon · rector.php     # Ch. 19
eslint.config.mjs (via @acme/eslint-config)       # Ch. 25
tsconfig.base.json                                # Ch. 21
```

Three mechanisms turn this from documentation into a living handbook:

1. **Templates generate structure.** New services start from the starter (Chapter 22) carrying the right configs, healthchecks, and CI includes automatically. Standards inherited at creation need no enforcement later.
2. **CI references the docs.** Failed pipelines link the relevant doc: the migration-append-only failure points at Chapter 15's section; the missing-correlation-ID lint links the observability guide. Feedback arrives at the moment of mistake, with explanation attached.
3. **Docs carry owners and review dates.** Every handbook file names a maintainer. Quarterly hygiene reviews (the same cadence as baselines and ops flags in earlier chapters) prune stale guidance — dead documentation is worse than none, because it's trusted.

When practices change, update the artifact *in the same PR* as the code change. A PR that introduces a new convention without updating the checklist or relevant doc is incomplete — make that explicit in the definition of done, and the handbook evolves at the speed of the system instead of lagging behind it.

---

That closes the loop. This book began with a mental model — layers with clear contracts — and ends with the mechanisms that keep those contracts honest after you've closed the cover: checklists for the routine, ADRs for the consequential, templates for the next person. The stack will change; PHP and Vue will age; something will replace RabbitMQ. What persists is the discipline: decide deliberately, record why, automate enforcement, and treat every layer boundary as a promise worth keeping.

---

# Glossary

**ABAC (Attribute-Based Access Control)** — Authorization model deciding access from attributes of the subject (who), resource (what), and environment (when/where). Layered on top of ReBAC in this book for conditions like business hours or device posture. *See Chapter 6.*

**Abandoned package** — Dependency whose maintainer archived it; it will never receive another fix. Every abandonment flag demands a recorded decision: replace, fork, or patch upstream. Ignoring them is deferred maintenance with interest. *See Chapter 17.*

**Access token** — Credential presented to a resource server to authorize API calls. Short-lived; opaque string or JWT. Never used to derive identity. *See Chapter 5.*

**ADR (Architecture Decision Record)** — A short document capturing one significant decision: context, options considered, choice made, consequences accepted. Immutable once accepted; superseded by new ADRs rather than edited.

**Alert fatigue** — Death of an on-call rotation by degrees: after enough non-actionable pages, engineers mute the channel and the one critical alert dies unheard. Rule: every alert must demand a human action right now, or it belongs on a dashboard instead. *See Chapters 29–30.*

**Allowlist** — Security pattern permitting only explicitly enumerated values (redirect origins, CORS origins, sort columns, outbound hosts) rather than blocking known-bad ones. Denylists age badly; allowlists compose with validation. *See Chapter 7.*

**AOF (Append Only File)** — Redis persistence mode logging every write operation; with `everysec` fsync, loses at most ~1 second of data on crash. Paired with RDB in production. *See Chapter 13.*

**At-least-once delivery** — Queue guarantee that a message arrives one or more times, never zero times. Duplicates are possible and expected; handlers must be idempotent. The foundational contract of Chapters 14 and 20.

**Authorization code** — Short-lived, single-use intermediate credential exchanged for real tokens in the OAuth flow. Exists so secrets and tokens never travel through the browser's front channel. *See Chapter 5.*

**Backfill** — Populating existing rows for a newly added column or table, done in batches rather than one giant transaction. Always sequenced within expand/contract, after the additive expand step ships. *See Chapters 9, 15.*

**Baseline (PHPStan)** — Generated file listing current static-analysis violations so enforcement applies only to new code. Debt with an expiry date: shrinks continuously or enforcement has quietly stopped. *See Chapter 19.*

**BFF (Backend-for-Frontend)** — A thin server layer built for one frontend client, aggregating backend responses into shapes the client needs. In this stack, a Node service holding tokens server-side so the browser never sees them.

**Bind mount** — Docker storage mapping a host directory into the container (`./src:/app/src`), giving instant edit visibility during development. Contrast: named volumes for data. *See Chapter 11.*

**Blackboxing** — Browser DevTools feature ignoring listed files during stepping, so the debugger skips through framework and vendor code and stops only in your own sources. *See Chapter 28.*

**Blackbox** — See *Blackboxing* in prior section.

**Bounded context** — A boundary inside which domain concepts have one specific meaning ("order" means different things to checkout vs. fulfillment). In this architecture, bounded by services and enforced through the API contract. *See Chapter 1.*

**Branded type** — TypeScript technique attaching a phantom compile-time marker to a base type (`UserId` vs. plain `string`) so distinct identifiers can't be passed interchangeably. Zero runtime cost. *See Chapter 21.*

**Bruno** — Git-friendly API client storing request collections as text files in the repository, generated from OpenAPI and assertable in CI. Replacement for cloud-hosted collections like Postman's. *See Chapters 8, 26.*

**BullMQ** — Redis-backed Node.js job queue offering retries with backoff, repeatable/cron jobs, priorities, rate limiting, and dashboards. The queue choice for Node-owned jobs in this stack. *See Chapter 14.*

**Bundle budget** — Hard ceiling on initial JavaScript payload enforced in CI; failing builds exceed it. Bundle size ratchets upward without a gate. *See Chapter 24.*

**Bundle budget** — See prior section.

**Cache aside** — Caching pattern where the application checks Redis first, reads MySQL on miss, writes back with a TTL. Fails safe (staleness expires) versus write-through's coupled failures. The book's default caching posture. *See Chapter 13.*

**Cache stampede** — Failure mode where a hot cache key expires and many concurrent requests simultaneously hit the database to rebuild it. Mitigated with jittered TTLs, single-flight locks, or stale-while-revalidate. *See Chapter 13.*

**Canary release** — Rolling out a change to internal accounts or a tiny traffic slice first, watching metrics before wider exposure. First rung of the rollout ladder in Chapter 10.

**Changed-code coverage** — Coverage measured only against lines touched by a pull request, replacing global percentage targets. Gaps in critical paths become review findings. *See Chapter 20.*

**Classmap (Composer)** — Explicit class-to-file map built by `composer dump-autoload`; optimized variants (`--classmap-authoritative`) trade rebuild requirements for faster lookups in production images. *See Chapter 16.*

**Client credentials grant** — OAuth flow for service-to-service calls with no user involved: the service authenticates *as itself* with an ID/secret pair and receives a scoped token. Tokens cached until near expiry. *See Chapter 5.*

**Clock port** — Injected dependency providing the current time (`PSR-20` `ClockInterface` in PHP) instead of calling `new DateTime()` inline. Frozen implementations make time-dependent logic deterministic in tests. *See Chapters 19, 27.*

**Code splitting** — Dividing the JavaScript bundle into chunks loaded on demand, typically one per lazily imported route, keeping the initial load small. Verified with bundle visualizers, bounded by budgets. *See Chapter 24.*

**Composer lockfile** — `composer.lock` recording the exact resolved dependency versions and hashes, making installs reproducible. Committed to git; `composer install` honors it, `composer update` regenerates it deliberately. *See Chapter 17.*

**Composition API / `<script setup>`** — Vue 3's standard component style: top-level bindings auto-exposed to the template, TypeScript native, no options-object ceremony. All examples in this book use it. *See Chapter 23.*

**Conditional breakpoint** — Breakpoint pausing execution only when an expression evaluates true — essential when a loop runs thousands of times but fails on one item. *See Chapter 28.*

**Confused deputy** — Attack where a legitimate client is tricked into using its authority for the attacker's benefit, e.g., processing another tenant's resource ID. Defended by re-checking object ownership server-side on every request. *See Chapter 5.*

**Consumer group (Redis Streams)** — Mechanism letting multiple consumers share a stream's messages with acknowledgment and replay semantics — at-least-once delivery with inspection. *See Chapter 13.*

**Context-aware encoding** — XSS defense escaping untrusted data according to its destination context (HTML body, attribute, JS string, URL), since each context has different dangerous characters. Framework defaults handle common cases; `v-html` bypasses them all and is a reviewed exception only. *See Chapter 7.*

**Correlation ID** — Identifier generated once per request and propagated through every hop (NGINX → PHP → queue → Node → logs), turning seven unrelated log streams into one traceable story. Returned in problem+json responses for support workflows. *See Chapter 29.*

**Cosign / Sigstore** — Tooling for signing container images and verifying signatures at deploy time; unsigned artifacts can't reach production when verification is enforced in the cluster. *See Chapter 30.*

**Coverage** — Measure of which code lines executed during tests. Used as a spotlight for uncovered critical paths, never as a target number — target-driven coverage produces assertion-free theater. *See Chapter 20.*

**CSP (Content-Security-Policy)** — HTTP header defining allowlists for scripts, styles, and connections; the strongest defense against XSS when strict. Rolled out via Report-Only before enforcement. *See Chapter 12.*

**CSRF (Cross-Site Request Forgery)** — Attack where an attacker page makes the victim's browser send authenticated requests using automatically attached cookies. Defended with SameSite cookies and synchronizer/double-submit tokens. *See Chapter 7.*

**Cursor pagination** — Pagination scheme using an opaque pointer to a position (typically encoded sort keys), giving stable, constant-cost pages immune to insert/delete races. Default for any growing list in this book. Contrast: offset pagination. *See Chapter 8.*

**Data provider** — PHPUnit mechanism feeding many input/output rows through one test method; each row reports as its own named case. The engine of authorization matrix tests. *See Chapter 20.*

**DATETIME(6)** — MySQL type storing wall-clock values at microsecond precision without time-zone conversion or the TIMESTAMP range limit. Book policy: DATETIME(6) holding UTC, application-side conversion. *See Chapter 9.*

**Dead letter queue (DLQ/DLX)** — Holding area for messages that exhausted retries or were rejected as permanently unprocessable. Every queue needs one; arrivals deserve tickets. *See Chapter 14.*

**Deadlock (MySQL)** — Two transactions each holding a lock the other needs; InnoDB detects and kills one (error 1213). Normal and expected: keep transactions short, lock consistently, retry on 1213. *See Chapter 9.*

**Definition of done** — The agreed completion criteria for a ticket, including accessibility checks, flag expiry, and cleanup — not merely "code works." Where accessibility becomes an acceptance criterion rather than an audit. *See Chapters 4, 32.*

**Dependency pre-bundling** — Vite's first-run transformation of CommonJS/node_modules dependencies into single ESM files via esbuild, cached in `node_modules/.vite`. Deleting that cache fixes most dev-server oddities. *See Chapter 24.*

**Distillation of 401/403/404** — Status-code semantics as an authorization concern: 401 "unknown caller," 403 "known but denied" (confirms existence), 404 preferred for object-level denials to prevent enumeration leaks. Matrix tests assert these choices. *See Chapters 6–7.*

**Distroless image** — Container image containing only the runtime — no shell, package manager, or utilities. Minimal attack surface; most valuable for compiled runtimes, awkward for PHP/Node entrypoint conventions. *See Chapter 11.*

**Docker Compose** — Tool describing multi-container applications in YAML; used throughout for local parity with production topology. *See Chapter 11.*

**Doctrine** — PHP's ORM: entities mapped via attributes, Unit of Work change tracking, identity map, DQL/QueryBuilder query language. *See Chapter 15.*

**Double-submit cookie** — CSRF defense storing a random token in both a cookie and a request header; server rejects mismatches. Attacker pages cannot read either copy to align them. Fits SPAs naturally. *See Chapter 7.*

**DPoP** — Demonstration of Proof-of-Possession: sender-constraining OAuth tokens to a client-held key so intercepted tokens are unusable elsewhere. *See Chapter 5.*

**DQL (Doctrine Query Language)** — Object-oriented query language targeting entities and associations rather than tables. Compiles to SQL; parameter binding remains mandatory — concatenating values into DQL is SQL injection. *See Chapters 7, 15.*

**DTO (Data Transfer Object)** — Explicit object defining exactly which fields a client may submit or receive. Never bind requests directly onto entities — mass-assignment protection lives here. *See Chapters 18, 21.*

**E2E journey** — One end-to-end Playwright test asserting a complete business outcome ("board shows archived") across all layers. Capped count, business-outcome assertions, traces on failure. *See Chapter 26.*

**Emits** — Vue component declaration of events a component may fire upward, typed like props. Props down, emits up is the core data-flow contract keeping components boring. *See Chapter 23.*

**Engines (npm)** — Package.json field declaring required runtime versions (`node >=22`), paired with `engine-strict=true` so mismatched environments fail loudly rather than misbehave subtly. *See Chapter 22.*

**Error budget / SLO** — Service-level objective defining acceptable reliability; the error budget is how much failure remains tolerable. Burn-rate alerts catch slow degradation that static thresholds miss. *See Chapter 29.*

**ESLint strict-typechecked rules** — TypeScript-aware linting catching unsafe async patterns (`no-floating-promises`, `no-misused-promises`) statically. An unawaited promise is a silent production failure; these rules make it a build failure. *See Chapter 25.*

**Event loop** — Node's single-threaded scheduler running I/O callbacks one at a time; enables cheap concurrency and means CPU-bound work blocks everything. Heavy computation belongs in workers or separate processes. *See Chapter 22.*

**Event subscriber/listener** — Symfony mechanism hooking kernel lifecycle moments (request, controller, exception, response). Home of cross-cutting HTTP concerns like problem+json error mapping. Distinct from Messenger (async, durable) and domain events (business facts). *See Chapters 8, 18.*

**Exchange (RabbitMQ)** — Routing entity producers publish to (direct, topic, fanout); bindings map routing keys to queues. Producers never publish to queues directly. *See Chapter 14.*

**Expand/contract** — Migration pattern sequencing schema changes safely: additive expand → dual-write code → backfill → contract (drop old shape). Required for zero-downtime deploys and functional rollbacks. *See Chapters 15, 31.*

**EXPLAIN** — MySQL command revealing a query's execution plan (index choice, row estimates, scans). First tool for any slow query investigation. *See Chapter 9.*

**Fail closed** — Security principle that failed checks deny access: if the authorization service errors, reject the request. Catch-and-continue patterns turn every dependency outage into an open door. *See Chapter 7.*

**FastCGI / `fastcgi_pass`** — Binary protocol between NGINX and PHP-FPM worker processes. Correct configuration matches only the front controller and marks that location `internal`. *See Chapter 12.*

**Fastify** — Schema-first Node microframework: JSON Schema validates input and strips output, structured logging ships built-in, plugins encapsulate concerns. Default choice for full-featured HTTP services in this stack. *See Chapter 22.*

**Feature flag** — Runtime switch decoupling deployment from feature release. Four types per Fowler: release, experiment, ops (kill switch), permission. Flags are inventory with owners and expiry dates. *See Chapter 10.*

**Flaky test** — Test failing intermittently without product changes. Quarantined with an owner under protocol; treated as a bug report, never rerun-until-green noise. *See Chapter 27.*

**Flat config** — ESLint's modern configuration format: one array of config objects evaluated top to bottom, replacing cascading `.eslintrc` inheritance. *See Chapter 25.*

**Foundry** — Modern PHP fixture factory library generating deterministic test entities programmatically per scenario, replacing rotting shared fixture files. *See Chapter 20.*

**Front controller pattern** — Architecture routing every request through one entry script (`public/index.php`) which dispatches internally. NGINX's job is enabling it safely: match only the known handler, mark it internal, use `$realpath_root`. *See Chapter 12.*

**FrozenClock** — Test double implementing the clock port at a fixed instant, making expiry logic, TTL math, and retry windows deterministic assertions instead of sleeps. *See Chapter 27.*

**Generated column** — MySQL column computed from an expression (virtual on read, or stored/materialized). Its killer feature: indexability, especially extracting JSON fields into queryable columns. *See Chapter 9.*

**Generated types** — TypeScript interfaces produced mechanically from the OpenAPI spec via tools like openapi-typescript, keeping frontend types synchronized with the API contract by construction. Regenerated in CI; never hand-edited; always ignored by linters/formatters. *See Chapters 21, 24.*

**Graceful shutdown** — Process teardown sequence on SIGTERM: stop accepting new work → drain in-flight jobs → close connections → exit. Sized inside the orchestrator's kill window; prevents losing queued messages mid-deploy. *See Chapter 22.*

**Healthcheck (liveness vs. readiness)** — Liveness asks "is this process alive?" (no dependency checks); readiness asks "can this instance serve traffic?" (checks what actually gates serving). Readiness failures remove instances from rotation without restarts. *See Chapters 11, 22.*

**HMR (Hot Module Replacement)** — Vite's development-time swap of changed modules in place, preserving component state where possible — no full reload, no rebuild wait. Requires WebSocket upgrade support through proxies. *See Chapter 24.*

**Horizontal privilege escalation** — Attack accessing another peer user's resources at equivalent privilege level ("Carlos reads Dana's board") rather than gaining admin rights. Tested explicitly via sequential-ID probing and cross-tenant assertions. *See Chapter 6.*

**HSTS (HTTP Strict Transport Security)** — Header forcing browsers to use HTTPS for future visits. Irreversible-ish once cached by visitors; raise max-age deliberately. *See Chapter 12.*

**HttpOnly** — Cookie flag forbidding JavaScript read access; mitigates XSS-based token theft (not XSS itself). Paired with `Secure` and SameSite attributes. *See Chapter 7.*

**Ice cream cone** — Inverted test pyramid anti-pattern: hundreds of slow brittle UI tests atop a thin unit layer. Arises from defaulting to E2E whenever unit testing feels hard. *See Chapter 27.*

**Idempotency** — Property of repeating an operation producing the same result. GET/PUT/DELETE are idempotent by HTTP semantics; dangerous POSTs require idempotency keys so network retries can't double-charge or double-send. *See Chapters 8, 14.*

**Identity map** — Doctrine guarantee that within one EntityManager, loading an entity twice returns the same object instance. Source of subtle bugs in long-running workers without periodic `clear()`. *See Chapter 15.*

**IDOR (Insecure Direct Object Reference)** — Vulnerability where sequential or guessable IDs grant access to other tenants' resources. Prevented by relationship-scoped query filters making unauthorized rows invisible, plus matrix tests asserting 404s. *See Chapters 6–7.*

**ILM (Index Lifecycle Management)** — Elasticsearch policy aging indices from hot (searchable) through warm (cheap storage) to deletion — retention enforced as code rather than manual cleanup. *See Chapter 29.*

**Immutable image** — Container artifact tagged by digest, never rebuilt under the same tag. Prerequisite for trustworthy rollbacks: the previous version still exists, byte-identical, in the registry. *See Chapters 30–31.*

**Jitter** — Random variance added to retry delays and cache TTLs, converting synchronized stampedes into trickles. Cheap insurance against retry storms. *See Chapters 13–14.*

**JSON Schema validation** — Fastify's native input/output validation: schemas validate requests, strip responses, generate docs, and feed TypeScript inference. *See Chapter 22.*

**JWKS (JSON Web Key Set)** — Published endpoint where an IdP exposes its signing keys; token validators fetch and cache keys from it to verify JWT signatures. *See Chapter 5.*

**JWT (JSON Web Token)** — Signed token format carrying claims. Validation requires checking signature against JWKS, exact issuer match, audience match, expiry with clock skew tolerance, and pinned algorithms. *See Chapter 5.*

**Key namespace** — Redis naming convention structuring keys by purpose and entity (`cache:board:{id}:view:v2`). Makes bulk inspection possible, collisions impossible, and versioned cache invalidation trivial. *See Chapter 13.*

**Kill switch** — An ops-type feature flag that instantly disables a misbehaving feature. Must be tested before launch; the fastest lever in the rollback playbook. *See Chapter 10.*

**Lazy loading (Doctrine)** — Association loading strategy deferring queries until first access. Convenient and dangerous in loops (the N+1 engine); kept per-mapping while fetch strategies are chosen per use case with query hints. Contrast: eager, extra-lazy. *See Chapter 15.*

**Leftmost prefix rule** — Composite index property: queries filter efficiently only by leading columns of the index, in order. Order composite indexes equality-first, range-column last. *See Chapter 9.*

**Library mode** — Vite build configuration producing a distributable package (design system, API client) rather than an application; peer dependencies stay external to prevent duplicate Vue instances. *See Chapter 24.*

**Lint-staged** — Tool running linters only on files staged for commit, keeping pre-commit feedback fast. Convenience layer; CI remains the enforcement authority. *See Chapter 25.*

**Live region** — ARIA mechanism (`aria-live`, `role="alert"`) announcing dynamic content changes to screen readers. Essential for SPA route changes and form errors. Rendered up front, updated later. *See Chapter 4.*

**Lockfile** — See *Composer lockfile*; the npm counterpart is `package-lock.json`, installed strictly via `npm ci`. Both are committed; both are supply-chain controls. *See Chapters 17, 22.*

**Logpoint** — Breakpoint variant that logs an expression without pausing execution — console.log without editing source code. Scatter freely; remove nothing. *See Chapter 28.*

**Logstash** — Pipeline component collecting structured log output from containers/files, enriching it with environment metadata, and shipping to Elasticsearch. Alternatives: Filebeat, Vector. *See Chapter 29.*

**Manual chunks** — Rollup configuration grouping specific modules into shared bundles, trading long-term cache stability against chunk size. Decide by measurement, not habit. *See Chapter 24.*

**Mass assignment** — Vulnerability where binding user input directly onto models lets attackers set fields they shouldn't (e.g., `status`, `isAdmin`). Prevented by DTOs defining exactly the writable surface. *See Chapter 18.*

**maxmemory-policy** — Redis eviction behavior when memory fills: `noeviction` fails writes loudly (correct for queues/locks), `allkeys-lru` evicts cold entries (correct for caches). One shared instance can't have both — isolate by instance. *See Chapter 13.*

**Merge gate** — CI job whose failure blocks merging to a protected branch. All enforcement lives here; local git hooks are convenience only and can always be skipped. *See Chapters 19, 25, 30.*

**Merge gate** — See prior section.

**Messenger** — Symfony's message bus abstraction mapping producer/consumer vocabulary onto transports (AMQP/RabbitMQ), with per-transport retry strategies and a failed-message transport as dead-letter destination. *See Chapters 14, 18.*

**Mix-up attack** — OAuth attack exploiting multi-IdP clients: the attacker redirects the client's authorization code to the wrong token endpoint. Defended by binding IdP selection into `state` and validating response issuers. *See Chapter 5.*

**Monorepo** — Repository holding multiple related projects (apps, services, shared packages) with shared tooling; enabled by npm workspaces, path repositories, and project references in this stack. *See Chapters 21–22.*

**Multi-stage build** — Dockerfile pattern compiling in one stage and copying only artifacts into a slim final stage, keeping compilers and dev dependencies out of production images. Also where Xdebug stays behind. *See Chapters 11, 28.*

**N+1 query** — ORM anti-pattern where accessing related entities triggers one additional query per parent record. Detected by SQL logging in dev; fixed with fetch joins. *See Chapters 9, 15.*

**Named volume** — Docker-managed persistent storage surviving container recreation. Mandatory for databases; the rule is bind mounts for source code in dev, named volumes for data everywhere. *See Chapter 11.*

**Navigation guard** — Vue Router hook running before route resolution completes; used for auth gates and coarse redirects. UX only — the server enforces regardless. Keep guards fast; anything async delays every navigation. *See Chapter 23.*

**NGINX** — Reverse proxy, static file server, and TLS terminator serving as the system's front door: routing `/api/*` to PHP-FPM, `/hooks/*` to Node, everything else to SPA assets. *See Chapter 12.*

**Nonce** — Random value included in OIDC authentication requests and verified against the ID token's claim, preventing token replay in authentication flows. *See Chapter 5.*

**npm ci** — Clean install strictly honoring the lockfile; the CI/production counterpart to developer `npm install`. Faster and stricter than install; never substitute update commands in pipelines. *See Chapter 22.*

**OAuth 2.0** — Delegated authorization framework answering "can this client access this resource?" Sanctioned grants: Authorization Code + PKCE (user-facing) and client credentials (service-to-service). Implicit and password grants are dead. *See Chapter 5.*

**Offset pagination** — Pagination via `LIMIT/OFFSET`: simple, supports jump-to-page, degrades linearly with depth and races on concurrent writes. Allowed only for small, bounded lists. Contrast: cursor pagination. *See Chapter 8.*

**OIDC (OpenID Connect)** — Authentication layer built atop OAuth 2.0, adding ID tokens carrying verified user claims. The only source of identity in OAuth flows. *See Chapter 5.*

**OPcache** — PHP bytecode cache eliminating recompilation per request. Always-on since PHP 8.5; production still tunes memory limits and JIT settings deliberately. *See Chapter 16.*

**OpenAPI** — Machine-readable API specification serving as the source-of-truth contract; types and Bruno collections generate from it, spec-first. *See Chapter 8.*

**Outbox pattern** — Technique making a database write and its event publish atomic: both inserted in one transaction; a relay publishes afterward. Eliminates divergence between state and events. Consumers stay idempotent. *See Chapter 14.*

**P95/p99 latency** — Response-time percentiles showing tail experience that averages hide. Primary capacity signal alongside queue lag and slow-query counts. *See Chapters 29, 31.*

**Path mappings** — Debugger translation between container file paths and host paths; misconfiguration is why Xdebug breakpoints silently fail to bind in Docker setups. Must match bind mounts exactly. *See Chapter 28.*

**Path repository** — Composer repository type mapping local directories as packages with symlinked live development — the monorepo mechanism for shared libraries developed alongside consumers. *See Chapter 17.*

**php-cs-fixer** — Automated code-style rewriter targeting PER coding style; config committed, CI check mode enforced, style debates ended permanently. Large initial reformats land as standalone PRs. *See Chapter 19.*

**PHPStan level** — Static analysis strictness scale (0–10); level 10 demands exhaustive null-safety and full typing. New projects start at max; inherited codebases ratchet up gradually with expiring baselines. *See Chapter 19.*

**PHPUnit / Vitest** — Unit test frameworks for PHP and TypeScript respectively. Both run without I/O at the pyramid base; design feedback from hard-to-test code is a feature, not friction. *See Chapters 20, 26.*

**Pinia** — Vue's official store library. Stores model domain-scoped application state (auth, UI preferences); server state lives elsewhere. Not a dumping ground, not a service locator. *See Chapter 23.*

**PITR (Point-in-Time Recovery)** — Restoring a database to any moment using full backups plus binary logs. The only defense against logical disasters ("someone ran the wrong DELETE") that replication faithfully copies everywhere. *See Chapters 9, 31.*

**PKCE (Proof Key for Code Exchange)** — Authorization-code extension where the client sends a hashed verifier upfront and proves possession at exchange time. Mandatory for public clients; protects intercepted codes. Pronounced "pixy." *See Chapter 5.*

**Playwright** — Browser automation framework for end-to-end journeys against the full Compose stack: auth fixtures, network interception for error cases, axe a11y scans, trace artifacts on failure. Deliberately capped at ~20–40 flows. *See Chapters 4, 26.*

**Poison message** — Message failing on every attempt due to malformed content or triggered bugs. Handled by classifying failures (transient → retry, permanent → dead-letter immediately) so one bad record cannot consume capacity forever. *See Chapter 14.*

**POUR** — WCAG's four principles: Perceivable, Operable, Understandable, Robust. The review lens applied to every component. *See Chapter 4.*

**Prefetch (RabbitMQ)** — Number of unacknowledged messages one consumer holds. Too high: one slow consumer hoards work. Too low: throughput suffers. Start around 10; tune under load. *See Chapter 14.*

**Problem details (RFC 9457)** — Standard JSON error format (`type`, `title`, `status`, `detail`, `instance`, extensions) superseding RFC 7807. One error handler branches on `type`; field errors ride in extensions. *See Chapter 8.*

**Project references** — TypeScript mechanism layering configs (`base` → app/node/vitest) across a monorepo, giving consistent strictness, whole-project incremental builds, and clean module boundaries. *See Chapter 21.*

**Protected branch** — Git branch configuration forbidding direct pushes, requiring reviews and passing CI, and protecting history from force-pushes. Without protection, every workflow discipline in this book is voluntary. *See Chapter 2.*

**Provide/inject** — Vue dependency-passing mechanism skipping prop drilling through deep trees. Reserved for genuine app-root concerns (theme, i18n); invisible dependencies otherwise. *See Chapter 23.*

**PSR / PER** — PHP standard recommendations (autoloading, HTTP interfaces, container, logging, caching, clock) and their living successor coding style. Type-hint interfaces, not implementations. *See Chapter 19.*

**Quarantine (flake protocol)** — Marking an intermittently failing test to run separately without blocking merges, with a named owner and a fix-or-delete deadline. Never silent deletion, never retry-and-forget. *See Chapter 27.*

**RabbitMQ** — AMQP broker used for PHP-side domain events: exchanges, bindings, ack/nack semantics, prefetch tuning, and dead-letter exchanges. Chosen over BullMQ because the owning runtime is PHP. *See Chapter 14.*

**RBAC (Role-Based Access Control)** — Roles map to permissions, users to roles. Fast to start, degrades into role soup when real-world exceptions accumulate. Roles remain useful as one attribute among several. *See Chapter 6.*

**Readiness probe** — See *Healthcheck*.

**Read replica** — MySQL secondary serving read traffic, scaling reads at the cost of replication lag and divergent-failure modes. Replicas are availability, not backup. *See Chapter 9.*

**ReBAC (Relationship-Based Access Control)** — Authorization modeled as graph traversal: "user is member of org that owns board." Handles ownership, sharing, and inheritance naturally. The core model in this book, layered with ABAC conditions. *See Chapter 6.*

**Rector** — Automated refactoring tool performing language and framework upgrades mechanically; diffs reviewed like PRs, shipped standalone. *See Chapter 19.*

**Referrer-Policy** — Header controlling URL leakage via the Referer header; `strict-origin-when-cross-origin` is the sane default set at the edge. *See Chapter 12.*

**Reflog** — Git's local record of HEAD movements; the undo button for "lost" commits. Proof that almost nothing in Git is truly lost. *See Chapter 2.*

**Refresh token rotation** — Practice of issuing a new refresh token on each use and invalidating the old one, with reuse detection revoking the whole family on theft signals. *See Chapter 5.*

**Retry storm** — Failure cascade where every dependent system's accumulated retries hammer a recovering service simultaneously, knocking it down again. Prevented by exponential backoff with jitter and capped attempts. *See Chapter 14.*

**Retry strategy (Messenger)** — Per-transport retry configuration: capped attempts with multiplying delays before dead-lettering. Handlers classify exceptions transient-vs-permanent; retries never apply to permanent failures. *See Chapter 14.*

**Review app** — Per-merge-request isolated environment running the MR's built images behind a clickable URL, auto-stopped after a few days. Makes UI changes reviewable by use, not imagination. *See Chapter 30.*

**RFC 9457** — See *Problem details*.

**Rollback playbook** — Ordered recovery levers from fastest to heaviest: ops flag → config change → immutable image rollback → point-in-time restore. Works only when deploys ship dark and migrations are backward-compatible. *See Chapters 10, 30–31.*

**RTO / drill-verified recovery time** — Recovery Time Objective as actually measured by monthly restore drills, not as claimed in documentation. First drills always find gaps; finding them costs an afternoon instead of an outage. *See Chapter 31.*

**Runbook** — Operational procedure stored beside the service it describes: symptoms, diagnosis commands, remediation steps, verification. Written during calm; linked directly from alerts. *See Chapter 31.*

**Same-Origin Policy (SOP)** — Browser rule isolating origins: cross-origin scripts may send requests but cannot read responses. The asymmetry explains CSRF (exploits sending) and CORS (relaxes reading). *See Chapter 7.*

**SameSite cookies** — Cookie attribute controlling cross-site sending: `Lax` (default; blocks classic CSRF form posts), `Strict` (never sent cross-site). Combined with `Secure` and `HttpOnly` as baseline session hygiene. *See Chapter 7.*

**SBOM (Software Bill of Materials)** — Machine-generated inventory of every component in a build artifact, enabling instant "where do we use X?" queries when CVEs land. Generated in CI alongside image signing. *See Chapter 30.*

**Scroll behavior** — Vue Router configuration restoring saved positions on back-navigation and scrolling to top/anchors on forward navigation. Part of making SPA routing indistinguishable from MPA behavior. *See Chapter 23.*

**Semver caret (`^`)** — Version constraint accepting compatible updates within a major (`^7.0` → `<8.0`). A contract: your test suite makes minor updates safe; without tests, caret just means unvetted changes reach production silently. *See Chapter 17.*

**Sender-constrained tokens** — Access tokens cryptographically bound to a specific client (DPoP, mTLS) so stolen copies are useless elsewhere. Direction of travel for token security. *See Chapter 5.*

**Slot (Vue)** — Component composition mechanism letting parents inject content: default slots, named slots for layout, scoped slots passing data back up to the parent's template. *See Chapter 23.*

**Slow query log** — MySQL feature recording queries exceeding a threshold with their plans; piped into observability and reviewed weekly. Finds the queries users feel, including ORM-generated ones nobody profiled. *See Chapter 9.*

**Source maps** — Files mapping minified production output back to original sources, enabling meaningful stack traces. Shipped openly for internal tools; generated-but-hidden (`hidden`) and uploaded privately for public apps. *See Chapters 24, 28.*

**SPA fallback** — NGINX directive (`try_files $uri $uri/ /index.html`) serving the app shell for any unmatched path so client-side routing owns deep links — partitioned carefully from API route prefixes. *See Chapter 12.*

**SSRF (Server-Side Request Forgery)** — Attack tricking your server into fetching attacker-chosen URLs, targeting internal networks and cloud metadata. Treated as an authorization problem: allowlists, DNS validation, redirect re-checks. *See Chapter 7.*

**State parameter** — Random value in OAuth authorization requests, verified on callback, binding responses to their initiating request and defeating CSRF-style code injection. Distinct from `nonce`, which addresses replay. *See Chapter 5.*

**Structured logging** — Emitting machine-readable JSON lines with consistent fields (request ID, actor, tenant, duration, event name) instead of prose. The entry ticket for everything ELK-based observability provides. *See Chapter 29.*

**Synchronizer token** — Classic CSRF defense embedding a random per-session token in forms that submissions must echo back. Attacker pages can't read it due to same-origin policy. Cookie-session architectures require this or double-submit; bearer-token SPAs largely don't. *See Chapter 7.*

**TanStack Query** — Server-state caching library (Vue adapter): declarative queries keyed by parameters with deduplication, background refetching, and invalidation. Owns all server state in this book's convention; Pinia owns client state; nothing owns both. *See Chapter 23.*

**Testcontainers** — Library spinning up throwaway Docker containers (MySQL, Redis) for integration tests, replacing mocks that guess at real infrastructure behavior. *See Chapter 27.*

**Test double taxonomy** — Precise vocabulary replacing generic "mock": dummy (fills space), stub (canned answers), fake (working lightweight implementation), mock (verifies interactions), spy (records for later verification). Mocks only when interaction itself is the requirement. *See Chapter 20.*

**Test pyramid** — Fowler's shape: many fast isolated units at the base, fewer integration tests at seams in the middle, few E2E journeys on top. Push coverage down wherever possible; E2E earns expense only for cross-layer truths. *See Chapter 27.*

**Threat modeling (per-endpoint)** — Ten-minute pre-implementation exercise asking four STRIDE-derived questions about one endpoint, recorded in the PR description. *See Chapter 7.*

**Token bucket** — Rate-limiting algorithm issuing bursts against a sustained refill rate; NGINX's `limit_req_zone` implements it keyed by IP or header. Returns 429 so clients know to back off. *See Chapter 12.*

**Trace ID** — Distributed-tracing identifier propagated alongside correlation IDs, linking spans across services into a full request timeline. Same propagation path, richer structure than plain request IDs. *See Chapter 29.*

**Tree shaking** — Build-process elimination of unused exports from bundles, enabled by ES module static structure. Broken by side-effectful imports; verified via bundle analysis. *See Chapter 24.*

**try_files** — NGINX directive trying literal paths before falling back (`$uri $uri/ /index.php` or `/index.html`). Implements both the front-controller pattern and SPA fallback; ordering relative to API location blocks determines correctness. *See Chapter 12.*

**Two-minute drill** — Observability maturity test: pick any recent transaction and answer "what happened to it, and where did it fail?" within two minutes using logs alone. Run quarterly; every failure reveals a missing log point or broken correlation. *See Chapter 29.*

**Unit of Work** — Doctrine's change-tracking mechanism comparing entity snapshots at flush time to generate minimal SQL. Powers implicit saves; requires care in long-running processes. *See Chapter 15.*

**utf8mb4** — Full Unicode character set (four-byte) supporting emoji and rare CJK; the modern MySQL default, set explicitly because legacy configs linger. Pairs with `0900_ai_ci` collation. *See Chapter 9.*

**Vertical slice** — One feature threading through every layer end to end — threat model, contract, schema, backend, queue, frontend, tests, rollout — rather than horizontal layer-by-layer delivery. Chapter 32 walks the canonical example.

**View transitions** — Animated route changes via wrapping `<RouterView>` in `<Transition>`, respecting `prefers-reduced-motion` throughout. Polish tier, not structure. *See Chapters 4, 23.*

**Vite** — Frontend build tool and dev server: native ESM serving and HMR in development, Rollup-bundled code splitting in production. Source maps bridge minified output back to original sources. *See Chapters 24–25.*

**Voter (Symfony)** — Class encapsulating one authorization decision (`isGranted('EDIT', $board)`), wired into firewalls and called from controllers/services. Home of object-level checks. *See Chapters 6, 18.*

**WCAG 2.2** — Web Content Accessibility Guidelines; conformance target AA. The standards layer beneath POUR. *See Chapter 4.*

**Webhook** — HTTP callback delivered to a registered URL when events occur. Inbound ones demand signature verification; outbound ones demand rate limits, timeouts, backoff, and dead-lettering. *See Chapters 1, 14.*

**WebTestCase** — Symfony integration test base booting the full kernel and issuing real HTTP requests through routing, firewall, controllers, and Doctrine — verifying seams unit tests structurally cannot. *See Chapter 20.*

**Worker threads** — Node's true parallelism module for CPU-bound computation sharing process state. Rarely needed here: separate processes scale more simply, and heavy work belongs in queue jobs anyway. *See Chapter 22.*

**Workspaces (npm)** — Monorepo mechanism hoisting dependencies and linking local packages across multiple Node projects in one repository. Adopt when two-plus Node projects share code. *See Chapter 22.*

**Worktree** — Git feature checking out multiple branches into separate directories from one repository — the clean answer to mid-refactor hotfixes. *See Chapter 2.*

**Write-through** — Caching pattern writing to cache and database together on every mutation. Always fresh, but couples failure domains and pays cache latency on every write. Reserved for hot counters; cache-aside is the default. *See Chapter 13.*

**Xdebug** — PHP's step debugger: breakpoints, stack inspection, path mappings through Docker. Development images only, never production. *See Chapter 28.*

**Zod / valibot** — TypeScript schema libraries validating data at runtime while inferring static types from the same definition. Runtime validation lives at the edge (API responses, WebSocket frames); downstream trusts the inferred types. *See Chapter 21.*
