# OpenClaw Architecture

OpenClaw is a **local-first, multi-channel AI agent gateway**. It listens on 15+ messaging platforms simultaneously, dispatches inbound messages to LLM agents (Claude, GPT-4, Gemini, etc.), and delivers replies back through the originating channel. A built-in scheduler fires cron jobs and heartbeats so the agent can act autonomously without human prompting.

---

## The Three Pillars

### 1. Listener &rarr; Dispatcher

Every messaging platform (Telegram, Discord, Slack, WhatsApp, Signal, iMessage, IRC, etc.) has a **channel listener** that converts platform-specific events into a normalized inbound message. The message flows through a pipeline:

```
Channel Listener
  → Message Preparation (debounce, media grouping, fragment assembly)
    → Route Resolution (agent selection, session key, access control)
      → Command Queue (lane-based serialization)
        → Pi Agent Runner (LLM call with tools)
          → Reply Dispatcher (stream blocks back to channel)
```

Key files:
- `src/channels/` — Channel dock abstraction and per-channel adapters
- `src/auto-reply/dispatch.ts` — `dispatchInboundMessage()` entry point
- `src/auto-reply/reply/reply-dispatcher.ts` — Serializes tool results, block replies, and final replies
- `src/routing/resolve-route.ts` — Determines which agent handles which message

### 2. Scheduler (Cron + Heartbeat)

Two independent timing mechanisms let the agent act without being spoken to:

**Cron Service** (`src/cron/`):
- Persistent job store with one-shot and recurring schedules (via `croner`)
- Each job fires an isolated agent turn with its own session context
- Optional webhook delivery (HTTP POST on job fire)
- Runs on its own command queue lane so it never blocks chat replies

**Heartbeat Runner** (`src/infra/heartbeat-runner.ts`):
- Per-agent periodic tick (configurable interval)
- Respects active-hours windows (e.g. only 9am–6pm)
- Fires a "no new messages" agent turn — the agent can check in, summarize, or stay silent
- Ghost reminder detection (notices when a user goes quiet)

### 3. Agent Core (the LLM bridge)

The Pi Embedded Runner (`src/agents/pi-embedded-runner/`) orchestrates multi-turn LLM sessions:

- **Model failover**: If Claude is down, fall through to OpenAI or Gemini
- **Auth profile rotation**: Round-robin API keys to avoid rate limits
- **Streaming**: Block-level chunking that respects word/code boundaries
- **Tool execution**: Browser (Playwright), bash (PTY), canvas (A2UI), cron manipulation, sub-session spawning
- **Context management**: Automatic transcript compaction when the context window fills up
- **Extended thinking**: Surfaces Claude's `<think>` reasoning blocks separately

Key dependency: `@mariozechner/pi-agent-core` (v0.52.12) — the underlying LLM orchestration library.

---

## Critical Abstractions

### Channel Dock

Every channel is described by a lightweight **Dock** — a metadata object declaring capabilities, routing rules, mention gating, and threading behavior. Four adapter interfaces compose the dock:

| Adapter | Responsibility |
|---------|---------------|
| `ChannelCommandAdapter` | Parse and execute slash commands |
| `ChannelGroupAdapter` | Group routing, mention gating |
| `ChannelThreadingAdapter` | Reply-to context, thread IDs |
| `ChannelMentionAdapter` | @mention stripping and validation |

This lets the core routing logic stay entirely platform-agnostic. Adding a new channel means implementing a dock and its adapters — the rest of the system doesn't change.

### Session Model

Sessions represent conversation state, keyed by a normalized string:

```
agent:<agentId>:<channel>:<channelSpecificId>
```

Example: `agent:default:telegram:123456789`

Sessions are persisted as JSON files under `~/.openclaw/sessions/`. The session system handles:
- File-level locking (prevent concurrent writes)
- Transcript compaction and repair
- Tool result guarding (prevent hallucinated tool outputs)

### Command Queue (Lane-Based Concurrency)

`src/process/command-queue.ts` implements a multi-lane async task queue:

| Lane | Concurrency | Purpose |
|------|-------------|---------|
| `main` | 1 (serial) | Chat auto-replies — preserves message order per session |
| `cron` | Configurable | Scheduled jobs — run in parallel, never block chat |
| `subagent` | Configurable | Agent-spawned sub-sessions |

Each lane has its own FIFO queue. Tasks within a lane execute up to `maxConcurrent` at a time. This prevents a flood of cron jobs from starving interactive replies.

---

## Gateway: The Control Plane

The Gateway (`src/gateway/`) is a combined WebSocket + HTTP server (default port 18789) that ties everything together:

**WebSocket**: Real-time bidirectional communication with the Control UI, mobile apps, and external nodes. JSON-RPC-style protocol (`{id, method, params}` → response/events).

**HTTP endpoints**:
- `POST /v1/chat/completions` — OpenAI-compatible API
- `POST /v1/responses` — OpenResponses API
- `POST /hooks/agent` — External webhook to send a message to an agent
- `POST /hooks/wake` — Trigger a heartbeat
- `GET /control/` — Serves the web-based Control UI

**Runtime registries**:
- `ChatRunRegistry` — Tracks in-flight agent runs per session
- `NodeRegistry` — Tracks connected external nodes (apps, satellites)
- `DispatcherRegistry` — Tracks pending reply dispatchers for graceful shutdown

---

## Delivery Guarantees

Outbound messages go through a **persistent disk-based delivery queue** (`src/infra/outbound/delivery-queue.ts`):

1. Message written to `~/.openclaw/delivery-queue/` before attempting send
2. Exponential backoff on failure: 5s → 25s → 2m → 10m (max 5 attempts)
3. Failed deliveries moved to `failed/` subdirectory for manual inspection
4. Gateway shutdown deferred until all pending deliveries complete

---

## Plugin & Hook System

OpenClaw is extensible through two mechanisms:

**Hooks** (`src/plugins/hooks.ts`):
- `beforeAgent` / `afterAgent` — Intercept or modify agent turns
- `onCompaction` — Custom logic during transcript compaction
- `modelOverride` — Dynamic model selection

**Plugins** (`src/plugins/`):
- Core plugins ship in `extensions/` (39+ channel and integration packages)
- Workspace plugins loaded from git repos or local paths
- Auto-discovered and registered at gateway startup

---

## Security Model

- **DM Pairing**: New senders must be approved before the agent responds (on by default)
- **Allowlists**: Per-channel sender allowlists (`allowFrom` config)
- **Auth Rate Limiting**: 20 failed auth attempts per 60-second window locks out
- **Tool Sandboxing**: Bash execution behind approval gates; tool policy enforcement
- **Local-First**: No cloud dependency — runs on the user's machine, keys stay local

---

## Directory Map

```
src/
  agents/           Pi agent runner, tool definitions, model resolution
  auto-reply/       Inbound message → agent → reply pipeline
  channels/         Channel dock abstraction, per-channel adapters
  gateway/          WebSocket + HTTP server, RPC methods, registries
  cron/             Scheduled job service with persistence
  infra/            Heartbeat, delivery queue, system events, diagnostics
  process/          Command queue, lane concurrency, PTY exec
  routing/          Session key resolution, route rules
  sessions/         Session persistence, locking, compaction
  config/           Config loading (Zod schema), validation, migration
  plugins/          Hook system, plugin registry, discovery
  browser/          Playwright-based browser tool
  canvas-host/      A2UI visual workspace server
  cli/              Commander.js CLI framework
  commands/         Individual CLI commands (gateway, agent, onboard, doctor)
  memory/           Session embeddings, context enrichment

extensions/         39+ channel and integration plugins (npm packages)
apps/               Native iOS, Android, macOS companion apps (Swift)
skills/             Bundled agent skills (tools agents can invoke)
ui/                 Web Control UI (Lit web components + Express)
docs/               Documentation (Mintlify-published markdown)
packages/           Monorepo workspaces
```

---

## What Makes It Unique

1. **Unified multi-channel abstraction**: One agent, one config, 15+ messaging platforms. The dock pattern keeps routing platform-agnostic.

2. **Local-first architecture**: Everything runs on the user's machine. No cloud intermediary. API keys and conversations stay local.

3. **Autonomous agent capabilities**: Cron jobs and heartbeats mean the agent can act on its own schedule — checking in, summarizing, or executing tasks without human prompting.

4. **Lane-based concurrency**: Chat replies stay serial per-session (preserving order), while cron and subagent work runs in parallel on separate lanes.

5. **Persistent delivery guarantees**: Disk-backed outbound queue with exponential backoff ensures messages survive crashes and network failures.

6. **Model-agnostic with failover**: Swaps between Claude, GPT-4, Gemini, and others. Auth profile rotation distributes load across API keys.

7. **Declarative YAML config**: A single config file defines agents, channels, routing, tools, cron jobs, and hooks. The `onboard` wizard and `doctor` command make it self-healing.
