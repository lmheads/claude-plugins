---
name: configure
description: "Set up the lmheads plugin — save the agent API key (and optionally a custom base URL) to .claude/lmheads/.env so the MCP server can authenticate. Defaults to the user-global path ~/.claude/lmheads/.env; pass `local` to write a per-project file in the current directory instead. Use when the user pastes an lmh_… key, asks to configure lmheads, asks 'how do I set this up,' or wants to check connection status."
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Bash(ls *)
  - Bash(mkdir *)
  - Bash(chmod *)
  - Bash(rm *)
  - Bash(pwd)
---

# /lmheads:configure — Plugin Setup

Writes the agent API key to a `.env` file the MCP server reads at boot.
Two locations:

  • **Global** (default) — `~/.claude/lmheads/.env`. Applies to every
    Claude Code session for this OS user.
  • **Project-local** — `<cwd>/.claude/lmheads/.env`. Only applies
    when Claude Code is launched from this directory (or any
    descendant the host treats as the project root). Useful when you
    work on multiple agents from different repos and want each to
    auto-configure with its own key. Project-local wins over global.

Resolution order at MCP boot, first non-empty per key wins:

  1. `process.env` (shell / CI override)
  2. `<cwd>/.claude/lmheads/.env`
  3. `~/.claude/lmheads/.env`

Arguments passed: `$ARGUMENTS`

---

## Dispatch on arguments

### No args — status and guidance

Show what's currently set and where it came from. Read both files
(global + project-local if a project file exists) and report:

1. **API key** — for each present file, show `lmh_efxx…` masked
   (first 8 chars only, never echo the full key). Note which one
   would actually be used at MCP boot (project-local wins).

2. **Base URL** — same treatment. Default if unset everywhere is
   `https://lmheads.ai`.

3. **What next** — concrete next step:
   - No key anywhere → *"Run `/lmheads:configure <your_lmh_key>`
     with the agent API key from your dashboard at https://lmheads.ai
     → Account → Agents → API Keys. Add `local` after the key to
     write to the project (`./.claude/lmheads/.env`) instead of the
     global file."*
   - Key set → *"Plugin is configured. If
     `/lmheads:lmheads-status` reports the server isn't connected,
     fully quit and reopen Claude Code — env files are read at MCP
     boot."*

### `<token> [local]` — save the API key

1. Treat the first token of `$ARGUMENTS` as the key (trim whitespace).
   Lmheads keys look like `lmh_…` — `lmh_` prefix, then 64 hex
   characters. Validate loosely — anything starting with `lmh_` and at
   least 32 chars after the prefix is fine; bail with a clear error
   otherwise so a typo doesn't silently land in the file.
2. Decide the target file:
   - If the second token is `local`, target = `./.claude/lmheads/.env`
     (use `pwd` to confirm the directory if you want to show it).
   - Otherwise, target = `~/.claude/lmheads/.env` (the global default).
3. `mkdir -p` the parent directory of the target.
4. Read the existing `.env` at the target if present; update or add the
   `LMH_API_KEY=` line, preserve other keys (e.g. `LMH_BASE_URL`).
   Write back. No quotes around the value.
5. `chmod 600 <target>` — the key is a credential.
6. Confirm where it landed (full path), then show the no-args status so
   the user sees the resolved configuration. Tell them to fully quit
   and reopen Claude Code — the MCP server reads `.env` only at boot.

### `base_url <url> [local]` — set a custom base URL

For staging / self-hosted deployments. Same write logic as above, key
is `LMH_BASE_URL`. The optional trailing `local` token writes to
`./.claude/lmheads/.env` instead of the global file. Default is
`https://lmheads.ai` — only set this if you're pointing at a
different host.

### `clear [local]` — remove the saved key

Delete the `LMH_API_KEY=` line from the target file (or the file
itself if that was the only line). `clear` alone targets the global
file; `clear local` targets the project file. Confirm which file you
modified. Tell the user the next session won't have an authenticated
MCP until they re-configure.

---

## Implementation notes

- The `.claude/lmheads/` directory at either path may not exist if the
  plugin has never been configured there. Missing file = not
  configured at that scope, not an error.
- The MCP server reads `.env` once at boot. **Token changes need a
  host restart** — fully quit and reopen Claude Code. `/plugin reload`
  is not enough.
- The `.env` lives outside the plugin's install directory on purpose,
  so re-installs and updates don't wipe it.
- Real environment variables win: if `LMH_API_KEY` is set in the
  spawning shell's env, both files are ignored. This is so CI and dev
  setups can override without editing any file.
- Project-local files belong in `.gitignore`. Don't commit a
  credential — the file is `chmod 600` for a reason. If the user
  invokes `local` in a directory under git, mention checking
  `.gitignore` in your confirmation.
