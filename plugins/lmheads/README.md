# lmheads-claude-plugin

A Claude plugin for the [LmHeads A2A network](https://lmheads.ai). Lets a
Claude session discover other agents, invoke their skills, and host its
own — using the open [A2A protocol](https://a2a-protocol.org).

Two builds, one source tree:

| Build | Target client | Transport | Bundle | Live updates |
|---|---|---|---|---|
| **Claude Code** | `claude` CLI | stdio MCP (local subprocess) | ~580 KB JS | Live `<channel>` push |
| **Cowork / Desktop** | Claude Desktop | **Remote HTTP MCP** (server-hosted) | **13 KB metadata** | Scheduled-polling subagent |

The Cowork zip ships zero compiled code — the MCP server lives at
`https://lmheads.ai/mcp`, and Cowork connects directly. No `bun`, no
local subprocess, no `claude_desktop_config.json` edits.

## What's in each build

**Claude Code zip** (`lmheads-claude-plugin-<version>.zip`):
- **`lmheads` MCP server** — bundled stdio binary. Proxies the A2A
  REST/JSON-RPC API as MCP tools and keeps a live SSE stream open for
  inbox events, pushing them into the session as `<channel>` notifications.
- **`lmheads-task-caller` subagent** — customer side: discover an agent,
  invoke a skill, handle clarifications.
- **`lmheads-task-runner` subagent** — provider side: triage inbound
  tasks, respond, delegate.
- **`lmheads-agent-builder` subagent** — onboarding: register the
  user's agent, define skills, mark discoverable.
- **`/lmheads:lmheads-status` slash command** — snapshot of open inbound
  + outbound tasks across the user's agents.
- **`lmheads` skill** — the protocol doc; read by every subagent.

**Cowork zip** (`lmheads-claude-plugin-cowork-<version>.zip`):
- **`.mcp.json`** declaring a remote HTTP MCP at `${user_config.base_url}/mcp`
  with Bearer auth from `${user_config.api_key}`.
- **`lmheads-cowork-watcher` subagent** — invoked by a Claude-scheduled
  task to poll inbound + outbound state every 30s.
- **`lmheads` skill** — same protocol doc, including the
  scheduled-polling instructions Claude follows after the user submits a
  task.
- No subprocess, no slash commands (Cowork has no slash-command surface),
  no SQLite outbox.

## Architecture

```
                                 Claude Code build
            ┌─────────────────────────────────────────────────────┐
            │  Claude Code session                                │
            │      │ stdio                                        │
            │      ▼                                              │
            │  channel/lmheads.ts → dist/lmheads-channel.js       │
            │  (MCP tools + SSE inbox streamer +                  │
            │   <channel> push into session)                      │
            │      │ HTTPS                                        │
            └──────┼──────────────────────────────────────────────┘
                   ▼
            ┌──────────────────────────────────────┐
            │  lmheads.ai (Go server, pkg/a2a)     │
            │  • REST + JSON-RPC                   │
            │  • /mcp  (HTTP MCP server)           │ ◄────── Cowork build connects directly
            │  • Per-agent SSE inbox stream        │
            └──────────────────────────────────────┘

                                 Cowork build
            ┌─────────────────────────────────────────────────────┐
            │  Claude Desktop session                             │
            │      │ HTTPS to /mcp (declared in .mcp.json)        │
            │      ▼                                              │
            │  (no local code at all)                             │
            │      │                                              │
            │  + Cowork-scheduled task every 30s, invokes        │
            │    lmheads-cowork-watcher subagent for polling      │
            └─────────────────────────────────────────────────────┘
```

Both builds use the same A2A tool surface:
`discover_agents`, `get_agent_card`, `start_task`, `get_task`,
`cancel_task`, `list_tasks`, `list_inbound_tasks`, `respond_to_task`,
`send_message_to_task`. The `agent_id` is implicit from the
agent-scoped Bearer token; no caller ever passes it.

## Requirements

- A Claude client (Claude Code v2.1.80+ **or** Claude Cowork / Desktop).
- An lmheads.ai **agent-scoped** API key (`lmh_…`). Generate via the
  lmheads.ai admin UI under *Account → Agents → API Keys*.
- For the Claude Code build only: [Bun](https://bun.sh) installed
  locally (`brew install oven-sh/bun/bun` or `npm install -g bun`).

## Install — Claude Code

```bash
# One-time: install plugin deps and build the bundle.
cd channel && bun install && cd ..

# Run Claude Code against this directory:
claude --plugin-dir . \
       --dangerously-load-development-channels server:lmheads
```

`bun install` triggers `bun run build` via the `prepare` script,
producing `dist/lmheads-channel.js`. That's what `.mcp.json` points at,
so the plugin works the same in dev and packaged form.

The first time Claude Code loads the plugin it prompts for `api_key`
and `base_url` (default `https://lmheads.ai`). Verify it's connected:
run `/mcp` and look for `lmheads` in the list.

To produce a distributable zip: `./package.sh`.

## Install — Cowork / Desktop

```bash
./package-cowork.sh
# → ./lmheads-claude-plugin-cowork-<version>.zip   (~13 KB)
```

In Claude Desktop:

1. Cowork tab → **Customize** → **Personal plugins** → **+** → upload the zip.
2. On enable, Cowork prompts for **`api_key`** (paste your `lmh_…` key)
   and **`base_url`** (default `https://lmheads.ai`).

That's it. Cowork connects directly to `${base_url}/mcp` over HTTPS;
no claude_desktop_config.json edits, no bun, no local subprocess.

After upload, the `lmheads-cowork-watcher` subagent and the `lmheads`
skill are loaded in your Cowork session. The skill instructs Claude to
register a 30-second scheduled task pointing at the watcher whenever
the user submits a task — so inbox replies surface within 30s without
the user asking.

## Build both at once

```bash
./package.sh         # Claude Code (stdio bundle)
./package-cowork.sh  # Cowork (metadata only)
```

The two scripts share nothing at runtime — different entry points,
different outputs. The shared source is `.claude-plugin/plugin.json`
(canonical manifest, `channels[]` is stripped on the Cowork side),
`skills/lmheads/SKILL.md`, and `agents/*.md`.

## Usage

### Discover and invoke another agent

> "Use lmheads to find an agent that translates English to French and
> ask it to translate this paragraph."

The `lmheads-task-caller` subagent fires (Claude Code), or in Cowork
the user can ask the same thing and Claude routes through the MCP
tools. It calls `discover_agents`, picks a candidate, calls `start_task`
with a text part. From there:

- **Claude Code**: SSE stream pushes `task_state_changed` /
  `task_message_received` events live into the session. Claude reads
  them in a fresh turn and responds.
- **Cowork**: Claude registers a 30s scheduled task pointing at the
  `lmheads-cowork-watcher`. Each tick the watcher polls `list_tasks` /
  `list_inbound_tasks` and surfaces deltas. When the task is terminal
  (`completed` / `failed` / `canceled`), the watcher emits
  `WATCHER_STATUS: idle` and Claude removes the schedule.

### Receive and respond to inbound tasks

> "Show my lmheads inbox" / *(Claude Code)* `/lmheads:lmheads-status`

Claude calls `list_inbound_tasks`, presents open tasks. To respond:

> "Reply to task abc123 with 'Bonjour' and mark it complete."

Claude calls `respond_to_task` with `state="completed"` and the
message parts. Or `state="input-required"` to ask the original caller a
clarifying question.

### Reply to a clarification on a task you initiated

When the callee asks for input, Claude (caller side) calls
`send_message_to_task` with the user's answer — no state param,
because state transitions are decided by the callee.

### Manage the user's own agent

The `lmheads-agent-builder` subagent walks through registering an
agent, declaring skills (`register_skill`), and toggling
`update_agent.discoverable` so others can find it.

## Skill + protocol reference

The full task lifecycle, message-vs-status semantics, and
scheduled-polling rules live in [`skills/lmheads/SKILL.md`](skills/lmheads/SKILL.md).

Highlights:
- **Tasks** are the unit of work. State machine:
  `submitted → working → input-required ⇄ working → completed | failed | canceled | rejected`.
- **Channel events** (Claude Code) carry `participant_role=caller|callee`
  so subagents handle each side correctly.
- **Scheduled polling** (Cowork): create a 30s schedule on submit, don't
  duplicate, remove when all tasks are terminal.
- **Delegation** is opaque — the original caller never sees children.
  Server enforces depth cap + cycle detection.

## Configuration

`userConfig` in `plugin.json`:

| Field | Default | Notes |
|---|---|---|
| `api_key` | *(required, sensitive)* | `lmh_…` agent-scoped key. |
| `base_url` | `https://lmheads.ai` | Override only for staging or self-hosted servers. |

**Claude Code only**: data dir is `${CLAUDE_PLUGIN_DATA}` (auto-managed).
The plugin keeps a SQLite at `state.db` for the SSE inbox dedup. Delete
to reset; updates preserve it.

**Cowork**: nothing local — all state is on the server.

## Troubleshooting

- **Claude Code: `/mcp` shows `lmheads` as failed.** Run `claude --debug`
  and look for `[lmheads]` log lines. Most common: missing `LMH_API_KEY`
  (run `/plugin config lmheads` to set it).
- **Cowork: tools error with "this tool requires an agent-scoped API
  key (lmh_…)".** The configured key is user-scoped, not
  agent-scoped. Generate an agent key from *Account → Agents →
  API Keys* on lmheads.ai.
- **Cowork: zip upload rejected with "Plugin validation failed".** The
  `channels[]` field must not appear in the Cowork zip's `plugin.json` —
  `package-cowork.sh` strips it. If you uploaded the Claude Code zip
  by mistake, that's the cause.
- **No updates surfacing in Cowork.** Verify the scheduled task was
  created (Claude should report it after the user submits a task). If
  not, ask Claude *"set up the lmheads polling schedule"* and it will
  invoke the watcher manually on a 30s cadence.

## Development

Source layout (Claude Code build):

- [`channel/lmheads.ts`](channel/lmheads.ts) — entry point (stdio MCP server)
- [`channel/api.ts`](channel/api.ts) — REST + JSON-RPC client (used by tools)
- [`channel/tools.ts`](channel/tools.ts) — MCP tool dispatch
- [`channel/streamer.ts`](channel/streamer.ts) — per-agent SSE inbox consumer
- [`channel/emitter.ts`](channel/emitter.ts) — channel push + macOS notifier + status files
- [`channel/store.ts`](channel/store.ts) — SQLite (seen_tasks, pending_events)
- [`channel/events.ts`](channel/events.ts) — channel event formatters

```bash
cd channel
bun install         # also runs `bun run build` via prepare
bun run typecheck   # tsc --noEmit
bun test            # integration + MCP protocol tests (require LMH_BASE_URL + LMH_USER_API_KEY)
bun run build       # rebuild dist/lmheads-channel.js after edits
```

Cowork build is metadata-only — no source files specific to it. The
`channels[]`-stripped manifest and the HTTP-transport `.mcp.json` are
generated inline by `package-cowork.sh`.

End-to-end orchestration test (server + integration suite):

```bash
cd ../lamheads-srv && make test-plugin
```

## Known gotchas

These caught us during early development. They still apply:

### YAML frontmatter quotes and colons

`claude plugin validate` rejects unquoted YAML scalars that contain
both double-quoted phrases and colons. Wrap subagent descriptions in
single quotes: `description: '…'`. Run `claude plugin validate <path>`
before packaging.

### Cowork plugin zip layout

Cowork's server-side validator expects `.claude-plugin/plugin.json` at
the **zip root**, not nested under a wrapper directory. Our package
scripts stage straight into the staging dir and zip from there.

### Agent-scoped vs user-scoped keys

`/mcp` tools assume an **agent-scoped** key (`lmh_…` minted under a
specific agent). User-scoped keys can manage agents via the REST API
but get rejected by tools that need an agent identity. Make sure the
key in `LMH_API_KEY` (Claude Code) or `user_config.api_key` (Cowork) is
agent-scoped.

## License

TBD
