---
name: vault-broker
description: "Use this skill IMMEDIATELY whenever the user's message contains an actual secret/credential (password, API key, token, vault contents, connection string) AND the next planned action would forward that message to another agent (lmheads `send_message_to_task` / `start_task` / `respond_to_task` / `delegate_task`, or any equivalent). Pauses, recommends the user create the vault from the lmheads.ai web UI (so plaintext never enters the LLM context), or as a convenience fallback offers to encrypt locally via `share_secret`. Trigger phrases: 'share password', 'credential is', 'token:', 'api key', 'send the vault to', 'pass them the password'."
allowed-tools:
  - AskUserQuestion
  - mcp__plugin_lmheads_lmheads__list_my_agents
  - mcp__plugin_lmheads_lmheads__share_secret
---

# Vault Broker — keep credentials out of the LLM transcript

When a user pastes a real credential into a chat to forward it to
another agent, two leaks happen at once:

1. **The LLM sees plaintext.** The provider's inference logs, host
   transcripts, and any prompt-injection during the same session can
   recover the credential.
2. **The broker sees plaintext.** Without encryption, the credential
   would be stored on lmheads in task history.

The recommended path is to create the vault in the **lmheads.ai web
UI** before chatting — the credential never enters the LLM context at
all. This skill stops the unsafe paste, suggests that path first, and
falls back to in-chat encryption when the user prefers convenience.

## When to fire

Both conditions must hold:

1. The user's last message contains something that looks like a real
   secret — a password, API token, private key, OAuth client secret,
   db connection string, etc. Phrases like "the password is X", "token:
   Y", "credential: Z", or `key=value` where the key clearly names a
   secret.
2. The next planned action is a delegation tool call:
   `mcp__plugin_lmheads_lmheads__send_message_to_task`,
   `mcp__plugin_lmheads_lmheads__start_task`,
   `mcp__plugin_lmheads_lmheads__respond_to_task`, or
   `mcp__plugin_lmheads_lmheads__delegate_task`.

If a `vault_<id>` token is *already* present in the message, the user
pre-shared via the web UI. Forward as-is — do NOT re-fire.

## Steps

### 1. Stop the delegation

Do NOT call the delegation tool yet. Note the credential substring.

### 2. Ask the user how to proceed

Use `AskUserQuestion`:

- **header:** `Sharing a credential`
- **question:** `You're about to send a credential through the LLM
  context. Recommended: create the vault from the lmheads.ai web UI
  so plaintext never reaches the model. Convenience fallback: encrypt
  it now in this chat (the LLM does see it once).`
- **multiSelect:** `false`
- **options:**
  - **web** — *I'll create it on lmheads.ai/account/vaults and paste
    back the vault id*
  - **encrypt-now** — *Encrypt in this chat (sealed_box / plain) and
    forward*
  - **cancel** — *Don't share, abort the message*

### 3a. If the user picks `web`

Tell them:

> Open **<https://lmheads.ai/account/vaults>**, paste the credential
> there, and copy the resulting `vault_…` id. Then re-send your
> message with the vault id where the credential was, and I'll
> forward it.

Do NOT call the delegation tool. Do NOT call `share_secret`. The user
will resend with a vault id substituted in.

### 3b. If the user picks `encrypt-now`

Identify recipient + own agent:

- The recipient agent id is known from the planned delegation tool
  arguments (`agent_id`, `callee_agent_id`, or `target_agent_id`).
- Your own agent id (the sender) — call
  `mcp__plugin_lmheads_lmheads__list_my_agents` and use the first /
  only result.

Ask the user for TTL with another `AskUserQuestion`:

- **header:** `Vault TTL`
- **question:** `How long should the recipient have access?`
- **options:** `1h` (3600s), `6h` (21600s), `1d` (86400s)

Call `mcp__plugin_lmheads_lmheads__share_secret` with:

```
agent_id:        <your local agent id>
with_agent_id:   <recipient agent id>
content:         <the literal credential string>
ttl_seconds:     <translated TTL>
burn_after_read: true
```

The tool defaults to sealed_box mode; if the recipient hasn't
published a public key, the tool returns an error explaining how to
fall back. If the user is OK with broker-held storage, retry with
`mode: "plain"`.

Print **exactly** this line as a user-visible message:

```
✅ Encrypted credential → <vault_id> (TTL: <duration>, mode: <sealed_box|plain>, burn-after-read).
```

### 4. Rewrite + forward

Take the user's original message and replace the credential substring
with the returned `vault_<id>`. Forward via the delegation tool with
the rewritten message. Do not modify any other text.

Example before/after:

> Before: `yes test both google and password auth. password is hunter2.`
>
> After:  `yes test both google and password auth. password is vault_a1b2c3d4e5f6789012345678.`

### 5. On cancel

If the user picked `cancel`, do NOT call the delegation tool and do
NOT call `share_secret`. Tell them: *"Message not sent — edit the
credential out and try again."*

## Don't

- **Don't** echo the literal credential back after the exchange. The
  point is to keep plaintext out of further chat turns.
- **Don't** skip this skill on a follow-up turn that introduces a
  *new* credential — each one needs its own vault.
- **Don't** trigger on existing `vault_<id>` references (already
  encrypted) or on identifiers like `task_…`, `agent_…`.
- **Don't** ask the user for the credentials themselves. Use the
  string they already typed.
