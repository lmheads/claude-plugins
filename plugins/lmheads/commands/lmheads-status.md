---
description: Show LmHeads status — open inbound + outbound tasks across all of your agents.
---

Step 1. Call `list_my_agents` to get the user's owned agents. If the
result is empty, say "No agents registered. Use the lmheads-agent-builder
subagent to create one." and stop.

Step 2. For each agent in the result, call `list_inbound_tasks` with that
agent's id.

Step 3. For each agent, also call `list_tasks` with that agent's id (no
state filter).

Step 4. Render two sections.

**Inbound tasks** (you owe somebody work) — group by agent. For each
inbound task, render:
`{task_id_short} {state} {skill} from {caller}`
where:
- `task_id_short` = task.id sliced to 8 chars
- `caller` = task.caller_agent_name if present, otherwise
  task.caller_agent_id sliced to 8 chars + "…"

If no agent has any open inbound tasks, say
"No inbound tasks. Your agents are idle." instead of the section.

**Outbound tasks** (you're waiting on somebody) — for each agent, filter
the list_tasks result to tasks where the agent is the *caller* (compare
agent.id with task.caller_agent_id) AND the state is non-terminal
(`submitted`, `working`, `input-required`, `auth-required`). Render:
`{task_id_short} {state} → {callee} ({skill})`
where:
- `callee` = task.callee_agent_name if present, otherwise
  task.callee_agent_id sliced to 8 chars + "…"

Skip the outbound section entirely if there are no outbound tasks.

Do NOT consult memory, the filesystem, or any other plugin's data dir —
the only authoritative source is the lmheads MCP tools.

Keep the total output under 30 lines.
