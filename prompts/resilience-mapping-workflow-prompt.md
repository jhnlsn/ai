# Spring Boot API Resilience Mapping — Claude Code Workflow Prompt

A reusable prompt for Claude Code that maps the resilience posture of a Spring Boot API service from
its architecture and calling patterns: it inventories the resilience patterns the service **already
implements** (with their full client-facing contract), derives which patterns it **should** have
given its dependencies and traffic, and **proposes** the missing or useful ones for a human to
review. The result is written to `resilience-patterns.md` — the authority consumed by the **API
Coverage & Hardening** prompt (`api-coverage-workflow-prompt.md`) and by further analysis (chaos
planning, capacity planning, runbooks, architecture review). Triggers a dynamic workflow (the word
`workflow` is present), so it runs across parallel subagents.

**This prompt is analysis-only.** It never modifies application code or the build. It writes exactly
one file — `resilience-patterns.md` — and within it only the **tool-owned `PROPOSED` block**. Humans
own what becomes real: a pattern is in force only once a person promotes it into the `DECLARED` block.

Defaults assume **Spring Boot 3.x + Maven**; it reads the Stack Profile from `CLAUDE.md` (written by
the coverage prompt's Phase 0) and adapts to what it finds.

---

## Before you run

1. **Fill the placeholders** in the prompt: `<SCOPE>` (the service / module / route group) and
   `<BASE PACKAGE>`. Leave `<BASE PACKAGE>` blank and Phase 0 infers it.
2. **It's read-and-propose only — no code or build changes, no tests run, no Docker needed.** Safe to
   run on a clean *or* dirty tree; it touches only the `PROPOSED` block of `resilience-patterns.md`.
3. **Allowlist the read-only inspection commands** so the run doesn't stall:
   ```
   Bash(mvn dependency:tree:*)
   Bash(rg:*)
   Bash(grep:*)
   Bash(find:*)
   ```
4. **Re-run on architecture change.** Re-running regenerates the proposals against the current code
   and the current `DECLARED` set, so adopted patterns stop being re-proposed.
5. **Save it.** Once a run does what you want, `/workflows` → select it → press `s` to keep it as a
   `/command`.

### What it inspects

| Signal source        | Examples                                                                                 |
| -------------------- | ---------------------------------------------------------------------------------------- |
| Build dependencies   | Resilience4j, Spring Retry, Bucket4j, Spring Cloud Circuit Breaker, spring-kafka, ShedLock |
| Annotations          | `@CircuitBreaker` `@Retry` `@RateLimiter` `@Bulkhead` `@TimeLimiter` `@Retryable` `@Version` `@Cacheable` |
| Config keys          | `resilience4j.*`, client connect/read timeouts, `spring.datasource.hikari.*`, `server.tomcat.threads.*`, `spring.kafka.*` |
| Code shapes          | `RestClient`/`WebClient`/Feign clients, `@RestControllerAdvice` → 503/429 + `Retry-After`, outbox tables, idempotency-key filters, DLQ recoverers, routing datasources |
| Architecture inputs  | sync vs async, edge/mid/leaf, downstream deps (type/criticality/latency/trust), data store & write pattern, idempotency, fan-out, SLA/timeout budget, traffic shape, multi-tenancy, consistency, deployment model |

---

## The prompt

> Run a **workflow** to map the resilience patterns of the Spring Boot API service at `<SCOPE: service,
> module, or route group>` (base package `<BASE PACKAGE: e.g. com.acme.orders>`) and record them in
> `src/test/resources/_generated/resilience-patterns.md` for the coverage prompt and further analysis.
>
> **Goal:** produce an accurate, machine-consumable map of (a) the resilience patterns the service
> **already implements**, each with its full client-facing contract, and (b) a reviewed set of
> **proposed** patterns that are missing or worth considering given the architecture — so a human can
> adopt them. **Analysis only:** never modify application code or the build; the only file you write is
> `resilience-patterns.md`, and within it only the tool-owned `PROPOSED` block.
>
> **Hard rules:**
> - **`DECLARED` = human-owned; `PROPOSED` = tool-owned.** Read the `DECLARED` block to understand
>   what's already in force and to dedupe, but **never modify a single byte between `DECLARED:BEGIN`
>   and `DECLARED:END`**. Regenerate only the `PROPOSED` block, wholesale. If the markers are missing
>   from an existing file, **abort** rather than risk overwriting curated content.
> - **No phantom patterns.** Never report a pattern as implemented without code/config evidence — cite
>   the file:line, annotation, dependency, or config key. Absence is also a finding (a `RestClient` with
>   no timeout, a read-modify-write with no `@Version`, a POST with no idempotency key).
> - **Nothing is auto-adopted.** Everything you detect that isn't already declared, and everything you
>   recommend, goes in `PROPOSED` — tagged `implemented-undeclared` (in code but not yet curated) or
>   `recommended-absent` (not present, advised). A human promotes a proposal by moving its YAML into
>   `DECLARED`.
>
> **Phases:**
>
> 0. **Discover stack & architecture.** Read the Stack Profile from `CLAUDE.md`. Gather the
>    architecture/calling-pattern inputs that drive pattern selection: communication style (sync /
>    async-event-driven / streaming); position in the call graph (edge / mid-tier / leaf); each
>    downstream dependency (type, criticality hard-vs-soft, p50/p99 latency, in-house vs third-party);
>    data store and write pattern (read-only / read-modify-write / multi-write); per-endpoint operation
>    semantics and idempotency; fan-out / aggregation; latency SLA and timeout budget; traffic shape and
>    multi-tenancy; consistency and ordering needs; deployment model and DB migrations. Record this as an
>    inputs profile. Locate `resilience-patterns.md`; if absent, create it with the banner and the
>    `DECLARED:BEGIN/END` + `PROPOSED:BEGIN/END` markers (empty `DECLARED`).
>
> 1. **Inventory implemented patterns.** For each pattern — request timeouts, retries + backoff/jitter,
>    retry budget, circuit breaker, bulkhead, rate limiting, load shedding, fallback / graceful
>    degradation, cache-aside / serve-stale, store-and-forward, transactional outbox, idempotency keys,
>    read-only / maintenance mode, backpressure, dead-letter queue, saga, optimistic lock, failover /
>    replica reads, async + eventual consistency, health/readiness — determine present vs. absent from
>    the Spring signals (dependencies, annotations, config keys, code shapes), citing evidence. For each
>    present pattern capture its **client-facing contract**: trigger, HTTP status, headers
>    (`Retry-After`, `RateLimit-*`, `ETag`…), the caller's safe/idempotent-retry semantics, server
>    behavior, and recovery.
>
> 2. **Map failure modes per endpoint and dependency.** For every endpoint in scope and every downstream
>    call, enumerate what can fail (timeout, 5xx, slow, malformed, unavailable, conflict) and how it
>    currently degrades — a defined contract (e.g. 503 + `Retry-After`) vs. an **unhandled 5xx** (a
>    defect candidate). Record the observable HTTP contract for each.
>
> 3. **Derive applicable patterns.** Apply the architecture → pattern mapping to the inputs profile to
>    determine which patterns the service *should* have, and why. Honor the interactions and conflicts:
>    idempotency keys are a prerequisite before any retry pattern; retries need a circuit breaker + retry
>    budget + backoff/jitter to avoid storms; `attempts × per-try timeout` must fit the parent deadline;
>    a breaker needs a fallback or a fast typed error behind it; caches/replica reads violate strong
>    consistency; load-shedding responses must carry `Retry-After`; deadline propagation vs. local
>    retries.
>
> 4. **Gap analysis & proposals.** Diff implemented against applicable. For each gap or improvement build
>    a `PROPOSED` entry: `state` (`implemented-undeclared` | `recommended-absent`), the gap, the failure
>    it mitigates, cited evidence, a `suggested_contract` that mirrors the declared schema (so promotion
>    is copy-paste), trade-offs, effort, and priority. Do not adopt anything.
>
> 5. **Emit `resilience-patterns.md`.** Regenerate **only** the bytes between `PROPOSED:BEGIN` and
>    `PROPOSED:END`, wholesale, with a `generated-by` + run-date comment. Dedupe against `DECLARED`
>    (never propose a pattern whose `category` + `applies_to` already matches a declared `id`). Leave the
>    banner and the entire `DECLARED` block untouched.
>
> **Entry schemas** (each entry carries a fenced ```yaml block as the machine-readable source of truth;
> surrounding prose is for humans):
> - **Declared:** `id` (stable), `name`, `category` (circuit-breaker | rate-limit | bulkhead | timeout |
>   retry | fallback | cache-fallback | shed-load | backpressure | store-and-forward | outbox |
>   idempotency | optimistic-lock | read-only | dlq | saga | failover | async-eventual |
>   health-readiness | hedged | coalescing), `status: declared`, `applies_to{endpoints[], dependencies[]}`,
>   `trigger{condition, detector}`, `response{http_status, body_contract, headers{}}`,
>   `caller_contract{retryable, safe_methods[], unsafe_without_idem_key[], backoff}`,
>   `server_behavior{mode, side_effects}`, `recovery{exit_condition, exit_state, max_duration}`,
>   `observability{metric, alert}`, `owner`, `last_reviewed`.
> - **Proposed:** `proposal_id`, `suggested_category`, `status: proposed`, `state`, `gap`,
>   `failure_mitigated`, `evidence[]` (file:line / annotation / config key), `suggested_contract{…}`
>   (mirrors the declared fields), `trade_offs`, `effort`, `priority`.
>
> **Definition of done:**
> - Every endpoint in scope and every downstream dependency has its failure modes mapped with the
>   observable HTTP contract, or marked `unhandled — defect candidate`.
> - Every implemented resilience mechanism is captured with cited evidence and its full contract.
> - Every gap/improvement from the mapping is a `PROPOSED` entry with all required fields and a
>   promotion-ready `suggested_contract`.
> - The output parses against the schema above; the `DECLARED` block is byte-for-byte unchanged and only
>   `PROPOSED` was regenerated; if the markers were missing the run aborted instead of overwriting.
> - No application code or build file was changed and nothing was auto-adopted.
>
> **Deliverables:** the updated `resilience-patterns.md` (regenerated `PROPOSED` block); the
> inputs/architecture profile; an **implemented-vs-applicable coverage matrix** (dependency × pattern,
> exposing dependencies with no resilience pattern and contracts that disagree across endpoints); and a
> prioritized proposal list for human review.
>
> **Model routing:** use a smaller model for the signal inventory and detection scaffolding; use the
> strongest model for the architecture mapping, the interaction/conflict analysis, and proposal design.

---

## Why each piece is there

- **Evidence-based inventory, not vibes.** Citing the file/annotation/config key for every "present"
  finding — and naming the absence signal for every gap — is what keeps the map trustworthy enough for
  downstream tools to act on.
- **`DECLARED` is human-owned, `PROPOSED` is tool-owned.** Resilience contracts encode intent (which
  retries are safe, what `Retry-After` to promise, what counts as recovered). The tool can *detect* and
  *recommend*, but a person owns the contract — so the workflow regenerates only the proposed block and
  never clobbers a curated declaration.
- **Patterns follow architecture.** A pattern isn't universally right; it's implied by calling patterns
  (a remote sync dependency → timeout + breaker; a non-idempotent POST exposed to retrying clients →
  idempotency key; a migration window → read-only + store-and-forward). Mapping from inputs is what makes
  the proposals specific instead of a generic checklist.
- **Interactions are where good patterns combine badly.** Idempotency before retries, retries + breaker +
  budget, timeout/retry co-budgeting — surfacing these prevents proposing a "fix" that becomes a retry
  storm.
- **Analysis-only, human-adoption gate.** Adding resilience changes the API's contract and operational
  behavior; that's a design decision. The prompt informs it (proposals) without making it (no code
  changes, no auto-declare).
- **One authoritative map, many consumers.** `resilience-patterns.md` is the single source the coverage
  prompt tests against and that chaos planning, capacity planning, runbooks, and architecture review all
  read — so the map is built once and reused, not re-derived per tool.

## How this links to the other prompts

```
resilience-mapping-workflow-prompt.md   ──writes PROPOSED──▶   resilience-patterns.md   ──read as authority──▶   api-coverage-workflow-prompt.md
        (producer, this file)                                  (human owns DECLARED)                              (consumer: tests DECLARED contracts;
                                                                                                                   any undeclared 5xx = defect)
                                                                       │
                                                                       └─ also feeds: chaos/failure-injection planning · capacity & load planning ·
                                                                          operational runbooks (from observability/recovery fields) · architecture review
```

- **Producer (this prompt)** owns only the `PROPOSED` block and names the consumer in the file banner.
- **Consumer (`api-coverage-workflow-prompt.md`)** parses the YAML inside `DECLARED` only, ignores
  `PROPOSED`, and treats any observed 5xx that matches no declared trigger as a defect to surface.
- **Promotion is human and one-directional:** a person cuts a proposal's YAML into `DECLARED`, assigns a
  stable `id`, sets `status: declared` + `owner` + `last_reviewed`, and deletes the proposal. The next
  run sees it's declared and stops re-proposing it.
- **Schema version** lives in the banner so a schema bump forces both prompts to update in lockstep.
