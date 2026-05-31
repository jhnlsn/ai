# Prompt Library

Reusable prompts for Claude Code and other agentic tooling.

## Contents

| File | Purpose |
| ---- | ------- |
| `resilience-mapping-workflow-prompt.md` | A Claude Code dynamic-workflow prompt for **Spring Boot** API services that maps resilience posture from architecture + calling patterns: inventories the patterns already implemented (with evidence), derives which apply, and **proposes** missing/useful ones for human review. **Analysis-only** (no code/build changes); produces `resilience-patterns.md` — a human-owned `DECLARED` block + a tool-owned `PROPOSED` block. Feeds the coverage prompt and further analysis (chaos, capacity, runbooks, architecture review). |
| `api-coverage-workflow-prompt.md` | A Claude Code dynamic-workflow prompt for **Spring Boot** REST APIs: maps an endpoint's case space (Spring's automatic 4xx surface, dependency-failure behavior, concurrency, and cross-endpoint consistency), generates JUnit 5 + jqwik + MockMvc/Testcontainers tests, measures with JaCoCo + layered PIT mutation testing, checks reachability/liveness for trap states, hardens the code **without breaking existing clients**, and independently verifies its own numbers before reporting done. **Reads `resilience-patterns.md`** (produced by the resilience-mapping prompt) to know which 5xx are designed vs. defects. Pinned to Maven (`mvn`); works on a branch and never auto-commits. |

## Shared artifacts & ordering

The two Spring Boot prompts chain through one file, `src/test/resources/_generated/resilience-patterns.md`:

1. Run **`resilience-mapping-workflow-prompt.md`** first. It writes the tool-owned `PROPOSED` block; a
   human promotes the proposals they accept into the human-owned `DECLARED` block.
2. Run **`api-coverage-workflow-prompt.md`** next. It reads the `DECLARED` block as authority — testing
   each designed degraded outcome (e.g. read-only-mode → 503 + `Retry-After`) and treating any 5xx that
   matches no declared pattern as a defect.

`DECLARED` is human-owned (tools never edit it); `PROPOSED` is tool-owned (regenerated each mapping run).

## Conventions

- One prompt per file, named `<area>-<purpose>-prompt.md`.
- Each prompt states its **definition of done** up front so success is measured, not asserted.
- Placeholders use `<ANGLE_BRACKETS>`; fill them before running.
- Workflow prompts include the word `workflow` so Claude Code orchestrates rather than working turn by turn.

## Usage

Paste a prompt into Claude Code (CLI, Desktop, or IDE extension). For workflow prompts, allowlist
any test/coverage/mutation commands first so the run doesn't stall on a permission prompt, then
save the run as a `/command` via `/workflows` → `s` if you'll reuse it.
