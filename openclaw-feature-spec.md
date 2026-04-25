# OpenClaw — Feature Specification

**Source:** https://github.com/openclaw/openclaw
**Docs:** https://docs.openclaw.ai/
**Local checkout analyzed:** `/Users/jhnlsn/wk/github.com/ai/openclaw` (shallow clone of `main`)
**Date:** 2026-04-25

---

## 1. Product Summary

OpenClaw is a self-hosted **personal AI assistant** designed to be run on the operator's own devices. It is explicitly *single-operator* (not multi-tenant): one gateway serves one user and their trusted circle of channels and providers. The product surface is a set of always-on chat/voice integrations, an interactive Canvas, native mobile/desktop apps, and a wake-word listener — all driven by a TypeScript core called the **Gateway**, which is described as "just the control plane — the product is the assistant."

**Tagline:** "EXFOLIATE! EXFOLIATE!"
**Wake word:** `clawd` (aliases: `claude`)
**Runtime:** Node 24 (recommended) or Node 22.14+, pnpm workspaces.
**Versioning:** Calendar `YYYY.M.D` (with optional `-beta.N`).
**License:** MIT.

---

## 2. High-Level Architecture

| Component | Path | Purpose |
|---|---|---|
| **Gateway (core)** | `src/gateway/` | Central orchestration: routing, model management, plugin lifecycle, RPC. |
| **Agents engine** | `src/agents/` | Multi-turn conversations, tool use, subagent spawning, streaming. |
| **Context engine** | `src/context-engine/` | Memory search, retrieval, RAG. |
| **Canvas host** | `src/canvas-host/` | Live interactive HTML/web rendering surface, separate from message channels. |
| **CLI** | `src/cli/` | Terminal-first control plane (200+ commands). |
| **Extensions** | `extensions/` | 124+ bundled plugins (channels, providers, tools). |
| **Skills** | `skills/` | 50+ task-domain skill bundles. |
| **Apps** | `apps/` | macOS, iOS, Android, macOS-MLX-TTS. |
| **Plugin SDK** | `packages/` (`openclaw/plugin-sdk/*`) | Public facades for plugin authors. |
| **Swabble** | `Swabble/` | Swift wake-word daemon for macOS 26 / iOS. |
| **Sandbox** | `Dockerfile.sandbox*`, `docker-setup.sh` | Container-isolated code/browser execution. |

**Boundary rule:** Plugins access core only through the public `openclaw/plugin-sdk/*` exports. Core never hardcodes plugin IDs.

---

## 3. Channels & Messaging Integrations

OpenClaw can read/write the operator on whichever channels they already use. Each channel is a plugin in `extensions/`.

### 3.1 Chat / Messaging

- WhatsApp
- Telegram
- Slack
- Discord
- Google Chat
- Signal
- iMessage (via BlueBubbles bridge)
- IRC
- Microsoft Teams
- Matrix
- Mattermost
- Feishu
- LINE
- Nextcloud Talk
- Synology Chat
- Tlon (Urbit)
- Twitch (chat)
- Nostr (decentralized)
- WeChat (qqbot)
- QQ
- Zalo / Zalo Personal
- WebChat (built-in browser channel)
- Bonjour (LAN-local)

### 3.2 Voice / Realtime

- Voice Call (`voice-call`) — realtime voice agent.
- Google Meet (`google-meet`).
- Realtime Transcription over WebSocket.

### 3.3 Capabilities Per Channel (typical)

Text, images, threads/replies, reactions, group/broadcast, and channel-specific media rendering. Exact feature support is per-adapter; not every channel exposes every capability.

---

## 4. Model Providers & LLM Features

### 4.1 Providers (50+)

**Hosted commercial:** Anthropic, OpenAI, Google (Gemini, Vertex AI), Mistral, Groq, Together, Perplexity, Fireworks, xAI, DeepSeek, Cohere-class providers, OpenRouter, Vercel AI Gateway, Cloudflare AI Gateway, HuggingFace Inference, LiteLLM proxy.

**China-region:** Qwen (Alibaba), Qianfan (Baidu), MiniMax, Moonshot, Stepfun, Tencent, Volcengine, BytePlus, Zai.

**Local / self-hosted:** Ollama, LMStudio, ComfyUI (image/video), and any OpenAI-compatible endpoint.

**Specialty / niche:** Arcee, Inferrs, Kilocode, Kimi-Coding, Venice, Runway (video), FAL (image/video).

**OAuth subscriptions:** OpenAI (ChatGPT/Codex).

### 4.2 LLM Capabilities

- Multi-turn tool use with 100+ built-in tools.
- Streaming responses; soft-chunk streaming with paragraph preference.
- Token-saving compaction (summarization fallback when context approaches limits).
- Block replies (chunked output without token-delta messages).
- Reasoning/thinking models (OpenAI o-series, Claude thinking) supported explicitly.
- Model aliases, auth profile rotation, cooldown/expiry, failover.
- Automatic catalog discovery per provider.

### 4.3 Memory

- Pluggable memory backends.
- Implementations include LanceDB-backed, wiki-style, and active-memory.
- Session-level and cross-session recall.

### 4.4 Vision & Media Understanding

- Image analysis.
- OCR via document extraction.
- Media Understanding Core for video/audio comprehension.

### 4.5 Image, Video, Music Generation

- **Image:** FAL, Runway, ComfyUI (via Image Generation Core).
- **Video:** Runway, FAL, ComfyUI (via Video Generation Core).
- **Music:** Provider registry exists; specific providers not enumerated in this pass.
- Async task polling for long-running generation jobs.

---

## 5. Voice (STT / TTS / Realtime)

### 5.1 Speech-to-Text

- Deepgram
- OpenAI Whisper (API + local CLI variants)
- SenseAudio
- Realtime Transcription over WebSocket

### 5.2 Text-to-Speech

- ElevenLabs
- Sherpa ONNX TTS (on-device)
- Local CLI TTS (`tts-local-cli`)
- Apple native (macOS 26)
- macOS MLX TTS app (separate native app using the MLX framework)

### 5.3 Realtime Voice

- Realtime Voice SDK with audio codec support.
- Agent consult mode (mid-conversation handoff).
- Wake-word activation via Swabble (see §11.1).

---

## 6. Skills System

Skills live in `skills/`, are TypeScript/ESM packages with a `SKILL.md` metadata file and an optional `onboard.js`. Skills are runtime-loadable. The project's stated direction is that net-new skills should ship through ClawHub rather than be bundled by default.

### 6.1 Bundled Skill Categories (50+ total)

- **Productivity:** Apple Notes, Apple Reminders, Bear Notes, 1Password, Notion, Obsidian, Trello, Things.
- **Communication:** GitHub (issues/PRs), Slack, Discord, imsg (iMessage), Telegram.
- **Development:** Coding Agent, GitHub Issues, Session Logs, Skills Creator, TaskFlow (workflow / inbox triage).
- **Media:** Camsnap, Video Frames, Gif Grep, Songsee (music ID), Spotify Player, Sonos Control.
- **Integrations:** Himalaya (email), Blogwatcher, Notion, Canvas.
- **System / Utilities:** Health Check, Model Usage, Weather, 8Ctl, Blucli (Bluetooth), TMux, Node Connect.
- **Search / Discovery:** GoPlaces, Xurl, Oracle.
- **Audio / Transcript:** OpenAI Whisper (CLI + API), Sherpa ONNX TTS.

---

## 7. Tools & Web/Document Capabilities

### 7.1 Search & Web

- Brave, DuckDuckGo, Exa, Tavily, SearXNG.
- Web Readability extraction.
- Link Understanding (URL summarization).

### 7.2 Document Processing

- Firecrawl (site → markdown).
- Document Extract.
- Nano PDF.

### 7.3 Browser

- Browser plugin with `evaluate` / script execution.
- Sandboxed browser context via `Dockerfile.sandbox-browser`.

### 7.4 Device & OS Control

- Device Pair (iOS/Android pairing).
- Phone Control.
- Xiaomi device control.

---

## 8. Canvas

- Live, interactive HTML/web rendering surface, distinct from message channels.
- A2UI bundle hash for reproducibility (`src/canvas-host/a2ui/.bundle.hash`).
- Supports `canvas.eval` directives and custom interactions (operator-trusted).
- Embedded in mobile apps via WKWebView (iOS) and Android equivalent.

---

## 9. Apps

| App | Platform | Notes |
|---|---|---|
| OpenClaw Desktop | macOS | Sparkle auto-updates, code signing, TCC permissions, `scripts/restart-mac.sh`. |
| OpenClaw Mobile | iOS | Native Swift, Canvas in WKWebView. |
| OpenClaw Mobile | Android | Kotlin/Gradle. |
| OpenClaw MLX TTS | macOS | Native local TTS using Apple's MLX framework. |
| Swabble | macOS 26 / iOS | Wake-word daemon (see §11.1). |

**Pairing:** Plaintext WebSocket allowed only on loopback or with `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1`; TLS (`wss://`) recommended for Tailscale or public networks.

---

## 10. CLI & Onboarding

The CLI is the primary control surface. Entry point: `openclaw.mjs`; sources in `src/cli/`. ~200+ commands, grouped:

- **Onboarding:** `openclaw onboard` (interactive wizard for gateway, workspace, channels, skills).
- **Config:** `openclaw config`, secrets, env management.
- **Plugins:** `openclaw plugins install|list|uninstall|update`.
- **Channels:** `openclaw channels` + per-channel auth flows.
- **Gateway control:** `openclaw gateway start|stop|restart|status`, RPC.
- **Models:** `openclaw models` + provider discovery and auth-profile management.
- **Execution:** `openclaw run`, node spawn, sandbox control.
- **Skills:** `openclaw skills list|info|exec`.
- **Security:** `openclaw security`, `openclaw secrets`, `openclaw exec-policy`.
- **Diagnostics:** `openclaw doctor`, `openclaw health`, `openclaw logs`, `openclaw docs`.
- **Shell:** completion scripts for Fish, Bash, Zsh.

Recommended setup path: `openclaw onboard` on macOS, Linux, or Windows-via-WSL2. Works with npm, pnpm, or bun.

---

## 11. Notable Subsystems

### 11.1 Swabble (Wake-Word Daemon)

- Swift 6.2, macOS 26 / iOS.
- On-device speech recognition via Apple `Speech.framework` (zero network).
- Wake word: `clawd` (aliases: `claude`).
- Spawns hook commands with configurable prefix/env/cooldown on detection.
- **SwabbleKit:** shared iOS/macOS utilities for wake-word gating.
- CLI subcommands: `serve`, `transcribe`, `test-hook`, mic management, `doctor`, launchd integration.

### 11.2 Canvas Host

See §8.

### 11.3 Dream Diary

- Two HTML preview files exist in repo root (`dream-diary-preview-v2.html`, `-v3.html`).
- Functionality not documented in repo; treat as preview/experimental.

### 11.4 QA Infrastructure

- Vitest-based tests.
- Live tests gated behind `OPENCLAW_LIVE_TEST=1`.
- Dedicated QA Lab / QA Matrix / QA Channel extensions for end-to-end testing.

### 11.5 Streaming & Compaction

- Soft-chunk streaming, paragraph-preferring.
- No token-delta channel messages — explicit chunked output with edits.
- Compaction summarizes context to extend effective window.

---

## 12. Sandboxing & Execution

| Mode | Description |
|---|---|
| Host bash | Local shell with PTY support, approval gating per policy. |
| Sandbox Node | Containerized Node.js (`Dockerfile.sandbox`). |
| Sandbox Browser | Isolated browser context (`Dockerfile.sandbox-browser`). |
| System call | `system.run` for OS-level commands, owner-gated. |

**Common base:** `Dockerfile.sandbox-common`. **Setup:** `docker-setup.sh`, `setup-podman.sh`. Both Docker and Podman are first-class.

**Tool policy:**
- Allow/blocklist per agent.
- Approval workflow for restricted ops.
- Workspace path guards (no host-root escape).
- Owner-only gating for sensitive operations.

---

## 13. Deployment

**Cloud:** Fly.io (`fly.toml`, `fly.private.toml`), Render (`render.yaml`), Heroku, DigitalOcean, Azure, GCP, Oracle Cloud, Hetzner, Hostinger.
**Container:** Docker, Docker Compose (`docker-compose.yml`), Podman, Kubernetes.
**Bare metal:** Node.js install, Raspberry Pi, macOS VM, Windows (exe / WSL2).
**IaC:** Ansible, Nix (`nix-openclaw` companion repo), custom scripts.
**Specialized:** ClawDock (Docker wrapper), `installer script (exe-dev)`.

**Updates:** Sparkle appcast (`appcast.xml`) for the macOS app.

---

## 14. Plugin / Extension SDK

### 14.1 Plugin Types

- **Channel plugins** — chat/voice integrations.
- **Provider plugins** — LLM/model adapters.
- **Tool plugins** — register new tools to the agent toolkit.
- **Memory plugins** — alternative memory backends.
- **Skill bundles** — packaged task libraries.
- **MCP servers** — exposed by OpenClaw or consumed from external sources.

### 14.2 SDK Surface

- `openclaw/plugin-sdk/channel-contract`
- `openclaw/plugin-sdk/provider-entry`
- `openclaw/plugin-sdk/plugin-entry`
- `openclaw/plugin-sdk/core`
- `openclaw/plugin-sdk/sandbox`

### 14.3 Discovery & Lifecycle

- Manifest-driven via `openclaw.plugin.json`.
- Bundle-style plugins (stable SKU) vs. code plugins (runtime hooks).
- Activation/deactivation lifecycle, before/after-tool-call hooks, system-prompt contribution hooks, auth/credential hooks.
- Zod schemas for config validation; types exported from `api.ts` / `runtime-api.ts` barrels.

### 14.4 MCP

- Bidirectional: OpenClaw can host MCP servers AND consume external MCPs.
- Embedded MCP runtime with stdio, HTTP, and Chutest proxy transports.
- LSP embedding for select tools.

---

## 15. Security & Trust Model

### 15.1 Trust Model (per `SECURITY.md`)

- Single-operator gateway — not multi-tenant.
- Operator intent is the trust boundary: features such as `canvas.eval`, browser script, and `node.invoke` are **operator-trusted by design**, not bypasses.
- Prompt injection of trusted features is explicitly out of scope as a vulnerability class — the operator chooses what to expose.

### 15.2 Authentication

- OAuth flows (e.g., Chutes integration) for channels/providers.
- API keys in `~/.openclaw/credentials/` and via env vars.
- Auth profiles with rotation, cooldown, and fallback.
- Gateway HTTP auth: token or password mode (`gateway.auth.mode`). Bearer auth = full operator access (no per-scope narrowing).
- ACP tool approval system with narrowed read-only classes.

### 15.3 Disclosure

- Private reports to `security@openclaw.ai` (or per-repo security contact).
- Required: title, severity, impact, demonstrated impact, remediation, environment.
- No bug bounty.

---

## 16. Build, Workspace & Repo Conventions

- **Package manager:** pnpm with workspaces (`pnpm-workspace.yaml`).
- **TypeScript projects:** layered `tsconfig.*.json` (core, extensions, tests, oxlint, plugin-sdk dts, scripts, ui).
- **Bundler / build:** `tsdown.config.ts`.
- **Lint/format:** oxlint configs; pre-commit formatting only (no validation enforcement at hook time).
- **Tests:** Vitest (`vitest.config.ts`); live-mode gated by `OPENCLAW_LIVE_TEST=1`.
- **CI workflows:** GitHub Actions (badge in README); `zizmor.yml` for workflow security review.
- **Patches & vendor:** `patches/`, `vendor/` for upstream pins.
- **Repo guides:** `AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`, `INCIDENT_RESPONSE.md`, `VISION.md`.

---

## 17. Documentation Map (https://docs.openclaw.ai/)

The README and docs site cross-link to:

- Getting Started — `/start/getting-started`
- Onboarding wizard — `/start/wizard`
- Showcase — `/start/showcase`
- Updating — `/install/updating`
- Docker install — `/install/docker`
- FAQ — `/help/faq`
- Onboarding — `/start/onboarding`
- DeepWiki mirror — https://deepwiki.com/openclaw/openclaw
- Nix port — https://github.com/openclaw/nix-openclaw
- Discord community — https://discord.gg/clawd

---

## 18. Open Questions / Gaps Not Resolved in This Pass

1. **Dream Diary** — preview HTMLs exist; no in-repo docs explaining the feature.
2. **Music generation** — provider registry confirmed; specific providers not enumerated.
3. **Canvas eval safety** — full security trace of `canvas.eval` not completed.
4. **Subagent spawning** — max nesting depth and resource limits not located.
5. **ClawHub** — third-party plugin index/discovery mechanism referenced but not explored.
6. **A handful of skills** (e.g., `peekaboo`, `ordercli`, `wacli`) — exact behavior would require source-level reads beyond this pass.

---

## 19. Coverage Summary

| Category | Count |
|---|---|
| Bundled extensions/plugins | 124+ |
| Messaging/voice channels | 30+ |
| LLM / model providers | 50+ |
| Bundled skills | 50+ |
| CLI commands | ~200 |
| Sandbox/execution modes | 4 (host bash, sandbox node, sandbox browser, system call) |
| Native apps | 4 (macOS desktop, iOS, Android, macOS MLX TTS) + Swabble |
| Documented deployment targets | 10+ |

---

*Generated from a thorough exploration of the cloned repository at `main`. For up-to-date capabilities, cross-reference https://docs.openclaw.ai/ and the project's CHANGELOG.md.*
