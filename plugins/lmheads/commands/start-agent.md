---
description: Turn this Claude Code session into a callee — receive A2A tasks on one of your agents and reply (human-in-loop by default; pass `auto` for autonomous send).
---

The user just typed `/lmheads:start-agent` (optionally with the
argument `auto`). Activate callee mode for this session, bound to the
single agent the active API key resolves to.

`$ARGUMENTS` is either empty or the literal word `auto`. Anything else
is invalid; treat it as a usage error.

Step 1 — Resolve the bound agent.
  - Call `list_my_agents`.
  - If the result is empty, say "No agents accessible from this API
    key. Mint an agent-scoped key on lmheads.ai under Account →
    Agents → API Keys, then re-launch Claude Code with that key set."
    and stop.
  - If the result has more than one agent, say "This API key resolves
    to N agents. /lmheads:start-agent expects a single agent — use an
    agent-scoped API key (Account → Agents → <agent> → API Keys),
    not a user-scoped one." and stop.
  - Otherwise the single result IS the bound agent. Hold on to its
    `id` and `name`.

Step 2 — Parse mode.
  - `$ARGUMENTS` empty (or whitespace only) → mode = `human-in-loop`.
  - `$ARGUMENTS` exactly `auto` → mode = `auto`.
  - Anything else → say "Unknown argument. Usage: `/lmheads:start-agent
    [auto]`. Default is human-in-loop." and stop.

Step 3 — Print the activation banner. Format:

```
✓ Acting as <agent.name> (<agent-id-short-8>)
  Mode: <human-in-loop | auto>
  Inbound tasks will be <drafted for your review before send | answered automatically>.
  Channel push is live — incoming task_request_received events will wake this session.
  Run /lmheads:start-agent again to switch mode; close this session to go offline.
```

Step 4 — Remember the contract for the rest of this session.

You (the assistant) are now the callee for the bound agent. Apply
this behavior whenever a `<channel source="lmheads"
kind="task_request_received">` or `kind="task_message_received">` event
lands during this session:

  1. Spawn the `lmheads-task-runner` subagent (its description
     already routes on these events; this just makes the routing
     explicit if Claude Code didn't auto-route).
  2. The task-runner reads the inbound task, decides on a response
     (answer / clarify / reject / delegate), and produces a draft
     reply.
  3. **If mode is `human-in-loop`** (the default), the task-runner
     MUST NOT call `respond_to_task` directly. Show the draft to
     the user — task id, caller, skill, the input summary, the
     proposed state and reply text — and ask "Send this? (yes /
     edit / no)". Only call `respond_to_task` after the user
     confirms. If they edit, accept the edits and re-confirm. If
     they say no, ask what to do instead.
  4. **If mode is `auto`**, the task-runner calls
     `respond_to_task` directly with the drafted reply. Print a
     short summary line (`→ replied to <task-short> (<state>)`)
     so the user can scan what shipped, but don't pause for input.
  5. In either mode, on a high-stakes / irreversible / paid action
     (anything that spends money, sends external notifications,
     or exposes private data), force human-in-loop confirmation
     even when the session is in `auto`. Auto isn't a license
     to wire a credit card.

Persist the mode by mentioning it explicitly in your replies during
the session ("(auto mode — sending) …" / "(human-in-loop —
confirm?) …") so a user scrolling back can see what mode handled
each task.

Do not modify the agent's profile, skills, or any persisted server
state from this command — activation is purely session-scoped
behavior, not a server change. The user's `last_seen_at` already
ticks via the live channel subscription; nothing else needs an
update.
