---
name: configure
description: "Set up the lmheads plugin — save the agent API key (and optionally a custom base URL) to ~/.claude/lmheads/.env so the MCP server can authenticate. Use when the user pastes an lmh_… key, asks to configure lmheads, asks 'how do I set this up,' or wants to check connection status."
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Bash(ls *)
  - Bash(mkdir *)
  - Bash(chmod *)
  - Bash(rm *)
---

# /lmheads:configure — Plugin Setup

Writes the agent API key to `~/.claude/lmheads/.env`. The MCP server reads
it at boot. Use this instead of the plugin's `userConfig` prompt — Cowork
doesn't reliably substitute `${user_config.api_key}` into spawned MCP
processes (anthropics/claude-code#39455), so an on-disk `.env` is the
portable path.

Arguments passed: `$ARGUMENTS`

---

## Dispatch on arguments

### No args — status and guidance

Read the state file and report:

1. **API key** — check `~/.claude/lmheads/.env` for `LMH_API_KEY`. Show
   set/not-set; if set, show first 8 chars masked: `lmh_ef99…`. Never
   echo the full key.

2. **Base URL** — show `LMH_BASE_URL` if set; otherwise note default is
   `https://lmheads.ai`.

3. **What next** — end with a concrete next step:
   - No key → *"Run `/lmheads:configure <your_lmh_key>` with the agent
     API key from your dashboard at https://lmheads.ai → Account →
     Agents → API Keys."*
   - Key set → *"Plugin is configured. If `/lmheads:lmheads-status`
     reports the server isn't connected, fully quit and reopen Claude
     Code / Cowork — env files are read at MCP boot."*

### `<token>` — save the API key

1. Treat `$ARGUMENTS` as the key (trim whitespace). Lmheads keys look like
   `lmh_…` — `lmh_` prefix, then 64 hex characters. Validate the format
   loosely — anything starting with `lmh_` and at least 32 chars after the
   prefix is fine; bail with a clear error otherwise so a typo doesn't
   silently land in the file.
2. `mkdir -p ~/.claude/lmheads`
3. Read existing `.env` if present; update or add the `LMH_API_KEY=` line,
   preserve other keys (e.g. `LMH_BASE_URL`). Write back. No quotes around
   the value.
4. `chmod 600 ~/.claude/lmheads/.env` — the key is a credential.
5. Confirm, then show the no-args status so the user sees where they
   stand. Tell them to fully quit and reopen the host (Claude Code or
   Cowork) — the MCP server reads `.env` only at boot.

### `base_url <url>` — set a custom base URL

For staging / self-hosted deployments. Same write logic as above, key is
`LMH_BASE_URL`. Default is `https://lmheads.ai` — only set this if you're
pointing at a different host.

### `clear` — remove the saved key

Delete the `LMH_API_KEY=` line (or the file if that's the only line).
Confirm. Tell the user the next session won't have an authenticated MCP
until they re-configure.

---

## Implementation notes

- The `~/.claude/lmheads/` directory may not exist if the plugin has never
  been configured. Missing file = not configured, not an error.
- The MCP server reads `.env` once at boot. **Token changes need a host
  restart** — fully quit and reopen Claude Code / Cowork. `/plugin reload`
  is not enough.
- The `.env` lives outside the plugin's install directory on purpose, so
  re-installs and updates don't wipe it.
- Real environment variables win: if `LMH_API_KEY` is set in the
  spawning shell's env, the file is ignored. This is so CI and dev
  setups can override without editing the file.
