# Spring Boot API Coverage & Hardening — Claude Code Workflow Prompt

A reusable prompt for Claude Code that maps a Spring Boot REST capability's case space — including
the automatic 4xx error surface the framework hands you, the failure behavior of its dependencies,
and its cross-endpoint consistency — builds JUnit 5 + jqwik + MockMvc/Testcontainers tests to prove
every input has a defined, *intended* outcome and the capability has no trap states, measures with
JaCoCo and PIT, hardens the code to close gaps **without breaking existing clients**, and verifies
its own numbers from a clean tree before reporting done. Triggers a dynamic workflow (the word
`workflow` is present), so the orchestration runs across parallel subagents with adversarial
verification.

Defaults assume **Spring Boot 3.x + Spring MVC** (Servlet stack, MockMvc) and **Maven (`mvn`)**.
You don't have to trust those defaults: **Phase 0 discovers the actual stack from the repo and
records a Stack Profile in `CLAUDE.md`**, then adapts commands and annotations — MockMvc →
`WebTestClient` for WebFlux, `@MockitoBean` → `@MockBean` for Boot < 3.4, `mvn` → `./gradlew` for
Gradle. Every later run reads that profile instead of re-discovering.

---

## Before you run

1. **Fill the placeholders** in the prompt: `<SCOPE>`, `<BASE PACKAGE>` (for PIT scoping), and the
   two thresholds. Leave `<BASE PACKAGE>` blank and Phase 0 infers it from the repo. The toolchain
   below is the default; Phase 0 reconciles it against the actual build and records the result.
2. **This workflow works on a branch and never auto-commits.** It edits production code, the build
   file, and `CLAUDE.md`. It creates a dedicated branch, leaves all changes uncommitted, and stops
   for human review of the full diff. Do not point it at a dirty working tree. A running **Docker
   daemon is required** — Testcontainers-backed slices are mandatory, so a run with no reachable
   Docker reports FAILED at the Phase 0 baseline rather than a lower-confidence pass.
3. **Allowlist the Maven + git commands first.** A workflow only pauses on permission prompts, so any
   command not in your allowlist stalls the run mid-way. Add these before starting:
   ```
   Bash(mvn clean test:*)
   Bash(mvn test:*)
   Bash(mvn test-compile:*)
   Bash(mvn jacoco:report:*)
   Bash(mvn org.pitest:pitest-maven:mutationCoverage:*)
   Bash(git checkout -b:*)
   Bash(git status:*)
   Bash(git diff:*)
   Bash(git stash:*)
   Bash(git worktree add:*)
   Bash(git worktree remove:*)
   Bash(pact-broker can-i-deploy:*)
   ```
4. **Missing plugins/deps are fixable mid-run — carefully.** If `pom.xml` lacks `jacoco-maven-plugin`,
   `pitest-maven` + `pitest-junit5-plugin`, `jqwik`, `spring-security-test`, Testcontainers,
   WireMock, or `swagger-request-validator`, Phase 0 adds them **test-scoped, version-pinned to the
   discovered Spring Boot / Java / JUnit Platform versions**, then confirms the build still compiles
   and the suite still passes before continuing. If an inject breaks the build it is reverted and
   reported — never left half-applied.
5. **Optional staging.** To eyeball the case matrix, conventions, and state model before committing
   tokens to the full loop, run phases 0–2 as their own workflow first, review the artifacts, then
   run the rest.
6. **Save it.** Once a run does what you want, `/workflows` → select it → press `s` to keep it as a
   `/command` you can re-run on every change.

### Toolchain (pinned for Spring Boot — Maven)

| Role               | Tool / mechanism                                              | Command or annotation                                              |
| ------------------ | ------------------------------------------------------------- | ------------------------------------------------------------------ |
| Base test + assert | JUnit 5 (Jupiter) + AssertJ (via `spring-boot-starter-test`)  | `mvn test`                                                         |
| HTTP layer         | MockMvc on a controller slice                                 | `@WebMvcTest`                                                      |
| Full stack         | Real server + RestAssured / `TestRestTemplate`                | `@SpringBootTest(webEnvironment = RANDOM_PORT)`                    |
| Persistence        | Repository slice on a real DB                                 | `@DataJpaTest` + Testcontainers (`@ServiceConnection`)            |
| Collaborators      | Replace a bean with a mock — and make it **fail**             | `@MockitoBean` (Boot ≥ 3.4; `@MockBean` below that)              |
| Outbound HTTP      | Stub the remote, including timeouts/5xx                       | WireMock / `MockWebServer`                                        |
| Concurrency        | Interleave requests / parallel writes                         | `ExecutorService` + `CountDownLatch`, `@Version`                  |
| Security paths     | Drive 401 / 403 / role checks                                 | `spring-security-test` (`@WithMockUser`, `SecurityMockMvc…`)      |
| Time / async       | Make non-determinism a seam                                   | injected `java.time.Clock` bean / Awaitility                      |
| Property-based     | Invariants + adversarial inputs over generated data           | jqwik (`@Property`, `@ForAll`)                                    |
| Coverage           | Branch coverage                                               | JaCoCo — `mvn clean test jacoco:report` → `target/site/jacoco/jacoco.xml` |
| Mutation           | The hard gate — **pure-logic layer only, fast unit tests**    | PIT — `mvn org.pitest:pitest-maven:mutationCoverage`              |
| Contract (schema)  | Responses match the published schema                          | `swagger-request-validator` vs springdoc `/v3/api-docs`           |
| Contract (consumer)| You haven't broken what real consumers depend on              | Pact (`pact-jvm` provider) + PactFlow broker — `@Provider`, `pact-broker can-i-deploy` |
| Degraded modes     | Declared 5xx/throttle patterns + caller retry semantics       | `resilience-patterns.md` authority — status + `Retry-After` + safe/idempotent retry |

---

## The prompt

> Run a **workflow** to evaluate and harden the Spring Boot REST capability at `<SCOPE: controller
> class, package, or route group>` (base package `<BASE PACKAGE: e.g. com.acme.orders>`).
>
> **Goal:** prove that every input maps to a defined, *intended*, documented outcome; that the
> capability stays correct when its dependencies fail and under concurrent access; that it has no
> trap states; and that it is consistent with the rest of the API — then improve the code to close
> any gap **without breaking existing clients**. Hard rules for the whole run:
> - **The running code is not the source of truth for correctness.** Where the only spec is
>   springdoc/OpenAPI generated *from* the code under test, treat it as documentation of *current*
>   behavior, not intended contract. Every expected outcome must trace to an authority (external
>   contract, written requirement, RFC, domain invariant); a behavior justified only by "this is
>   what the code does today" is recorded as unverified, not locked in by a test.
> - **4xx / client-error responses are valid terminal outcomes.** An *unhandled* 5xx — an uncaught
>   exception, a leaked stack trace, or any 5xx matching no declared pattern — is a defect to fix (add
>   the validation or handler that makes it the correct 4xx). But a 5xx that conforms to a **declared
>   resilience/degradation pattern** in `resilience-patterns.md` is a legitimate, intended outcome:
>   e.g. database read-only during an upgrade → 503 + `Retry-After` triggering *store-and-forward*, a
>   429 + `Retry-After` under rate limiting, or a 503 from an open circuit breaker. For those, assert
>   the pattern's full contract — correct status, `Retry-After`/headers, the safe/idempotent-retry
>   semantics the caller may rely on, a clean `ProblemDetail` body with no stack trace, and that a
>   retry after recovery succeeds with no duplicate side effect — instead of flagging it. A 5xx that
>   matches **no** declared pattern is surfaced for human classification: never silently accepted (that
>   would ossify a bug) and never auto-rewritten to a 4xx (that would break a designed pattern).
> - **Pin non-determinism behind a seam and also drive the seam to fail.** Injected `Clock`, seeded
>   RNG, `@MockitoBean`, WireMock, Awaitility — assert the *contract*, not a fixed value. The mocked
>   collaborators are where real outages come from, so test them throwing, timing out, and returning
>   malformed data, not only succeeding.
>
> **Consistency, defined:** the scoped capability conforms to a single repo-wide API-conventions
> contract covering error media type + body shape, HTTP status usage (e.g. absent entity → 404 not
> empty-200; create → 201 + `Location`; conflict → 409), pagination/sorting params and envelope,
> date/time + number serialization, null-vs-absent field policy, resource/field naming case, content
> negotiation, and versioning/deprecation signaling. Conformance is measured against the shared
> `api-conventions.md` artifact, not asserted endpoint-locally.
>
> **In scope:** functional correctness, the framework error surface, dependency-failure behavior,
> per-request concurrency/idempotency, transaction atomicity, and cross-endpoint consistency.
> **Out of scope** (configure sane timeouts + a circuit breaker on every outbound call as a
> prerequisite, but do not try to verify them here): load/soak behavior, connection- or thread-pool
> exhaustion under volume, and circuit-breaker/backoff tuning — these need a separate load-test pass.
>
> **Definition of done — do not report success until ALL hold, each backed by pasted evidence:**
> - Branch coverage ≥ `<95%>` on the scoped logic (JaCoCo `BRANCH` counter), generated/boilerplate
>   excluded — Lombok/MapStruct `@Generated`, `@Configuration`, and pure DTOs do not count. Every
>   exclusion is disclosed with its file and reason; a class containing a handler, `@ExceptionHandler`,
>   validation, or transition logic is **not** excludable. A genuinely unreachable defensive branch is
>   reported with its reason, never colored in with a contrived input. Evidence: paste the
>   `<counter type="BRANCH" missed=… covered=…/>` lines and the ratio you computed.
> - Mutation score ≥ `<80%>` (PIT) on the **pure-logic packages** (`service`, `domain`, mappers,
>   validators) measured by **fast unit tests only**. Controllers, `@RestControllerAdvice`, and
>   integration code are excluded from the mutation gate and instead held to the contract/slice gate
>   below — do not chase a mutation score on the web layer (it makes the run never finish). No
>   surviving mutants except equivalents *proven* so (mutated line + operator + argument why no input
>   can distinguish them), each verified by a different agent. Evidence: paste the PIT summary line and
>   survivor count.
> - **Web layer (controllers, advice): contract gate, not mutation gate** — every case-matrix
>   partition has an asserting slice/contract test (`@WebMvcTest` / `swagger-request-validator`).
> - **Consumer contracts hold and degraded modes conform.** Where consumer pacts exist, Pact provider
>   verification passes and `can-i-deploy` is green — no published consumer expectation is broken. Every
>   pattern in `resilience-patterns.md` has a conformance test asserting its status, `Retry-After`/headers,
>   and safe/idempotent-retry semantics.
> - Every input partition — valid, invalid, boundary, **every framework error path**, and the
>   **reliability partitions** (see phase 1) — has an asserting test that checks HTTP status **and the
>   semantic correctness of the body's key fields** (the error `detail`/code names the actual offending
>   input, body `status` equals the HTTP status, validation errors name the violated field). Shape-only
>   assertions (field presence/type without values) do not satisfy this. Round-trip and schema-validity
>   properties are self-consistency checks and do **not** count as covering a case.
> - The state model shows every non-terminal state reachable from init with at least one defined exit
>   **whose guard and target match the documented lifecycle**; every transition maps to a named
>   executing test; transitions present in code but absent from / contradicting the lifecycle are
>   reported, not silently accepted; trap and unreachable states are listed.
> - Every scoped endpoint conforms to `api-conventions.md`, or each deviation is fixed (phase 6) or
>   recorded as a human-approved exception.
> - Measurement integrity: every numeric claim is backed by the literal artifact text that produced it.
>   If any measurement command exits non-zero, times out, OOMs, or can't reach Docker/DB, the run is
>   **FAILED** — never estimated, interpolated, or carried forward. Across the scoped suite,
>   `Skipped: 0` and `Errors: 0`; no required partition is covered by an `@Disabled`,
>   `Assumptions.assume*`-skipped, empty-assertion, or exception-swallowing test.
> - An **independent verifier** that wrote none of the tests re-derived every number from a clean tree
>   (phase 7) and they all hold. Falling short of any gate is a FAILED run with an itemized gap list —
>   "no further progress is possible" is acceptable only as a per-item, evidence-backed claim, never a
>   blanket reason to declare success below threshold.
>
> **Phases:**
>
> 0. **Discover the stack & establish the baseline.** Create a dedicated branch
>    (`api-hardening/<scope>`); refuse to start on a dirty tree. Read `CLAUDE.md` for a documented
>    **Stack Profile** (build tool + wrapper, Boot/Java version, MVC vs WebFlux, security scheme,
>    persistence/DB + Testcontainers, `@MockitoBean` vs `@MockBean`, `ProblemDetail` vs custom error
>    body, API versioning/deprecation scheme, OpenAPI location, which of JaCoCo / PIT
>    (+ `pitest-junit5-plugin`) / jqwik / `spring-security-test` / WireMock / `swagger-request-validator` /
>    `pact-jvm` (with consumer pacts published to a broker / PactFlow)
>    are on the build, base package, and the **pure-logic package names** — service/domain/mapper/validator
>    — that PIT will target). If present, use it but spot-check against the build files on disk;
>    the repo wins on conflict. If missing/incomplete, evaluate `pom.xml`/`build.gradle`, the
>    `@SpringBootApplication` class, starters, and security/persistence config, and **append the Stack
>    Profile to `CLAUDE.md` inside a single delimited, regenerable block**
>    (`<!-- STACK-PROFILE:start -->` … `<!-- STACK-PROFILE:end -->`) — never modify content outside
>    that block. Add any missing test plugin/dependency test-scoped and version-pinned, then verify the
>    build still compiles and passes; **smoke-test PIT** on one trivial service with a fast unit test
>    and confirm a non-zero mutation count (a "0 tests found" / `MalformedClassNameException` means the
>    JUnit 5 plugin is mismatched — fix before trusting any score). Produce or update a repo-wide
>    **`api-conventions.md`** (in `src/test/resources/_generated/`) capturing the consistency dimensions
>    above; if it already exists, treat it as the authority the scope must conform to, never overwrite it
>    with this scope's local behavior. Then **read `resilience-patterns.md`** — produced and maintained by
>    the resilience-mapping prompt (`resilience-mapping-workflow-prompt.md`) — as the authority for
>    *acceptable* degraded outcomes (read-only-mode/store-and-forward, rate-limit, circuit-open, planned
>    maintenance). Parse the YAML inside the `DECLARED:BEGIN/END` block only — its trigger, status,
>    `Retry-After`/headers, caller retry contract, and recovery exit — and **ignore the `PROPOSED` block**
>    (proposals are not in force). Do not invent patterns: if the file is absent or declares none, note that
>    the resilience-mapping prompt should be run first, and treat every undeclared 5xx-producing behavior you
>    observe as a defect candidate for human confirmation. Finally, **measure the baseline**: run `mvn clean test jacoco:report`
>    and the scoped PIT goal against the unmodified code, save the reports to
>    `src/test/resources/_generated/baseline/`, and paste the raw counter/score lines — the "before"
>    numbers must come from these saved artifacts, never reconstructed later.
>
> 1. **Derive the case space.** Enumerate every endpoint in scope and partition each into equivalence
>    classes: valid, invalid, boundary, and every declared error path. Cross-reference the springdoc
>    spec including its negative space. For each case record the expected status, the expected **body
>    semantics**, and the **authority** for that expectation, *separately from* the actual current
>    response (capture the latter as a characterization baseline; where intended ≠ current, mark it a
>    proposed contract change for phase 6, do not let a test silently lock in current behavior).
>    Explicitly include Spring's **automatic error surface** — branches you own through your handlers:
>    - 400 — Bean Validation (`MethodArgumentNotValidException` / `ConstraintViolationException`),
>      malformed JSON (`HttpMessageNotReadableException`), type mismatch, missing required param/header
>    - 401 / 403 (Security), 404 (absent entity / no handler), 405, 415, 406 — and verify the *choice*
>      of status matches `api-conventions.md`, not just that it is reachable.
>
>    And the **reliability partitions** (mark each endpoint safe / idempotent / unsafe-mutating):
>    - **Concurrency** — for unsafe-mutating endpoints, two interleaved requests; if the entity carries
>      `@Version`, a stale write must yield 409, not an unhandled 500; if it does read-modify-write with
>      no `@Version`, flag the lost-update risk.
>    - **Idempotency under retry** — replay the identical request (POST included); assert no duplicate
>      side effect, or record the duplicate as a finding where no idempotency key exists.
>    - **Adversarial-but-parseable input** — oversized body, deeply nested / large-array JSON, and
>      mass-assignment (fields like id/role/audit set via `@RequestBody`); assert predictable rejection,
>      not a crash or silent bind.
>    - **Unbounded collections** — collection endpoints clamp an oversized/absent page size and avoid
>      N+1 (assert statement count via Hibernate statistics in a `@DataJpaTest`).
>    - **Declared degraded modes** — for each pattern in `resilience-patterns.md`, a case asserting the
>      designed response (e.g. read-only-mode → 503 + `Retry-After`) and that the caller's permitted retry
>      (safe / idempotent) succeeds after recovery with no duplicate side effect.
>
>    Write the result to `src/test/resources/_generated/case-matrix.md`.
>
> 2. **Build the state model.** Extract states, transitions, and guards from the domain status enums,
>    any `spring-statemachine` config, and the service-layer transition methods into
>    `src/test/resources/_generated/state-model.json`. Run reachability and liveness checks; cross-check
>    each transition against the documented lifecycle (a code transition absent from the spec is a
>    finding); map every transition to a named test. List trap states (a status with no outgoing
>    transition) and unreachable states explicitly. 4xx terminals are expected sinks, and declared
>    degraded modes (read-only, circuit-open, maintenance) are legitimate non-terminal states whose exit
>    is the recovery transition — not trap states.
>
> 3. **Generate tests.** Against the case matrix, write:
>    - **(a) example-based** — `@WebMvcTest` + MockMvc for validation and error-mapping (status +
>      `ProblemDetail` body semantics); `@SpringBootTest(RANDOM_PORT)` + RestAssured for representative
>      happy and error paths end-to-end; `@DataJpaTest` + Testcontainers for repository constraints,
>      N+1, and transaction rollback; security paths (authenticated / anonymous / wrong role).
>    - **(b) property-based** with jqwik for invariants (idempotency where claimed, GET safety) and for
>      adversarial size/depth inputs. Note: round-trip and schema-validity are consistency checks, not
>      correctness checks.
>    - **(c) failure & concurrency** — make each seam throw the realistic exception
>      (`DataAccessException`, `HttpServerErrorException`, `ResourceAccessException`/timeout), respond
>      slowly (WireMock delay → caller timeout), and return malformed/empty payloads; assert a defined
>      5xx `ProblemDetail` and no hung thread. For unsafe-mutating endpoints add the concurrency test;
>      for multi-write handlers force a mid-operation failure and assert full rollback (verify
>      `@Transactional` is on a proxied, externally-invoked method with rollback rules covering the
>      thrown exceptions). For each declared resilience pattern, simulate its trigger (e.g. a read-only
>      datasource) and assert the designed degraded response *and* that a permitted retry succeeds after
>      recovery with no duplicate side effect.
>
>    Add schema contract tests against the OpenAPI spec with `swagger-request-validator`, and — where
>    consumer pacts exist — **Pact provider verification** (`@Provider` against the broker / PactFlow) so
>    every published consumer expectation is checked against the live provider.
>    **Every test must trace to a contract authority; a test whose only justification is "kills mutant X"
>    by pinning an incidental value is rejected** — if a mutant is only killable by asserting wrong-but-
>    current behavior, mark the code "suspected-bug" for phase 6 instead. No `@Disabled`, `assume*`-skip,
>    empty assertions, or `catch`-swallow on a required partition.
>
> 4. **Measure (layered, separate invocations).** Run `mvn clean test jacoco:report` for branch
>    coverage (paste per-package `BRANCH` counters; disclose exclusions). Run PIT **only over fast unit
>    tests on the logic packages**, never the Spring-context/Testcontainers tests (each would reload the
>    context per mutant and the run would hang):
>    `mvn org.pitest:pitest-maven:mutationCoverage -DtargetClasses=<BASE PACKAGE>.service.*,<BASE PACKAGE>.domain.*,<BASE PACKAGE>.mapper.*,<BASE PACKAGE>.validation.* -DtargetTests=<BASE PACKAGE>.**.*UnitTest -DexcludedTestClasses=*IT,*IntegrationTest,*WebMvcTest,*DataJpaTest -DtimeoutConstant=10000 -DtimeoutFactor=2`
>    — use the actual pure-logic package names recorded in Phase 0; mappers/validators that live in
>    their own packages must be included or the ≥80% gate is measured on a narrower surface than it claims.
>    Run JaCoCo and PIT as separate Maven invocations (PIT must not run with the JaCoCo agent attached).
>    Confirm a non-zero PIT test count. **If any command exits non-zero / times out / OOMs / can't reach
>    Docker, STOP and report FAILED with the output — do not estimate.** Report JaCoCo and PIT per layer
>    side by side; a branch with coverage but no killed mutant is an *untested* branch. Collect uncovered
>    branches, surviving mutants, unmet contract/consistency/reliability cases into one gap list.
>
> 5. **Close the loop.** Independent agents attack the gap list: for each item, first write down the
>    intended contract (citing its authority), then a test that asserts it — re-measure. Bound the loop:
>    at most 3 rounds and a wall-clock ceiling; if a threshold is still unmet, report the remaining gap
>    list and FAIL rather than looping. Never raise the score by deleting or weakening a defensive guard.
>
> 6. **Improve the code.** For every real defect — undefined behavior, a 5xx that should be a 4xx, an
>    unhandled dependency failure, a missing rollback, a trap state, a consistency deviation, or a
>    genuinely-wrong mutant survivor — propose a fix. **Contract-preservation gate:** classify each fix
>    as *wire-compatible* (changes only previously-undefined or 5xx behavior) or *breaking* (changes an
>    existing 2xx shape, field, status code a client observes, validation strictness on a previously-
>    accepted input, error-body format, or auth/authorization). **Auto-apply only wire-compatible fixes
>    covered by a test that traces to an independent contract source** (a test written this same loop to
>    match the fix does not count). List every breaking fix — and *all* security tightening — as a
>    recommendation with the before→after wire diff and affected consumers; never auto-apply. Pact
>    provider verification is the objective test here: a fix that fails verification against any published
>    consumer pact is breaking by definition — recommend it, never auto-apply. Stay inside
>    `<SCOPE>`: a fix needing a shared file (global advice, security config, shared state machine) is a
>    recommendation noting the blast radius, not an edit. Serialize edits to any shared file across
>    parallel agents. Re-run the baseline suite after changes — a previously-passing test that now fails
>    is a regression that blocks.
>
> 7. **Independent verification.** Spawn a verifier subagent that wrote none of the tests. From a fresh
>    `git worktree` off the branch HEAD (a clean tree), it runs `mvn clean test jacoco:report` and the
>    scoped PIT goal itself, parses `jacoco.xml` and
>    `mutations.xml` with its own arithmetic, re-enumerates the framework error surface from the
>    controller signatures, audits each coverage exclusion and equivalence claim, confirms `Skipped: 0` /
>    `Errors: 0`, and checks that every case-matrix row and state transition maps to an asserting test.
>    It rejects — does not trust — the writers' self-reported numbers. The run is DONE only if the
>    verifier's independently-derived numbers meet every gate.
>
> **Deliverables:** the Stack Profile block and `api-conventions.md` (conforming to the declared `resilience-patterns.md` it read); the case matrix (with authority,
> current-behavior, and reliability partitions); the state model + reachability/liveness report +
> transition→test map; the generated tests; the baseline and after JaCoCo + PIT reports per layer with
> the raw counter/score lines pasted; the list of recommended breaking changes with wire diffs; a
> **reliability ledger** in behavior terms (defects found and fixed, contract discrepancies surfaced,
> behaviors confirmed wrong-but-current, breaking changes); and a summary of code changes made vs.
> recommended. Do not claim the API is "more reliable" on coverage/mutation deltas alone — cite concrete
> defects closed.
>
> **Model routing:** use a smaller model for enumeration and test scaffolding; use the strongest model
> for state analysis, mutation triage, code fixes, and the independent verifier (which must be a
> different agent than the writers).

---

## Why each piece is there

- **The thresholds are the contract, but the mutation gate is layered.** Coverage alone is gameable; a
  surviving PIT mutant is a precise, machine-checkable "case you don't actually cover." But PIT re-runs
  the covering tests per mutant, so it only completes on fast, context-free unit tests — it gates pure
  logic. The web layer is gated by contract/slice completeness instead. Trying to mutation-test
  controllers is what makes the run never finish.
- **The running code is not the oracle.** If tests pin current behavior and springdoc is generated from
  that same code, a real bug becomes a test-protected, mutation-locked invariant. Recording an
  *authority* for each expected outcome and diffing against current behavior is what prevents ossifying
  the bug.
- **A more correct API can be a less stable one.** Changing a 200 to a 404, tightening validation, or
  requiring auth is "more correct" and breaks every client that depended on the old behavior. The
  contract-preservation gate keeps "stable" honest: breaking changes are recommended, never auto-applied.
- **Consumer-driven contracts catch the breakage schema checks can't.** OpenAPI validation proves shape;
  Pact provider verification proves you haven't broken what a real consumer actually depends on — the
  precise guard the backward-compatibility goal needs, and `can-i-deploy` gates the release on it.
- **A designed 5xx is a contract, not a bug — when it's declared.** Read-only/store-and-forward, 429
  rate-limit, and circuit-open 503 are legitimate degraded outcomes with their own retry/`Retry-After`
  contract that the workflow asserts. A 5xx matching no declared pattern is surfaced for a human to
  classify, so designed degradation is allowed without silently swallowing an unhandled exception.
- **The seams are where outages live.** Mocking a collaborator to succeed proves nothing about what
  happens when it times out or 503s — the single most important reliability question. Failure injection
  and concurrency tests cover the bug classes that no happy-path example test will.
- **Assert the body, not just the status.** A status-only or shape-only assertion passes even when the
  error body is wrong, misrouted, or leaks a stack trace. The body's *meaning* is the contract.
- **Consistency needs a shared authority.** A per-scope run can't see cross-endpoint drift, so two runs
  can each pass while disagreeing. `api-conventions.md` is the single normative artifact every scope
  conforms to, converting "consistent" from an aspiration into a gate.
- **The case matrix, state model, and conventions are files, not prose.** Durable artifacts the later
  phases and future re-runs read from; they survive `mvn clean` under `src/test/` and keep the
  enumeration out of the model's context window.
- **Evidence over assertion, and an independent verifier.** The agent that writes the tests is
  incentivized to clear the gates, so it cannot be the one that certifies them. Pasted artifact lines,
  treating any errored measurement as FAILED, a measured-first saved baseline, forbidding skipped/empty
  tests, and a clean-tree re-derivation by a different agent are what make "done" mean done.
- **Discovery writes back to `CLAUDE.md`.** Phase 0 turns the stack into a durable, shared fact in a
  regenerable block, so the next run reads the Stack Profile instead of re-deriving it — paid once.
