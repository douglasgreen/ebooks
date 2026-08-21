# A practical handbook for PHP 8.5 / Symfony and TypeScript / Vue engineers

**Identity, APIs, security, testing, and delivery — from local Docker to production GitLab CI.**

---

# Part I — Foundations

## Chapter 1. The Dual-Stack Mental Model

Before we write a single line of code, we need to settle a question that trips up almost every team running more than one backend technology: **who is responsible for what?**

This book builds a real application with a real architecture. That architecture uses PHP (specifically Symfony), Node.js, and Vue.js at the same time. That combination sounds like a recipe for chaos, and it *will* be — unless you adopt the mental model this chapter lays out. The model is simple: each runtime has a job, and each job has one runtime.

### Why This Shop Is Not "PHP *or* Node"

Framework wars waste time. The honest answer to "Symfony or Node?" is: **they solve different problems, so we use both — deliberately.**

- **Symfony owns the domain and persistence.** Your business rules live here: what an order is, who can cancel one, how a refund affects inventory. Symfony's maturity, Doctrine ORM, and ecosystem (forms, security, validation, migrations) make it the right home for logic that must be correct, auditable, and long-lived.
- **Vue owns interaction.** The browser is where users click, type, drag, and expect instant feedback. Vue components talk to the backend over an API and manage everything the user sees and touches in real time.
- **Node microframeworks own glue.** Webhooks from Stripe, background workers crunching jobs, a backend-for-frontend (BFF) that reshapes API responses for the Vue app, small internal admin tools — these are thin, I/O-heavy services where Node's speed of development and its enormous npm ecosystem shine.

Notice the pattern: the *core* of the business sits in one place. The *edges* — the parts that touch external services, queues, and browsers — use lighter tools.

A useful analogy: Symfony is the bank vault, Vue is the branch office lobby, and Node services are the couriers running between buildings. You wouldn't store cash in the lobby, and you wouldn't make customers wait outside the vault.

### Request Lifetime: Following One Request Through the System

Full-stack development makes far more sense once you can trace a request end to end. Here is the journey of a typical "save my shopping cart" request:

1. **Browser.** Vue intercepts the click, validates the input, and sends an HTTP request (usually JSON over `fetch` or Axios).
2. **NGINX.** The reverse proxy receives the request and routes it based on rules: `/api/*` goes to PHP-FPM; `/api/bff/*` or `/webhooks/*` goes to a Node service; static assets are served directly.
3. **PHP-FPM or Node.** The application layer runs. A Symfony controller validates the payload, invokes domain services, and Doctrine persists changes. A Node worker, meanwhile, might be pulling a job off a queue instead.
4. **MySQL.** The source of truth. Orders, users, products — durable relational data lives here and only here.
5. **Redis.** Fast, temporary data: sessions, caches, rate-limit counters, and the queues that feed Node workers.

The key insight: **most requests only touch part of this chain.** A webhook from a payment provider enters at NGINX, hits Node, and ends in the queue. A cached product listing may never reach MySQL at all. Understanding which segments a request visits — and why — is the difference between debugging and guessing.

### Bounded Contexts and the API as the Contract

When two runtimes share a system, you need a hard boundary between them. That boundary is the **API**, and the discipline that keeps it clean comes from a concept called **bounded contexts**.

A bounded context is a region of the system with its own vocabulary and its own data. "Order" in the checkout context means something slightly different from "Order" in the fulfillment context — and that's fine, because they don't share tables or objects. They share *messages*.

In this architecture:

- Symfony exposes a versioned REST (or JSON:API) interface describing its contexts: `/api/orders`, `/api/customers`, and so on.
- Vue and the Node services are **consumers** of that contract. They never read MySQL directly. They never import Symfony's PHP classes. They only know the API.
- Node BFF services may expose their *own* smaller contracts, stitched together from upstream APIs, tailored to what a specific screen actually needs.

Why does this matter so much? Because the contract is what buys you freedom:

- You can refactor Symfony internals without touching Vue.
- You can rewrite a Node worker in another language entirely, and nothing else notices.
- You can version the API (`/api/v2/...`) and migrate consumers gradually.

The moment a Vue component makes a raw SQL query, or a Node service reaches into Doctrine entities, the contract is broken and the whole model collapses. **The API is the only doorway. Use the doorway.**

### What "Full-Stack" Means Here

Many people hear "full-stack developer" and picture someone equally expert in every layer, from CSS animation to database indexing. That's a myth, and chasing it produces exhausted generalists.

In this book, full-stack means: **you own a feature end to end.**

Building a "product reviews" feature means you write:

- The Doctrine entities and migration in Symfony
- The API endpoint that validates and serves reviews
- The Node worker that sends the "review posted" notification
- The Vue component with the star rating and the form

You'll go deeper on some layers than others — that's normal and fine. What matters is that you understand how the pieces connect and can change any of them without fear. Depth where it counts, competence everywhere, and a map of the whole territory.

### When *Not* to Introduce a New Runtime or Store

Every additional runtime and datastore has a standing cost: deployment pipelines, monitoring, security patching, on-call knowledge, and the cognitive load of "which thing do I put where?"

So before adding anything new, ask these questions:

- **Is there an existing tool that can do this, even imperfectly?** MySQL can store JSON. Redis can serve as a lock manager. Symfony can process queues with Messenger. An imperfect fit you already run often beats a perfect fit you have to operate.
- **Is the new tool solving a *scaling* problem you actually have?** Teams routinely adopt a document database for data that fits comfortably in three MySQL tables. Solve today's problem today.
- **Who will maintain it at 3 a.m.?** If only one person on the team knows the new store, you've added a bus-factor risk, not a capability.
- **Can the boundary be an API call instead of a new service?** Sometimes the right move is a new Symfony endpoint, not a new microservice.

A good rule of thumb: **add a runtime or store when a concrete, measured need forces it — not when a blog post inspires it.** The stack in this book is already two runtimes and three stores; that's the ceiling we'll defend throughout, and we'll only lean on each piece within its job description.

### Chapter Summary

- Three runtimes, three jobs: Symfony for domain and persistence, Vue for interaction, Node for glue.
- Every request is traceable: browser → NGINX → PHP-FPM or Node → MySQL, Redis, queue.
- Bounded contexts communicate only through the API. The API is the contract; never go around it.
- Full-stack means owning features end to end, not mastering every layer equally.
- New runtimes and stores are liabilities until proven otherwise. Default to what you already run.

With this mental model in place, we're ready to set up the development environment — which is where the dual-stack rubber meets the road.

# Chapter 2. Git as Daily Craft

Most developers learn Git as a survival skill: commit, push, pray. This chapter treats Git as a craft instead — a tool whose daily habits shape how well your team reviews code, hunts bugs, and stays out of trouble six months from now.

The core idea is simple: **Git is not a backup system. It's a communication system.** Every commit you write will be read by humans (reviewers, future teammates, your future self via `git blame`). Writing Git history for those readers changes how you work.

---

## The Mental Model: Commits Are Decisions, Not Save Points

If you treat a commit as "saving your work," you'll end up with histories like this:

```
wip
fix
fix again
actually fix
asdfgh
```

That history is write-only. Nobody can read it, review it, or reason about it.

A better model: **a commit is a single, reviewable decision.** Each one should answer the question, *"What did you change, and why?"* If you can't summarize the decision in one sentence, the commit is probably doing too much.

### What this looks like in practice

- Refactoring a function and changing its behavior? That's two commits.
- Fixing a typo you noticed along the way? That's a separate commit (or a follow-up).
- Adding a feature plus its tests? One commit — the tests are part of the decision, not decoration.

Small, purposeful commits give you three concrete payoffs:

1. **Reviewable diffs.** Reviewers can actually understand each change.
2. **Safe rollbacks.** When a bug appears, you can revert one decision without unwinding a week of work.
3. **Accurate `git blame` and `git bisect`.** When someone runs `git blame` on a buggy line, they land on a commit that explains the intent — not a 4,000-line monster called "Friday."

### Building the habit

Committing this way takes discipline. Two techniques help:

- **Stage deliberately.** Use `git add -p` (patch mode) to stage only the hunks belonging to one decision, leaving the rest for the next commit.
- **Commit early, reorganize later.** If you're deep in the zone, commit messy snapshots locally — then clean them up with an interactive rebase *before* anyone else sees them.

That last point matters and we'll return to it: local history is a scratchpad; pushed history is a contract.

---

## Branching: Short Lived, Protected by Default

Branching strategy causes endless team arguments, but a durable consensus has emerged in modern web development. It has three rules.

### Rule 1: Short-lived feature branches

A branch should live for **hours to a few days**, not weeks. The moment a branch ages past that, two things rot simultaneously: the branch drifts away from mainline, and the merge gets painful enough that people start avoiding it.

Long-lived branches don't just delay integration — they *hide* design decisions from the team. Two people can spend a month building conflicting architectures on two branches without noticing.

### Rule 2: A protected default branch

`main` (or `trunk`) should be a protected branch: no direct pushes, all changes arrive via pull request, and CI must pass before merge. The default branch is your deployable artifact — treat tampering with it the way you'd treat editing production by hand.

### Rule 3: No long-lived personal mains

Every team eventually meets a branch like `dmitri/main-final-v2-really`. Personal long-lived branches are where code goes to be forgotten. They defeat the point of shared history, create merge debt, and often hide unreviewed changes that eventually ship in a panic.

### Feature flags over long branches

If your feature is too big to finish in a few days, the answer isn't a longer branch — it's **breaking the work into smaller pieces** and hiding incomplete behavior behind a **feature flag** (a runtime switch that turns the feature off until it's ready). Small, merged, inert code beats big, unmerged, invisible code.

---

## History Hygiene: Rebase, Merge, Squash — and When to Leave History Alone

Once you have meaningful commits, you'll want tools to keep history clean. These tools are also how people destroy each other's work, so the rules matter.

### Rebase vs. merge

Both commands combine your branch with the latest `main`. The difference is the history they produce:

- **Merge** creates a merge commit — a node with two parents that says "these lines of work came together here." History is preserved exactly as it happened.
- **Rebase** replays your commits on top of the new `main`, as if you'd written them yesterday. History becomes linear and clean, but it's a *rewrite* — the original commits are replaced with new ones.

```bash
# Update your feature branch with the latest main
git fetch origin
git rebase origin/main
```

**Practical rule of thumb:**

- Rebase your **own, unpushed (or team-local) feature branch** onto `main` before opening a PR. Reviewers see only your decisions, not merge noise.
- Merge your branch into `main` through the PR — the merge commit records that integration happened.
- Use merge commits when merging **between shared branches** or when the parallel history genuinely matters (e.g., a release branch cherry-picking fixes).

### When to squash

**Squashing** collapses many commits into one. It's the right call when your branch's intermediate commits are scaffolding — `wip`, `fix tests`, `address feedback` — and the whole thing is really one decision.

Modern platforms make this a policy choice: GitHub's "Squash and merge," GitLab's squash option. Many teams squash-merge *every* PR by default, which keeps `main` perfectly linear: one PR, one commit.

The tradeoff: squashing discards fine-grained history. If you carefully built a branch as five well-crafted commits, squashing erases that work. This is why the "commits as decisions" habit still matters — squashing a branch of good commits is a choice, and some teams instead use "Rebase and merge" to preserve them.

### When NOT to rewrite history

Rebase, squash, amend, and filter-branch all **rewrite history** — they replace old commits with new ones. This is safe on your own unpushed work and dangerous everywhere else.

**Never rewrite commits that other people may have pulled.** If you force-push a rebased branch that a teammate based work on, their history and yours diverge, and recovering is misery.

Rules to keep yourself safe:

- Rewrite freely **before pushing**.
- After pushing to a **shared branch**, either don't rewrite, or coordinate and use `git push --force-with-lease`, which fails if someone pushed after you (unlike plain `--force`, which silently destroys their work).
- Never rewrite `main`. Protected branches should reject force-pushes outright; make sure they do.

And if something goes wrong anyway, that's what `git reflog` is for — more on that below.

---

## Commit Messages That Survive Code Review and `git blame`

Your commit message is read at two very different moments:

1. **In review**, where it frames the diff before a reviewer reads a single line.
2. **In `git blame`**, months later, when someone hits a bug and runs `git blame src/payments/charge.ts` to find out why that line exists.

A message like `fix` fails at both. Here's the standard format that works:

```text
Charge prorated amount for mid-cycle upgrades

Previously, upgrades billed the full new plan price immediately,
overcharging users with 20 days left in their cycle.

Calculate the unused remainder of the old plan and apply it as
credit toward the new plan's price.

Refs: #4821
```

### Anatomy of a good message

- **First line: imperative mood, ≤ 50-ish characters.** "Charge prorated amount," not "charging" or "fixed bug." It should complete the sentence *"If applied, this commit will…"*
- **Blank line, then the body.** The body answers *why*, not *what* — the diff already shows what changed. Explain the reasoning, the bug's cause, the tradeoff you chose.
- **Reference the ticket or issue** so the future reader can find the full context.

### The `git blame` test

Before pushing, ask: *if someone blames this code in a year and reads only my first line, will they understand the decision?* "Charge prorated amount for mid-cycle upgrades" passes. "fix billing" does not.

---

## The Tools You Actually Need: Bisect, Reflog, Worktrees, Sparse Checkout

Four Git features are dramatically underused relative to how useful they are.

### `git bisect` — find the commit that broke it

Binary search over your history. You tell Git a known-good commit and a known-bad commit; it checks out the midpoint, you test, say `good` or `bad`, and it halves the search space each round. Finding the culprit in a month of history takes ~10 steps instead of reading every diff.

```bash
git bisect start
git bisect bad                 # current commit is broken
git bisect good v2.3.0         # this version worked
# Git checks out a midpoint; test it, then:
git bisect good   # or: git bisect bad
# ...repeat until Git announces the first bad commit
git bisect reset
```

If you have a test that reproduces the bug, bisect can even run it for you:

```bash
git bisect run npm test
```

**Prerequisite:** this only works when commits are small and each one builds. A broken commit in the middle of the range breaks the search. This is the strongest practical argument for the "commits as decisions" model.

### `git reflog` — your safety net

The reflog records every place `HEAD` has pointed — every commit, checkout, rebase, reset. Even "lost" commits live here.

The classic panic scenario: you ran `git reset --hard` and wiped out uncommitted work, or a botched rebase ate your branch. The recovery:

```bash
git reflog
# find the entry from before the disaster, e.g. abc1234
git reset --hard abc1234
```

Commits aren't garbage-collected for ~30 days by default, so "I lost my work" is almost always recoverable. `git reflog` is the reason you can experiment boldly with history rewriting — the undo button exists.

### `git worktree` — multiple branches at once

Normally one repository checks out one branch at a time, which forces the `git stash` dance whenever you need to switch tasks. Worktrees let you check out **several branches simultaneously** in separate directories:

```bash
git worktree add ../hotfix-login main-hotfix
```

Now `../hotfix-login` is a full working copy on the hotfix branch while your main directory stays untouched. Ideal for: urgent bugfixes mid-feature, running two versions side by side, or long-running builds that would otherwise block branch switching.

### Sparse checkout — only the files you need

Monorepos can contain gigabytes of files irrelevant to your task. Sparse checkout limits your working directory to a subset of paths:

```bash
git sparse-checkout set apps/storefront packages/ui
```

You still have the full history — you just don't clutter your disk and editor with every directory in the repo. For web teams working in shared monorepos, this is often the difference between tolerable and miserable.

---

## `.gitignore`, LFS Policy, and What Never Belongs in the Repo

A repository should contain **source, and the definition of everything else** — not everything else itself.

### `.gitignore` fundamentals

`.gitignore` lists paths Git should never track. Every web project needs at minimum:

```gitignore
# dependencies — reproducible from package-lock.json, not vendored
node_modules/

# build output
dist/
build/
.next/
.cache/

# environment & secrets
.env
.env.*
*.pem

# editor & OS noise
.vscode/
.DS_Store

# logs & dumps
*.log
*.sql
```

A hard-won lesson: **add to `.gitignore` before the file exists.** Once a secret is committed, removing the file doesn't remove the history — the secret lives in every clone forever (more below).

### Git LFS (Large File Storage)

Git stores every version of every file forever, which is great for text and terrible for binaries. A 50 MB image edited ten times costs 500 MB of history — and Git can't meaningfully diff or merge binaries anyway.

**Git LFS** solves this: the repo stores a small pointer file, and the actual binary lives on a separate server, downloaded on demand:

```bash
git lfs install
git lfs track "*.psd" "*.mp4" "*.woff2"
git add .gitattributes
```

Have a **written policy** for what goes into LFS (commonly: images, fonts, video, design files above a size threshold) and what doesn't go into the repo at all. Left unmanaged, repos quietly grow to gigabytes and every clone slows down.

### What never belongs in the repository

Some things shouldn't be in Git at all, LFS or not:

- **Secrets — credentials, API keys, private keys, `.env` files.** This is a security incident in waiting. Secrets belong in a proper secrets manager or your platform's environment-variable store. If a secret *was* committed, treat it as compromised: rotate it immediately, then scrub history if needed. A leaked key in public history gets found by automated scanners within minutes.
- **`node_modules` and vendor directories.** They're reproducible from your lockfile. Committing them bloats the repo, invites merge conflicts in files nobody reviews, and — worse — smuggles unreviewed third-party code into your repo, bypassing every supply-chain control you have (see the end of this chapter).
- **Database dumps and large datasets.** They compress badly, diff as noise, and often contain real user data — a compliance problem disguised as a convenience.
- **Generated files.** Build output, coverage reports, docs generated from source. Commit the source; generate the rest in CI.

A good gut-check question for any file: *"Does a human need to review changes to this?"* If the answer is no, it almost certainly doesn't belong in version control.

---

## Code Review: A Security and Design Control

Code review tends to slide into style nitpicking — bikeshedded naming, quote preferences, things your linter should have caught. That wastes everyone's time and hides what review is actually for.

**Review is a control.** It's the last human checkpoint before code becomes part of the system your users depend on. Its real jobs:

1. **Design review.** Does this change fit the architecture? Are the tradeoffs sound? Is it testable and maintainable?
2. **Security review.** Does it validate input? Handle auth correctly? Introduce an injection, XSS, or access-control hole?
3. **Knowledge sharing.** At least one other person now understands this code — you've eliminated a single point of failure.

Reviews also enforce everything else in this chapter. A reviewer looking at five clean commits with honest messages can *actually review*. A reviewer staring at one 3,000-line "misc changes" commit will rubber-stamp it — and rubber-stamped review is worse than no review, because it produces a false sense of safety.

### Practical review discipline

- **Small PRs.** Review quality collapses somewhere past a few hundred lines of diff. Keep PRs small enough to read carefully.
- **Automate the nitpicks.** Linting, formatting, and type checks belong to machines. Humans review what machines can't: design, security, correctness.
- **Ask questions, don't issue commands.** "What happens if this array is empty?" reviews better than "Handle the empty case."
- **Blocking vs. non-blocking comments.** Distinguish "this must change" from "consider this" so authors know what's negotiable.
- **Approve means accountable.** "LGTM" is a statement that you'd be comfortable debugging this code at 3 a.m.

---

## Signing, Protected Branches, and Supply-Chain Basics

The final layer of Git craft: proving that commits are what they claim to be.

### Commit signing

Anyone can configure Git to claim they're anyone:

```bash
git -c user.name="The CEO" -c user.email="ceo@company.com" commit -m "approve my own raise"
```

**Commit signing** cryptographically proves a commit came from the holder of a private key. With SSH keys or GPG:

```bash
git config --global commit.gpgsign true
git config --global user.signingkey <your-key-id>
```

Signed commits display a "Verified" badge on GitHub and GitLab. For teams, signing closes the identity gap that review depends on — a reviewer approving "Alice's" commit should know it's really Alice's.

### Protected branches, revisited

Pull all the controls together on your default branch:

- Require PRs and at least one approval.
- Require CI to pass.
- **Reject force-pushes** — this makes history rewriting impossible by policy, not just convention.
- Require signed commits where your threat model justifies it.
- Require linear history (or a squash/rebase merge policy) if your team values it.

These settings are five minutes of configuration that prevent entire categories of accidents and attacks.

### Supply-chain basics

Here's why this all matters beyond Git hygiene. **OWASP's A03 (Injection) sits alongside Supply Chain vulnerabilities in the OWASP Top 10** — and modern supply-chain attacks increasingly start where this chapter's habits are weakest:

- An attacker who can push to a protected branch can ship malicious code directly — unless branch protection blocks them.
- An attacker who can forge commit identities can slip a malicious "fix" into review as a trusted colleague — unless commits are signed.
- A compromised dependency committed as vendored code bypasses your lockfile and dependency scanning entirely — which is why vendor directories stay out of the repo.

Your repository's integrity is the first link in your software supply chain. Signed commits, protected branches, and a clean, reviewable history aren't bureaucratic ceremony — they're the foundation the security chapters later in this book build on.

---

## Chapter Summary

- Treat commits as **reviewable decisions**, not save points. Small, purposeful, well-messaged.
- Keep branches **short-lived**, protect the default branch, and replace long-running work with feature flags.
- Rebase your own work before review; squash scaffolding; **never rewrite shared history** without coordination and `--force-with-lease`.
- Write commit messages for two readers: the reviewer today and the `git blame` user next year.
- Master the power tools: `bisect` for hunting bugs, `reflog` for undoing disasters, `worktrees` for parallel work, sparse checkout for monorepos.
- Keep the repo clean: `.gitignore` everything generated, use LFS deliberately, and **never commit secrets, vendor code, or dumps**.
- Treat review as security and design control — small PRs, real scrutiny, no rubber stamps.
- Sign commits, protect branches, and understand your repo as the first link in your supply chain.

Git habits are like posture: easy to neglect, painful to correct later, and invisible to everyone except those who have to work with you. Build them now, in the small daily moments, and the rest of this book stands on solid ground.

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
