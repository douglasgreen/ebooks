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

Vue gives you a component model, a reactivity system, and a compiler. But none of that runs in a vacuum. Every component you write is ultimately compiled into DOM mutations, every style you write is resolved against the cascade, and every promise you return is settled by an event loop you did not build. This chapter is about the substrate underneath the framework — the parts of the web platform that are *real* regardless of which UI library you reach for, and the places where confusing the framework for the platform will quietly cost you.

We'll cover six things: what HTML actually commits you to, how CSS really resolves, the modern JavaScript you can rely on, the browser APIs an SPA leans on, an honest take on progressive enhancement, and how your browser support policy becomes a hard constraint on your build.

---

### HTML as a contract with the browser, not a Vue template leftover

A common mental model to correct early: **the `<template>` block in a `.vue` file is not what the browser sees.** It is source code. The compiler turns it into a render function, and that function produces DOM nodes at runtime. The browser never parses your SFC template. It parses *two* things: the `index.html` you ship, and the DOM your JavaScript produces.

That distinction matters because it changes what HTML is *for*.

**`index.html` is a contract.** When you write it, you are making a promise to the browser — and through the browser, to the user, to search engines, and to assistive technology — about what the document is and how it should behave *before a single line of your JavaScript runs*. The browser honors that contract: it parses, lays out, paints, and exposes the accessibility tree for standard HTML without asking your permission or waiting on your bundle.

Treat that contract seriously:

- **Use semantic elements for structure, not just presentation.** `<header>`, `<nav>`, `<main>`, `<article>`, `<section>`, `<footer>` are not decorative. They build the landmark structure a screen reader uses to navigate. A `<div class="nav">` gives a user of VoiceOver nothing; a `<nav>` gives them a jump point.
- **Prefer native controls to hand-rolled ones.** `<button>`, `<input type="date">`, `<dialog>`, `<details>`, `<select>`, `<label>`. Each one comes with keyboard handling, focus management, and screen reader semantics for free. Reimplementing a dialog in `<div>`s is a tax you pay forever.
- **Get the `<head>` right.** Character encoding, viewport, and a meaningful `<title>` and `<meta name="description">` are the baseline of a document that behaves and is indexable.

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Orders — Acme Dashboard</title>
    <meta name="description" content="Track and manage your orders." />
  </head>
  <body>
    <div id="app"><!-- Vue mounts here --></div>
    <script type="module" src="/src/main.js"></script>
  </body>
</html>
```

Notice the `<div id="app">`. In a client-side-rendered app, the HTML you ship is a *shell*: a contract that says "a document will appear here, and here is the script that builds it." That is a weaker contract than a fully server-rendered page, and it has real consequences for first paint, for users with JavaScript disabled, and for crawlers. We'll return to that trade-off in the progressive enhancement section.

The second half of the contract is the **DOM your components produce**. Vue's job is to keep that DOM in sync with your state, but the browser's job — the part you cannot override — is to interpret whatever ends up there. A `<p>` inside a `<p>` is still invalid. A `<button>` containing a `<button>` is still a problem. A `role` that contradicts the element is still a lie to assistive tech. The framework does not absolve you of writing correct HTML; it just makes it easier to get wrong by hiding the markup behind abstractions.

**The habit to build:** when you reach for a `div` with a class, ask whether a semantic element or a native control would do the job. The browser is already paying the implementation cost. Use it.

---

### CSS: cascade, specificity, custom properties, container queries, logical properties

CSS has a reputation for being unpredictable, and most of that reputation comes from one place: people writing rules without a shared model of *how a declaration wins*. Once the model is explicit, most "mystery CSS" disappears.

#### The cascade, in order

When several declarations target the same property on the same element, the browser resolves them in a fixed order. Roughly, from weakest to strongest:

1. **Origin and importance.** Author styles lose to user-agent (browser default) styles only when the author uses `!important` in reverse; in practice, author `!important` beats normal author and user styles, and normal author styles beat user-agent styles.
2. **Specificity** (next).
3. **Source order.** When specificity ties, the later rule in the stylesheet wins.

Modern CSS adds a fourth lever that sits *above* specificity: **cascade layers** (`@layer`). Rules in a later layer beat rules in an earlier layer regardless of specificity, which is how you stop fighting specificity altogether by declaring an explicit order — for example, a `tokens` layer, then `base`, then `components`, then `utilities`. We'll use layers to structure our stylesheets so that "the last rule wins" becomes a deliberate decision instead of an accident.

#### Specificity

Specificity is a four-part score, compared left to right:

| Part | Example | Weight |
| --- | --- | --- |
| Inline style | `style="color: red"` | highest (among normal declarations) |
| ID selector | `#nav` | 1-0-0-0 |
| Class, attribute, pseudo-class | `.card`, `[type="text"]`, `:hover` | 0-1-0-0 |
| Type, pseudo-element | `div`, `::before` | 0-0-1-0 |
| Universal | `*` | 0-0-0-0 |

```css
/* All three target the same element. The third wins on specificity. */
.card              { padding: 1rem; }   /* 0-1-0-0 */
#profile .card     { padding: 2rem; }   /* 1-1-0-0  ← wins */
.card.highlighted  { padding: 1.5rem; } /* 0-2-0-0 */
```

The failure mode is **specificity escalation**: one team member adds an ID to win, the next adds `!important`, the next adds an inline style, and the stylesheet becomes a record of who shouted loudest. The modern counter-strategies:

- **`@layer`** to make order explicit instead of implicit.
- **`:is()` and `:where()`** to compose selectors without adding specificity. `:where()` contributes *zero* specificity, which is exactly what you want for a utility that must not win by accident.

```css
/* :where() adds no specificity, so it can't escalate the game. */
:where(.card, .tile, .panel) { border-radius: 8px; }
```

#### Custom properties

Custom properties (CSS variables) are the platform's answer to design tokens. They inherit, they resolve at computed-value time, and they let you parameterize a design system without a build step.

```css
:root {
  --color-surface: #ffffff;
  --color-text: #1a1a1a;
  --space-2: 0.5rem;
  --space-4: 1rem;
  --radius-md: 8px;
}

.card {
  background: var(--color-surface);
  color: var(--color-text);
  padding: var(--space-4);
  border-radius: var(--radius-md);
}

@media (prefers-color-scheme: dark) {
  :root {
    --color-surface: #121212;
    --color-text: #f5f5f5;
  }
}
```

Two capabilities worth knowing because they remove whole categories of JavaScript:

- **`color-mix()`** blends colors in CSS: `background: color-mix(in srgb, var(--color-primary) 10%, transparent);`
- **`light-dark()`** picks a value by color scheme without a media query: `color: light-dark(#1a1a1a, #f5f5f5);`

The discipline: define tokens once at the root (or in a tokens layer) and reference them everywhere. If a raw hex value appears in a component rule, that is a code smell — it is a token that has not been promoted yet.

#### Container queries

For a decade, responsive design meant "how wide is the *viewport*?" Container queries flip the question to "how wide is *this component's container*?" — which is what you actually care about when a card is rendered in a sidebar at one size and full-width at another.

```css
/* Declare a container. */
.card-container {
  container-type: inline-size;
  container-name: card;
}

/* Query the container, not the viewport. */
@container card (min-width: 400px) {
  .card {
    display: flex;
    gap: 1rem;
  }
}
```

`container-type: inline-size` makes the element's inline size queryable without requiring you to fix its height. This is the difference between a component that adapts to its context and one that adapts to the screen — and it is the CSS-native version of the responsive component you would otherwise build with a ResizeObserver and a prop.

#### Logical properties

Physical properties (`margin-left`, `padding-top`) are tied to the screen. Logical properties are tied to the *writing flow*, which is what makes layouts work across text directions and writing modes without a separate RTL stylesheet.

| Physical | Logical |
| --- | --- |
| `margin-left` / `margin-right` | `margin-inline-start` / `margin-inline-end` (or `margin-inline`) |
| `margin-top` / `margin-bottom` | `margin-block-start` / `margin-block-end` (or `margin-block`) |
| `width` / `height` | `inline-size` / `block-size` |
| `border-top-left-radius` | `border-start-start-radius` |

```css
.card {
  margin-inline: 1rem;        /* left+right in LTR; right+left in RTL */
  padding-block: 0.5rem;      /* top+bottom */
  border-start-start-radius: 8px;
}
```

The payoff: write the layout once, and it is correct in an Arabic or Hebrew interface with zero changes. If your product has any chance of internationalization, default to logical properties and treat physical ones as the exception.

#### What Stylelint will enforce later

We will add Stylelint to the project later, and its configuration is where the conventions from this section stop being advice and start being rules the toolchain will reject. Expect it to enforce, at minimum:

- **No ID selectors** in component styles — they are a specificity escalation and they break reuse.
- **No `!important`** — if you need it, the cascade is wrong and the layering should be fixed.
- **A property and declaration order** so diffs stay stable and reviewable.
- **No unknown or misspelled properties**, and no vendor prefixes you do not need (modern Autoprefixer handles those).
- **A selector-class pattern** (e.g. BEM or a naming convention) so the design system stays consistent.
- **Color and unit rules** — no raw hex values outside the tokens file, consistent spacing scale.

The point is that the "right way" from this section is not a matter of taste we argue about in review. It is encoded in the linter, and the build fails if you break it.

---

### Modern ECMAScript: what to use, and what to leave to TypeScript

JavaScript is no longer the language of `var` and callback pyramids. The platform ships a modern language, and you should write to it — while being clear about which problems are the *language's* and which are the *type system's*.

#### Modules

ES modules are the standard unit of composition. Two properties matter more than the syntax:

- **Static analysis.** `import` and `export` are resolved before code runs, which is what lets a bundler do tree-shaking (drop exports nobody uses) and what makes circular dependencies at least detectable.
- **Hoisting and single evaluation.** Modules are evaluated once, in dependency order, and their bindings are hoisted.

```js
// cart.js
import { formatPrice } from './currency.js';

export function cartTotal(items) {
  return items.reduce((sum, it) => sum + it.price, 0);
}

export { formatPrice }; // re-export
```

In the browser, `type="module"` scripts are deferred by default and run in a strict mode. In Node, the line between ESM and CommonJS is still a live source of pain, but in a Vite project you are almost always on the ESM side, so write ESM and stop thinking about it.

#### Iterators and iterables

An **iterable** is anything with a `[Symbol.iterator]()` method; an **iterator** is the object that method returns, with a `next()` that yields values. `for...of` is the consumer. You already use this everywhere:

```js
const items = ['a', 'b', 'c'];
for (const item of items) { /* ... */ }

// Sets, Maps, and generator functions are iterables too.
const seen = new Set(['a', 'b', 'a']);
for (const v of seen) { /* 'a', 'b' */ }

function* ids() { yield 1; yield 2; yield 3; }
[...ids()]; // [1, 2, 3]
```

The practical reason to care: `Array.from()`, the spread operator, and destructuring all accept iterables, so you can normalize "anything that can be looped" into an array in one step.

#### `async` / `await`

Async/await is syntactic sugar over promises that makes asynchronous code read like synchronous code — and, crucially, makes **error handling** use the `try`/`catch` you already know instead of chained `.catch()` calls.

```js
async function fetchOrders(signal) {
  try {
    const res = await fetch('/api/orders', { signal });
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return await res.json();
  } catch (err) {
    if (err.name === 'AbortError') return; // expected, not an error
    throw err;
  }
}
```

The companion statics are worth internalizing because they map to real UI patterns:

- **`Promise.all`** — run concurrently, fail fast. Use when you need every result.
- **`Promise.allSettled`** — run concurrently, never reject; inspect each outcome. Use when partial failure is acceptable (e.g. loading a dashboard of independent widgets).
- **`Promise.any`** — resolve with the first success, reject only if all fail.
- **`Promise.race`** — first to settle wins; the basis of a timeout.

```js
const timeout = new Promise((_, reject) =>
  setTimeout(() => reject(new Error('timed out')), 5000));
await Promise.race([fetchOrders(signal), timeout]);
```

#### Temporal and the collection APIs you will actually use

**Temporal** is the long-awaited replacement for `Date`, and it is finally real: it reached TC39 Stage 4 and is part of ECMAScript 2026, shipping in Chrome 144 and Edge 144 (January 2026) and Firefox 139 (May 2025). Safari has it in Technology Preview but not yet in a stable release as of mid-2026. That last detail is the whole story of how to use it: **it is not yet Baseline "widely available,"** so a strict build target will not assume it, and you should either polyfill it or hold off until your support policy allows. When you do use it, the model is a set of explicit types instead of one ambiguous `Date`:

```js
const today = Temporal.Now.plainDateISO();        // '2026-08-18'
const next  = today.add({ days: 7 });             // a PlainDate, no DST surprises
const dt    = Temporal.Now.zonedDateTimeISO();    // an Instant + a time zone
```

The collection and object APIs that are safe to use today and that quietly remove a lot of boilerplate:

- **`Map` / `Set` / `WeakMap`** — keyed by reference or by value, not just by string. `WeakMap` is the right home for attaching data to objects without leaking memory.
- **`structuredClone()`** — a deep clone that understands `Map`, `Set`, `Date`, and cyclic references. Replace `JSON.parse(JSON.stringify(x))`.
- **Non-mutating array methods** — `toSorted()`, `toReversed()`, `toSpliced()`. These return a new array instead of mutating, which is a better default in reactive code where mutation is a footgun.
- **`at(-1)`** — index from the end.
- **`Object.hasOwn(obj, key)`** — the safe replacement for `hasOwnProperty.call`.
- **`Object.groupBy()` / `Map.groupBy()`** — group an array by a key in one call.
- **`String.prototype.replaceAll()`** — and the `findLast` / `findLastIndex` pair.

```js
const byStatus = Object.groupBy(orders, (o) => o.status);
const last = items.at(-1);
const copy = structuredClone(draft);
```

#### What to leave to TypeScript

The line to draw: **JavaScript is the runtime; TypeScript is the contract you check before it runs.** The language gives you the *mechanism* (modules, async, collections). TypeScript gives you the *assurance* (shapes, null-safety, exhaustiveness) without any runtime cost.

Concretely, leave to TypeScript:

- **Object and API shapes.** Do not document a response format in comments; type it.
- **Null and undefined handling.** `strictNullChecks` turns a whole class of "can this be undefined?" bugs into compile errors.
- **Exhaustiveness.** A `switch` over a union that the compiler proves covers every case is a correctness argument, not a hope.
- **Generics for reusable logic.** A typed `debounce<T extends (...args: any[]) => void>` is safer than an untyped one and still compiles to the same JavaScript.

The discipline: do not reach for JavaScript workarounds (manual runtime guards, defensive `typeof` checks, `as any` casts) for problems the type system already solves. And do not let TypeScript become a second runtime — it should erase cleanly to the modern JavaScript above, not to a pile of polyfilled ceremony.

---

### Web APIs that matter in an SPA

An SPA is not just UI state. It is a small application that talks to a server, manages navigation, remembers things, and interacts with the device. The platform provides the primitives; the framework provides the structure. Here are the primitives you will actually touch.

#### Fetch

`fetch` is the modern replacement for `XMLHttpRequest`. Two things trip people up:

1. **It does not reject on HTTP error.** A 404 or 500 resolves with a `Response` whose `ok` is `false`. You must check it.
2. **It is cancellable via `AbortController`**, which is how you stop a request when a component unmounts or the user navigates away.

```js
const controller = new AbortController();

async function fetchOrders() {
  const res = await fetch('/api/orders', {
    signal: controller.signal,
    headers: { 'Content-Type': 'application/json' },
  });
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json();

}

// Later, to cancel:
controller.abort();
```

Remember that `fetch` is subject to **CORS**: the *server* must send the right `Access-Control-Allow-*` headers, or the browser blocks the read. That is a backend configuration problem, not a frontend one, but it is the cause of most "fetch works in dev, fails in prod" reports.

#### History

The History API is what makes an SPA feel like a multi-page site: real URLs, a working back/forward button, and shareable deep links — without a full page reload.

```js
// Push a new entry.
history.pushState({ page: 2 }, '', '/orders?page=2');

// React to back/forward.
window.addEventListener('popstate', (event) => {
  // event.state holds the object you passed to pushState/replaceState
  renderPage(event.state?.page ?? 1);
});
```

This is exactly the mechanism your router (Vue Router) is built on. Knowing it means you understand *why* the router does what it does, and how to debug a broken back button.

#### Storage

There is a stack, and choosing the right layer matters:

- **`localStorage` / `sessionStorage`** — synchronous, string-keyed, a few megabytes, survives reloads (`local`) or the tab (`session`). Fine for a theme preference or a draft. The traps: it is **synchronous** (it blocks the main thread on every read/write), it only stores strings (you will be `JSON.stringify`-ing constantly), and it is **not** a database.
- **`IndexedDB`** — asynchronous, structured, large. This is where real client-side data lives. It has an awkward callback/`IDBRequest` API, so in practice you wrap it (the `idb` library is the common choice) rather than speak to it raw.
- **The Cache API** — caches `Request`/`Response` pairs; the foundation of service workers and offline.
- **Cookies** — sent to the server on every request; the right home for auth tokens in some architectures (with `HttpOnly`/`Secure`/`SameSite`), the wrong home for application state.

The rule of thumb: if it is a small preference, `localStorage`; if it is data the app genuinely needs to query, `IndexedDB`; if it must reach the server, a cookie or a header — not storage.

#### Intersection Observer

`Intersection Observer` tells you when an element enters or leaves the viewport, *without* a scroll listener. It replaces the old pattern of throttling `onscroll` and comparing `getBoundingClientRect()` on every frame — a pattern that janks.

```js
const io = new IntersectionObserver((entries) => {
  for (const entry of entries) {
    if (entry.isIntersecting) {
      entry.target.src = entry.target.dataset.full;
      io.unobserve(entry.target); // done with this one
    }
  }
}, { rootMargin: '200px' }); // start loading 200px before visible

document.querySelectorAll('img[data-full]').forEach((img) => io.observe(img));
```

This is the primitive behind lazy-loaded images, infinite scroll, and "element became visible" analytics. If you find yourself attaching a `scroll` listener to do any of those, reach for the observer instead.

#### Clipboard

The async Clipboard API writes and reads the user's clipboard. Two requirements: a **secure context** (HTTPS or localhost) and, for reading, a **user gesture** (it must be triggered by a click or keypress, not by a timer).

```js
// Usually inside a click handler:
await navigator.clipboard.writeText('Order #12345');

// Reading requires a gesture and permission:
const text = await navigator.clipboard.readText();
```

It is the correct way to build a "copy to clipboard" affordance — with a graceful fallback to `execCommand` only if you must support a context where it is unavailable.

#### Credential Management

The Credential Management API is a unified surface for authentication, and it is where **passkeys** live. It abstracts three credential types:

- **`PasswordCredential`** — username/password.
- **`FederatedCredential`** — "sign in with Google/Apple."
- **`PublicKeyCredential`** — WebAuthn: the platform or a security key proves possession of a private key. This is the passkey path, and it is the most important one for security.

```js
// Sign in: ask the platform for a credential.
const cred = await navigator.credentials.get({
  password: true,
  // mediation: 'silent' for a passkey
});

// Sign up: create a new public-key credential.
const newCred = await navigator.credentials.create({
  publicKey: { /* challenge, rp, user, pubKeyCredParams, ... */ },
});
```

Even if you start with passwords, know that this is the API your auth flow will move toward, because passkeys are the direction the platform — and user expectation — is heading.

---

### Progressive enhancement vs. "SPA-only" — when each is honest

There is a spectrum between two honest positions, and a dishonest middle where most projects accidentally live.

**Progressive enhancement** is the position that the *core* of the experience works with no JavaScript, and JavaScript layers on the interactivity. It is honest — and usually correct — when **content is the product**: documentation, marketing, a blog, a catalog, an article. The user's goal is to *read* or *find* something, and that goal is fully servable by HTML and CSS. Enhancement (instant navigation, filtering, optimistic updates) is a bonus, not the point.

**SPA-only** is the position that **interactivity *is* the product**: a dashboard, an editor, a design tool, a game, a real-time collaborator. The value is the shared, mutable state and the responsiveness of it. Stripping JavaScript does not degrade the experience — it *removes* the experience. Pretending otherwise is dishonest.

The dishonest middle is where the trouble is:

- A **content site shipped as a full SPA** — a wall of JavaScript to render paragraphs that HTML would have rendered for free. You have taken on bundle cost, first-paint delay, SEO fragility, and a no-JS dead end to deliver text.
- A **JS-dependent app that claims "progressive enhancement"** while its core features simply do not exist without the bundle. The label is doing work the implementation is not.

The honest move is to **name what you are shipping** and design to it:

- Decide, per area of the product, whether the content or the interactivity is the point.
- For content areas, ship real HTML first (server-rendered or statically generated) and enhance.
- For app areas, be explicit that JavaScript is required, and make the *degradation* graceful and visible — a clear "this feature needs JavaScript" state, not a blank `<div id="app">`.
- Be honest with the crawlers and the assistive tech, not just with the user. A page that is empty until a bundle resolves is a page that is empty to a lot of consumers.

Most real products are a **blend**: a content shell that is fully servable, with app-like islands (a cart, a dashboard, an editor) that are honestly SPA-only. The skill is drawing that boundary deliberately and labeling each side correctly, rather than defaulting everything to one mode.

---

### Browser support policy and how it constrains Vite targets

"Which browsers do we support?" is not a nice-to-have question. It is a **build-time constraint** that determines what syntax you may write, what the bundler will transpile, and which platform features you can assume.

#### The policy

Write the policy down, in machine-readable form. The two common encodings:

- **`browserslist`** — a query list in your `package.json` or a `.browserslistrc`, e.g. `> 0.5%, last 2 versions, not dead`.
- **Baseline** — the web-platform-dx classification of features as *limited*, *available*, or *widely available* across current browser releases. "Baseline widely available" is the practical definition of "safe to use without a polyfill today."

The policy should answer: *what is the oldest browser we will actively test and not abandon?* That single number is the anchor for everything below.

#### How Vite consumes it

Vite's `build.target` tells esbuild the minimum browser to emit code for. **In Vite 7 the default changed from `'modules'` to `'baseline-widely-available'`** — a special value that targets the minimum browser versions compatible with Baseline *Widely Available* as of a date fixed for each major release (for the current major, 2026-01-01). In other words, Vite now defaults to "whatever the web considers safe right now," and it re-anchors that date on every major.

```js
// vite.config.js
export default defineConfig({
  build: {
    // Vite 7 default; set explicitly to pin or widen your support window.
    target: 'baseline-widely-available',
    // or an explicit list: target: ['es2022', 'chrome110', 'safari16'],
  },
});
```

The relationship you must keep straight: **your support policy and your build target should agree.** If your policy says "we support Safari 16" but your target is `baseline-widely-available` and that baseline has moved past Safari 16, you have a silent gap — the build will emit code that your oldest supported browser cannot run, and nothing will tell you.

#### Transpilation is not polyfilling

This is the single most important distinction in the section:

- **Syntax is transpiled.** If you write an optional chain or a class field and your target is older, esbuild rewrites the *syntax* so the old parser accepts it. This is automatic and safe.
- **Runtime APIs are not polyfilled by Vite.** If you call `Array.prototype.toSorted()`, `structuredClone()`, or `Temporal.PlainDate` and the target browser does not have that method, transpiling the syntax does nothing — the method still does not exist at runtime. You get a `TypeError`, not a build error.

So the practical workflow is:

1. Set `build.target` to match your oldest supported browser.
2. For any **syntax**, trust the transpiler.
3. For any **runtime API**, check your target (caniuse / Baseline) and, if it is missing, add a polyfill deliberately — typically via `core-js` with an appropriate usage config, or a small hand-rolled shim.
4. **Test on the oldest browser in your policy.** The build passing is not the same as the oldest browser passing.

#### The loop

The policy is not static. As Baseline moves and your users' browsers update, you can *raise* the floor: drop a polyfill, stop transpiling a syntax you no longer need, and ship a smaller bundle. That is the whole point of a written policy — it turns "can we use this feature?" from a tribal-knowledge argument into a lookup: *is it Baseline widely available, and is it at or below our target?* If yes, use it. If no, polyfill it or wait.

---

### Summary

The framework is a layer you can change. The platform is the layer you are shipping. Keep these six things straight and the rest of the book becomes a set of choices on a stable foundation:

- **HTML is a contract** the browser honors before your JavaScript runs — write semantics and native controls, not just a mount point.
- **CSS resolves by a fixed order** — origin, layers, specificity, source order. Encode the "right way" in Stylelint so it is a rule, not a debate.
- **Modern ECMAScript is the runtime** — modules, iterators, `async`/await, and the collection APIs are your tools; leave shapes and null-safety to TypeScript.
- **The Web APIs are the app's organs** — Fetch, History, Storage, Intersection Observer, Clipboard, and Credential Management are what an SPA actually does.
- **Be honest about the mode** — progressive enhancement when content is the product, SPA-only when interactivity is, and a deliberate, labeled boundary between them.
- **The support policy is a build constraint** — set `build.target` to match it, remember that Vite transpiles syntax but not runtime APIs, and test on the oldest browser you promise to support.

## Chapter 4. Accessibility as a Feature Constraint

Most teams treat accessibility the way they treat performance in 2011: as a number to hit at the end of a project, if at all. This chapter argues the opposite. Accessibility is a **feature constraint**—a set of requirements that should shape how you model your components, manage state, and write tests from the first commit, in the same way that "must not break on mobile" or "must not leak secrets" constrains your design.

The payoff is that when you *do* design for assistive technology up front, most of the work disappears. You stop bolting `role` and `tabindex` onto divs and start writing markup that is correct by construction. The rest of this chapter is about how to do that in a modern SPA, with a focus on Vue.

A quick framing note on standards: this chapter targets **WCAG 2.2** (published October 2023), which is the current W3C Recommendation and the de facto legal and procurement baseline in many jurisdictions. WCAG 2.2 is backward-compatible with 2.1; it *adds* nine success criteria and does not remove any. We'll flag the new ones where they matter.

---

### 4.1 WCAG 2.2 orientation: POUR

WCAG is organized around four principles, remembered by the acronym **POUR**. These are not four checklists; they are four *questions* you ask of every piece of UI you ship.

- **Perceivable** — Can a user *get* the information? If it's visual, is there a non-visual equivalent? If it's auditory, a visual one? Text is perceivable to everyone: screen readers, magnification, and search all run on text. Color, shape, and position are not.
- **Operable** — Can a user *act* on the interface? This is where keyboard support, focus management, and "no interaction that requires a precise motor skill" live. If a feature needs a drag, a double-tap, or a timed click, it has an operability problem.
- **Understandable** — Can a user *predict* what will happen? Consistent navigation, clear labels, helpful error messages, and no sudden, unexplained page changes.
- **Robust** — Can *assistive technology* actually parse it? Robustness is about the contract between your markup and the accessibility tree. It's why "it looks fine in the browser" is not the same as "a screen reader can navigate it."

The four principles map to a practical workflow:

| Principle | The question you ask | Where it shows up in this chapter |
|---|---|---|
| Perceivable | "Is there a text equivalent?" | Semantic HTML, labels, contrast |
| Operable | "Can I do this with a keyboard?" | Keyboard model, focus, target size |
| Understandable | "Can I predict this?" | Forms, errors, consistent help |
| Robust | "Can a screen reader parse this?" | ARIA, robust markup, testing |

**WCAG 2.2 adds nine success criteria.** Most of them are about real friction points rather than abstract rules, and several are exactly the kind of thing that's easy to get wrong in a SPA:

- **2.4.11 Focus Not Obscured (Minimum)** — AA. When an element is focused, it must not be *completely* hidden behind a sticky header, banner, or modal. (A sticky cookie banner covering the focused menu item is the canonical failure.)
- **2.4.12 Focus Not Obscured (Enhanced)** — AAA. The focused element must be *fully* visible, not partially covered.
- **2.4.13 Focus Appearance** — AAA. The focus indicator needs a minimum area and a 3:1 contrast ratio against the unfocused state. This is the formal version of "don't delete the default outline and replace it with nothing."
- **2.5.7 Dragging Movements** — AA. Anything you can do by dragging, you must also be able to do with a single-pointer (click/tap) alternative. Sliders, kanban boards, and signature pads all need a non-drag path.
- **2.5.8 Target Size (Minimum)** — AA. Interactive targets should be at least 24×24 CSS pixels (with narrow exceptions for inline text and spacing). Tiny footer icons are the most common failure.
- **3.2.6 Consistent Help** — A. If a help mechanism (chat widget, contact link, "need help?") appears on multiple pages, it must appear in the same relative place on each.
- **3.3.7 Redundant Entry** — A. Within the same process, don't make the user re-enter data they already provided; pre-fill or offer to reuse it.
- **3.3.8 Accessible Authentication (Minimum)** — AA. Don't force a cognitive function test (memorize and retype a string, transcribe a captcha, identify objects) without an alternative; allow pasting into password fields; offer SSO or magic links.
- **3.3.9 Accessible Authentication (Enhanced)** — AAA. No cognitive function test at all, even with an alternative.

Two of these are reliably caught by automated tools (target size, and part of focus-not-obscured); the rest need human review. That asymmetry is a theme we'll return to in the testing section.

**A note on levels.** WCAG success criteria are labeled **A**, **AA**, and **AAA**. In practice, **AA is the target** for most public and commercial products: it's the level most laws and procurement policies reference, and it covers the highest-impact barriers. AAA is a stretch goal for specific content (and several AAA criteria, like reflow and no-visual-obstruction, are genuinely hard on complex apps). Treat A and AA as non-negotiable and AAA as aspirational.

---

### 4.2 Semantic HTML first; ARIA only to fill gaps

The single most important sentence in accessibility is the **First Rule of ARIA**:

> If you can express it with a native HTML element, do that instead of reaching for ARIA.

Native elements come with behavior for free. A `<button>` is focusable, responds to Enter and Space, and is announced as a button by every screen reader on Earth—no JavaScript required. A `<label>` is programmatically associated with its input. A `<nav>`, `<main>`, or heading gives structure that assistive tech uses to jump around a page. The moment you replace a `<button>` with a `<div onclick>`, you have to re-implement all of that by hand: focusability, keyboard handling, and the role. You have just written more code to get less.

**Use the right element for the job:**

- Actions that change state → `<button>`.
- Navigation to a new resource → `<a href>`.
- Group related form controls → `<fieldset>` + `<legend>`.
- Associate a label → `<label for>` (or wrap the control).
- Page structure → `<header>`, `<nav>`, `<main>`, `<article>`, `<section>`, `<footer>`.
- Headings → `<h1>`–`<h6>`, in order, no skips.

**So when *do* you use ARIA?** ARIA exists to fill the gaps that native HTML can't express—usually when you build a custom widget that has no native equivalent, or when you need to convey state that the DOM alone can't. Common legitimate uses:

- `aria-expanded` / `aria-controls` on a disclosure or menu trigger.
- `aria-current` on the active item in a nav or pagination.
- `aria-live` regions to announce dynamic content (covered in 4.4).
- `role` to *correct* a native element's implicit role when you have no choice (e.g., a `role="switch"` on a custom toggle).
- `aria-label` / `aria-labelledby` / `aria-describedby` to supply or refine an accessible name or description when visible text can't.

A useful mental model: **ARIA describes the accessibility tree; it does not create behavior.** Setting `role="button"` on a div tells a screen reader "this is a button," but it will *not* make the div respond to Enter or Space. You must add `tabindex="0"` and a `keydown` handler yourself. This is the trap that produces the most "accessible" components that are actually worse than the native element they replaced.

A compact decision procedure:

1. Is there a native element that does this? **Use it.** Stop.
2. No native element, but the widget is standard (tabs, dialog, combobox)? Use the native element where possible and add the *minimal* ARIA to express state.
3. No native element and it's truly custom? Add `role`, an accessible name, keyboard handling, and focus management—and then test with a real screen reader.

Two more rules worth internalizing:

- **Don't override a native role needlessly.** Putting `role="presentation"` on a `<table>` that actually conveys tabular data, or `role="none"` on a real form, strips the semantics you paid for.
- **ARIA is not a substitute for correct DOM.** A `role="heading"` on a `<div>` is announced as a heading, but it won't be in the document outline the way a real `<h2>` is. Prefer the real thing.

---

### 4.3 Keyboard model, focus management, and SPA route-change announcements

#### The keyboard model

A keyboard user (or a power user, or a screen-reader user who has turned off mouse support) navigates with a small, predictable set of keys. Your interface must respect this model:

- **`Tab` / `Shift+Tab`** move focus forward and backward through the *focusable* elements in DOM order. This order should match the visual order. If a user has to Tab five times to reach the second button on screen, your DOM order is wrong.
- **`Enter`** activates a focused link or button (and submits a form).
- **`Space`** activates a focused button (and toggles checkboxes/radios).
- **`Escape`** closes dialogs, menus, and popovers.
- **Arrow keys** navigate *within* a composite widget (a listbox, a tablist, a grid)—this is where the "roving tabindex" pattern lives.

The core principle: **every interactive element must be reachable and operable with the keyboard alone, in a logical order, with a visible focus indicator.** No exceptions. If you build a carousel, the "next" control must be a real button (or keyboard-operable). If you build a custom dropdown, it must be openable and selectable without a mouse.

#### Focus management

Focus is the cursor for keyboard and screen-reader users. Mismanaging it is one of the most common and most disorienting accessibility bugs. The rules:

1. **Never trap focus unintentionally.** A modal *should* trap focus (you don't want Tab to escape into the background), but a popover that accidentally traps it is a trap. Provide a way out—`Escape`, a visible close button, or focus returning to the trigger.
2. **Move focus deliberately on state changes.** When a dialog opens, move focus into it (usually to its heading or first control). When it closes, return focus to the element that opened it. When an inline error appears, move focus to the error summary or the first invalid field.
3. **Keep the focus indicator visible and high-contrast.** This is now literally a WCAG criterion (2.4.13). The most common sin is `outline: none` with no replacement. If you restyle the focus ring, make it obvious.
4. **Don't let focus fall off the edge.** If you remove the focused element from the DOM (a `v-if` that deletes a focused button, a list item that disappears), focus silently jumps to `<body>` and the user is lost. Move focus to a sensible element first.

#### SPA route-change announcements

This is where single-page apps genuinely differ from classic multi-page sites, and where most teams get it wrong.

In a classic site, navigating to a new page replaces the document. Screen readers announce the new page's title and heading, and focus naturally lands at the top. In an SPA, **no document load happens**—you're swapping DOM inside the same page. A screen-reader user who "navigates" from Home to Settings may hear *nothing*, because from their perspective nothing changed. They are now on a different page with no idea.

You have two complementary fixes, and you should usually do both:

**1. Announce the route change in a live region.** Keep a persistent, visually-hidden `aria-live` region in your app shell (it must exist in the DOM *before* the content changes, or some screen readers won't announce it). Update its text on navigation:

```vue
<!-- AppShell.vue -->
<template>
  <a class="skip-link" href="#main">Skip to main content</a>
  <header>…</header>
  <main id="main">
    <RouterView />
  </main>

  <!-- Persistent, visually hidden, present from first render -->
  <div class="visually-hidden" aria-live="assertive" aria-atomic="true">
    {{ routeAnnouncement }}
  </div>
</template>

<script setup>
import { ref, watch } from 'vue'
import { useRoute } from 'vue-router'

const route = useRoute()
const routeAnnouncement = ref('')

watch(
  () => route.fullPath,
  (to, from) => {
    if (from.name) {
      routeAnnouncement.value = ''
      // Clear then set so repeated announcements of the same text re-fire
      requestAnimationFrame(() => {
        routeAnnouncement.value = `Navigated to ${to.meta.label ?? to.name}`
      })
    }
  }
)
</script>
```

```css
.visually-hidden {
  position: absolute;
  width: 1px; height: 1px;
  padding: 0; margin: -1px;
  overflow: hidden;
  clip: rect(0 0 0 0);
  white-space: nowrap;
  border: 0;
}
```

**2. Move focus to the new page.** After a route change, move focus to the main content (or the page's `<h1>`) so keyboard users land somewhere meaningful. Do this *after* the new view has rendered:

```js
import { nextTick } from 'vue'

watch(
  () => route.fullPath,
  async () => {
    await nextTick()
    const main = document.querySelector('#main h1, #main')
    main?.focus()
  }
)
```

For focus to land on a heading or the main region, give it `tabindex="-1"` (programmatically focusable but not part of the Tab order):

```html
<main id="main" tabindex="-1">
  <h1 tabindex="-1">Settings</h1>
</main>
```

A few refinements that make this feel professional:

- **Skip link.** Always ship a "Skip to main content" link that is the first focusable element and becomes visible on focus. It's the single highest-value keyboard feature on a content-heavy page.
- **Use `meta.label`** on your routes so announcements are human-readable ("Billing settings"), not route names (`account-billing-edit`).
- **Don't announce on the initial load.** Announcing "Navigated to Home" when the app first mounts is noise; guard on `from.name` as above.
- **`aria-atomic="true"`** ensures the whole message is read, not just the changed fragment.

---

### 4.4 Forms, errors, and live regions

Forms are where accessibility either shines or collapses, because a form that works with a mouse can be completely unusable with a screen reader.

#### Labels: non-negotiable

Every input needs a **programmatic** label. Placeholder text is *not* a label (it disappears on input, has poor contrast, and isn't reliably exposed). Use `<label for>` or wrap the control:

```html
<!-- Good: explicit association -->
<label for="email">Email address</label>
<input id="email" type="email" name="email" required>

<!-- Also good: wrapping -->
<label>
  Email address
  <input type="email" name="email" required>
</label>

<!-- Bad: placeholder is not a label -->
<input type="email" placeholder="Email address">
```

Group related controls with `<fieldset>` and `<legend>` (think: radio groups, checkbox groups, "delivery options"). The legend is the group's label.

#### Errors: associate, announce, and focus

An error has three jobs: it must be **visible**, **associated** with the field, and **announced**. The robust pattern:

1. Mark the field invalid: `aria-invalid="true"`.
2. Point the field at the error text: `aria-describedby="email-error"`.
3. Give the error text an `id` and render it when (and only when) there's an error.
4. On submit failure, move focus to an **error summary** (or the first invalid field) and announce it.

```vue
<template>
  <form @submit.prevent="onSubmit" novalidate>
    <p v-if="hasErrors" id="form-errors" tabindex="-1" role="alert">
      Please fix the {{ errorCount }} problem{{ errorCount === 1 ? '' : 's' }} below.
    </p>

    <div class="field">
      <label for="email">Email address</label>
      <input
        id="email"
        v-model="form.email"
        type="email"
        :aria-invalid="errors.email ? true : undefined"
        :aria-describedby="errors.email ? 'email-error' : undefined"
      />
      <p v-if="errors.email" id="email-error" class="error">
        {{ errors.email }}
      </p>
    </div>

    <button type="submit">Create account</button>
  </form>
</template>
```

The `role="alert"` on the summary (equivalent to `aria-live="assertive"`) means it's announced the moment it appears. Combined with `tabindex="-1"` and a `focus()` call on submit failure, a keyboard or screen-reader user is taken straight to the problem.

#### Live regions

Live regions are how you tell assistive technology "this part of the page updates on its own, and here's what changed." You use them for anything that changes without a user action on that element: async results, "3 items added to cart," validation that runs as the user types, connection status, toasts.

The mechanics:

- `aria-live="polite"` — announce when the user is idle. Use this for the vast majority of cases (status updates, non-urgent results).
- `aria-live="assertive"` — announce immediately, interrupting. Reserve for genuine emergencies (a destructive action confirmed, a critical error). `role="alert"` is shorthand for assertive.
- `role="status"` is shorthand for polite.
- **The region must exist in the DOM before the content changes.** If you create the live region *at the same time* as you fill it, many screen readers will not announce it. Render the empty container permanently and update its text.
- **`aria-atomic="true"`** reads the entire region's new content; without it, some readers read only the changed fragment.
- **Don't put a live region in a spot that gets torn down** (a `v-if` that unmounts the container kills the announcement).

```html
<!-- Persistent status region; empty until there's something to say -->
<p class="visually-hidden" role="status" aria-live="polite" aria-atomic="true">
  {{ saveStatus }}
</p>
```

```js
// saveStatus.value = 'Saving…'  →  'Saved just now'
```

A common mistake is over-using `assertive`. A polite region that updates a few times a second is fine; an assertive one that fires on every keystroke is hostile. Default to polite, escalate only when the user genuinely cannot afford to wait.

---

### 4.5 Color, contrast, motion, and `prefers-reduced-motion`

#### Color and contrast

Two independent rules matter here.

**1. Don't rely on color alone to convey meaning** (WCAG 1.4.1). "The red items are errors" fails for the roughly 8% of men with a color-vision deficiency. Pair color with a second channel: an icon, an underline, text ("Error:"), or a border.

**2. Meet the contrast ratios.** Contrast is the ratio of relative luminance between text and its background.

- **Normal text:** 4.5:1 (AA). Large text (roughly 18pt / 24px, or 14pt / 18.5px bold) needs 3:1.
- **AAA** raises these to 7:1 (normal) and 4.5:1 (large).
- **Non-text UI** (icons, input borders, focus indicators) needs 3:1 against adjacent colors (WCAG 1.4.11).

Practical guidance:

- Design your palette to hit 4.5:1 for body text *in its normal state*, not just against white. Gray-on-gray "subtle" text is the most frequent real-world failure.
- Test **both** light and dark themes, and test **focus rings** and **disabled states** (disabled elements are exempt from contrast, but "disabled-looking but still interactive" elements are not disabled).
- Use a contrast checker in review, and consider a CI check for your design tokens.

#### Motion and `prefers-reduced-motion`

Some users are genuinely harmed by motion—vestibular disorders and motion sickness can make parallax, auto-playing video, and aggressive transitions physically painful. The OS-level signal is the `prefers-reduced-motion` media query.

```css
/* Default: your normal animations */
.card { transition: transform 0.3s ease; }
.card:hover { transform: translateY(-4px); }

/* Respect the user's OS setting */
@media (prefers-reduced-motion: reduce) {
  .card { transition: none; }
  .card:hover { transform: none; }

  /* Kill scroll-driven and decorative animation */
  .hero { animation: none; }
}
```

Rules of thumb:

- **Respect the query for anything decorative or non-essential.** If an animation isn't conveying information, turn it off under `reduce`.
- **Provide an alternative for essential motion.** If a transition *is* the information (a progress bar, a "slide in to reveal"), don't just delete it—switch to a non-motion equivalent (a fade, an instant state change, or text).
- **Auto-playing content** should be pausable, stoppable, and controllable (WCAG 2.2.2). A hero video that autoplays with no pause control is a failure.
- **Avoid three or more flashes per second.** This is a photosensitivity safety issue (WCAG 2.3.1), not just a preference.
- There are also `prefers-reduced-transparency` and `prefers-contrast` queries worth checking for in design systems.

In Vue, you can also branch in JS if you're doing canvas or JS-driven animation:

```js
const reduceMotion = window.matchMedia('(prefers-reduced-motion: reduce)')
const animate = !reduceMotion.matches
```

---

### 4.6 Testing: axe, Playwright accessibility snapshots, keyboard-only QA

Accessibility testing has three layers, and you need all three. Relying on only the first is the most common mistake.

#### Layer 1: Automated rules (axe)

Automated tools catch a meaningful subset of issues—missing labels, bad contrast, missing alt text, invalid ARIA, missing form associations. **axe-core** is the standard engine. In a Playwright suite, use the official wrapper:

```bash
npm i -D @axe-core/playwright
```

```js
// a11y.spec.js
import { test, expect } from '@playwright/test'
import AxeBuilder from '@axe-core/playwright'

test.describe('Accessibility', () => {
  for (const path of ['/login', '/checkout', '/settings/billing']) {
    test(`has no detectable a11y violations on ${path}`, async ({ page }) => {
      await page.goto(path)
      const results = await new AxeBuilder({ page })
        .withTags(['wcag2a', 'wcag2aa', 'wcag22a', 'wcag22aa'])
        .analyze()
      expect(results.violations).toEqual([])
    })
  }
})
```

The critical thing to understand about axe: **it reports what it finds, and what it finds is a lower bound.** A clean axe run does *not* mean the page is accessible. axe cannot tell you whether your focus order makes sense, whether a screen-reader user can complete the checkout, whether your live regions announce correctly, or whether your custom widget's keyboard model is coherent. Treat axe as a *regression net for the mechanical stuff*, not a verdict.

Run it in CI as a gate, tag it to your target level (`wcag2aa`), and fail the build on new violations.

#### Layer 2: Accessibility-tree snapshots (Playwright)

axe checks *rules*; **Playwright's ARIA snapshots** check *structure*. They capture the page's accessibility tree as YAML—roles, names, attributes, and hierarchy—and let you assert that the tree a screen reader would see is exactly what you intend. This is a powerful way to catch the "it works but the tree is wrong" class of bugs that axe misses.

```js
import { test, expect } from '@playwright/test'

test('checkout summary exposes the right roles and names', async ({ page }) => {
  await page.goto('/checkout')

  await expect(page).toMatchAriaSnapshot(`
    - main:
      - heading "Your order" [level=1]
      - region "Order summary":
        - list:
          - listitem: "Widget × 2 — $40"
          - listitem: "Shipping — $5"
        - text: "Total: $45"
      - button "Place order"
  `)
})
```

How it works and how to use it well:

- Each node is `role "name" [attribute=value]`. Names can be exact strings, or regex in slashes for dynamic text: `- heading /Total: \\$\\d+/.`
- **Partial matching** is the default: you can omit attributes or names you don't care about, and omit children you don't want to pin. This keeps snapshots maintainable.
- **Strict matching** via `/children: equal` (or `deep-equal`) asserts the *exact* set of children, in order—use it when the structure must not change. You can set this globally in `playwright.config.ts` under `expect.toMatchAriaSnapshot.children`.
- **Generate and update snapshots** with the codegen "Assert snapshot" action, by passing an empty template (`await expect(locator).toMatchAriaSnapshot('')`), or by running `npx playwright test --update-snapshots` (alias `-u`). Updates produce reviewable patch files.
- **Store them as files** with the `name` option and a `.aria.yml` extension so they live in source control and diffs are reviewable:

  ```js
  await expect(page.getByRole('main')).toMatchAriaSnapshot({ name: 'main.aria.yml' })
  ```

- **Generate programmatically** with `await page.ariaSnapshot()` or `await locator.ariaSnapshot()` when you need the tree at runtime (e.g., to log it on a failure).

The value of this layer is *intent as a test*: you encode, in a reviewable YAML file, "this is the accessibility tree this page must present." When a refactor silently turns a `<button>` into a `<div>` or drops a label, the snapshot fails and shows you the exact node that changed.

#### Layer 3: Keyboard-only QA

No tool replaces doing the thing. **Keyboard-only QA** means completing your core flows with the keyboard alone, with the mouse in another room. For each critical path (signup, search, checkout, the primary task of your app), verify:

1. **You can reach every control** with `Tab`/`Shift+Tab`, and the **order matches the visual order**.
2. **Every control is operable**: Enter/Space activate buttons and links; `Escape` closes dialogs and menus; arrow keys work inside composite widgets.
3. **The focus indicator is always visible** and high-contrast, and never lands behind a sticky header (2.4.11).
4. **There are no focus traps** you didn't intend, and no focus "falls off the edge" when elements are removed.
5. **Modals trap focus correctly** and return it to the trigger on close.
6. **Route changes announce and re-focus** (see 4.3).

Do this with a real screen reader at least once per release—**NVDA** (free, Windows) and **VoiceOver** (built into macOS/iOS) cover the two biggest platforms. Turn off mouse/pointer usage in the screen reader so you're testing the pure keyboard+SR path.

**The honest summary of the three layers:**

| Layer | Tool | Catches | Misses |
|---|---|---|---|
| Rules | axe-core | Labels, contrast, alt text, invalid ARIA | Focus order, task completion, live-region behavior |
| Structure | Playwright ARIA snapshots | Wrong/missing roles, names, hierarchy | Whether the *behavior* is correct |
| Human | Keyboard-only + SR QA | Everything, including the intangible | Nothing—but it's slow and must be scheduled |

None substitutes for the others. axe keeps you honest on the mechanical floor; snapshots pin your structural intent; keyboard and screen-reader QA confirm a real person can actually use it.

---

### 4.7 Vue-specific pitfalls

Vue is a friendly framework for accessibility when you use it the way it's meant to be used—but it has a few sharp edges. These are the ones that bite most often.

#### Pitfall 1: Custom components that destroy native semantics

The most common Vue accessibility bug is a "component" that renders a `<div>` or `<span>` and pretends to be a button, link, or input. The template looks clean, the click handler works, and then a keyboard or screen-reader user finds a wall.

```vue
<!-- Bad: a div pretending to be a button -->
<template>
  <div class="btn" @click="save">Save</div>
</template>
```

This is not focusable, doesn't respond to Enter or Space, and has no role. If you must build a custom control (because you need a native element to do something it can't), you have to rebuild the semantics *and* the behavior:

```vue
<!-- Better: a real button, styled -->
<template>
  <button type="button" class="btn" @click="save">Save</button>
</template>
```

When a custom element is genuinely unavoidable, the full contract is:

```vue
<template>
  <div
    class="btn"
    role="button"
    tabindex="0"
    :aria-disabled="disabled"
    @click="save"
    @keydown.enter.prevent="save"
    @keydown.space.prevent="save"
  >Save</div>
</template>
```

Note the four things you're now responsible for: the `role`, the `tabindex`, the keyboard handlers, and the disabled state. That's a lot of code to *not* have a `<button>`. Reach for the native element first, every time.

A related Vue-specific trap is **`v-html`**. It injects raw markup that bypasses your component's semantics and your CSP, and it's a frequent source of unlabeled images and structureless content. Audit anything rendered through `v-html` for accessibility, or sanitize and structure it.

#### Pitfall 2: `RouterLink` vs. real buttons

This one is subtle and extremely common. The rule is about **intent**, not appearance:

- **Navigation** (going to a different resource/route) → a link: `<RouterLink>` (renders an `<a>`).
- **Action** (changing state on the current page, submitting, toggling, opening a dialog) → a button: `<button>`.

The failure mode is styling a `RouterLink` to look like a button (or vice versa) and letting the *look* decide the element. The element must match the *job*.

Why it matters beyond semantics: a real `<a href>` gives you middle-click to open in a new tab, right-click "copy link address," and correct screen-reader announcement as a link. A `<button>` gives you form submission and correct "button" announcement. Using a link to fire an action means no middle-click, no "open in new tab," and a screen reader announcing "link" for something that isn't navigation. Using a button for navigation loses all the link affordances.

```vue
<!-- Navigation: go to the user's profile page -->
<RouterLink :to="{ name: 'profile' }">View profile</RouterLink>

<!-- Action: submit the form on this page -->
<button type="submit">Save changes</button>

<!-- Action that looks like a link but isn't navigation -->
<button type="button" class="link-like" @click="removeItem">Remove</button>
```

Two Vue-router-specific notes:

- **`RouterLink` with the `custom` prop** lets you render a link's behavior inside a custom element (e.g., a component that needs to be a button *and* navigate). Use it deliberately, and keep the resulting element's role correct.
- **`aria-current`** is your friend for the active nav item. Vue Router doesn't set it for you, so add it on the active `RouterLink` (or use a small helper) so screen-reader users know where they are:

  ```vue
  <RouterLink
    :to="{ name: 'settings' }"
    :aria-current="route.name === 'settings' ? 'page' : undefined"
  >
    Settings
  </RouterLink>
  ```

#### A few more Vue edges worth knowing

- **Dynamic components (`<component :is>`)** and **`v-if`** can swap in elements with different implicit roles mid-interaction. Make sure the swapped-in element is focusable and correctly labeled, and that focus doesn't get stranded when a `v-if` removes the focused node.
- **`<Teleport>`** (used for modals and tooltips) moves DOM to another location. The accessibility tree follows the DOM, so a teleported dialog is announced from its *new* position—make sure focus management and `aria-modal` are set so the dialog is still a proper modal.
- **Keep live regions in a stable place** (the app shell), not inside a component that gets unmounted on navigation, or your announcements will vanish along with the view.

---

### 4.8 Accessibility is not a sprint-end audit — it is an acceptance criterion

The through-line of this chapter is that accessibility is cheapest when it's a constraint you design against, and most expensive when it's a report you read at the end. A "sprint-end audit" has a specific, predictable shape: a consultant or a tool produces a list of 200 violations, the team scrambles to fix the cheap ones, the expensive ones (a custom widget with no keyboard model, a checkout flow that's incoherent to a screen reader) get deferred, and the product ships "mostly accessible"—which is not a state a user experiences.

Treat accessibility as an **acceptance criterion** instead. Concretely:

1. **Put it in the Definition of Done.** A feature is not done if it fails keyboard-only use or a screen-reader pass. "Works with the mouse" is not done.
2. **Make the mechanical floor a CI gate.** axe (tagged to your target level, e.g. `wcag2aa`) runs on every PR and blocks merges on *new* violations. This is non-negotiable and automated.
3. **Pin structural intent with ARIA snapshots.** For your key pages, commit `.aria.yml` snapshots so a refactor that silently degrades the accessibility tree fails the build and shows the exact node.
4. **Schedule keyboard + screen-reader QA as a first-class task**, not an afterthought. Estimate it, staff it, and do it per release on the critical paths—NVDA and VoiceOver, mouse off.
5. **Make the new WCAG 2.2 criteria explicit in review.** Target size, focus-not-obscured, dragging alternatives, redundant entry, and accessible authentication are the ones most likely to be missed, and most of them need a human eye, not a tool.
6. **Name the owner.** Accessibility without an owner is accessibility without a budget. Someone is accountable for the a11y bar, the same way someone owns performance or security.

The reframe is the point. When accessibility is a constraint, it's a question you ask *while* designing the component: "What's the native element here? What's the focus order? What does the tree look like? Can I do this with a keyboard?" When it's an audit, it's a question you ask *after* the fact, and the answers are expensive.

Design for the constraint. The audit takes care of itself.

---

#### Chapter summary

- **POUR** (Perceivable, Operable, Understandable, Robust) is the lens, not a checklist. Target **WCAG 2.2 AA**; know the nine new criteria, especially focus-not-obscured, target size, and dragging alternatives.
- **Semantic HTML first.** Native elements give you behavior for free; ARIA fills the gaps and describes state, but it never creates behavior.
- **Respect the keyboard model**, manage focus deliberately (open/close dialogs, errors, route changes), and **announce SPA route changes** with a persistent live region plus a focus move to the new page.
- **Forms** need real labels, `aria-invalid` + `aria-describedby` error association, and focus + announcement on failure. **Live regions** must exist before they update and default to `polite`.
- **Meet contrast ratios**, never rely on color alone, and honor **`prefers-reduced-motion`** for anything non-essential.
- **Test in three layers:** axe for the mechanical floor, Playwright ARIA snapshots for structural intent, and keyboard-only + screen-reader QA for the rest. No layer substitutes for the others.
- **In Vue,** don't let custom components destroy native semantics, and match the element to the intent—`RouterLink` for navigation, `<button>` for actions.
- **Accessibility is an acceptance criterion**, baked into the Definition of Done and CI, not a sprint-end audit.

---

A couple of notes on choices I made while writing, in case you want to adjust: I assumed **Vue 3 Composition API** for the code samples (the `RouterLink`/`vue-router` references in your outline point that way), and I used **AA as the target level** throughout since that's the prevailing legal/procurement baseline. If your book targets Vue 2's Options API or a different WCAG level, those sections are the ones to swap.

---

# Part II — Identity, Authorization, and Security

## Chapter 5. OAuth 2.0 and OpenID Connect

> **The one-sentence rule for this chapter:** OAuth 2.0 answers *"is this client allowed to do X to this resource?"* OpenID Connect (OIDC) answers *"who is the user?"* They are two layers, not synonyms. OAuth 2.0 is delegated **authorization**; OIDC is **authentication** built on top of OAuth. The single most common architectural mistake is treating an OAuth access token as an identity claim and "deriving identity from an access token." You must not. Identity comes from the OIDC **ID token** (or the `userinfo` endpoint), never from the access token.

This chapter is written for a stack of Symfony backends, a Vue single-page app (SPA), and Node microservices. Where the standard is still moving, we say so explicitly.

---

### 5.1 Why two protocols?

OAuth 2.0 (RFC 6749) is deliberately *not* an authentication protocol. It was designed to let a third-party app get a *scoped, revocable, time-limited* credential (the **access token**) to call an API on a user's behalf, without ever seeing the user's password. It says nothing about *who* the user is.

A long and painful history of misuse — apps calling a "me" endpoint with an access token and treating the returned user ID as a login — showed that people *wanted* login, and that bolting it onto OAuth by convention was unsafe. OIDC (a thin identity layer on OAuth 2.0) fixed this by adding one well-defined artifact: the **ID token**, a signed JWT whose claims describe the *authenticated user* and whose signature the relying party can verify. OIDC also defines a `/.well-known/openid-configuration` metadata document, a `userinfo` endpoint, and a standard `nonce` binding.

So the clean mental model is:

| Question | Protocol | Artifact you trust for the answer |
|---|---|---|
| "Who is the user?" | OpenID Connect | **ID token** (JWT) / `userinfo` |
| "May I call this API, and to what?" | OAuth 2.0 | **access token** (opaque or JWT) |

A single OIDC login can mint *both* an ID token (for identity) and an access token (for API calls). That is normal. What is *not* normal is reading the access token to learn the user's identity.

---

### 5.2 Concepts

#### 5.2.1 Roles

Four roles. A single physical server can play several at once — the names describe *function*, not *processes*.

- **Resource Owner** — the entity (usually a human, "the end user") that owns the protected resource and can grant access to it.
- **Client** — your application. It makes requests to the protected resource *on behalf of* the resource owner. Note: "client" is client-of-the-protocol, not "browser." A backend service is a client too. Clients split into two security classes that matter enormously later:
  - **Confidential client** — can keep a secret (a server-side app that can hold a `client_secret` safely).
  - **Public client** — *cannot* keep a secret (SPAs, mobile, desktop, CLIs). Everything in the client is visible to the user. This is why public clients *must* use PKCE.
- **Authorization Server (AS)** — authenticates the resource owner, obtains consent, and issues tokens. In OIDC the same server is the **OpenID Provider (OP)**.
- **Resource Server (RS)** — hosts the protected resource and accepts/validates access tokens. The AS and RS are often the same deployment, but they are distinct *roles*, and keeping them distinct in your head prevents most authorization bugs.

A useful picture of the OAuth 2.1 "abstract flow" (the roles and the six steps are identical in 2.0):

```
   +--------+                               +---------------+
   |        |--(1)- Authorization Request ->|   Resource    |
   |        |                               |     Owner     |
   |        |<-(2)-- Authorization Grant ---|               |
   |        |                               +---------------+
   |        |
   |        |--(3)-- Authorization Grant -->| Authorization |
   | Client |                               |     Server    |
   |        |<-(4)----- Access Token -------|               |
   |        |                               +---------------+
   |        |
   |        |--(5)----- Access Token ------>|    Resource   |
   |        |                               |     Server    |
   |        |<-(6)--- Protected Resource ---|               |
   +--------+                               +---------------+
```

Step (1)–(2) is where OIDC adds its identity value: the user authenticates *to the AS/OP* (never to the client), the OP issues an ID token, and the client learns *who* the user is without ever touching the password.

#### 5.2.2 Tokens

Know exactly what each one is, what it's for, and where it may live.

- **Authorization code** — a short-lived, *single-use* ticket returned to the client's redirect URI. It is *not* a token you use against an API; it is exchanged for real tokens at the token endpoint. It exists precisely so that the real tokens are never placed in the browser/URL.
- **Access token** — the credential presented to the *resource server* to access a protected resource. It may be **opaque** (a random string the RS looks up) or a **JWT** (self-contained, signed). It encodes *authorization*: `client_id`, `scope`, audience, expiry. It is *not* an identity document.
- **Refresh token** — a long-lived credential used to obtain a fresh access token (and, in a well-behaved system, a fresh refresh token) without re-prompting the user. It is high-value: treat it like a password. It should be **rotated** (see 5.4).
- **ID token (JWT)** — *OIDC-only*. A signed JWT the **client** verifies to learn the **user's identity**. Its audience is the *client* (the `aud` is your `client_id`), not the resource server. Key claims:
  - `iss` — the OP's issuer (must match exactly).
  - `sub` — the user's stable, OP-unique subject identifier.
  - `aud` — your client ID.
  - `exp` / `iat` — expiry / issued-at.
  - `auth_time` — when the user actually authenticated (important for session semantics).
  - `nonce` — echoes the value you sent, binding the token to your request (replay protection).
  - optional profile claims (`name`, `email`, `email_verified`, …) per the requested scopes.

A JWT is three base64url parts: `header.payload.signature`. The **signature** is what you verify (the OP's key, fetched from its JWKS endpoint). The header tells you the algorithm (`alg`) and key ID (`kid`). The payload is the claims. Never "verify" by decoding the payload alone — that proves nothing.

#### 5.2.3 Grant types

A *grant* is the mechanism by which the client obtains tokens. The field has shrunk a lot since 2012.

**Still acceptable**

- **Authorization Code + PKCE** — the default for *any* client that has a browser in the loop, and *mandatory* for public clients (SPAs, mobile). The authorization code is exchanged at the token endpoint for an access token (+ refresh token, and, in OIDC, an ID token). PKCE (below) is what makes this safe for clients that can't hold a secret.
- **Client Credentials** — for **service-to-service** (machine-to-machine) calls where the client acts *on its own behalf* (there is no resource owner / user). The client authenticates to the AS with its identity (client ID + secret, or a signed assertion / certificate) and receives an access token whose `aud` is the target API. This is the correct tool for "PHP worker calls internal API" — see 5.3.4.

**Dead — do not use, do not offer**

- **Implicit grant** — returned the access/ID token *directly in the redirect URI fragment*. It is dead because the token lands in the browser, is exposed to any in-page script, and (historically) to referrer leakage. Authorization Code + PKCE supersedes it entirely. The OAuth 2.1 draft and RFC 9700 both remove/recommend against it.
- **Resource Owner Password Credentials ("password" grant)** — the user hands their *password* to the client, which forwards it. It is dead because it defeats the entire reason OAuth exists (the client should never see the password), breaks MFA, and makes the client a credential-harvesting target. There is no acceptable use left in a modern system.

> **Rule of thumb:** if you are building a new client in 2026 and you find yourself reaching for implicit or password, stop. You want Authorization Code + PKCE (user-facing) or Client Credentials (machine).

#### 5.2.4 PKCE

**Proof Key for Code Exchange** (RFC 7636) is the mechanism that lets a *public* client use the authorization code flow safely. The idea: the client proves at the token step that it is the same entity that started the authorization step, *without* sharing a secret.

1. The client generates a high-entropy random string, the **`code_verifier`** (43–128 characters).
2. It derives **`code_challenge` = BASE64URL(SHA256(code_verifier))** and sends `code_challenge` + `code_challenge_method=S256` in the authorization request.
3. The AS stores the challenge and returns an authorization code.
4. At the token endpoint, the client sends the original **`code_verifier`**.
5. The AS recomputes `BASE64URL(SHA256(code_verifier))` and checks it equals the stored challenge. If it matches, it issues tokens.

Why it matters: an attacker who *intercepts the authorization code* (e.g., via a malicious browser extension or a shared device) cannot redeem it, because they don't have the `code_verifier`. For confidential clients PKCE is still recommended as defense-in-depth; for public clients it is *required*.

#### 5.2.5 Exact redirect URI matching

The `redirect_uri` the client sends at the *token* step must **exactly match** (string equality) one of the URIs registered for the client, and must match the one used in the authorization request. This is the control that prevents **authorization code injection / open-redirect** attacks, where an attacker tricks the AS into sending a code to *their* URI.

Practical consequences:

- Register the exact URI(s) per environment (dev/stage/prod). No wildcards, no "close enough."
- A trailing slash or a different port is a *different* URI. Test this.
- Never let a user-supplied or config-injected value become the registered redirect URI.

#### 5.2.6 `state` and `nonce`

These are two different protections that people constantly conflate.

- **`state`** — an opaque, unguessable value the client generates, sends in the authorization request, and *must* receive back unchanged in the redirect. It protects against **CSRF** on the authorization request: it binds the response to *this* client's *this* request, so a forged redirect can't be mistaken for a real one. It is an OAuth concept (also used by OIDC). It is *not* encrypted, not signed, just a correlation + integrity check.
- **`nonce`** — an OIDC-specific, unguessable value embedded in the ID token. The client generates it, sends it in the authorization request, and then checks that the `nonce` claim in the returned ID token equals what it sent. This binds the ID token to *this* authentication session and prevents **ID-token replay** (an attacker capturing a valid ID token and presenting it later).

A clean client sends *both*: `state` to protect the redirect, `nonce` to protect the ID token. Keep them independent random values; don't reuse one for the other.

#### 5.2.7 Sender-constrained tokens

A plain **bearer** token is a gun: *whoever holds it can use it.* Every leaked bearer token (logs, URLs, referrers, XSS) is a full compromise. **Sender-constrained tokens** tie the token to *its presenter* so that possession alone is not enough. The two main families:

- **DPoP (Demonstrating Proof of Possession)** — the client proves possession of a key pair with each request (a `DPoP` proof header + `ath` access-token hash). The token is only valid when presented with a valid proof.
- **mTLS / client-certificate binding** — the token is bound to a client TLS certificate; only the holder of the private key can present it.

For service-to-service and for any token that crosses trust boundaries, prefer sender-constrained tokens where your AS and RS support them. Where you must use bearer (which is still common), *assume it can leak* and design storage, logging, and transport accordingly (5.4).

#### 5.2.8 OAuth 2.1 — "secure defaults made mandatory"

**Status as of this writing (2026):** OAuth 2.1 is a **Standards-Track Internet-Draft** (`draft-ietf-oauth-v2-1`, latest revision in the IETF OAuth WG), *not yet a published RFC*. Treat it as the authoritative *consolidation* of where OAuth 2.0 has actually landed, and cite it as a draft.

The 2.1 draft does not invent new magic. It **removes the insecure parts** and **promotes the security best practices (RFC 9700) into the spec**, so that the "secure by default" reading of 2.0 becomes the *only* reading. Concretely:

- **Authorization Code + PKCE** is the required flow for browser-based and native clients; **PKCE is mandatory** for public clients and recommended for all.
- **Implicit** and **Resource Owner Password** grants are **removed**.
- The remaining grant types are **authorization code**, **refresh token**, and **client credentials** (plus the extension mechanism).
- It consolidates RFC 6749, RFC 6750 (bearer), RFC 8252 (native apps), RFC 9700 (security BCP), and the browser-based-apps draft, and *obsoletes* the older core spec.

For a book, the practical guidance is: **design to 2.1.** You will be implementing against real-world servers that still speak 2.0, but 2.1 tells you which 2.0 features are deprecated and which defaults are now expected. If you build your own AS, implement 2.1's reduced surface.

---

### 5.3 Implementation

#### 5.3.1 Symfony: `league/oauth2-server` or an external IdP; firewalls; token storage; clock skew

You have two fundamentally different jobs, and you should not confuse them:

1. **You are the *resource server* (most common).** You *consume* tokens issued by an external IdP (Auth0, Keycloak, Okta, Azure AD/Entra, Google, or your own). You validate them; you do not mint identity.
2. **You are the *authorization server*.** You *issue* tokens to other clients. This is rare and hard; do it only if you genuinely need to be an IdP.

**Option A — external IdP (the default recommendation).** Use a battle-tested IdP and make your Symfony app a resource server / relying party. You get MFA, session management, consent, and key rotation for free. Validate the ID token / access token in a firewall (below). This is the path for 95% of apps.

**Option B — `league/oauth2-server` (you are the AS).** The `thephpleague/oauth2-server` library plus the `thephpleague/oauth2-server-bundle` (root config key `league_oauth2_server`, current releases support recent Symfony) let a Symfony app *be* the authorization server. Be honest about what this buys and costs you:

- **Buy:** a standards-conformant AS in PHP, with grants, scopes, and encryption.
- **Cost:** you now own the *entire* identity and security surface — user accounts, MFA, consent UX, key management, PKCE, refresh rotation, revocation, JWKS rotation, and the security patching of the library itself (it has had CVEs; pin and update). `league/oauth2-server` is an OAuth *authorization* server — it does **not** give you OIDC out of the box. If you need *login* (identity), you need OIDC on top, which `league/oauth2-server` does not provide natively. **If you need SSO/login, use an external OIDC provider or a dedicated OIDC-capable stack, not raw `league/oauth2-server`.**

A representative (version-sensitive) bundle config, shown to illustrate the *shape* — check the current docs for exact keys and supported grants:

```yaml
# config/packages/league_oauth2_server.yaml
league_oauth2_server:
    storage:
        # DB-backed repositories for clients, access tokens, refresh
        # tokens, auth codes, scopes. (Entity/Doctrine adapters exist;
        # confirm the adapter for your current major version.)
        user: App\Repository\UserRepository
        client: App\Repository\ClientRepository
        access_token: App\Repository\AccessTokenRepository
        refresh_token: App\Repository\RefreshTokenRepository
        auth_code: App\Repository\AuthCodeRepository
        scope: App\Repository\ScopeRepository

    encryption_key: '%env(OAUTH_ENCRYPTION_KEY)%'   # for encrypted storage (e.g. defuse)
    private_key: '%env(resolve:OAUTH_PRIVATE_KEY)%' # RS256 signing key (PEM)
    public_key: '%env(resolve:OAUTH_PUBLIC_KEY)%'

    default_scope: 'read'
    scope_map:
        'read':  'Read access'
        'write': 'Write access'
    grant_types:
        - League\OAuth2Server\Grant\AuthorizationCodeGrant
        - League\OAuth2Server\Grant\RefreshTokenGrant
        - League\OAuth2Server\Grant\ClientCredentialsGrant
    # NOTE: do NOT register ImplicitGrant or PasswordGrant.
```

**Wiring a firewall that *validates* inbound tokens** (this is the part you do in both Option A and B):

```php
// config/packages/security.yaml
security:
    firewalls:
        api:
            pattern: ^/api
            stateless: true
            provider: app_user_provider
            # Validate a Bearer/JWT access token. For an external IdP, use a
            # JWT validation listener (e.g. lexik/jwt or a custom authenticator)
            # that checks iss, aud, sig, exp. The authenticator below is the
            # extension point.
            custom_authenticators:
                - App\Security\JwtAuthenticator
```

A stateless `Authenticator` (the modern Symfony extension point) that validates an access token and — *only for the identity you need* — resolves the user from a trusted claim:

```php
// src/Security/JwtAuthenticator.php
namespace App\Security;

use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
use Symfony\Component\Security\Http\Authenticator\AbstractAuthenticator;
use Symfony\Component\Security\Http\Authenticator\Passport\Passport;
use Symfony\Component\Security\Http\Authenticator\Passport\SelfValidatingPassport;

final class JwtAuthenticator extends AbstractAuthenticator
{
    public function supports(Request $request): ?bool
    {
        return $request->headers->has('Authorization');
    }

    public function authenticate(Request $request): Passport
    {
        $jwt = str_replace('Bearer ', '', $request->headers->get('Authorization', ''));

        // 1) Verify signature + iss + aud + exp (see 5.3.3 for the full list).
        //    For an external IdP, fetch/verify against its JWKS.
        $claims = $this->validator->validate($jwt, audience: 'https://api.example.com');

        // 2) Map a *trusted* subject claim to a local user. This is an
        //    authorization mapping, NOT identity derivation from the access
        //    token for login purposes. For login, use the OIDC ID token.
        $sub = $claims['sub'];
        $user = $this->userRepo->findOneBy(['externalId' => $sub]);

        return new SelfValidatingPassport($user, new BearerJWT($jwt));
    }

    public function onAuthenticationSuccess(Request $request, TokenInterface $token, string $firewallName): void
    {
        return; // stateless: nothing to do
    }
}
```

**Storing tokens (server side):**

- **Access tokens:** short-lived. If you keep them in a session/DB, keep them out of URLs and out of logs. Prefer stateless validation (verify signature + claims) over server-side lookup when the token is a signed JWT — it scales and avoids a hot DB path.
- **Refresh tokens:** store **hashed** (never plaintext), with an expiry, an `issued_to` client, and a **rotation counter / family** so you can detect reuse (5.4). Use an encrypted column or an application-level encryption key (the `encryption_key` above) if the DB is a trust boundary.
- **ID tokens:** these are for the *client*, not for your DB. If you store a session, store the verified claims (or a session id), not the raw JWT.

**Clock skew.** Every `exp`/`iat`/`auth_time` check is subject to clock drift between your server and the issuer. Always validate with a **leeway/skew** (commonly 30–60 seconds) so a token that is "expiring in 20s" on the issuer's clock isn't rejected because your clock is 25s slow. In JWT libraries this is a `leeway` parameter. The corollary: a generous leeway slightly widens the window a *stolen-but-not-yet-expired* token is valid — 30–60s is the standard trade-off, not 10 minutes.

#### 5.3.2 Vue / SPA: Authorization Code + PKCE; token storage (memory vs. httpOnly cookie BFF); silent renew

**The flow (Authorization Code + PKCE, no client secret — the SPA is a public client):**

1. Generate `code_verifier` (random) and `code_challenge = BASE64URL(SHA256(code_verifier))`.
2. Generate `state` (CSRF) and `nonce` (ID-token replay).
3. Redirect the user to the OP's `/authorize` with:
   `response_type=code`, `client_id`, `redirect_uri` (exact), `scope`, `state`, `nonce`, `code_challenge`, `code_challenge_method=S256`, `prompt=login` (first time) / `prompt=none` (silent).
4. User authenticates at the OP; OP redirects back to `redirect_uri?code=...&state=...`.
5. **Verify `state`** matches what you stored (e.g., `sessionStorage`/a pre-redirect value). Abort on mismatch.
6. Exchange the code at `/token` with `code_verifier` (and `client_id`). **Do this from the browser only if you have no BFF** — see storage below.
7. Receive `access_token` (+ `id_token` in OIDC). **Verify the ID token** (`iss`, `aud`, `exp`, `nonce`, signature) before trusting any identity claim.

A minimal PKCE helper (Web Crypto):

```js
// src/lib/pkce.js
const toB64Url = (buf) =>
  btoa(String.fromCharCode(...new Uint8Array(buf)))
    .replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');

export function generateVerifier() {
  const bytes = crypto.getRandomValues(new Uint8Array(64)); // 43..128 chars
  return toB64Url(bytes);
}

export async function challengeFromVerifier(verifier) {
  const data = new TextEncoder().encode(verifier);
  const digest = await crypto.subtle.digest('SHA-256', data);
  return toB64Url(digest);
}
```

**Token storage: the decision that actually matters.**

- **In-memory only (JS variable) + httpOnly cookie BFF — the recommended architecture.** The SPA keeps tokens in memory (lost on refresh, which is fine), and a thin **Backend-for-Frontend** holds the *refresh token* in an **httpOnly, Secure, SameSite** cookie. The SPA calls the BFF; the BFF calls the OP's `/token` and the real APIs. Because the refresh token lives in an httpOnly cookie, **XSS cannot read it** — this is the single biggest win. The SPA never sees the refresh token at all.
- **`localStorage`/`sessionStorage` — avoid for refresh tokens.** Any XSS can exfiltrate them. Storing only a *short-lived* access token in memory is acceptable; persisting a refresh token in web storage is not.
- **Why not just put everything in httpOnly cookies with no BFF?** You can (the OP issues tokens, your *own* backend sets the cookie), but then the SPA's backend must do the token exchange and proxy calls — which *is* a BFF. The BFF is the clean way to express this.

**Silent renew.** When the access token expires (or on app load), re-authenticate *without* user interaction:

- **With a BFF:** the BFF uses the stored (httpOnly) refresh token to mint a new access token. If the refresh token is rotated, the BFF stores the new one.
- **Without a BFF (pure SPA, not recommended):** use `prompt=none` with a hidden iframe (or the OP's JS SDK) to get a fresh ID/access token from the OP's existing session. This depends on third-party cookies / same-site behavior in the browser, which is increasingly unreliable — another reason the BFF is the robust choice.

**Practical SPA rules:**

- Send the access token as `Authorization: Bearer <token>` to *your* APIs (or the BFF). Never put tokens in URLs (query strings leak into logs, referrers, history).
- On a 401, attempt one silent renew, then retry once; if that fails, redirect to login. Don't spin in a refresh loop — guard with a single in-flight promise.
- Respect `auth_time` from the ID token if you have a "re-authenticate for sensitive action" requirement (step-up).

#### 5.3.3 Node microservices validating JWTs (issuer, audience, signature, expiry)

A Node service that is a **resource server** validating a JWT access token (or ID token) must check, *in this order*, and reject on any failure:

1. **Signature** — verify against the issuer's public key (JWKS). Confirm the `alg` in the header is one you allow (defend against `alg=none` and HS/RS confusion attacks — pin the expected algorithm).
2. **`iss` (issuer)** — must exactly equal the expected issuer URI. This is your defense against **mix-up attacks** (5.4).
3. **`aud` (audience)** — must include *your* service's audience. A token minted for a different client/service must be rejected.
4. **`exp` (expiry)** and **`nbf` (not-before)** — with a small **leeway** for clock skew.
5. **`iat`** — sanity check (e.g., not in the far future).
6. **Optional but recommended:** `jti` (dedupe/replay), `azp` (authorized party, when `aud` is shared), and any app-specific claims.

A representative Express middleware using a JWKS-backed verifier (e.g., `jose` or `jsonwebtoken` + `jwks-rsa`):

```js
// middleware/verifyJwt.js
import { createRemoteJWKSet, jwtVerify } from 'jose';

const ISSUER  = process.env.OIDC_ISSUER;   // e.g. https://idp.example.com/realms/app
const AUDIENCE = process.env.SERVICE_AUD;  // e.g. https://api.internal.example.com
const JWKS = createRemoteJWKSet(new URL(`${ISSUER}/protocol/openid-connect/certs`));

export async function verifyJwt(req, res, next) {
  const header = req.headers.authorization || '';
  const token = header.startsWith('Bearer ') ? header.slice(7) : null;
  if (!token) return res.status(401).json({ error: 'missing_token' });

  try {
    const { payload } = await jwtVerify(token, JWKS, {
      issuer: ISSUER,        // exact match
      audience: AUDIENCE,    // must be present for us
      algorithms: ['RS256'], // pin; reject 'none' / unexpected algs
      clockTolerance: 30,    // seconds of clock skew
    });

    // Do NOT derive *identity/login* from an access token. If you need the
    // user, use a claim the issuer guarantees, or use the ID token/userinfo.
    req.auth = { sub: payload.sub, azp: payload.azp, scope: payload.scope };
    return next();
  } catch (err) {
    return res.status(401).json({ error: 'invalid_token' });
  }
}
```

Notes:

- **Cache the JWKS** but handle key rotation: on an unknown `kid`, re-fetch once (don't hammer the endpoint per request).
- **`aud` can be an array** — check membership, not equality.
- **Pin `algorithms`.** A verifier that "accepts whatever the header says" is how `alg=none` and HMAC/RSA confusion attacks happen.
- For **bearer** access tokens, remember the token is only as strong as its secrecy (5.2.7). Prefer **mTLS** or **DPoP** for service-to-service where available.

#### 5.3.4 Machine-to-machine between PHP workers and internal APIs

This is **Client Credentials** — there is no user, so there's no authorization code, no consent, no ID token. The PHP worker authenticates *as itself* to the AS and gets an access token scoped to the internal API.

**Flow:**

1. Worker (a **confidential client**) calls the AS `/token` with `grant_type=client_credentials`, its `client_id`, and its credential (a `client_secret`, or — better — a **client assertion** / **mTLS** for a sender-constrained token).
2. AS returns a short-lived access token with `aud` = the internal API and the needed `scope`.
3. Worker calls the internal API with `Authorization: Bearer <token>`.
4. Cache the token in the worker and refresh *before* `exp` (e.g., at 80% of lifetime), with a single in-flight refresh to avoid a thundering herd across worker processes.

**Best practices:**

- **Short-lived tokens** (minutes), not hours. M2M tokens should be the most tightly scoped and shortest-lived things in your system.
- **Scope minimization:** a worker that only reads metrics should not hold a `write` scope.
- **Prefer sender-constrained credentials:** a **client assertion (JWT signed with a per-service key)** or **mTLS** beats a shared `client_secret` in a config file, because the secret isn't a bearer string that can be copied. If you must use a secret, store it in a secrets manager, never in the repo, and rotate it.
- **Per-service identities:** give each worker/service its *own* client so you can audit and revoke independently. One shared "backend" client is an audit black hole.
- **Honor `aud`:** the internal API must reject tokens whose audience isn't itself.

A small PHP helper sketch (using a generic HTTP client):

```php
// src/Service/M2MTokenProvider.php
final class M2MTokenProvider
{
    private ?array $token = null; // ['access_token' => ..., 'expires_at' => int]

    public function accessToken(): string
    {
        // Reuse until ~80% of its lifetime, then refresh (single-flight).
        if ($this->token && time() < $this->token['expires_at'] - 60) {
            return $this->token['access_token'];
        }

        $res = $this->http->post($this->tokenUrl, [
            'grant_type'    => 'client_credentials',
            'client_id'     => $this->clientId,
            'client_secret' => $this->clientSecret, // or a signed client_assertion
            'scope'         => 'metrics:read',
        ]);

        $data = json_decode($res->body, true);
        $this->token = [
            'access_token' => $data['access_token'],
            'expires_at'   => time() + (int) $data['expires_in'],
        ];

        return $this->token['access_token'];
    }
}
```

---

### 5.4 Failure modes

These are the attacks that actually happen in the wild. Learn them as *patterns*, not trivia.

#### 5.4.1 Confused deputy

The client (the "deputy") is trusted by the resource server to act only within its granted scope — but it acts on *user-supplied input* that widens that reach. Classic shapes:

- The app takes a `?user_id=` or `?account=` parameter and calls the API "as that user," letting an attacker read *anyone's* data through a single legitimate token.
- The app forwards an unvalidated `scope` or an attacker-chosen `redirect_uri`/`aud` so the deputy requests more than the user consented to.

**Defenses:** the resource server authorizes on the *token's* claims (subject, scope, audience), never on client-supplied identity; the client never lets user input select *whose* token or *which* scope is used; scopes are minimal and enforced server-side.

#### 5.4.2 Token leakage via logs / URLs

Bearer tokens are the canonical "leak and you're done" artifact. They leak through:

- **URLs** — tokens in query strings end up in server access logs, proxy logs, browser history, and `Referer` headers. (This is exactly why the *implicit* grant was killed.)
- **Logs** — a request logger that dumps headers or the full request will print `Authorization: Bearer …`.
- **Error pages / crash reports** — full URLs and headers in stack traces and APM tools.
- **Client-side storage** — refresh tokens in `localStorage` exfiltrated by XSS.

**Defenses:** tokens **only** in the `Authorization` header (or httpOnly cookies via a BFF); **redact** `Authorization`, `Cookie`, and token-bearing query params in every logger and APM; keep access tokens short-lived so a leak has a short half-life; use sender-constrained tokens (mTLS/DPoP) where the token crosses trust boundaries.

#### 5.4.3 Refresh-token rotation mistakes

Refresh tokens should be **rotated**: each use returns a *new* refresh token and invalidates the old one. Done wrong, rotation creates new bugs:

- **No reuse detection.** If a client presents an *already-rotated* (stale) refresh token, that's a strong signal the token was **stolen** (the thief used the "current" one, forcing rotation, and now the legit client's old one is replayed). The correct response is to **revoke the whole token family / session**, not just reject the one request.
- **Race conditions.** Two concurrent requests both try to use the same refresh token; one must win, the other must be handled deterministically (idempotent window or reject), not cause a spurious revocation.
- **Rotating the refresh token but not the access token** (or vice versa) inconsistently, leaving a long-lived credential alive after you thought you "logged out."
- **Storing rotated tokens in plaintext** so a DB leak hands out the current refresh token.

**Defenses:** store refresh tokens **hashed**, with a **family/rotation counter** and an expiry; on detecting reuse of a stale token, **revoke the family**; make rotation idempotent within a small window; revoke the full chain on logout and on suspected theft.

#### 5.4.4 Mix-up attacks

The client is pointed at the **wrong identity provider** — one with the *same* metadata shape but a different `iss` — and ends up accepting an ID token from an attacker-controlled or unintended OP. Because the token is well-formed and signed (by the *other* OP), a naive "verify the signature" check passes.

**Defenses:** **validate `iss` for exact equality** against the expected issuer (this is the primary control); pin the OP's **JWKS URI** and fetch keys from *that* URI, not from whatever the token's header implies; pin expected `aud`; ideally pin the OP's TLS certificate / use a fixed discovery document. The general principle: *validate the issuer before you trust the signature.*

#### 5.4.5 A few more worth naming

- **Authorization-code injection / open redirect** — a non-exact or user-influenced `redirect_uri` lets an attacker receive the code. (Exact-match, 5.2.5.)
- **CSRF on the authorization request** — missing/unchecked `state`. (5.2.6.)
- **ID-token replay** — missing/unchecked `nonce`. (5.2.6.)
- **`alg=none` / algorithm-confusion** — a verifier that trusts the header's `alg`. (Pin algorithms, 5.3.3.)
- **Unbounded token lifetime** — long-lived bearer tokens multiply every other failure above.

---

### 5.5 Checklist for reviewing any new client

Run this before any new OAuth/OIDC client ships. It is a *gate*, not a suggestion.

**Identity & flow**

- [ ] Is this a **public** or **confidential** client? (Determines whether PKCE is mandatory.)
- [ ] Using **Authorization Code + PKCE** for any browser/native flow? **Client Credentials** for M2M?
- [ ] **Not** using implicit or password grant. If it is, it does not ship.
- [ ] **PKCE** present with `code_challenge_method=S256`; verifier high-entropy; verifier *not* logged.
- [ ] **Identity comes from the OIDC ID token / `userinfo`** — *not* derived from the access token.

**Request integrity**

- [ ] `state` generated (unguessable), stored before redirect, **verified** on return; mismatch aborts.
- [ ] `nonce` generated and **verified** in the ID token (OIDC).
- [ ] `redirect_uri` **exactly matches** a registered URI, per environment. No wildcards, no user input.

**Token validation (resource server / relying party)**

- [ ] **Signature** verified against the correct JWKS; `alg` **pinned** (no `none`, no confusion).
- [ ] **`iss`** exact-matched (mix-up defense).
- [ ] **`aud`** includes this service/client.
- [ ] **`exp` / `nbf`** checked with a small **leeway** (clock skew).
- [ ] `jti` / replay checks where applicable; `azp` checked when `aud` is shared.

**Storage & lifecycle**

- [ ] Access tokens short-lived; sent only in `Authorization` header (or httpOnly cookie via BFF) — **never** in URLs.
- [ ] Refresh tokens **hashed** at rest, **rotated** with **reuse detection** that revokes the family; revoked on logout and suspected theft.
- [ ] Refresh token out of reach of XSS (httpOnly cookie / BFF), not in `localStorage`.
- [ ] **Logging redacts** `Authorization`, `Cookie`, and token-bearing query params across all services and APM.

**Least privilege & blast radius**

- [ ] **Minimal scopes**; enforced server-side on the token's claims, not on client input.
- [ ] No user-supplied parameter selects *whose* token/scope is used (confused deputy).
- [ ] M2M: per-service identities, short-lived tokens, sender-constrained credentials (mTLS/DPoP) preferred over shared secrets.
- [ ] A leak of any single token has a **short half-life** and a **bounded scope**.

**Ops**

- [ ] JWKS **key rotation** handled (re-fetch on unknown `kid`, cached otherwise).
- [ ] **Revocation** path exists and is tested (token + session + refresh family).
- [ ] Clocks are NTP-synced; leeway is documented.
- [ ] Dependencies (e.g., `league/oauth2-server`) pinned and patched; you know its CVE history.

---

#### Chapter summary

- **Two layers:** OAuth 2.0 = authorization (access tokens); OIDC = authentication (ID tokens). Never derive identity from an access token.
- **Two live grants:** Authorization Code + PKCE (user-facing, mandatory for public clients) and Client Credentials (M2M). Implicit and password grants are dead.
- **Three integrity controls you must not skip:** PKCE (code binding), exact `redirect_uri` matching (code-injection defense), and `state` + `nonce` (CSRF + ID-token replay).
- **Validate in order:** signature → `iss` → `aud` → `exp` (with leeway). Pin the algorithm.
- **Store defensively:** tokens in headers/httpOnly cookies, never URLs; refresh tokens hashed and rotated with reuse detection; logs redacted.
- **Design to OAuth 2.1** (the secure-defaults consolidation; a Standards-Track draft as of this writing) even while interoping with 2.0 servers.
- **When in doubt, the checklist in 5.5 is the contract.** A client that fails any red-line item (implicit/password grant, identity-from-access-token, missing `iss` check, refresh token in `localStorage`, token in a URL) is not a "minor issue" — it is a release blocker.

## Chapter 6. Authorization Models: RBAC, ABAC, ReBAC

Authentication answers *"who are you?"* Authorization answers *"what may you do with this thing?"* The two are routinely conflated in code — a session cookie that proves identity is then treated as proof of permission — and that conflation is one of the most common sources of Broken Access Control, the top entry of the OWASP Top 10 for years. OWASP's position is explicit and worth internalizing before writing any policy code: **prefer attribute- and relationship-based access control over role-only designs**. Roles remain useful as *one* attribute, not the whole model. [[3]](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)

This chapter walks through the three models — RBAC, ABAC, ReBAC — maps them onto the project board domain we've been building, decides *where* in the stack each check belongs, and closes with the testing discipline that keeps authorization honest.

---

### 6.1 The three models in one breath

| Model | Decision is based on | Mental model |
|---|---|---|
| **RBAC** | Roles assigned to the user | *flags* |
| **ABAC** | Attributes of subject, resource, and environment, evaluated against policies | *rules* |
| **ReBAC** | Relationships between entities (user → board, org → board) | *graphs* |

They are not rivals; they are different levels of expressiveness. A mature system typically uses all three, each where it fits.

---

### 6.2 RBAC: roles → permissions

RBAC is the classic: permissions attach to roles, users get roles, and the user inherits the union of their roles' permissions.

```yaml
# config/packages/security.yaml
security:
    role_hierarchy:
        ROLE_ORG_ADMIN: [ROLE_MEMBER]
        ROLE_ADMIN: [ROLE_ORG_ADMIN]
```

**Why it's fast to start.** It matches how most teams talk about the product ("admins can do X"), it's trivial to audit ("who has `ROLE_ADMIN`?"), and debugging is easy: print the user's roles and compare against a table. For a two-tier app (users + one admin panel) it is genuinely enough.

**Why it explodes.** The moment permissions depend on *which* resource, roles stop scaling:

- `can_edit_any_board` vs. `can_edit_own_board` — now the role encodes a relationship.
- `board_editor_org_A`, `board_editor_org_B` — now the role encodes a tenant.
- "Managers can approve their own team's requests" — now the role encodes a hierarchy *and* a relationship.

You end up with **role soup**: dozens of near-duplicate roles, users wearing five of them, and no way to answer "what can Alice do?" without enumerating combinations. The failure mode is structural, not accidental: RBAC's decision logic is *presence or absence of a role*, which is a poor fit for object-level (horizontal) decisions. That is precisely OWASP's objection [[3]](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html).

**Rule of thumb:** RBAC is fine as long as every permission you can name is expressible as "has role R." The day you write "has role R *and* the resource belongs to them," you have outgrown it.

---

### 6.3 ABAC: attributes, not flags

ABAC generalizes the decision: a request is granted or denied based on **attributes of the subject** (user), **attributes of the resource** (object), **conditions in the environment**, and **policies** written in terms of those three. NIST SP 800-162 defines attributes simply as name–value pairs — job role, time of day, project name, MAC address, creation date are all legitimate attributes.

```php
// A policy, in pseudo-code:
//   permit
//     subject.role == "member"
//     and resource.visibility == "org"
//     and environment.time_in_business_hours
//     and environment.device.posture == "compliant";
```

Note that **"role" appears as just one subject attribute** — RBAC demoted to a special case, exactly as OWASP recommends [[3]](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html).

**The upside is least privilege.** You can express "an accountant may access the customer database from the internal network during working hours, but not from home at midnight" — something no role list can say. Decisions can vary per resource, per time, per device, without inventing new roles.

**The downside is reasoning.** Policies are programs. They compose, they interact, and a change to one attribute's meaning ("does `posture == compliant` include 'pending attestation'?") can shift dozens of decisions at once. You need:

1. **One place** where policies live (see §6.7 — no scattering).
2. **A way to evaluate a policy offline** — "what would this policy decide for Alice on board 42?" — which is really just a pure function, and pure functions are testable.
3. **A periodic review for *privilege creep***: after deployment, re-check that effective permissions haven't drifted beyond the design-phase intent. It is always easier to grant than to revoke. [[3]](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)

ABAC is where "business hours + device posture" lives. It is also the model that makes **deny-by-default** natural: a policy engine with no matching rule returns *deny*, and you must explicitly justify every grant.

---

### 6.4 ReBAC: relationships, not attributes

ReBAC (the model behind Google's Zanzibar and most modern multi-tenant systems) answers a different question: not "what attributes does this user have?" but **"what is this user's relationship to this resource?"**

- *"user **is owner of** board"* — direct relationship.
- *"user **is member of** org that **owns** board"* — relationship *through* another entity.

```
alice --member_of--> acme_org --owns--> board_42
bob   --member_of--> acme_org --owns--> board_42
carol --member_of--> globex
```

The policy is now a **graph traversal**: *grant `board:read` to users related to the board by `member_of ∘ owns`*. That single line encodes "any member of the owning org can read the board" — in RBAC this was a role per org, in ABAC it was a join condition, in ReBAC it is a relationship path.

Why graphs beat flags for this:

- **The data is already a graph.** `users → orgs → boards → cards` is your schema. ReBAC stops *re-encoding* relationships as role names and flags, and reads them from where they live.
- **Transitivity is explicit.** "Member of the org that owns the board" is a two-hop path you can see, test, and log. In role soup, the equivalent is a role named `org_member_board_reader` whose meaning only a 2023 ticket explains.
- **Granularity is free.** "Owner can delete; members can edit; org admins can manage membership" is three relationship paths on the same graph — no new roles, no new attribute schemes.

The cost: you need a place to store and query relationships (in our stack, the relational schema *is* the graph — `org_memberships`, `boards.organization_id`), and policies become path expressions rather than boolean expressions. For a project board, the relational schema already is the relationship store; ReBAC here costs you a policy DSL, not a new database.

---

### 6.5 Mapping the project board

Here is how the three models divide the work in this application. Notice that **no single model owns the whole story** — that is the point.

| Concern | Model | Why |
|---|---|---|
| "Alice can read board 42 because she's in the org that owns it" | **ReBAC** | It *is* a relationship path: `user → member_of → org → owns → board`. Re-encoding it as `ROLE_ORG_A_MEMBER` is role soup. |
| "Alice can delete board 42 because she is the org's admin" | **ABAC** (with role as an attribute) | "Admin" is a property of the *membership*, not a global badge. `membership.role == "admin"` composes with the ReBAC path: *admin of the org that owns the board*. |
| "Nobody can edit from a non-compliant device," "exports only in business hours" | **ABAC** (environment attributes) | Pure environment conditions. No role or relationship expresses this. |
| "The platform operator can view anything, for support" | **RBAC** | A true global role, with no resource-relative meaning. This is where RBAC earns its keep. |

A unified policy for `board:delete` reads:

```
permit board:delete
  if  subject.role == "platform_operator"                       // RBAC escape hatch
  or  subject --member_of[role=admin]--> org --owns--> board    // ReBAC + ABAC
  or  subject --owner_of--> board;                               // ReBAC
```

Two disciplines from OWASP apply to every line of that policy:

1. **Deny by default.** No matching clause ⇒ deny. New endpoints and new resource types start *locked*; access is granted by explicit policy, never by framework default. Configure this explicitly — don't rely on a library's default posture, which can change under you. [[3]](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
2. **Validate on every request.** AJAX, server-side, background jobs, webhooks — every entry point re-checks. One unguarded route is enough; "the majority of requests are protected" is not a security property.

---

### 6.6 Enforcement points: where the check actually lives

A policy is only as strong as its *enforcement points*. The same decision can be enforced at several layers, and each layer catches a different class of mistake:

| Layer | Catches | Can't catch |
|---|---|---|
| **API gateway / firewall** | Unauthenticated traffic, coarse route guards (`/admin/**` requires `ROLE_ADMIN`) | Anything resource-specific. "This admin route" ≠ "this admin's org." |
| **Controller** | Wrong *action* for the user (`#[IsGranted('board:delete', $board)]`) | Wrong *resource* when the resource is fetched by ID — **this is where IDOR lives** (below). |
| **Domain service** | Business invariants ("can't archive a board with open billing") that imply access | Nothing on its own; it *assumes* the caller already holds the board. |
| **Query filter** | **Everything, for reads and lists.** The only layer that can stop a user from even *seeing* rows they don't own. | Write-path checks (a query filter can't stop `DELETE /boards/42`). |

#### IDOR lives in the query

Insecure Direct Object Reference — the canonical horizontal privilege escalation — is born in the data-access layer, not the controller:

```php
// ✗ The controller check is a speed bump; the query is the door.
public function show(string $id): Response
{
    $board = $this->boards->find($id);          // ← anyone with a valid ID gets the board
    $this->denyAccessUnlessGranted('board:read', $board); // too late: the object is already out
    // ...
}
```

The controller check above *can* be correct for a single read — but it fails everywhere it actually matters:

- **List endpoints.** `GET /boards` returns every board the query can see; a per-item check after the fact means you already serialized the data.
- **Any endpoint that takes a bare ID** and fetches before checking.
- **Subtle paths**: a `find()` in a service called from two controllers, where one forgot the check.

The fix is to make the query itself policy-aware — scope every read to what the subject may see:

```php
// ✓ The query filter *is* the authorization for reads.
public function scopeFor(User $user, QueryBuilder $qb): QueryBuilder
{
    $qb->andWhere('b.organization IN (SELECT o.id FROM App\Entity\Organization o
                                      JOIN o.memberships m WITH m.user = :user)')
       ->setParameter('user', $user);
    return $qb;
}
```

Now `find($id)` under that scope returns `null` for a foreign board, and the endpoint 404s — the IDOR is closed at the only layer where it can be closed. Write paths still need the controller/service check, because a scoped query can't stop a `DELETE`.

**Defense in depth, explicitly:** the gateway enforces coarse routes, the controller enforces action-level grants, the domain service enforces invariants, and the query scope enforces visibility. OWASP is blunt that you should not depend on any single framework, library, or control as the *sole* enforcer [[3]](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html) — and in practice, the layer that saves you is the one you'd have forgotten to check manually.

---

### 6.7 Symfony: Voters and expressions vs. a dedicated policy module

Symfony's built-in authorization is `AccessChecker` + **Voters** + **security expressions**. It is the right starting point, and it degrades gracefully.

#### Voters

A Voter answers "does this token hold this attribute on this subject?" The current API (Symfony 7.1+):

```php
use Symfony\Component\Security\Core\Authorization\Voter\Voter;
use Symfony\Component\Security\Core\Authorization\Vote;
use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
use App\Entity\Board;

class BoardVoter extends Voter
{
    public function supports(string $attribute, mixed $subject): bool
    {
        return str_starts_with($attribute, 'board:') && $subject instanceof Board;
    }

    protected function voteOnAttribute(
        string $attribute,
        Board $board,
        TokenInterface $token,
        ?Vote $vote = null,
    ): bool {
        $user = $token->getUser();
        if (!$user instanceof User) {
            return false; // deny by default: anonymous can't touch boards
        }

        return match ($attribute) {
            'board:read'   => $this->org()->isMember($user, $board->getOrganization()),
            'board:update' => $this->org()->isMember($user, $board->getOrganization()),
            'board:delete' => $this->org()->isOrgAdmin($user, $board->getOrganization())
                              || $board->getOwner()->getId() === $user->getId(),
            default        => false, // deny by default: unknown attribute ≠ grant
        };
    }
}
```

And in the controller:

```php
#[Route('/boards/{id}', name: 'board_show', methods: ['GET'])]
public function show(Board $board): Response
{
    $this->denyAccessUnlessGranted('board:read', $board);
    // ...
}
```

#### Security expressions

For one-off or config-level checks, expressions keep you out of PHP entirely:

```php
#[IsGranted(new Expression(
    'is_granted("ROLE_ADMIN") or (is_authenticated() and user.isPlatformOperator())'
))]
public function supportPanel(): Response { /* ... */ }
```

Two notes: `has_role()` was deprecated in Symfony 4.2 — use `is_granted('ROLE_ADMIN')` (or the `role_names` variable) everywhere. And expressions are great for *subject-only* decisions ("is this user a platform operator?") — the moment the decision needs the *resource* ("…and this board is theirs?"), expressions get stringly-typed and Voters take over.

#### When to graduate to a dedicated policy module

Voters are excellent until they aren't. You'll feel the pain as:

- the same relationship traversal (`isMember`, `isOrgAdmin`) gets copy-pasted across five voters;
- an ABAC condition ("business hours + device posture") has no natural home in a `match` on attribute strings;
- you want to *evaluate* a policy in tests, in a CLI command ("what can Alice do?"), or in the query scoping of §6.6 — none of which have a `TokenInterface` handy.

The move is to extract the **pure decision logic** into a policy module, and make the Voter a thin adapter:

```php
// src/Security/Policy/BoardPolicy.php
final class BoardPolicy
{
    public function __construct(
        private readonly OrganizationRepository $orgs,
    ) {}

    /** The single source of truth: (subject, resource, action) -> bool. */
    public function can(User $user, Board $board, string $action): bool
    {
        if ($user->hasRole('ROLE_PLATFORM_OPERATOR')) {
            return true; // RBAC escape hatch, audited, logged
        }

        $org = $board->getOrganization();

        return match ($action) {
            'read'   => $this->orgs->isMember($user, $org),
            'update' => $this->orgs->isMember($user, $org),
            'delete' => $this->orgs->isOrgAdmin($user, $org)
                         || $board->isOwnedBy($user),
            default  => false, // deny by default
        };
    }
}

// src/Security/Voter/BoardVoter.php — now a 6-line adapter
class BoardVoter extends Voter
{
    public function __construct(private readonly BoardPolicy $policy) {}

    public function supports(string $attribute, mixed $subject): bool
    {
        return str_starts_with($attribute, 'board:') && $subject instanceof Board;
    }

    protected function voteOnAttribute(string $attribute, Board $board, TokenInterface $token, ?Vote $vote = null): bool
    {
        $user = $token->getUser();
        return $user instanceof User
            ? $this->policy->can($user, $board, substr($attribute, 6))
            : false;
    }
}
```

Now `BoardPolicy::can()` is a **pure function of (user, board, action)** — no tokens, no HTTP. That buys you:

- **Query scoping** (§6.6) that calls the same logic the controller does;
- **CLI and tooling**: `bin/console debug:policy alice board:42` to answer "what can she do?" during support;
- **Trivially testable** policies with plain fixtures and no kernel;
- One place to add the ABAC layer (environment attributes) when a compliance requirement lands — a new parameter to `can()`, not a new voter.

**Bottom line:** start with Voters and expressions — they are Symfony's native idiom and cover 80% of a first version. The moment decision logic is *shared* (controller, query scope, tests) or *environmental* (ABAC), extract it into a policy module and keep the Voter as its adapter. The Voter is the *enforcement point*; the policy is the *decision*. Don't confuse the two.

---

### 6.8 Vue: hiding the UI is UX, not authorization

The frontend will be tempted to do authorization. Resist, and be precise about what the UI is *for*:

```vue
<script setup>
import { can } from '@/composables/useAuthorization'
</script>

<template>
  <BoardHeader :board="board">
    <button v-if="can('board:update', board)" @click="openRename">Rename</button>
    <button v-if="can('board:delete', board)" class="danger" @click="confirmDelete">
      Delete board
    </button>
  </BoardHeader>
</template>
```

What this code is: **progressive disclosure**. It keeps the interface honest — a member doesn't see a Delete button that would just 403 — which is good UX and good *security theater in the right direction*: it reduces the attack surface an attacker can enumerate by clicking, and it prevents confused users from triggering errors.

What this code is **not**: authorization. The `v-if` runs in the attacker's browser, against data the attacker already received, with logic the attacker can read. An attacker does not click your button; they send:

```
DELETE /api/boards/42
```

directly, with a valid session for a user who is not the owner. The only thing that decides the outcome is the server-side `denyAccessUnlessGranted('board:delete', $board)` (or the scoped query returning nothing).

**The rule, stated once and enforced everywhere: every mutation — and every read, if you want list-level IDOR closed — is re-checked server-side, on every request, regardless of what the UI showed.** The client's `can()` is a *cache of the server's decision for rendering purposes only*; it must never be the decision. Two practical consequences:

1. **Never gate on client state alone.** `v-if="user.role === 'admin'"` is doubly wrong: it trusts a role string that lives in the browser, and it duplicates policy. Derive `can()` from the same permission list the server computed and sent (e.g., the user's granted actions per resource type), and treat it as advisory.
2. **Design the 403/404 contract deliberately.** For resources the user can't even *see* exist (foreign org's board), return **404**, not 403 — a 403 confirms the resource exists and is merely off-limits, which is information an IDOR scanner wants. Reserve 403 for "you exist, the resource exists, and this *action* is denied to you."

---

### 6.9 Testing authorization: the matrix

Authorization bugs are rarely "the policy is wrong" — they are "the policy is right but *this combination* was never exercised." So you test the **cartesian product: actor × resource × action**, with the expected outcome for every cell.

#### The matrix

Actors (fixtures): `owner`, `member_same_org`, `org_admin_same_org`, `member_other_org`, `platform_operator`, `anonymous`.
Resources: `own_board`, `colleague_board_same_org`, `board_other_org`, `nonexistent_board`.
Actions: `read`, `update`, `delete`, `invite_member`.

| actor \ action → | read | update | delete | invite_member |
|---|---|---|---|---|
| **owner / own_board** | 200 | 200 | 200 | 200 |
| **member_same_org / colleague_board_same_org** | 200 | 200 | **403** | 200 |
| **org_admin_same_org / colleague_board_same_org** | 200 | 200 | 200 | 200 |
| **member_other_org / board_other_org** | **404** | **404** | **404** | **404** |
| **platform_operator / board_other_org** | 200 | 200 | 200 | 200 |
| **anonymous / own_board** | **401** | **401** | **401** | **401** |
| **any authenticated actor / nonexistent_board** | **404** | **404** | **404** | **404** |

The cells that matter most are the **bolded ones** — they are the bugs. Note the asymmetry: `member_same_org` gets **403** on delete (the board is visible to them; the *action* is denied), while `member_other_org` gets **404** on everything (the resource must not even appear to exist). A test that asserts only status codes will catch both; a test that asserts "not 200" will catch neither.

#### As a data provider

```php
/**
 * @dataProvider boardAccessMatrix
 */
public function testBoardAccessMatrix(
    string $actor,
    string $action,
    string $resource,
    int $expectedStatus,
): void {
    $user = $this->createUserWithProfile($actor);
    $board = $this->createBoardFixture($resource);
    $client = $this->createClient();
    $this->logInAs($user); // anonymous fixture → no login

    $client->request($this->methodFor($action), $this->uriFor($action, $board));

    $this->assertSame($expectedStatus, $client->getResponse()->getStatusCode());
}

public static function boardAccessMatrix(): iterable
{
    yield 'owner reads own board'            => ['owner', 'read', 'own_board', 200];
    yield 'owner deletes own board'          => ['owner', 'delete', 'own_board', 200];
    yield 'member updates colleague board'   => ['member_same_org', 'update', 'colleague_board_same_org', 200];
    yield 'member CANNOT delete colleague board' => ['member_same_org', 'delete', 'colleague_board_same_org', 403];
    yield 'org admin deletes colleague board'    => ['org_admin_same_org', 'delete', 'colleague_board_same_org', 200];
    yield 'outsider CANNOT read other org board' => ['member_other_org', 'read', 'board_other_org', 404];
    yield 'outsider CANNOT delete other org board' => ['member_other_org', 'delete', 'board_other_org', 404];
    yield 'platform operator reads other org board' => ['platform_operator', 'read', 'board_other_org', 200];
    yield 'anonymous CANNOT read'            => ['anonymous', 'read', 'own_board', 401];
    yield 'nobody reads a deleted board'     => ['owner', 'read', 'nonexistent_board', 404];
    // ... one yield per cell of the matrix
}
```

#### Horizontal privilege escalation, explicitly

OWASP calls it out by name: **horizontal privilege elevation — accessing another user's resources — is the most common authorization weakness** an authenticated user can exploit. [[3]](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html) The matrix above encodes it, but add two escalation-shaped tests that the matrix's regularity can hide:

1. **ID substitution on every ID-accepting route.** For each endpoint taking `{id}`, log in as user A, then request A's resource ID ± 1 (or a known resource ID belonging to user B in the *same* org — the nastier case, since org membership makes the ID "plausible"). Expect 404/403 per the matrix. This is the test that catches the unscoped `find()` of §6.6.
2. **Privilege creep on org boundaries.** A user who was *removed* from an org can still read the board for one request if membership is cached — test the *revocation* path, not just the grant path. It is easier to grant than to take away; your tests should verify the taking-away.

Two final disciplines, both from the cheat sheet [[3]](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html):

- **Test the configuration, don't assume it.** Framework defaults and third-party components change; a "deny by default" you inherited from a library is a deny by default only until the next upgrade. Encode the default in your own config and test it.
- **Review periodically.** After deployment, re-run the matrix against the *live* permission set to catch privilege creep — privileges that quietly exceeded the design-phase intent.

---

### 6.10 Key takeaways

1. **Roles are an attribute, not a model.** RBAC gets you to v1 fast; role soup is what you buy when permissions start depending on *which* resource. OWASP's preference for ABAC/ReAC over role-only designs is a warning about that trajectory. [[3]](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
2. **Each model owns a slice of the board:** org membership is ReBAC (relationship paths), "admin" is an ABAC attribute on the membership, business-hours/device-posture are ABAC environment conditions, and a global platform-operator role is the one place plain RBAC is right.
3. **IDOR lives in the query.** Controller checks can be correct and still lose; scope your reads so unauthorized rows are invisible, and keep per-action checks for writes. Defense in depth across gateway → controller → domain service → query, with **deny by default** at every layer.
4. **Voters are enforcement; policies are decisions.** Start with Symfony Voters and expressions, extract a pure `BoardPolicy::can(user, resource, action)` the moment logic is shared or environmental, and keep the Voter as a thin adapter.
5. **The UI is UX.** `v-if="can(...)"` is progressive disclosure in the attacker's browser. Every read and every mutation is re-checked server-side, on every request.
6. **Test the matrix, not the happy path.** Actor × resource × action, with 401/403/404 asserted *exactly* — and horizontal privilege escalation (ID substitution, same-org neighbors, and the revocation path) as first-class test cases.

## Chapter 7. High-Level Web Security

Security is not a feature you add before launch; it is a set of properties your code has or doesn't have on every commit. This chapter treats the OWASP Top 10:2025 the way an engineering team actually uses it — as a checklist that maps to code review questions, CI jobs, and design decisions — and then goes deep on the specific failure modes a modern PHP + Vue stack hits in production.

The list this chapter works from (per the [OWASP Top 10:2025](https://owasp.org/Top10/2025/en/)):

1. A01:2025 — Broken Access Control
2. A02:2025 — Security Misconfiguration
3. A03:2025 — Software Supply Chain Failures
4. A04:2025 — Cryptographic Failures
5. A05:2025 — Injection
6. A06:2025 — Insecure Design
7. A07:2025 — Authentication Failures
8. A08:2025 — Software or Data Integrity Failures
9. A09:2025 — Security Logging and Alerting Failures
10. A10:2025 — Mishandling of Exceptional Conditions

Two things changed that matter for how you read the list. **SSRF was folded into A01 (Broken Access Control)** — OWASP's position is that a server fetching a URL on behalf of a user is an authorization decision, and we'll treat it that way in §7.2.6. And **A10 is new**: mishandling of exceptional conditions (fail-open behavior, unhandled states, error handling that leaks or misroutes) is now a first-class category rather than a footnote under other items.

### 7.1 The Top 10 as an engineering checklist

A poster version of the Top 10 tells you *what* is risky. A checklist version tells you *what to ask* in review, *what to run* in CI, and *what to verify* in design. Here is the translation.

| Category | The review question | The concrete check |
|---|---|---|
| **A01 Broken Access Control** | Does this route verify the *caller's* relationship to the *resource*, not just that they're logged in? | Every ID in a route is resolved and ownership/role-checked in one place (a voter or policy, not scattered `if`s). IDOR test: swap the ID in a request for another user's. Any outbound URL fetch is allowlisted (§7.2.6). |
| **A02 Security Misconfiguration** | If this app is deployed with defaults, is it safe? | No `debug=true` in prod, no default credentials, no directory listing, HSTS on, framework security middleware enabled, secrets from the environment not the repo, containers run non-root. |
| **A03 Supply Chain Failures** | What code runs that we didn't write, and can we prove what it was? | Lockfiles committed and built from; `composer audit` / `npm audit` in CI with a failure policy; CI artifacts signed; container images pinned by digest (§7.2.7). |
| **A04 Cryptographic Failures** | Where do we store or transmit secrets, and are the algorithms current? | TLS everywhere; bcrypt/argon2 for passwords (never MD5/SHA1, never home-rolled); no secrets in logs, URLs, or client bundles; random tokens from CSPRNGs only. |
| **A05 Injection** | Does any untrusted string reach a query, shell, template, or HTML sink without parameterization or encoding? | Parameterized queries are the floor, including in ORM-adjacent code (§7.2.4); all user data encoded for its output context (§7.2.5); no `eval`-family functions on input. |
| **A06 Insecure Design** | Is the *design* of this feature safe, assuming every library works correctly? | Threat model written before the endpoint exists (§7.3.2); rate limits and business-rule invariants (e.g., "a coupon applies once") enforced server-side, not in the UI. |
| **A07 Authentication Failures** | Can someone become someone else, or stay logged in too long? | Password policy + breach-list check; MFA where it matters; session revocation on password change/logout; lockout or rate limiting on login; no account enumeration in error messages. |
| **A08 Software or Data Integrity Failures** | Do we trust what we deserialize, load, or upgrade without verifying it? | No unsafe deserialization of untrusted data; webhooks verified (signature check); migrations and deploys verified; client-side code integrity (SRI) where third-party scripts are loaded. |
| **A09 Security Logging & Alerting Failures** | If an attack happened tonight, would we know by morning — and could we reconstruct what happened? | Auth failures, authz denials, and admin actions logged with actor + target; logs shipped somewhere the app can't tamper with; alerts wired to a human, because "great logging with no alerting is of minimal value" (OWASP's words). |
| **A10 Mishandling of Exceptional Conditions** | When something unexpected happens, does the system fail *closed* and stay quiet? | Authorization errors deny, not allow; no stack traces to clients; 401/403/404 used correctly (§7.2.8); timeouts and resource limits on every external call. |

Notice the shape of the checklist: most items are **decisions made in design and review**, not detections made by a scanner. Scanners find symptoms; the checklist targets root causes. That's also why OWASP 2025 deliberately groups CWEs by root cause — "injection" and "cryptographic failure" are causes; "sensitive data exposure" is what you see in the incident report.

### 7.2 Deep dives the stack actually hits

#### 7.2.1 Same-Origin Policy: what the browser actually isolates

The Same-Origin Policy (SOP) is the browser's foundational isolation mechanism, and understanding exactly what it does — and doesn't — do explains where CSRF, XSS, and CORS all come from.

**Origin** is the triple `(scheme, host, port)`. `https://api.example.com` and `http://api.example.com` are different origins. `https://api.example.com` and `https://api.example.com:8443` are different origins. `https://a.example.com` and `https://b.example.com` are different origins.

What the SOP actually isolates:

- **DOM access.** A script on origin A cannot read or modify the DOM of a cross-origin frame or document. This is the wall that keeps a malicious page from reaching into your embedded checkout iframe.
- **Read access to fetch/XHR responses.** A page on origin A can *send* a cross-origin request to origin B, but if B hasn't granted CORS permission, the browser blocks the page from *reading the response*. The request may still have executed on the server — which is precisely why CORS is a read gate, not a request gate.
- **Storage APIs.** `localStorage` and `sessionStorage` are partitioned per origin. A script on `evil.com` cannot read `example.com`'s storage.
- **WebSockets and other same-origin-only APIs** follow the same origin rules.

What the SOP does *not* isolate, historically or by design:

- **Network visibility.** The SOP is enforced in the browser, not on the wire. A network observer, a corporate proxy, or a malicious dependency on your *own* page sees everything. Never treat "the browser can't read it" as "no one can."
- **Sending requests.** Form submissions, `fetch` without a need to read the response, and image/script loads cross origins freely. This is the loophole CSRF exploits: the browser will happily send your session cookie to your own API from a page the attacker controls.
- **Cookies, strictly speaking.** Cookies are scoped by *domain and path*, not by full origin — a cookie set for `.example.com` is sent to every subdomain, and (absent `Secure`) over both HTTP and HTTPS. The `SameSite`, `Secure`, and `HttpOnly` attributes (§7.2.3) are how you close the gaps between "domain-scoped" and "origin-scoped."

The practical upshot for a web team: the SOP is a **browser-side sandbox for JavaScript**, not a security boundary for your API. Your API must authenticate and authorize every request as if the SOP doesn't exist, because for non-browser clients it doesn't.

#### 7.2.2 CORS: simple vs. preflight, credentials, and why `*` + cookies is forbidden

CORS (Cross-Origin Resource Sharing) is the mechanism by which a server *opts in* to letting a cross-origin page read its responses. It is implemented by the browser, driven by headers from the server, and it exists to let your Vue frontend on `app.example.com` talk to your API on `api.example.com` while still honoring the SOP.

**Simple requests.** A request is "simple" if it's `GET`, `HEAD`, or `POST`, uses only safelisted headers, and has a `Content-Type` of `text/plain`, `multipart/form-data`, or `application/x-www-form-urlencoded`. The browser sends it immediately and then checks the response for `Access-Control-Allow-Origin`. If it's missing or doesn't match, the response is delivered to the server but the page can't read it.

**Preflighted requests.** Anything else — `PUT`, `DELETE`, `PATCH`, a JSON content type, or *any custom header* — triggers a preflight: the browser first sends an `OPTIONS` request asking "if I send this, will you let me read the answer?" The server must answer with `Access-Control-Allow-Origin`, `Access-Control-Allow-Methods`, and `Access-Control-Allow-Headers` before the real request goes out.

This has a security property you should know about: **a custom request header forces a preflight, and a preflight that fails blocks the request entirely.** Cross-origin attackers can't send your API a request with an `X-CSRF-Token` header they don't know — the preflight fails first. This is the backbone of the SPA CSRF strategy in §7.2.3.

**Credentialed requests.** When the frontend sends cookies or an `Authorization` header (`fetch(url, { credentials: 'include' })`), the rules tighten:

- `Access-Control-Allow-Origin: *` is **not permitted** on credentialed responses. The browser rejects it. The server must echo the specific origin: `Access-Control-Allow-Origin: https://app.example.com`.
- The server must also send `Access-Control-Allow-Credentials: true`.

This is why "**never `*` + cookies**" is a rule rather than a suggestion: with cookies in play, a wildcard origin would let *any* site on the internet read your authenticated responses if the browser allowed it — and the spec doesn't, precisely because it would be catastrophic.

**Explicit allowlists, always.** The production pattern:

```php
// CORS service: the allowlist lives in configuration, not in code paths
class CorsService
{
    public function __construct(private array $allowedOrigins) {}

    public function headersFor(?string $origin): array
    {
        if ($origin === null || !in_array($origin, $this->allowedOrigins, true)) {
            return []; // no CORS headers at all: cross-origin pages get nothing
        }

        return [
            'Access-Control-Allow-Origin'      => $origin,
            'Access-Control-Allow-Credentials' => 'true',
            'Vary'                             => 'Origin',
        ];
    }
}
```

Two details that matter in production:

1. **`Vary: Origin`** — if a CDN or cache sits in front of the API, it must cache separate responses per origin, or origin A's page could read origin B's cached response. This is a real, recurring cache-poisoning bug.
2. **The allowlist is the security control.** The check belongs in one server-side place, driven by configuration. "Allow the origin if it ends with `example.com`" is a classic allowlist bug: `evil-example.com` also ends with `example.com`. Compare full origins, exactly.

And the framing that keeps CORS honest: **CORS protects browser users from malicious *other* sites. It is not authentication.** A script running on your own compromised page, a curl from an attacker's laptop, or a malicious dependency all bypass it entirely. CORS is the SOP's permission slip, nothing more.

#### 7.2.3 CSRF: cookie semantics and the two token strategies

Cross-Site Request Forgery works because of a quirk the browser never fixed: **cookies are sent automatically with requests to the domain that set them, regardless of which page initiated the request.** If a logged-in user visits `evil.com` and that page contains:

```html
<form action="https://api.example.com/transfer" method="POST">
  <input name="amount" value="1000">
  <input name="to" value="attacker">
</form>
<script>document.querySelector('form').submit()</script>
```

the user's browser will submit the form *with their session cookie*. The server sees a perfectly authenticated POST from a perfectly real user. The user did not choose to do this.

Note what the attack relies on: the attacker can *cause* a request, but (thanks to the SOP) cannot *read* your responses or *read* your cookies. Every defense exploits that asymmetry.

**The cookie attributes are your first line:**

- **`SameSite`** controls whether the cookie is sent on cross-site requests at all.
  - `Lax` (the default in modern browsers): the cookie is *not* sent on cross-site subresource requests or cross-site POSTs, but *is* sent on top-level cross-site GET navigations (clicking a link). This alone breaks the classic form-POST CSRF.
  - `Strict`: never sent cross-site, even for top-level navigation. Stronger; occasionally breaks legitimate "click a link in an email" flows.
  - `None`: always sent; **requires `Secure`**. Only use this when you genuinely need cross-site credentialed reads (and then you *must* have CORS configured for it deliberately).
- **`Secure`**: cookie only travels over HTTPS. Non-negotiable in production; it also prevents the cookie from being captured in plaintext on a network.
- **`HttpOnly`**: the cookie is invisible to JavaScript. This does not stop CSRF by itself (the browser sends the cookie whether JS can see it or not), but it stops the *other* half of the picture: an XSS payload stealing the session cookie. Set it unless you have a specific reason not to.

The production baseline for a session cookie: `HttpOnly; Secure; SameSite=Lax; Path=/`.

**But `SameSite=Lax` is a browser default, not a contract.** Older browsers, embedded webviews, and API clients don't all honor it the same way. Defense in depth adds a token the attacker can't forge:

**Synchronizer token.** The server generates an unpredictable token, ties it to the session (or to a form), and requires it on state-changing requests:

```php
// generating: store per session, expose to the frontend once
$token = bin2hex(random_bytes(32));
$_SESSION['csrf'] = $token;

// validating: on every POST/PUT/PATCH/DELETE
if (!hash_equals($_SESSION['csrf'], $request->headers->get('X-CSRF-Token'))) {
    throw new AccessDeniedException('Invalid CSRF token');
}
```

The attacker's page can't know the token (SOP blocks reading your session), so the request is rejected. Use `hash_equals` for constant-time comparison; use `random_bytes` for generation.

**Double-submit cookie.** The server sets a random value in a cookie that is *not* `HttpOnly`, and the client echoes that value back in a header on state-changing requests. The server compares the two. The attacker can cause the browser to send the cookie, but can't read it to fill in the header — so the comparison fails. A subtle variant worth knowing: in an SPA, requiring a custom header like `X-Requested-With` on all state-changing routes achieves the same effect, because the custom header forces a CORS preflight that cross-origin forgeries can't pass.

**The SPA decision: bearer token vs. cookie session.** This is where the architecture choice *is* the CSRF answer:

- **Bearer token in `Authorization: Bearer …`** (typical for a token-authed SPA): the browser never attaches it automatically, so there is no CSRF vector by construction. The trade moves to XSS — a token in `localStorage` is readable by any script that reaches your page, so your XSS hygiene (§7.2.5) becomes the token's protection. Keep token lifetimes short, support revocation, and treat the token like a password.
- **Cookie session in an SPA** (typical when the API and frontend share a domain or use credentialed CORS): you inherit the full CSRF problem. The working combination is `SameSite=Lax` (or `Strict` where flows allow) **plus** a synchronizer token on every state-changing route. Don't pick one and call it done.

A useful rule of thumb: **state-changing endpoints require a token or a preflight-forcing header; read-only GETs with `SameSite=Lax` are acceptable without one.** And because CSRF is an authorization-shaped problem, log rejected tokens with actor and target — that's A09 doing its job.

#### 7.2.4 SQL injection: parameterized queries are the floor

SQL injection is the canonical injection bug, and the fix is the oldest one in the book: **separate the query structure from the data.** Parameterized queries (prepared statements) send the SQL and the values to the database as separate things; the values are never parsed as SQL.

```php
// Wrong: interpolation. The input *is* the query.
$sql = "SELECT * FROM users WHERE email = '$email'";

// Right: the input is a value. It cannot become syntax.
$stmt = $conn->prepare('SELECT * FROM users WHERE email = :email');
$stmt->execute(['email' => $email]);
```

With Doctrine, the same principle:

```php
// Safe: named parameter
$users = $em->createQuery(
    'SELECT u FROM App\Entity\User u WHERE u.email = :email'
)->setParameter('email', $email)->getResult();
```

**The floor, not the ceiling.** "We use an ORM, so we're safe" is the most expensive false belief in this chapter, because ORMs hand you a knife:

```php
// DQL concatenation: the ORM is not magic
$qb->select('u')
   ->from(User::class, 'u')
   ->orderBy('u.' . $sortBy, $sortDir);   // ← $sortBy is attacker-controlled
```

`$sortBy = 'email'` is fine. `$sortBy = 'email; DROP TABLE users --'` (or, in DQL, a crafted expression that errors out your schema or leaks data) is not. **Identifiers — table names, column names, sort keys, join fragments — cannot be bound as parameters.** They are part of the query's *structure*, and structure must come from a whitelist:

```php
$allowedSorts = ['email' => 'u.email', 'createdAt' => 'u.createdAt'];
if (!isset($allowedSorts[$sortBy])) {
    throw new BadRequestException('Invalid sort field');
}
$qb->orderBy($allowedSorts[$sortBy], $sortDir === 'DESC' ? 'DESC' : 'ASC');
```

The pattern generalizes: **values bind, identifiers whitelist, everything else doesn't enter the query at all.** The same logic applies to raw `DBAL` calls, to `executeQuery` with string-built SQL, and — don't skip this — to anything that reaches a shell, a template engine, or a NoSQL query builder with the same "I'll just interpolate" instinct.

Defense in depth still applies: the web app's database user should have the minimum privileges it needs (a read-heavy feature doesn't need `DROP`), and query errors must surface as generic failures, not as database diagnostics (§7.2.8). But the *primary* control is the floor: if you can point to a code path where untrusted input shapes query structure, you have an injection bug regardless of your ORM.

#### 7.2.5 XSS: context-aware encoding, and `v-html` as a reviewed exception

Cross-Site Scripting succeeds when untrusted data reaches the browser and is interpreted as *something more than text* — as HTML, as JavaScript, as a URL. The fix is encoding, and the crucial insight is that **encoding is context-specific**: the same string needs different treatment in an HTML body, an HTML attribute, a JavaScript string, a CSS value, or a URL.

```
Hello <script>alert(1)</script>

HTML body:      Hello &lt;script&gt;alert(1)&lt;/script&gt;
HTML attribute: same, plus quotes encoded: &quot;
JS string:      'Hello \x3cscript\x3ealert(1)\x3c/script\x3e'
URL:            Hello%20%3Cscript%3Ealert(1)%3C%2Fscript%3E
```

Getting the context wrong — HTML-encoding a value that lands inside a `<script>` block, for instance — produces a broken page *and* a vulnerability, because the script context has its own escape rules. This is why "just escape everything" is a slogan, not a strategy: the strategy is **knowing the sink and using the encoder for that sink.**

In a Vue frontend, the framework does the default case for you, and that's the point of using a framework:

```vue
<!-- Safe: Vue escapes the value as HTML text -->
<p>{{ userComment }}</p>

<!-- DANGER: raw HTML injection, no escaping, no sanitization -->
<div v-html="userComment"></div>
```

`{{ }}` interpolation escapes HTML entities automatically. `v-html` deliberately does not — it renders the string as live markup. That makes `v-html` a **reviewed exception**, and the review discipline is:

1. **Default is no.** New `v-html` in a PR asks "why?"
2. **Trusted content, documented.** Rendering server-generated, already-sanitized HTML (a reviewed rich-text editor output, for example) is defensible — with a comment naming the trust boundary.
3. **Untrusted content gets sanitized.** If the HTML comes from users, run it through a sanitizer (e.g., DOMPurify) *before* it reaches `v-html`, with an explicit allowlist of tags and attributes.
4. **Never** `v-html` on anything shaped by a URL parameter, an API passthrough, or a third-party feed without (3).

The backend has the same obligation: if your API returns HTML fragments, *it* is responsible for encoding or sanitizing them, because other clients will consume the same endpoint. And one backstop worth deploying regardless — a Content-Security-Policy with a locked-down `script-src` (no `unsafe-inline`, no `unsafe-eval`) turns a successful XSS from "attacker runs arbitrary JavaScript" into "attacker's script never executes." CSP is a backstop, not a substitute for encoding, but it is the difference between an incident and a near-miss.

#### 7.2.6 SSRF, folded into Broken Access Control: outbound fetch is an authorization problem

In 2025, OWASP moved Server-Side Request Forgery into A01. The reasoning is worth internalizing because it changes how you review code: **when your server fetches a URL that a user influenced, the server is acting on the user's behalf — so the question is not "is this URL safe?" but "is this user *authorized* to make the server reach that destination?"**

The attack surface is anywhere input becomes an outbound request: image proxies, "import from URL," webhook test buttons, preview endpoints, URL shorteners, feed importers. The danger is that your server sits inside your network with privileges your users don't have:

- **Cloud metadata services.** `http://169.254.169.254/latest/meta-data/iam/security-credentials/` hands you the instance's IAM credentials on AWS, GCP, and Azure. One SSRF and the attacker owns your cloud identity.
- **Internal services.** Intranet admin panels, CI systems, message queues, and other hosts that are firewalled from the internet but not from your app tier.
- **Reconnaissance.** Even without a crown jewel, SSRF lets an attacker port-scan your internal network through your server's responses and timings.

The mitigations, in order of importance:

1. **Allowlist destinations, don't blocklist sources.** "Only fetch from `images.trusted-cdn.com`" beats "block private IPs" every time, because blocklists have holes (IPv6, decimal-encoded IPs, DNS rebinding, `file://` and `gopher://` schemes). If the feature genuinely needs arbitrary destinations, the allowlist becomes a *scheme + domain* policy, and everything else is denied.
2. **Treat the check as authorization, and enforce it where authorization lives.** The "may this user cause the server to fetch this host?" decision belongs in the same policy layer as every other authz check — so it's tested, logged (A09), and denied on error (A10).
3. **Resolve, then pin.** DNS rebinding: the attacker's DNS server answers your first lookup with a public IP (passes the check) and the second with `127.0.0.1` (connects internally). Defend by resolving once and connecting to the resolved IP (with SNI set for TLS), or by re-validating the IP after resolution and rejecting private, loopback, and link-local ranges: `10/8`, `172.16/12`, `192.168/16`, `127/8`, `169.254/16`, `::1`, `fc00::/7`, `fe80::/10`.
4. **Restrict schemes and bound the request.** HTTP/HTTPS only; timeouts on connect and read; cap response sizes; no redirects to non-allowlisted hosts (follow redirects only after re-checking each hop).
5. **Network-layer egress controls** as the backstop: if the app tier can't reach the metadata IP or the internal management network at all, SSRF degrades from "game over" to "inconvenience."

The review question for any endpoint that accepts a URL: *what is the set of hosts this user is authorized to make us touch, and where in the code is that set enforced?* If the answer is "everywhere," the endpoint is a broken access control.

#### 7.2.7 Supply chain: lockfiles, audits, and pins

A03 is the 2025 expansion of the old "vulnerable components" item into the whole ecosystem: your dependencies, their dependencies, your build system, your CI, your distribution. The working assumption: **you will run code you did not write, you cannot read all of it, and one of it will eventually be compromised or broken.** The engineering response is to make the running set of code *known, pinned, and audited* at every step.

**Lockfiles are the contract.** `composer.lock` and `package-lock.json` record the exact resolved dependency tree. The rules:

- **Commit them.** A lockfile in `.gitignore` means "production runs whatever resolves today" — a moving target you never reviewed.
- **Build from them, not from the manifest.** `composer install --no-dev` and `npm ci` install *exactly* what the lockfile says; `npm ci` even fails if the manifest and lockfile disagree. `npm install` in CI is how a "harmless" version bump sneaks into production.
- **Update deliberately.** Dependabot/Renovate open PRs against the lockfile; the PR *is* the review. Merging a dependency bump without reading what changed is how a malicious or broken package becomes your problem.

**Audit in CI, with a failure policy.** Both ecosystems ship auditors against public advisory databases:

```yaml
# CI job sketch
steps:
  - run: composer audit --format=json | audit-gate --fail-on=high
  - run: npm audit --audit-level=high
```

The part teams skip is the *policy*: `npm audit` exiting non-zero must actually fail the pipeline (or page someone), and "high/critical fails, medium pages a human" is a reasonable starting line. An audit that only writes to a log is a poster, not a control.

**Watch what *runs* on install.** `npm` executes `postinstall` scripts from dependencies by default; Composer can load plugins. A compromised package doesn't need a vulnerability in its code if it ships an install script. Keep an eye on which of your dependencies run lifecycle scripts, consider `--ignore-scripts` for third-party packages that don't need them, and treat a new install script in a dependency diff as a red flag.

**Pin and sign the artifacts you ship.**

- **Container images by digest, not tag.** `node:20` is a moving alias; `node@sha256:abcd…` is a specific image. Pin base images to digests in CI so "the same build" means the same build, and re-pin deliberately.
- **Sign CI artifacts.** Build outputs (deployables, release bundles) should carry signatures your deploy pipeline verifies, so a compromised runner or registry can't substitute a payload downstream.
- **Verify what you import.** This is the A08 side of the same coin: webhooks verified by signature, released packages checked before install, client-side third-party scripts loaded with Subresource Integrity.

The one-line version for code review: *can you point to the exact bytes of every dependency this build will run, and to the audit result for them?* If yes, you're doing supply-chain security. If the answer is "it resolves fine," you're doing hope.

#### 7.2.8 Exceptional conditions: fail closed, stay quiet, and use the right 4xx

A10 is the new category, and its core claim is that **abnormal states are security states.** Most vulnerabilities found in the wild under this heading are not exotic: the authorization service timed out and the code fell through to `allow`; the exception handler dumped a stack trace; the "user not found" path returned a different error than "wrong password"; the rate limiter's counter overflowed and wrapped to zero.

Three rules cover most of the category.

**1. Fail closed.** When a security-relevant check can't complete — the authz service errors, the token validation library throws, the risk-check API times out — the only correct answer is *deny*. "Fail open" (let the request through, log a warning, deal with it later) converts every upstream flake into an authorization bypass, and it converts *your own* bugs into bypasses. In code, this means the exception path of an authorization check is a rejection, not a fallback:

```php
try {
    $allowed = $policy->can($user, 'edit', $document);
} catch (Throwable) {
    $this->logger->error('authz check failed; denying', ['document_id' => $document->id]);
    throw new AccessDeniedException();   // fail closed
}
```

The same instinct applies to business invariants: a payment that is *both* `refunded` and `disputed` is an exceptional state; the system should freeze the record and alert, not pick a branch silently.

**2. Don't leak.** Production error responses carry a generic message and a reference ID; the stack trace, query, and environment details go to the log, where they're useful, not to the client, where they're a map. Concretely: framework debug mode off in production, a global exception handler that maps internal exceptions to safe responses, and no `display_errors`-style behavior at the web server layer. A stack trace that reveals your ORM version, table names, and framework is a head start for every other category in this chapter.

**3. Use 401/403/404 correctly — they are not interchangeable.**

- **401 Unauthorized** = "I don't know who you are." The client should authenticate (the response carries `WWW-Authenticate`). Use it for missing/invalid credentials.
- **403 Forbidden** = "I know who you are, and you may not." Use it for authenticated callers who lack permission.
- **404 Not Found** = "This doesn't exist" — or, deliberately, "I will not tell you whether this exists."

The classic mistakes: returning **401 for an authorization failure** (the client goes off to re-login, loops, or logs a misleading "session expired" — the user *was* authenticated; they just can't do this); and **returning 200-with-error or a distinct 403 vs. 404 that reveals resource existence** to users who shouldn't see the resource at all. For "can user A see user B's profile?", the correct behavior for a non-visible profile is the same response as a nonexistent one — usually 404 — so that enumeration is not possible. The same discipline applies to login ("invalid credentials," one message for both wrong-user and wrong-password) — that's A07 wearing A10's hat.

Tie it to A09: every denied request, every failed-closed fallback, and every 4xx burst is a log line worth alerting on. "Fail closed" without logging is just an outage; "fail closed" *with* alerting is a security control.

### 7.3 Resources the chapter teaches you to *use*

#### 7.3.1 The OWASP Cheat Sheet Series: first lookup, not blog posts

When you hit a specific problem — "how do I do CORS credentialed requests correctly," "what are the SameSite edge cases," "how do I validate a JWT in this framework" — the first stop should be the [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/), not a search-engine scroll. The reasons are practical:

- **They're organized by problem, and maintained as a series.** CORS, CSRF Prevention, XSS Prevention, SQL Injection Prevention, Session Management, Secure Headers, Authentication — each is a focused document that states the threat, the requirements, and concrete mitigations, and they cross-reference each other.
- **They're the upstream source.** Blog posts are usually someone's (sometimes dated, sometimes wrong) summary of a cheat sheet. Start at the source; read the blog if you want the opinions.
- **They're written for implementation.** A cheat sheet will tell you the exact headers, the exact attribute flags, and the failure modes to test — which is what you need when you're writing the code, not what you need when you're writing a postmortem.

The habit to build: **when a security decision comes up in design or review, the cheat sheet for that topic is linked in the ticket.** It makes the decision reviewable ("we followed the CORS sheet, section on credentialed requests") and catches the case where you *didn't* follow it and need to say why.

#### 7.3.2 Threat-model a single endpoint before you write it

Full-team threat modeling (STRIDE workshops, data-flow diagrams) has its place, but the practice that actually changes code is the small one: **before implementing an endpoint, spend ten minutes answering a fixed set of questions.** It's cheap, it's repeatable, and it maps 1:1 onto the Top 10, which is the point — the model *is* the checklist from §7.1 applied to one function.

The template:

| Question | Top 10 hook |
|---|---|
| What does this endpoint do, in one sentence? What are its inputs and outputs? | — (clarity) |
| Who may call it? (authentication) | A07 |
| What may each caller *do* with it? How do I verify the caller's relationship to the specific resource? (authorization) | A01 |
| Does it fetch anything outbound based on input? What destinations are allowed? | A01 (SSRF) |
| What data does it read or write? Who else can see it? Is any of it secret? | A04 |
| Which inputs reach a query, shell, template, or HTML sink? How are they parameterized or encoded, in which context? | A05 |
| What third-party code or data does it depend on? Is that dependency pinned and verified? | A03 / A08 |
| What are the business invariants? What happens if two requests race, or a value is in an impossible state? | A06 / A10 |
| When it fails — timeout, dependency down, bad state — what does it do? Deny or allow? What does the client see? | A10 |
| What does it log, and would an alert fire if someone abused it? | A09 |

Worked example, briefly — `POST /api/documents/{id}/share` (share a document with another user):

- *Authn:* authenticated user only. *Authz:* the caller must be the document's owner or have `edit` permission — checked against the resolved document, not the raw ID (IDOR is the A01 test: swap the ID).
- *Inputs:* `email` of the recipient. It reaches the DB (parameterized) and — if you send a notification email — a mail sink (template injection check). No outbound URL input, so no SSRF; if a future "share to a URL" field appears, this model forces the allowlist decision *now*.
- *Invariants:* sharing to yourself, sharing to a suspended account, re-sharing an already-shared document — each has a defined, logged outcome.
- *Failure:* permission-service timeout → deny (fail closed); error response is a generic 403 with a reference ID, no stack trace.
- *Logging:* every grant and denial logs actor, target document, and recipient; a burst of denials on one document alerts.

That's ten minutes of writing that would otherwise surface as an IDOR in production, a leaked stack trace, and a silent fail-open. Do it for every endpoint; keep the answers in the ticket so the next reviewer is checking a list, not rediscovering the threat model.

### 7.4 Chapter checklist

Before you close this chapter, you should be able to answer yes to each of these:

- [ ] I can state, for any route, where authentication and authorization are enforced — and the authz check resolves the *resource*, not just the session.
- [ ] My session cookie is `HttpOnly; Secure; SameSite=Lax` (or stricter), and state-changing endpoints additionally require a CSRF token or a preflight-forcing header.
- [ ] My CORS configuration is an explicit origin allowlist with `Vary: Origin`, credentials handled per the spec, and no wildcard anywhere near cookies.
- [ ] No untrusted string shapes a query's structure; values bind, identifiers whitelist.
- [ ] Every `v-html` (and every backend HTML fragment) has a documented trust boundary or a sanitizer in front of it.
- [ ] Every endpoint that fetches a URL from input has an explicit destination allowlist enforced in the authz layer.
- [ ] Lockfiles are committed and built from; `composer audit` / `npm audit` fail CI at a defined severity; images are pinned by digest.
- [ ] Security-relevant failures deny by default, return generic errors with reference IDs, and use 401/403/404 correctly.
- [ ] Denials, auth failures, and admin actions are logged with actor and target, and at least some of them alert a human.
- [ ] New endpoints get the ten-question model before the first line of the handler is written.

The OWASP Top 10 is not a list of things to fear; it's a list of questions to ask, ten times per feature. Ask them in review, encode them in CI, and the category you're most likely to see in an incident report is the one nobody asked about — so keep asking.

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
