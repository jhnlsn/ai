# Multi-Turn Agent — Implementation Specification

A practical, layered spec for building a multi-turn agent runtime. Three tiers:

1. **Tier 0 — MVP** (you can hold a tool-using conversation)
2. **Tier 1 — Production** (you can ship it to a real user)
3. **Tier 2 — Advanced** (Claude-Code- / pi.dev- / Hermes-tier capability)

Each feature is listed with **what it does**, **why it matters**, and **the minimum implementation primitives** you need. Treat tiers as *cumulative*: don't skip Tier 0 to chase Tier 2 features.

> Vendor reference points used throughout:
> - **Claude Code** — Anthropic's terminal coding agent. Strong on tools, permissions, hooks, subagents, skills, MCP.
> - **pi.dev** — Inflection's personal AI. Strong on persistent memory, voice, persona continuity.
> - **Hermes** — Nous Research's open-weights model family. Strong on tool/function calling fine-tune and steerable system prompts.
>
> Hermes is primarily a *model*; Claude Code and pi.dev are *systems*. Where this matters, the spec calls out whether a feature lives in the model, the harness, or both.

---

## Tier 0 — MVP

The smallest thing that can hold a multi-turn, tool-using conversation. If any of these are missing you don't have an agent — you have a chatbot.

### 0.1 Conversation Loop

**What:** A loop that, given a message history, calls the model, parses the response, and either returns to the user (if no tool calls) or executes tools and re-invokes the model with their results, until a stop condition is reached.

**Why:** This *is* the agent. Everything else is plumbing around it.

**Minimum primitives:**
- A typed `Message` discriminated union: `system | user | assistant | tool_result`.
- A `ToolCall` type with `id`, `name`, `arguments` (JSON).
- A `runTurn(history) → { newMessages, stopReason }` function.
- A driver loop: `while stopReason === "tool_use": execute tools, append results, runTurn() again`.
- A hard cap on iterations (e.g., 50) to prevent runaway loops.

**Pseudocode:**

```ts
async function runAgent(history: Message[]): Promise<Message[]> {
  for (let i = 0; i < MAX_TURNS; i++) {
    const { messages, stopReason } = await callModel(history);
    history.push(...messages);
    if (stopReason !== "tool_use") return history;
    const toolResults = await executeToolCalls(extractToolCalls(messages));
    history.push(...toolResults);
  }
  throw new Error("Agent exceeded max turns");
}
```

### 0.2 Tool Calling Protocol

**What:** A way for the model to declare "I want to run tool X with args Y" and for the harness to declare available tools to the model.

**Why:** Without tools, the model is just generating text. Tools are how it acts on the world.

**Minimum primitives:**
- A `ToolSpec` schema: `{ name, description, inputSchema (JSON Schema or Zod) }`.
- A `ToolHandler = (args) → Promise<string | structured>` registry.
- Pass tool specs to the model on every call (most providers require this).
- Validate tool arguments against the schema before executing.

**Provider-specific notes:**
- Anthropic, OpenAI, Google all have native tool-calling APIs — use them, don't roll your own JSON parsing on top of plain text.
- Hermes-style open-weights models often expect a chat template with `<tool_call>` tags; verify the exact template — they're not interchangeable across model families.

### 0.3 System Prompt

**What:** A persistent instruction at the head of the conversation that defines persona, capabilities, constraints.

**Why:** Sets the agent's identity and rules. Without it, behavior drifts wildly between turns.

**Minimum primitives:**
- A static string or template.
- Inject once at the start; do not repeat each turn (it lives in conversation state).
- Prompt-cache it (see §1.6).

### 0.4 Stop Conditions

**What:** Rules for when to exit the loop and return to the user.

**Why:** A loop with no termination is a billing incident.

**Minimum primitives:**
- Provider stop reason (`end_turn`, `stop`, `tool_use`).
- Max turn count.
- Max wall-clock time.
- User-initiated cancellation (abort signal).

### 0.5 Error Handling

**What:** Graceful behavior when a tool throws, the model errors, or the network drops.

**Why:** Real systems fail; the agent must recover or surface clearly.

**Minimum primitives:**
- Tool errors return as `tool_result` with `is_error: true` — let the model see and react.
- Model API errors: retry with exponential backoff for 5xx/timeouts, fail-fast for 4xx.
- Distinguish *transient* (retry) from *terminal* (abort) errors.

---

## Tier 1 — Production

Everything required to run the agent for a real user without a human babysitter watching the logs.

### 1.1 Streaming

**What:** Surface tokens / blocks to the user as the model generates them, instead of waiting for the full response.

**Why:** Latency perception. A 30-second silent wait feels broken; a 30-second stream feels alive.

**Minimum primitives:**
- Provider streaming API (`stream: true`).
- An event channel out of `runTurn`: `text_delta | tool_call_start | tool_call_delta | tool_call_end | message_stop`.
- Backpressure handling (don't drop events under load).

**Advanced flavors (see §2.10):**
- Block-based streaming with edits (replaces last block) instead of token deltas — used by OpenClaw for messaging channels where token-by-token edits are too noisy.
- Reasoning streams as a separate channel from final output.

### 1.2 Conversation Persistence

**What:** Store conversation history so a session can resume across process restarts and the user can scroll back.

**Why:** Users expect chat history. Without persistence, every restart is a fresh session.

**Minimum primitives:**
- A `SessionStore` interface: `load(sessionId)`, `append(sessionId, messages)`, `list()`.
- Filesystem JSON or SQLite is fine for single-user. Postgres if multi-user.
- Atomic writes (write to temp + rename) to survive crashes.

### 1.3 Context Window Management

**What:** Don't exceed the model's context limit. Don't pay for tokens you don't need.

**Why:** Long conversations hit the limit. The naïve "always send everything" approach fails at turn ~50.

**Minimum primitives:**
- Token counting (provider tokenizer, not heuristics).
- A budget: `model_context_limit - reserved_for_response - safety_margin`.
- A truncation policy when over budget — at minimum, drop oldest non-system messages.

**Better:** see §2.4 (Compaction).

### 1.4 Auth & Secrets

**What:** Manage API keys for one or more providers without hardcoding them.

**Why:** You'll rotate keys. You'll add a fallback provider. You don't want to redeploy for either.

**Minimum primitives:**
- Env var → fallback to credential file in user dir.
- A redaction layer for logs (never log full keys).
- A `Provider` interface so swapping providers doesn't touch agent code.

**Production-tier:** auth profile rotation with cooldowns on rate limits — `last-good` round-robin ordering, mark-failure with cooldown expiry. (OpenClaw's `auth-profiles.*` is a good reference shape.)

### 1.5 Observability

**What:** Logs, metrics, and traces sufficient to debug "why did the agent do that?" after the fact.

**Why:** Agents fail in non-deterministic ways. Without traces you cannot root-cause.

**Minimum primitives:**
- Per-turn structured log: `{ turn_id, model, input_tokens, output_tokens, tool_calls, latency_ms, stop_reason }`.
- Full transcript dump with input redacted secrets, addressable by `session_id`.
- OpenTelemetry spans around `runTurn` and each tool call.

### 1.6 Prompt Caching

**What:** Mark stable prefixes (system prompt, tool specs, long context) as cacheable so the provider charges 10× less for cache hits.

**Why:** Multi-turn agents re-send the same prefix every turn. Without caching you pay 5–20× more than necessary.

**Minimum primitives:**
- `cache_control: { type: "ephemeral" }` markers on the last stable block (Anthropic). OpenAI auto-caches.
- **Deterministic ordering** of dynamic content — sort tools, files, registries before stringifying. Non-deterministic order = zero cache hits. (OpenClaw enforces this; it matters more than people realize.)
- Preserve old transcript bytes when editing: don't re-stringify with different whitespace.

### 1.7 Cancellation

**What:** Let the user interrupt mid-turn and the harness cleanly tear down model streams + in-flight tools.

**Why:** Long tool calls hang. Users press Ctrl-C and expect it to actually stop.

**Minimum primitives:**
- `AbortController` plumbed into model fetch and every tool handler.
- Tool handlers must honor the signal (especially long ones: bash, browser, network).
- On abort: append a synthetic `tool_result: { is_error: true, content: "cancelled" }` so the model can recover on resume.

### 1.8 Rate Limiting & Backoff

**What:** Provider 429s. Rate-limited tools. Honor retry-after headers.

**Why:** You will hit limits. Without graceful backoff you'll thrash.

**Minimum primitives:**
- Token bucket per provider × auth profile.
- Exponential backoff with jitter on 429/503.
- Surface "rate limited, retrying in Xs" to the user — don't silently stall.

### 1.9 Approvals (Human-in-the-Loop)

**What:** For destructive or sensitive tool calls, require user confirmation before execution.

**Why:** Bash with shell access can `rm -rf /`. Production agents need a brake pedal.

**Minimum primitives:**
- Per-tool `requires_approval: boolean | predicate(args)`.
- An approval prompt UI (terminal y/n, modal, Slack message) that blocks tool execution.
- "Always allow" / "Always allow this exact command" / "Allow once" granularity.
- Approval state survives session restarts.

---

## Tier 2 — Advanced

This is the gap between "I built an agent over a weekend" and "Claude Code / pi.dev / Hermes-tier."

### 2.1 Rich Tool Suite

**What:** A built-in set of well-designed, composable tools that cover most real tasks without the user having to write any.

**Why:** The model is only as good as its hands. A great tool surface is the single biggest force-multiplier.

**Reference: Claude Code's tools**
- `Read` — read files (with line ranges, image support)
- `Write` — create files
- `Edit` — exact-string replacement (forces the model to read first; prevents drift)
- `Glob` / `Grep` — search by pattern
- `Bash` — shell, with PTY support and approvals
- `Task` — spawn a subagent
- `WebFetch` / `WebSearch`
- `TodoWrite` — structured task tracking visible to user

**Design principles that separate good tools from bad:**
- **Single responsibility per tool.** A "do anything with files" tool fails; `Read` + `Edit` + `Write` succeeds.
- **Schemas that prevent footguns.** `Edit` requires `old_string` to be unique in the file; if not, error and force the model to add context. This eliminates an entire class of silent corruption.
- **Force preconditions.** `Edit` errors if you haven't `Read` the file first in this session. Cheap to enforce, catches huge classes of model errors.
- **Output that fits in a tool result.** Cap stdout, paginate, summarize. A tool that returns 200K tokens poisons the context window.
- **Idempotency where possible.** Re-running a tool with same args should be safe.

### 2.2 Subagents / Task Spawning

**What:** A tool the parent agent calls that spawns a child agent with its own context window, runs it to completion, and returns only the summary.

**Why:**
- **Context preservation** — child does noisy exploration, parent only sees the answer.
- **Specialization** — different system prompts / tool subsets per child.
- **Parallelism** — fan out independent tasks.

**Minimum primitives:**
- A `spawn` tool with: `description`, `prompt`, `subagent_type` (selects system prompt + tool allowlist), optional `model` override.
- Per-child session isolation (separate transcript, separate sandbox if applicable).
- Return value = single final message (the parent does not see child's intermediate tool calls).
- Background spawning with completion notification (`run_in_background`).
- Configurable max nesting depth (children spawning children).

**Reference shapes:**
- Claude Code's `Task` tool with named agent types (general-purpose, Explore, etc.).
- OpenClaw's ACP spawn (`acp-spawn.*`) — full bidirectional protocol with allowlists, lifecycle, model selection, scope, steer-failure handling.

### 2.3 Permission System

**What:** A policy engine that gates tool execution by tool name, argument shape, file path, and command pattern.

**Why:** "Approve every bash call" is unusable. Users want "always allow `git status`, never allow `rm`, ask for everything else."

**Minimum primitives:**
- Pattern-based allowlist/blocklist: `Bash(git status:*)`, `Edit(/safe/dir/**)`, `Write(*)`.
- Permission modes: `default | acceptEdits | bypassPermissions | plan`.
- Per-session, per-user, and per-project scopes (project overrides user overrides default).
- A persisted approval store keyed by `(tool, normalized_args)`.

**Pattern syntax to support:**
- Glob for paths (`**`, `*`)
- Prefix match for commands (`Bash(git:*)` matches any `git ...`)
- Argument-shape matching for structured tools

### 2.4 Compaction (Auto Context Engineering)

**What:** When context fills up, summarize old turns into a single message and continue, instead of truncating.

**Why:** Truncation loses information. Compaction preserves it lossily but coherently. Required for sessions >100 turns.

**Minimum primitives:**
- Trigger: `tokens_used > threshold` (e.g., 70% of window).
- A summarization pass: feed old messages + a "summarize" prompt to the model, get back a single condensed message.
- Replace the old slice with the summary; preserve the system prompt, the most recent N turns, and any pinned tool results.
- Mark compaction in transcript so user can see it happened.

**Advanced:** *streaming compaction* that runs in the background as token usage rises, not as a stop-the-world event.

### 2.5 Memory (Cross-Session)

**What:** Information that survives across sessions and is recalled when relevant.

**Why:** This is the single biggest UX differentiator between "tool" and "assistant." It's pi.dev's whole thesis.

**Three memory archetypes — pick at least two:**

**a) File-based instruction memory (Claude Code, OpenClaw)**
- A `MEMORY.md` / `CLAUDE.md` / `AGENTS.md` file the agent reads at session start.
- Structured by topic; user can edit directly.
- Cheap, transparent, user-controllable.

**b) Vector / semantic memory (pi.dev, RAG systems)**
- Embed each user utterance + agent response.
- On new turn, semantic-search top-K relevant past memories, inject into context.
- Requires: embedding model, vector store (LanceDB, sqlite-vec, pgvector), retrieval policy.

**c) Structured / typed memory**
- "Facts about the user": role, preferences, projects, deadlines, names.
- LLM-extracted from conversation, stored as typed records, surfaced when keys match.
- More reliable than vector for atomic facts; worse for narrative context.

**Memory writeback policy:**
- Save *what was surprising or non-obvious*, not the whole transcript.
- Decay/expiry for stale facts (deadlines pass, projects end).
- Versioning for facts that update (user changes job → don't accumulate both).
- User can inspect, edit, and delete (privacy + correctness).

### 2.6 Plan Mode

**What:** A mode where the agent is forbidden from taking actions and must produce a written plan first; the user approves/edits the plan before execution.

**Why:** For ambitious tasks, "let it rip" produces sprawling, half-right work. Plan-then-execute halves the iteration cost.

**Minimum primitives:**
- A flag in the agent runtime: `mode: "execute" | "plan"`.
- In plan mode: tools that write/mutate are blocked; only read-only tools allowed.
- A dedicated `ExitPlanMode(plan)` tool that hands the plan back to the user for approval.
- After approval, the same plan is fed into a fresh execution turn.

### 2.7 Hooks / Lifecycle

**What:** User-configurable shell commands or callbacks that fire at lifecycle events.

**Why:** Users have repo-specific needs (auto-format after edit, run tests after write, log every prompt). Hooks let them extend the harness without forking it.

**Events to expose:**
- `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `Stop`, `Notification`.

**Hook contract:**
- Receives JSON over stdin (event payload).
- Returns JSON on stdout (decision: continue, block, modify).
- Can short-circuit: `PreToolUse` returns `block` → tool doesn't run.

**Reference:** Claude Code's hooks system (`settings.json`).

### 2.8 Skills (Loadable Instruction Bundles)

**What:** Named, discoverable units of "how to do X" that the agent loads on demand.

**Why:** System prompts grow unboundedly. Skills let you keep the prompt small and load expertise lazily.

**Anatomy of a skill:**
- `SKILL.md` with frontmatter: `name`, `description`, `triggers` (when to load).
- A body: instructions, checklists, examples, *links to other files*.
- Optional bundled tools or scripts.

**Loading model:**
- At session start, the harness lists *only the names + descriptions* (cheap).
- The model invokes a `LoadSkill(name)` tool when the description matches the task.
- The full skill body is injected for the rest of the session.

**Why this beats RAG for instructions:** instructions are not chunks. A skill is a coherent procedure; semantic chunking corrupts it. Named skills + LLM-driven selection is more reliable.

### 2.9 MCP / Extensibility

**What:** A protocol for third-party tool servers the agent can dynamically discover and call.

**Why:** Your built-in tool suite cannot cover every user's stack. MCP (Model Context Protocol, Anthropic's open spec) lets users plug in their own tools without modifying the harness.

**Minimum primitives:**
- An MCP client that speaks stdio or HTTP transport.
- Server discovery: list servers from config; start each; enumerate tools.
- Tool naming with namespace (`mcp__notion__create_page`) to avoid collisions.
- Per-server permission scoping.

**Bidirectional bonus:** also *expose* your agent as an MCP server so other agents can call it.

### 2.10 Advanced Streaming

**Three streaming modes; pick the right one per surface:**

**a) Token-delta streaming** — best for terminals, raw chat.
- Append-only; cheap.

**b) Block-streaming with edits** — best for chat platforms (Slack, Teams, iMessage) where every edit is a notification.
- Coalesce deltas into paragraph-sized blocks.
- Edit the last block in place; flush a new block on paragraph break.
- (OpenClaw's approach for messaging channels — see `streaming-message.ts`, `reply-stream-controller.ts`.)

**c) Reasoning channels** — separate stream for `<thinking>` content.
- Render in a collapsed/dimmed UI, distinct from final output.
- Required for thinking-mode models (Claude o-series, OpenAI o-series, DeepSeek R1).

### 2.11 Multi-Modal

**What:** Images in, images out, PDFs, audio, video.

**Why:** Real users send screenshots. Agents need to read them.

**Minimum primitives:**
- Provider-side: pass image blocks (URL or base64) per provider's spec.
- Tool-side: tools that *return* images (screenshots, generated images) emit `ImageBlock` content the model can see next turn.
- Token accounting for image tokens (different from text — a 1024×1024 image ≈ 1.5K Anthropic tokens).
- PDF: chunk by page, OCR if scanned.

### 2.12 Voice / Realtime

**What:** Speak in, speak out, ideally in a continuous full-duplex stream.

**Why:** pi.dev's killer feature. Increasingly table stakes.

**Three layers:**

**a) Push-to-talk batch** (easy)
- Mic → STT (Whisper / Deepgram / SenseAudio) → text → agent → text → TTS (ElevenLabs / Sherpa / Apple) → speaker.
- 1–3s latency, single turn.

**b) Realtime streaming** (medium)
- Streaming STT + streaming TTS, agent emits text deltas, TTS reads ahead.
- ~500ms perceived latency.

**c) Native multimodal realtime** (hard)
- Provider's realtime API: speech in → speech out, no STT/TTS round-trip.
- OpenAI Realtime, Gemini Live. ~200ms.
- Tool calling mid-conversation, interruption handling.

**Always include:**
- Wake-word activation (Apple Speech.framework, Picovoice) — see OpenClaw's Swabble.
- Echo cancellation (or you record yourself).
- Voice activity detection (VAD) for turn detection.

### 2.13 Sandboxing

**What:** Run agent-executed code in an isolated environment that cannot harm the host.

**Why:** Tool-using agents will run untrusted code (the model's own output). On a developer's machine, a hostile or buggy command can wipe `~`.

**Sandbox levels (pick per workload):**

| Level | Implementation | Use case |
|---|---|---|
| **Process isolation** | New process, no shared state | Cheap, weak — model can still touch fs |
| **Filesystem jail** | chroot, bind-mount workspace, deny everything else | Lightweight, blocks fs escape |
| **Container** | Docker / Podman with networking + fs limits | Strong, slower startup |
| **MicroVM** | Firecracker, gVisor | Strongest, harder ops |
| **Browser sandbox** | Headless browser in container | For agentic web tasks |

**Approval-vs-sandbox is orthogonal:** approvals tell the user *what's about to happen*; sandboxing limits *what damage is possible if approved*.

### 2.14 Model Routing & Failover

**What:** Pick the best model per turn; fall back when it errors.

**Why:** No single model wins on cost × latency × quality. Fast model for simple turns, smart model for hard ones.

**Routing dimensions:**
- **By task type** — coding → Sonnet/Opus, summarization → Haiku, vision → multi-modal, reasoning → o-series.
- **By latency budget** — interactive < 2s, background batch jobs OK at 30s.
- **By cost budget** — explicit $/turn caps.
- **By region / sovereignty** — data residency.

**Failover primitives:**
- Provider health probes.
- Rate-limit-aware fallback (don't fail over to a sibling that's also rate-limited).
- Auth profile rotation per provider.
- Preserve transcript across model swaps; some providers refuse foreign tool-call IDs — re-canonicalize.

### 2.15 Determinism for Caching

**What:** Treat cache hit rate as a first-class quality metric.

**Why:** Cache misses are 5–20× more expensive. Non-determinism in your prompt is silent dollars.

**Sources of non-determinism to eliminate:**
- Map / Set iteration order (sort before stringify).
- Object key order in JSON.stringify (use a stable serializer).
- Plugin / tool registration order (sort by id).
- Network result order (sort fetched files, embeddings).
- Timestamps in system prompt (only if needed; otherwise omit).
- Per-request UUIDs in cacheable prefixes (move to suffix).

### 2.16 Slash Commands

**What:** Named user shortcuts (`/help`, `/clear`, `/plan foo`) that map to harness actions or pre-canned prompts.

**Why:** Discoverability + muscle memory. Power users live in commands.

**Minimum primitives:**
- A registry: `name → handler(args, context)`.
- Tab completion.
- Both *harness commands* (`/clear` resets context) and *prompt commands* (`/explain` expands to a templated prompt).

### 2.17 Background / Async Tools

**What:** Long-running tools that return a handle immediately and emit completion notifications later.

**Why:** Some tasks (long shell commands, training jobs, scrapes) take minutes. Blocking the agent loop on them wastes tokens and time.

**Minimum primitives:**
- Tool return type allows `{ status: "running", handle }`.
- A reaper loop that polls handles or listens for completion events.
- Notifications appended to transcript: `system: bash[abc] completed with exit 0`.
- The model sees the notification on its next turn and reacts.

### 2.18 Context Engineering Primitives

Beyond compaction, advanced agents *actively shape* their context.

- **Pinned messages** — never truncate this (e.g., user's project goal).
- **Sliding windows of file content** — when reading large files, the harness windows over them; the model requests offsets.
- **Tool-result summarization** — if a tool returned 50K tokens, summarize it into the transcript and store the full result on disk addressable by ID.
- **Lazy expansion** — show short paths on first reference, expand to full content if the model asks.
- **Conversation forking** — branch from turn N to explore an alternative without polluting the main thread.

### 2.19 Multi-Channel / Multi-Surface

**What:** The same agent reachable from terminal, web, mobile, chat platforms (Slack/Discord/Teams/iMessage), and voice.

**Why:** Personal-AI products (pi.dev) win by being where the user already is.

**Minimum primitives:**
- A channel abstraction: `inbound(message, ctx) → agent → outbound(channel, message)`.
- Per-channel rendering (markdown for chat, plain for SMS, adaptive cards for Teams).
- Per-channel feature flags (no streaming on SMS; threads on Slack; reactions on iMessage).
- Routing: same `userId` → same session, regardless of channel.
- (See OpenClaw's `extensions/<channel>/` family for a concrete instantiation across 30+ surfaces.)

### 2.20 Personality & Style Layer (pi.dev-tier)

**What:** Persistent voice, tone, and emotional calibration that doesn't drift across long sessions.

**Why:** "Smart" is solved. "Feels like a person" is the moat.

**Minimum primitives:**
- A locked style document in the system prompt (pi.dev does this aggressively).
- Style-eval suites — automated regression tests for tone, refusal patterns, warmth.
- Adversarial prompt resistance — user pushes for off-character behavior; agent declines without breaking immersion.

### 2.21 Function Calling Quality (Hermes-tier, model-side)

This one lives in the *model*, not the harness. Hermes' edge is being trained on synthetic function-calling data. If you're training/fine-tuning your own model:

- **Strict JSON schema adherence** — the model emits valid JSON every time, not "usually."
- **Parallel tool calls** — emit multiple tool calls in one turn when independent.
- **Tool selection precision** — picks the right tool from a 30-tool palette.
- **Argument hallucination resistance** — doesn't invent paths/IDs that don't exist.
- **Failure recovery** — when a tool errors, retries with corrected args instead of giving up.

If you're not training, the harness equivalent is **schema enforcement**: validate every tool call against its JSON schema, refuse and re-prompt on validation failure.

### 2.22 Eval Infrastructure

**What:** Automated tests that catch regressions in agent quality across releases.

**Why:** Without evals, every prompt change is roulette. Production agents that don't eval are slowly getting worse and don't know it.

**Eval types:**
- **Unit evals** — single-turn, scored by string match or LLM judge.
- **Trajectory evals** — multi-turn task completion, scored by a final state check (file exists with expected content, test passes, etc.).
- **A/B evals** — old prompt vs. new prompt, side-by-side preference.
- **Adversarial evals** — prompt injection, jailbreaks, off-policy requests.

**Run them in CI; gate releases on them.**

### 2.23 Security Surface

**What:** A coherent threat model and the controls that match it.

**Single-operator self-hosted (OpenClaw, local Claude Code):**
- Trust = the operator. Sandbox protects the host from agent mistakes, not from a malicious operator.
- Prompt injection is a UX issue, not a security boundary, *if* the operator chose to expose the tool.
- Secrets in OS keychain / `~/.config`.

**Multi-tenant (any hosted product):**
- Isolation between users is the boundary. Per-tenant sandboxes, per-tenant credentials, no shared state.
- Prompt injection IS a security boundary — agent must not exfiltrate cross-tenant data on adversarial input.
- Audit logs of tool calls, retained.

**Either case:**
- Redact secrets from logs, transcripts, and crash dumps.
- Permission system gates destructive ops.
- Rate limiting per identity.

### 2.24 Onboarding & Configuration

**What:** A guided setup flow that gets a new user from "installed" to "working" in one sitting.

**Why:** Agents have a lot of moving parts (provider keys, channels, skills, sandboxes). Without onboarding, drop-off is brutal.

**Minimum primitives:**
- An `init` / `onboard` command that walks: pick provider → enter key → test → pick channels → install skills → first conversation.
- Diagnostics command (`doctor`) that probes every dependency and reports actionable errors.
- Config repair / migration on version upgrade.

---

## Implementation Order (Pragmatic)

If you're building from scratch, this order has the highest payoff per week of work:

1. **Tier 0 in full** — week 1.
2. **Streaming + persistence + observability** (1.1, 1.2, 1.5) — week 2.
3. **A real tool suite + permissions** (2.1, 2.3) — week 3.
4. **Prompt caching + compaction** (1.6, 2.4) — week 4. (Cost goes from "scary" to "fine.")
5. **Subagents** (2.2) — week 5. (Quality jump is large.)
6. **MCP + skills** (2.8, 2.9) — week 6. (Extensibility unlocks community.)
7. **Memory** (2.5) — week 7. (UX jump.)
8. **Plan mode + hooks** (2.6, 2.7) — week 8.
9. **Eval infra** (2.22) — start week 1, but you'll only feel the value at week 8+.
10. Multi-channel, voice, advanced sandboxing — when the product needs them.

---

## Common Anti-Patterns

Things that look like progress but aren't:

- **Building a giant tool that does many things.** Always lose to small composable tools. Resist.
- **Letting the model write tools at runtime.** Looks magical; produces unmaintainable mess.
- **Vector memory as the only memory.** Embeddings retrieve *vibes*, not facts. Use file/structured memory for facts.
- **Token-by-token streaming everywhere.** Great in terminal, hostile on chat platforms.
- **Skipping deterministic ordering.** "Cache hit rate is fine probably" — measure it. It's not.
- **Doing your own JSON tool-call parsing on raw text.** Use the provider's native tool API. Hermes is the rare case where you implement the chat template; even then, follow the official template byte-for-byte.
- **Treating the system prompt as a dumping ground.** Skills + memory exist for a reason. A 30K-token system prompt cache-busts itself on every minor edit.
- **No max-turn cap.** One bug → unbounded loop → very bad bill.
- **Approvals UX that says "approve y/n?" with no detail.** Show the exact command, the diff, the file path. Otherwise users hammer `y` and the approval is theater.

---

## Where Each Reference Product Sits

| Capability | Claude Code | pi.dev | Hermes (model) |
|---|---|---|---|
| Tier 0 loop | ✅ | ✅ | model-side only |
| Rich tool suite | ✅✅✅ | minimal | n/a |
| Subagents | ✅ | ✅ (internal) | n/a |
| Permissions | ✅✅ | n/a (cloud) | n/a |
| Compaction | ✅ | ✅ | n/a |
| Memory | file-based | ✅✅✅ | n/a |
| Plan mode | ✅ | n/a | n/a |
| Skills | ✅ | n/a | n/a |
| MCP | ✅✅ | n/a | n/a |
| Hooks | ✅ | n/a | n/a |
| Voice | n/a | ✅✅✅ | n/a |
| Multi-channel | terminal-first | ✅✅ | n/a |
| Sandbox | local approvals | hosted | n/a |
| Function-call training | n/a | n/a | ✅✅✅ |
| Personality | terse | ✅✅✅ | model-side |

The tells:
- **Claude Code's edge** is a coherent tool/permission/hook/skill system targeted at coding workflows.
- **pi.dev's edge** is memory + voice + persona — UX, not capability.
- **Hermes's edge** is at the model layer — the harness around it can be any of the above.

Different products win on different axes. Decide which axis is yours before deciding which features to invest in.

---

## Minimum Viable "Advanced" Agent — Feature Matrix

If you want a single answer to "what's the smallest set that puts me in Tier 2?":

**Required:**
- Tier 0 + Tier 1 in full.
- Subagents (2.2).
- Permissions (2.3).
- Compaction (2.4).
- File-based memory (2.5a) — at minimum.
- Skills (2.8) — even a simple version.
- Prompt-cache-aware determinism (2.15).
- Eval infra (2.22).

**Pick at least one differentiator:**
- Coding-tier tools + sandboxing → Claude-Code-shaped product.
- Vector + structured memory + voice → pi.dev-shaped product.
- Trained function-calling model + open weights → Hermes-shaped product.

Trying to ship all three at once is how projects die.
