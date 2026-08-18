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

# Part I — Foundations

## Chapter 1. The Dual-Stack Mental Model

Before you write a line of code in this shop, you need a map. Not a diagram pinned to a wall — a *mental model* you can hold while you're debugging at 11 p.m. This chapter builds that model. It explains how the pieces fit together, who is responsible for what, and — just as importantly — where the boundaries are so you know when *not* to cross them.

The one-sentence version: **Symfony is where the truth lives, Vue is where the user touches it, and Node is where the loose ends get tied up.** Everything else in this chapter is an elaboration of that sentence.

---

### 1.1 Why this shop is not “PHP *or* Node”

The first misconception to kill is the idea that this architecture is a *choice* between PHP and Node, as if we sat in a room and picked a side. We didn't. There is no contest, because the two runtimes are not competing for the same job.

A false dichotomy only looks real when you assume every line of code is the *same kind* of line of code. It isn't. The work in a web application falls into at least three different shapes, and each shape has a runtime that fits it well:

**Symfony owns the domain and persistence.** This is the system of record. It's where the business rules live — how an order is priced, what makes an invoice valid, what invariants must never be broken. It's where data is read and written through Doctrine, inside transactions, with the full weight of the ORM and the domain model behind it. This work is *correctness-critical* and *state-critical*: a bug here can corrupt real money or real records. Symfony is a mature, batteries-included framework for exactly this. It has a first-class ORM, a service container, validation, security, and a long track record of boring reliability. "Boring" is a feature when you're the thing of record.

**Vue owns interaction.** The browser is a different machine from the server, with different constraints: it's about rendering, state, and responding to human input at 60 frames per second. Vue is the framework for that world — components, reactivity, routing, and talking to the backend over HTTP. It doesn't care about your Doctrine entities. It cares about what the user sees and what happens when they click.

**Node microframeworks own the glue.** This is the catch-all for the work that is *I/O-bound and event-shaped* but isn't core domain: receiving a webhook from a payments provider, fanning a single request out to several internal APIs and stitching the result together (a BFF), consuming a queue and sending a confirmation email, or running an internal tool that a human pokes occasionally. This work is chatty, asynchronous, and short-lived. Node's event loop is a natural fit, and a microframework like Fastify or Express gets you a working HTTP endpoint in minutes without dragging in a full domain stack you don't need.

Notice what's happening: the boundary between runtimes is drawn along the **nature of the work**, not along the language. You're not asking "PHP or Node?" You're asking "what *kind* of thing is this?" and the answer tells you where it belongs.

- Is it a business rule or a write to the system of record? → Symfony.
- Is it something a human sees or touches? → Vue.
- Is it wiring, reacting, aggregating, or a fire-and-forget job? → Node.

When the work is ambiguous, that's a signal to stop and think, not to grab whichever runtime you're more comfortable in. We'll come back to exactly how to make that call in §1.5.

---

### 1.2 Request lifetime: browser → NGINX → PHP-FPM / Node → MySQL / Redis / queue

A mental model is only useful if you can trace a request through it. Let's do that with a concrete flow: **a customer places an order.**

```
Browser ──► NGINX ──► PHP-FPM (Symfony) ──► MySQL
   │            │              │
   │            │              └─► Redis (cache)
   │            │              └─► Queue (order.placed)
   │            │
   │            └─► Node (BFF / worker / webhook) ──► MySQL / Redis / 3rd party
   │
   └─ (Vue renders the result, polls for status)
```

Here's the walk-through, step by step:

1. **The browser** fires `POST /api/orders`. The Vue component that rendered the "Place order" button is what produced this request, but Vue's job is essentially done — it's now waiting.

2. **NGINX is the edge.** It terminates TLS, and then it makes the first routing decision of the request's life. This is the most important decision in the whole system, and it's cheap: *where does this URL belong?* NGINX serves static assets (Vue's compiled bundle, images) directly from disk and never touches a runtime. Everything else is proxied. A typical `location` block looks like this:

   ```nginx
   server {
       listen 443 ssl;

       # Static frontend — served directly, no runtime involved
       location / {
           root /var/www/frontend/dist;
           try_files $uri /index.html;   # Vue SPA fallback
       }

       # Domain API — the system of record
       location /api/ {
           fastcgi_pass unix:/run/php/php-fpm.sock;
           include fastcgi_params;
           fastcgi_param SCRIPT_FILENAME /var/www/symfony/public/index.php;
       }

       # BFF + webhooks + internal tools — the glue
       location /bff/ {
           proxy_pass http://127.0.0.1:4000;
       }
       location /webhooks/ {
           proxy_pass http://127.0.0.1:4000;
       }
   }
   ```

   The point is that **NGINX is where the split between "PHP-FPM" and "Node" is actually decided.** It's a routing table, and getting it right means each runtime only ever sees the traffic it's meant to handle.

3. **PHP-FPM runs Symfony.** The order request lands in a Symfony controller. This is where the domain work happens: the payload is validated, the pricing service runs, inventory is checked, and — inside a single database transaction — the order and its line items are written to **MySQL** through Doctrine. This is the *synchronous* path, and it's where correctness is non-negotiable. If two customers grab the last unit, the transaction and a proper lock (or an optimistic version check) are what stop you from selling it twice.

4. **Symfony hands off the rest to a queue.** The order is *saved*, but the downstream work — sending the confirmation email, updating the search index, notifying the warehouse — is not urgent and doesn't need to block the customer. So instead of doing it inline, Symfony publishes an `order.placed` event onto the **queue** and immediately returns `202 Accepted` to the browser. The request is done. The customer sees "Order placed" in about the time it took to write a row.

   **Redis** appears in two roles here. As a *cache* — say, the product catalog that's read far more than it's written — Symfony checks Redis before hitting MySQL. And, in some setups, as the *queue transport* itself. The exact broker (Redis, RabbitMQ, SQS) is a detail; the *concept* — decoupling the synchronous request from the asynchronous follow-up — is the part that matters.

5. **A Node worker picks up the job.** Somewhere else, a long-running Node process is blocked on the queue. It pops `order.placed`, sends the email, updates the index, calls the warehouse API. This is the *asynchronous* path. It can be slower, it can retry on failure, and it can be **eventually consistent** — the email might arrive a second after the page says "order placed," and that's fine. The queue is what lets the two paths have different consistency and latency requirements without stepping on each other.

6. **Back in the browser,** the Vue component may poll a status endpoint (often through the **BFF**, a thin Node layer that aggregates "order status + shipping ETA + recent activity" from several internal calls into one payload shaped for the UI) until the order shows as confirmed.

The mental model to carry away: **there is a synchronous spine and an asynchronous tail.** The spine (browser → NGINX → PHP-FPM → MySQL) is short, fast, and correct. The tail (queue → Node workers) is longer, chatty, and eventually consistent. The queue is the membrane between them, and it's the single most important design decision in keeping the system responsive under load.

---

### 1.3 Bounded contexts and the API as the contract

"Domain" is doing a lot of work in this chapter, so let's make it precise with a term from domain-driven design: the **bounded context**.

A bounded context is a boundary inside which a particular model of the world is *true and consistent*. The crucial insight is that the same real-world thing can have *different* models in different contexts, and that's not a bug — it's the point. Consider "customer":

- In the **billing** context, a customer has a payment method, a billing address, and a credit limit.
- In the **shipping** context, a customer has a delivery address, preferred carrier, and a signature requirement.
- In the **support** context, a customer has a ticket history and a sentiment score.

These are all "the customer," but they are *different models serving different purposes*. A bounded context is the fence that keeps them from bleeding into each other. Inside each fence, one model is authoritative and consistent. Crossing the fence, you translate.

In this shop, bounded contexts map onto **modules and services**, and each context exposes its own **API**. That API is the *contract*.

The contract is the agreement between two sides of a boundary: *this is what you can send me, this is what I will send back, and this is what each response means.* It specifies the shapes (request/response schemas), the semantics (what a `409` means here), the error format, the pagination rules, and the guarantees (is this read-your-writes? eventually consistent?).

Why does the contract matter so much? Three reasons:

**It's the seam.** The contract is the line where you can change one side without breaking the other. The Vue team can build and test a component against a documented contract — or a mock — while the Symfony team is still writing the domain service. They're decoupled by the agreement, not by the implementation. When the implementation changes, the contract doesn't have to.

**It keeps the domain from leaking.** A common mistake is exposing your Doctrine entities directly as your API. Don't. The database shape of an `Order` entity and the API shape of an order are *different models in different contexts* — the persistence context and the client context. Symfony should translate between them (via DTOs or API resources) so that a rename of a database column never breaks a frontend, and a sensitive column never accidentally ships to the browser. The contract is where you make that translation deliberate.

**It's where versioning and compatibility live.** Contracts get versioned, deprecated, and evolved. When you add a field, is it backward-compatible? When you change a status code, who's affected? Thinking in terms of contracts forces you to answer these questions *at the boundary*, where they're cheap, instead of discovering them in production.

So when you hear "bounded context" in this shop, translate it to: *a self-consistent slice of the domain with its own model and its own API.* And when you hear "the API is the contract," translate it to: *the API is the stable agreement that lets the people on either side of a boundary move independently.* Design the contract first, implement around it, and you'll spend far fewer afternoons untangling "but I thought that endpoint did X."

---

### 1.4 What “full-stack” means here

In a lot of shops, "full-stack developer" quietly means "person who is a competent expert in PHP, Node, Vue, SQL, *and* DevOps, all at once." That's not what it means here, and holding it to that standard will just make people avoid the label.

In this shop, **full-stack means you own the feature end to end — not that you hold every runtime to the same level of mastery.**

Concretely, a full-stack developer here can take a feature from the user's finger to the database and back, and — critically — *debug it at any layer along the way*. That might look like:

- Writing the **Vue** component and its state handling (the interaction),
- Writing the **Symfony** controller, domain service, and Doctrine mapping (the domain and persistence),
- Writing a **Node** BFF route or a queue worker when the feature needs aggregation or async follow-up (the glue),
- Reading the **NGINX** config to figure out why a request is landing in the wrong runtime,
- Running a query against **MySQL** to see what actually got written.

"End to end" is about **ownership and unblockability**, not parity. You should be able to look at a broken feature and say "the problem is in the Vue state, the Symfony service, the queue worker, or the SQL" — and then *go fix it*, even if that layer isn't your strongest. The boundaries in §1.1 exist precisely so that you can be effective across them without being a grandmaster of each one.

A realistic and healthy shape for a person here is to have a **home base** and extend from it. You might live mostly in Vue and Symfony and write Node glue only when a feature demands it. Or you might be strongest in Symfony and lean on the team's Vue conventions for the frontend. Neither is a failure. What would be a problem is being *blocked* by a layer you can't touch — stuck waiting on someone else to change one line in the worker because you "don't do Node." Full-stack, in this shop, is the ability to not be blocked.

There's a psychological payoff to the framing, too: the boundaries are a *gift*. Because the work is already sorted into three shapes, you don't have to decide "what language should this be?" from scratch every time. You ask "what *kind* of work is this?" and the map answers. That's a developer who can ship a feature across the whole stack, with depth where it counts and working competence everywhere else.

---

### 1.5 When *not* to introduce a new runtime or store

A polyglot shop has a specific disease: **the temptation to add.** A new language for the new feature. A new database because the access pattern *feels* different. A new framework because it's fashionable. Each addition is individually rational and collectively expensive, so this section is about the discipline of *not* adding.

Every new runtime or store you introduce is not free. It costs you:

- **Operations** — a new thing to deploy, monitor, back up, patch, and keep alive.
- **Skills** — now the team has to know it, and you can't assume everyone does.
- **Integration** — a new seam to build, a new place for data to get out of sync, a new failure mode.
- **Cognitive load** — every developer now has to hold one more mental model just to be effective.

The default position should be **reuse**. Before you reach for something new, ask whether the tools you already have can do the job. They usually can, and they can do it *boringly*, which is what you want for most of a system.

Here are the reflexes to build:

- **Don't add a second relational database "for the new feature."** If it's rows with relationships and you need transactions, it goes in MySQL. A new schema in the existing database is almost always the right answer.
- **Don't add a message broker when the queue you have is enough.** If you already have a queue (even one backed by Redis), use it. You don't need a second one because the new jobs are "different."
- **Don't add a new language for a cron job.** A scheduled task that a PHP console command or a Node script can handle doesn't need its own runtime and deployment.
- **Don't add a microservice (and thus a new runtime instance) when a Symfony module will do.** "Microservice" is an *operational* decision — it means you now own independent deployment, scaling, and failure isolation. If you don't genuinely need that, a module inside the existing app is cheaper and correct.
- **Don't add a cache layer where a single query is fine.** Redis earns its keep on hot, read-heavy data. If you're reading a row twice a day, a database index is your cache.

The test to run before adding anything: **is the ongoing cost of the new thing (ops + skills + integration) lower than the ongoing cost of *not* having it?** If you can't answer "yes" with confidence, you don't add it.

And to be fair, there *are* cases where adding is right — but they're specific and you should be able to name the reason:

- A **genuinely different access pattern** that the existing store handles badly: heavy full-text or fuzzy search (→ a search engine), graph traversals (→ a graph store), or a schema that's truly document-shaped and evolves per-record (→ a document store).
- A **different consistency or scale requirement** that justifies isolation: a high-write, low-latency subsystem that would degrade the system of record if it shared the database.
- A **different runtime shape**: you need a long-lived process that holds in-memory state and streams, where a request/response PHP-FPM worker is the wrong model.

The pattern in every legitimate case is the same as the pattern in §1.1: **the nature of the work genuinely requires it.** "It's trendy," "I know this one better," and "it seemed cleaner" are not the nature of the work. When you can point to a concrete property of the workload that the existing stack can't serve, you've earned the new runtime or store. When you can't, you reuse, and you sleep fine.

---

### Key takeaways

- **It's not PHP *or* Node.** Symfony owns the domain and persistence, Vue owns interaction, Node owns the glue. The boundary is drawn along the *nature of the work*, not the language.
- **Trace every request:** browser → NGINX (the routing decision) → PHP-FPM or Node → MySQL / Redis / queue. There's a synchronous, correctness-critical spine and an asynchronous, eventually-consistent tail, with the queue as the membrane between them.
- **Think in bounded contexts.** Each is a self-consistent slice of the domain with its own model and its own API, and the API is the *contract* — the stable seam that decouples the people on either side and keeps the domain from leaking.
- **"Full-stack" means owning the feature end to end and being unblockable at every layer** — not holding every runtime to the same mastery. Have a home base, extend from it.
- **The default is reuse.** Add a new runtime or store only when a concrete property of the workload requires it, and be ready to name that property. Everything else is cost, not capability.

Hold this map in your head and the rest of the book becomes a series of zoom-ins: each later chapter takes one region of this model and goes deeper. For now, just make sure the shape is right — *truth in Symfony, touch in Vue, glue in Node* — because everything you build will hang off it.

## Chapter 2. Git as Daily Craft

Most developers learn Git as a set of commands to memorize. This chapter treats it differently: as a *craft*, the way a mason learns stone or a carpenter learns wood. The commands are trivial; the value lives in the habits. A team that has internalized this chapter will write fewer tickets, debug faster, survive the inevitable "who broke production" morning, and — quietly but measurably — ship more secure software.

We'll build the chapter in two movements. First, the working model: how to think about commits, branches, and history. Second, the infrastructure: the tooling, policies, and controls that make that model hold up under real team pressure.

---

### 2.1 The Mental Model: Commits as Reviewable Decisions, Not Save Points

The most consequential error a new Git user can make is treating a commit as a *save point* — the way you save a Word document: often, cheaply, without thinking about what you just did. Under that model, a history looks like this:

```
fix
fix again
wip
asdf
more fixes
??
try 2
```

Nothing is wrong with any individual command here. Everything is wrong with the *model*.

Reframe the commit. A commit is a **reviewable decision**: the smallest unit of change that (a) makes the codebase correct and complete on its own, (b) can be described in one coherent sentence, and (c) could be reverted without surgery. "Correct and complete on its own" is the key clause. If commit N requires commit N+1 to compile, you have made two commits where you should have made one.

This is not academic. The decision-unit model is what makes three of Git's most valuable properties work:

1. **`git revert` becomes a real safety valve.** You can undo a bad decision in one atomic step because the decision was atomic.
2. **`git blame` becomes an investigation tool** rather than a guessing game (more in §2.4).
3. **`git bisect` becomes a precision instrument** (more in §2.5), because "the last commit before this broke" is a meaningful phrase.

A practical definition of "atomic" for daily work:

- **One concern per commit.** The refactor and the behavior change are two commits, even if they touch the same lines.
- **The tree builds and tests pass at every commit.** If you're mid-migration, the intermediate state compiles — use temporary stubs, delete them in the follow-up commit.
- **The message could stand in front of a reviewer** without you needing to explain context in a chat thread. If the commit message is "fix," the decision is not reviewable.

A useful drill: before you commit, run `git diff --staged` and ask, *"Could I defend this as one decision?"* If the diff contains a rename, a bug fix, and a formatting sweep, you have three decisions wearing one costume. Split them.

**What about work-in-progress?** WIP exists, and pretending otherwise breaks people. But WIP belongs in *uncommitted working-tree state* or a *throwaway local branch* — not in your mainline history. The working tree is your scratch space; the branch is your save space; the commit is the decision. Keep those three layers distinct and the rest of this chapter follows naturally.

---

### 2.2 Branching: Short-Lived Features, a Protected Default, No Long-Lived Personal Mains

Branches are the cheapest object Git has, which is why teams misuse them the way people misuse free cloud storage: hoard it. The branching model we recommend is deliberately boring:

#### The default branch is sacred

`main` (or `master`) is **protected**: nobody pushes to it directly, no one force-pushes to it, and every change arrives via a pull/merge request that must pass CI and at least one review. This is not bureaucracy; it's the load-bearing wall of everything in this chapter. A protected default branch means:

- `main` is always deployable, because every commit on it has been reviewed and tested.
- Release, hotfix, and "what's in production" questions have a single unambiguous answer.
- The cost of a bad commit is contained: you `git revert` it, and you're done.

#### Feature branches are short-lived by design

A feature branch is a *transaction*, not a *residence*. The target: **hours to a couple of days**, measured in commits, not calendar time. The mechanics:

1. Cut the branch from a fresh `main`: `git switch -c feat/billing-retry main`.
2. Work on it, committing reviewable decisions per §2.1.
3. Rebase onto `main` regularly (see §2.3) so merges stay trivial.
4. Open the merge request when the *decision* is ready — not when the feature is "done," which is a different and larger unit.

The discipline that keeps branches short is **decomposition**. If a feature needs three weeks, that's a signal to split it into a sequence of small, independently reviewable PRs: the schema migration and its backfill, the API endpoint behind a feature flag, the client wiring, the flag flip, the cleanup. Each is a branch, a review, a merge. The feature is the *sum*; the branches are the *terms*.

#### No long-lived personal mains

The anti-pattern to name explicitly: the `alice` branch. Everyone has one — a personal trunk where they park half-finished work, experiments, and "I'll sort this out later" diffs. It accumulates for months, drifts further and further from `main`, and then one day the team needs a change that only exists there, and the merge is a three-hour archaeology expedition.

Long-lived personal branches fail for a specific, predictable reason: **the cost of merging grows with the distance from `main`, and distance grows with time.** A branch that's two weeks behind `main` in a fast-moving codebase may touch every file you're about to merge. At that point the "convenient" branch has become the most expensive object in the repository.

The replacement: if you have work you can't finish today, either (a) get it to a *committable* state — even if small — and open a draft PR, or (b) shelve it honestly: `git stash`, a worktree (§2.5), or a clearly-named local branch with a deadline. Draft PRs are underrated: they freeze the diff, start the review conversation early, and make "what's in flight" visible to the whole team.

A compact policy statement for the team:

> **Branch policy.** `main` is protected and always deployable. Feature branches live days, not weeks; they are rebased on `main` at least daily, opened as draft PRs the moment they contain a reviewable decision, and deleted after merge. No branch outlives the feature it implements.

---

### 2.3 History Hygiene: Rebase vs. Merge, When to Squash, When *Not* to Rewrite

History is a *contract*, not a diary. The question is never "which history looks nicer" but "who is relying on this history, and what does rewriting it cost them?"

#### The one rule that decides everything

**Never rewrite commits that other people have pulled.** Period. This single rule resolves ninety percent of rebase-vs-merge arguments:

- Your **unpushed, local-only** commits: yours to reshape. Rebase, squash, reword, reorder — freely.
- **Pushed, shared** commits: frozen. Fix them with *new* commits (or `git revert`), never by rewriting the shared past.

When you rewrite a shared branch, you don't just change history — you invalidate every teammate's local copy of it. Their next `git pull` is a conflict explosion, and the fix (`git pull --rebase`, or worse, a force-push from someone who didn't ask) is how trust in a codebase gets broken.

#### Rebase vs. merge, concretely

**Merge** (`git merge`) creates a junction commit with two parents. It preserves the exact shape of when things happened, including "these two lines of work were happening in parallel." Merges are the right tool for *joining* histories: merging a feature into `main`, merging a release branch forward, merging a fork.

**Rebase** (`git rebase`) replays your commits on top of a new base, producing a linear sequence. It's the right tool for *resyncing* your in-flight work:

```bash
# you're on feat/billing-retry, main moved on
git fetch origin
git rebase origin/main
# resolve conflicts commit-by-commit, then:
git push --force-with-lease   # safe because this branch is yours
```

Note `--force-with-lease`, which refuses to overwrite if someone else pushed to the branch in the meantime. It's the force-push that's safe to put in a policy document.

The daily rhythm that keeps history clean:

1. Work on your feature branch; commit reviewable decisions.
2. `git rebase origin/main` **frequently** — daily, or whenever you rebase, whichever comes first. Small rebases are painless; large ones are funerals.
3. Open the PR. If `main` moves again before review, rebase the PR branch (on GitHub/GitLab, enable "auto-merge on upstream update" or rebase manually).
4. Merge with your platform's **squash** option (below).

This is the "rebase locally, squash at the boundary" model, and it gives you the best of both: a linear, readable `main` history, with the messy iterative work preserved in the PR's commit list for anyone who wants it.

#### When to squash

Squash-merge a PR when the PR's *value unit* is the feature or fix as a whole. This is the common case: a 9-commit PR that fixes the billing retry logic should land on `main` as **one** commit titled "Retry failed billing charges with exponential backoff." The reasons:

- `main` history reads as a changelog of *decisions*, not of your debugging session.
- `git revert` of the PR is a single atomic operation.
- `git bisect` steps over decisions, not over "fix typo in variable name" noise.
- `git blame` points at the decision and its PR, not at the third iteration of a helper function.

Squash also lets you **reword the final commit** to describe the decision *as reviewed and as merged* — which is often better than any individual commit message in the PR, since the review conversation usually improves the description.

#### When *not* to squash

Squash is a tool, not a religion. Don't squash when:

- **The PR contains genuinely independent decisions.** A PR that "adds the new parser and also fixes the log rotation bug" is two PRs. If it's already one PR, it should have been two — and if you squashed it, you've now welded two unrelated changes into one commit, which is *worse* for bisect and revert than the pre-squash state. The fix is process, not history surgery.
- **A commit must survive as its own unit for operational reasons.** A database schema migration that must be applied (and rolled back) independently of the code that uses it. A security fix that must be identifiable and revertable on its own. These deserve their own commits on `main` — which is another argument for splitting the PR.
- **You're rewriting *shared* history to squash.** Never. If a teammate has already built on top of those commits, squashing them away rewrites everyone's world. Let them stand.

#### The emergency case: rewriting `main`

There are rare moments when a team *does* rewrite a protected branch — a secret committed to `main` and pushed to a public remote, a catastrophic merge that must vanish. The procedure: act fast (rotate the secret — rewriting history does **not** un-leak it; see §2.6), rewrite with `git rebase -i` or `git filter-repo`, force-push with coordination, and make every teammate re-clone. It works, but notice what it takes: a coordinated outage of the repo. That cost is exactly why the prevention in §2.6 exists.

---

### 2.4 Commit Messages That Survive Code Review and `git blame`

A commit message is a piece of software: it has users (future you, the reviewer, the person running `git blame` at 2 a.m. during an incident), and it fails silently. The format is settled and cheap to adopt:

```
<imperative subject, ≤ ~72 chars, no trailing period>

<body: why, not what. Wrap at ~72 chars. Reference the
issue, the decision context, and any non-obvious tradeoff.>

<footer: Fixes #482 · Reviewed-by: ... (platform-specific)>
```

The subject answers **"what changed"** in the imperative mood ("Add retry to billing client," not "Added retry" or "adding retry"), because `git log` reads as a changelog and the imperative is what makes it read as a list of decisions rather than a diary. The body answers **"why"**, which is the part no diff can show you:

- The diff shows *what*. It can never show *why you chose X over Y*.
- The diff shows the final state. It can never show the three approaches you rejected.
- The diff is a snapshot. It can never show the incident ticket, the customer complaint, or the constraint that will be relaxed next quarter.

The body is where you write those things down, briefly. A good body is two to six lines. It is not a novel; it is the difference between a `git blame` hit that answers the question and one that sends the reader into the PR archaeology.

Examples, for calibration:

```
Fix race between session cleanup and token refresh

The cleanup timer could fire between the refresh request being
sent and its response arriving, logging out users mid-refresh.
Guard the cleanup with the in-flight refresh set.

Fixes #1147
```

```
Split auth middleware into verify and authorize stages

verify() checks the token is well-formed and unexpired;
authorize() checks the caller may perform the action. The old
single stage re-parsed the token on every check, which made the
rate-limit keying in §7.2 impossible to express.
```

```
Update PostgreSQL driver 15.2 → 15.4

Point releases only; no API changes. 15.3 contains a fix for a
memory leak in the extended query protocol (GHSA-xxxx).
```

Contrast with the messages that don't survive: `fix`, `update`, `wip`, `asdf`, `changes`. Each one is a promise the history will later break. The rule that makes this stick is the same rule as §2.1: **if the commit is a reviewable decision, the message is its reviewable summary.** You wouldn't submit a PR with the title "changes." Don't submit a commit either.

One more habit: **reference the issue or ticket number** in the body. It's the thread that ties the commit, the PR, the ticket, and (later) the incident review into one searchable record. When the billing bug recurs in six months, `git log --grep=1147` is the fastest path back to everything you learned the first time.

---

### 2.5 The Tools You Actually Need: Bisect, Reflog, Worktrees, Sparse Checkout

Git ships with roughly two hundred commands. You will use maybe twenty. Four of them are worth knowing deeply because they convert the scary moments — "it's broken," "I lost my work," "I need two checkouts," "this repo is huge" — into routine moments.

#### `git bisect`: finding the commit that broke it

When a regression appears, you don't guess which change caused it; you let Git binary-search it:

```bash
git bisect start
git bisect bad                 # current HEAD is broken
git bisect good v2.4.1          # last known-good tag
# Git checks out a midpoint. Run your test. Then:
git bisect bad                 # or: git bisect good
# ...repeat; each answer halves the search space
# when done:
git bisect reset
```

For a regression spanning 1,000 commits, that's about **10 test runs** (log₂ 1000 ≈ 10). The pro move is to skip the manual loop entirely:

```bash
git bisect run ./scripts/smoke-test.sh
```

`bisect run` executes your script at each midpoint and reads its exit code (0 = good, anything else = bad), driving the whole search unattended. This is why the "every commit passes tests" rule from §2.1 pays off: bisect is only as good as the invariant that *each commit is a valid state*.

#### `git reflog`: the undo button for everything

Every time Git moves a reference — commit, reset, rebase, checkout — it records it in a **local, private** log. The reflog is why "I lost my work" is almost never true:

```bash
git reflog                          # recent HEAD movements, with hashes
git reflog show main                # history of the main branch itself
git switch -c rescue abc123         # recover a "lost" commit
git reset --hard abc123             # undo a bad reset
```

Typical rescues:

- **`git reset --hard` went too far:** `git reflog`, find the commit from before, `git reset --hard <that>`.
- **A rebase went sideways:** `git reflog` shows the pre-rebase HEAD; restore it, try again more carefully.
- **"Deleted" a branch:** its tip commit is still in the reflog; `git switch -c branch-name <hash>` brings it back.

Two caveats. First, the reflog is **local**: it doesn't travel with pushes, so it can't recover a commit that was never pushed and was garbage-collected (default expiry: 90 days for reachable, 30 for unreachable). Second, it's *per-machine* — it's your undo history, not a team backup. (Team-level safety comes from pushing early and often, which is also why short-lived branches in §2.2 are pushed as draft PRs.)

#### `git worktree`: multiple checkouts, one repository

A worktree attaches a **second (or third) working directory** to the same repository, on a different branch:

```bash
git worktree add ../checkout-hotfix hotfix/session-timeout
git worktree list
git worktree remove ../checkout-hotfix
```

This solves the daily pain of "I'm mid-feature but I have to look at / fix / demo something else." Previously that meant stashing, switching, un-stashing, and praying. With worktrees you get a clean second directory — full history, full tooling — while your main checkout stays exactly where it was. Use cases: a quick hotfix while a feature branch is mid-rebase; running the old version and the new version side by side; keeping a checkout dedicated to a long-running test run. The constraint to know: a branch can only be checked out in one worktree at a time (Git enforces this to prevent exactly the footgun it's guarding against).

#### `git sparse-checkout`: a repo, minus most of the files

In a monorepo, you often need one service and not the other forty. Sparse checkout materializes **only the paths you asked for**:

```bash
git sparse-checkout set apps/billing libs/shared
```

Your working tree now contains just those paths (plus your cone-mode defaults), while the *repository* is fully cloned — history intact, `git log` on other paths still works, merges still work. You're smaller on disk and faster on checkout, without the pain of a shallow or partial clone. This is the standard answer to "the repo takes 11 minutes to clone" for teams that can't split the repo (and usually can't, for good reasons).

#### The working set

Keep these four in your muscle memory and the rest of Git recedes into a reference section: **bisect** finds the bad commit, **reflog** recovers the lost one, **worktree** gives you a second desk, **sparse-checkout** shrinks the office. Everything else is a variation on these four ideas.

---

### 2.6 What Belongs in the Repo: `.gitignore`, LFS, and the Never List

The repository is a **public artifact** in the sense that matters: assume everything in it — including everything it has *ever* contained, on any branch, in any reflog — will eventually be seen by more people than you intended. That assumption drives the whole policy.

#### `.gitignore`: ignore what's derived, generated, or local

The principle: **the repo stores what you *write*; it never stores what you can *build*.** Anything reproducible from checked-in sources plus a documented toolchain is a build artifact and belongs in `.gitignore`:

```gitignore
# Dependencies — installed from the lockfile, never stored
node_modules/
vendor/            # or: commit vendor/ deliberately, as a policy, and say so

# Build output
dist/
build/
*.o
*.pyc
__pycache__/

# Local environment and secrets
.env
.env.*
!.env.example
*.pem
*.key
secrets/

# Databases and data
*.sqlite
*.dump
data/

# Editor/OS noise
.DS_Store
.idea/
```

Details that matter:

- **Commit the lockfile** (`package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `Gemfile.lock`, `go.sum`). The lockfile is the *reproducible build* — it's exactly the kind of derived-but-authoritative artifact you want versioned. Ignoring it is how "works on my machine" gets born.
- **Commit an empty `.env.example`** that documents every variable the app needs, with no values. It's onboarding documentation and a checklist in one.
- `.gitignore` only prevents *new* tracking. **If a file is already tracked, adding it to `.gitignore` does nothing.** Untracking requires `git rm --cached <file>` — and at that point, read the next section, because the file's full history is still in the repo.

#### LFS policy: the repo is for text, roughly

Git is a text-diffing, content-addressed system. Large binaries defeat both properties: a 200 MB video re-committed with a 1-pixel change stores *another* 200 MB, forever, in every clone. **Git LFS** replaces the blob with a small pointer file; the binary lives in LFS storage and is fetched on demand.

A sane LFS policy is a *list*, not a vibe:

- **In LFS:** design assets (PSD, Figma exports), model weights, recorded test fixtures (video/audio), large datasets used by tests, anything above ~1–2 MB as a standing rule.
- **Never in LFS:** source code, config, lockfiles, anything a build reads. LFS pointers can be missing on fresh clones if the LFS server is unreachable — your build must not depend on LFS.
- **Decide at commit time, not after.** Converting a file to LFS after it's been committed repeatedly means rewriting history (see §2.3: only do that when the branch is shared by no one) or accepting the old blobs stay. The rule "anything over 1 MB goes to LFS, enforced by a pre-push hook or CI check" is the only version of this policy that doesn't rely on memory.

#### The never list: what must not enter the repo at all

Some things are not "in LFS vs. not" questions. They must never be committed, on any branch, ever:

| Never commit | Why | Where it goes instead |
|---|---|---|
| **Secrets** (API keys, tokens, passwords, signing keys, connection strings with credentials) | Once committed, they're in history *forever* on every clone, fork, and mirror; reflog and force-push don't make them un-leaked | Env vars / secret manager (Vault, AWS SM, 1Password CLI); `.env.example` documents names only |
| **`node_modules/`, `vendor/`** (unless a *deliberate*, documented vendoring policy) | Derived, huge, and a supply-chain attack surface you can't audit by eye | Installed from the lockfile at build time |
| **Database dumps, `.sql` exports, CSVs of real data** | PII and production state; also binary-ish and volatile | Backups live in storage, not Git; test fixtures are *synthetic* |
| **Build artifacts, binaries** (except deliberately-versioned release binaries) | Reproducible from source; bloat every clone | CI builds them; artifacts go to a registry |
| **Machine-local config** (IDE state, absolute paths, local certs) | Not reproducible, not shared, occasionally sensitive | `.gitignore`, or a local untracked config file |

The incident pattern to internalize: a developer commits `.env` with a production API key. They notice in an hour, delete the file, commit the deletion. **The secret is not removed** — it sits in the history of every clone, and if the remote is public, it's already indexed by secret-scanning bots within minutes. The actual remediation, in order:

1. **Rotate the secret first.** History rewriting is optional; rotation is not.
2. Scrub history (`git filter-repo`), force-push with coordination, ask everyone to re-clone — *if* the exposure window makes it worth it.
3. Add detection so it can't happen again: a **pre-commit secret scanner** (e.g., gitleaks, trufflehog) on every developer machine, and **server-side scanning** on every push and PR — the server check is the one that actually works, because developers disable pre-commit hooks.

This is where Git hygiene stops being a convenience topic and becomes a security control: **the repository is part of your attack surface**, and its history is part of your secret-management boundary.

---

### 2.7 Code Review as a Security and Design Control, Not a Style Nitpick

The common failure mode of code review is that it becomes a font-size tribunal: reviewers burn their attention on naming and blank lines, and the things that actually ship broken — the race condition, the unvalidated input, the missing migration rollback — sail through because nobody was looking at them. The fix is to be explicit about **what review is for**, and to allocate attention accordingly.

Review is a control with three jobs, in roughly descending order of importance:

**1. Security: the diff is an attack-surface change, and the reviewer is the second pair of eyes on it.**
The reviewer asks the questions the author, who is thinking about the feature, isn't thinking about:

- *What new input crosses a trust boundary here?* User input, request headers, file names, webhook payloads, data from the database — anything not produced by your own code. Is it validated, parameterized, and bounded? (This is the daily, human layer beneath OWASP A05: Injection — see Chapter 4.)
- *What can this do with elevated privileges?* New admin routes, new file access, new shell/exec calls, new deserialization.
- *What did we add to the supply chain?* New dependencies, new build plugins, new CI actions. Every one is code you don't write, executed in your build (more in §2.8).
- *What does this log?* Secrets in logs is the most common self-inflicted secret leak; review is where it's caught.
- *What's the failure mode?* What happens when the external call times out, the disk fills, the payload is 10× the expected size?

**2. Design: the review is the last cheap moment to change the shape.**
A design flaw found in review costs an afternoon; found in production it costs a quarter. The design questions:

- *Is this the right boundary?* Is business logic in the controller, is the controller in the database trigger?
- *Does this scale with the next requirement, or does it fork?*
- *Is this the simplest thing that could work?* Reviewers have an obligation to delete code, not just to approve it.
- *Is the interface honest?* Does the function do what its name says, and nothing else?

**3. Correctness and maintainability: the ordinary stuff, but scoped.**
Edge cases, error handling, test coverage of the *interesting* paths, naming — yes, all of it, but as a *bounded* pass, not the main event.

The mechanics that make this work are the ones this chapter has been building toward, and they're not incidental:

- **Small PRs are a review-quality control, not a formatting preference.** A 400-line diff gets a 400-line diff's attention — which is to say, none of it. The decomposition discipline of §2.2 exists so that every PR is small enough to review *properly*.
- **Reviewable commits (§2.1) make the diff readable.** A PR whose commits each state a decision is a PR a reviewer can follow commit by commit.
- **The squash commit message (§2.4) is the review's output artifact** — the record of the decision, the tradeoffs, and the issue, for the next person.
- **The protected default branch (§2.2) is what makes "must pass review" mean something.** A review requirement with a force-push override is a review *suggestion*.

A practical review rubric, in review order: (1) Does it do what the PR says, and nothing else? (2) Security pass: inputs, privileges, dependencies, logs. (3) Design pass: boundaries, simplicity, next requirement. (4) Correctness pass: errors, edges, tests. (5) *Then* style — and style issues get a "nit:" prefix, which is a social contract meaning "this does not block the merge."

---

### 2.8 Signing, Protected Branches, and Supply-Chain Basics

The controls so far assume a threat model of *accident*: the tired developer, the shared machine, the secret in history. This section adds the controls for *adversary*: the compromised laptop, the malicious dependency, the attacker who has valid credentials. The through-line is that **your software supply chain starts at the repository** — the code you commit, the tools that build it, and the dependencies it pulls are all part of the product, and all of it needs integrity guarantees. (This is the foundation for OWASP Top 10:2025 **A03 — Software Supply Chain Failures**, which we treat in depth in the security chapter; the point here is that the Git layer is where A03 *starts*.)

#### Commit and tag signing: "who" becomes provable

Git's default identity is a name and email in a config file — assertable, not verifiable. **Signing** attaches a cryptographic signature to each commit (and tags), so anyone can verify both *that* the commit is unmodified since signing and *who* signed it.

The modern setup:

```bash
# Generate an Ed25519 signing key (GPG works too; Ed25519 is simpler)
ssh-keygen -t ed25519 -f ~/.ssh/signing -C "you@example.com"

git config --global user.signingkey ~/.ssh/signing
git config --global commit.gpgsign true      # sign all commits
git config --global tag.gpgsign true         # sign all tags
```

Upload the public key to your hosting platform (GitHub/GitLab verify and display a "Verified" badge), and verify locally with `git verify-commit <sha>` or `git log --show-signature`.

What signing buys you:

- **Tamper detection on history.** A forged or modified commit fails verification. Combined with signed tags, your release tags are *authentic artifacts*: `v2.4.1` signed by the release manager is a different object than `v2.4.1` pushed by anyone else.
- **Attribution you can audit.** In an incident, "which human's key signed the bad commit" is a question with a cryptographic answer.
- **A policy hook.** Hosts can *require* signed commits on protected branches, and CI can reject unsigned PRs. Signing then becomes a team invariant, not a personal habit.

Sign **tags especially** — an unsigned tag is just a movable label; a signed one is a statement.

#### Protected branches, done properly

§2.2 established the policy; here's the control set that enforces it, which every major host provides:

- **No direct pushes** to `main` (and to release branches): all changes via PR.
- **Required status checks** before merge: CI build, tests, lint, and — per §2.6 — the **secret scan** and **dependency audit** must be green.
- **Required review:** at least one approval; for security-sensitive paths (auth, payments, CI config, deployment), two, or a specific team.
- **No force-push, no deletion** on protected branches — this is what makes the history a contract (§2.3).
- **Branch protection for the protectors:** the CI config, the branch-protection config, and the deployment scripts should live behind the same (or stricter) review bar as the application code. An attacker who edits `.github/workflows/deploy.yml` owns your deployment.

#### Supply-chain basics: the repo's perimeter is wider than the repo

The software you ship is a graph: your code, its dependencies, the build tools, the CI runners, the registry, the container base image. Each edge is a trust decision. The Git-layer controls:

1. **Lock everything.** Lockfiles for application dependencies; pinned versions for build tools and CI actions (pin to a full commit SHA, not a floating `v3` tag — a tag you don't control can be moved). The lockfile plus dependency auditing (Snyk/OSV/Dependabot-style checks in CI, required per the branch protection above) is the baseline for "I know what I'm shipping."
2. **Treat CI as production.** A workflow file is code that runs with your credentials, on your infrastructure, with write access to your artifacts. Review CI changes with more care than application code, keep the set of third-party actions small, and prefer first-party or well-audited ones.
3. **Reproducible builds.** Same commit + same lockfile + same toolchain versions → same artifact, verifiable. This is what makes "we shipped exactly what we reviewed" a checkable claim rather than a hope, and it's the end-state that signed artifacts and signed commits are working toward.
4. **Assume compromise, contain blast radius.** Least-privilege CI tokens, ephemeral runners, and the rule that *rotating a leaked credential is faster than scrubbing history* (the lesson of §2.6, generalized).

The one-paragraph summary to carry into the security chapter: **signing proves authorship and integrity of what you commit; branch protection and required checks prove what is allowed to reach `main`; lockfiles, pinning, and dependency audits prove what your code pulls in. Together they make the repository a controlled boundary of the supply chain — and OWASP A03 is what happens when any part of that boundary is treated as decoration.**

---

### Chapter Summary

- A commit is a **reviewable decision**: atomic, self-consistent, described in a message that survives `git blame`.
- **`main` is protected and always deployable**; feature branches are short-lived transactions, rebased often, merged as squash; long-lived personal branches are a debt instrument.
- **Rewrite only your own unpushed commits.** Rebase locally, squash at the merge boundary, and never rewrite shared history — except in a coordinated emergency, after rotating whatever leaked.
- **Bisect** finds the bad commit, **reflog** recovers the lost one, **worktrees** give you parallel desks, **sparse checkout** shrinks the repo to the part you need.
- The repo stores what you *write*, never what you can *build*; **secrets, dumps, and `node_modules` never enter**, and a committed secret is rotated, not just deleted.
- **Review is a security and design control** with a defined rubric; small PRs and reviewable commits are what make the review possible.
- **Signing, branch protection, and dependency pinning** turn the repository into an integrity boundary — the first line of defense against the supply-chain failures covered under OWASP A03 in the security chapter.

*Exercises: (1) Take a week of your real commit history and rewrite it, in a scratch clone, as reviewable decisions with proper messages. (2) Deliberately break a build across ten commits, then find the culprit with `git bisect run` without looking at the diffs. (3) Commit a fake secret, let it live for a day, then run through the full rotation-and-scrub procedure and time it. (4) Set up commit signing and a signed tag, and verify both from a clean clone.*

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
