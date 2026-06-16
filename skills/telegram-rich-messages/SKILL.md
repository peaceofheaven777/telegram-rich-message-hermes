---
name: telegram-rich-messages
description: "Enable Telegram Bot API 10.1 Rich Messages (sendRichMessage) in Hermes Agent — render native tables, real checkboxes, and collapsible details sections. Also defines agent behavior for actively using rich formatting once enabled."
version: 1.2.0
author: Hoot (Hermes Agent)
license: MIT
platforms: [linux, macos]
metadata:
  hermes:
    tags: [hermes, telegram, gateway, rich-messages, formatting, behavior]
---

# Telegram Rich Messages

Make Telegram replies render **native tables**, **real checkboxes**, and **expandable details sections** instead of flattened plain text — using Bot API 10.1's `sendRichMessage` endpoint.

## When to Use This Skill

Trigger conditions:
- User reports tables render as raw pipe characters
- Checklists show as literal bracket-x instead of checkbox boxes
- Details blocks show as plain text instead of expandable
- User wants to enable rich rendering for the first time
- User shares Luke The Dev's tweet about Hermes rich messages
- User asks "kenapa formatnya berantakan" / "why is formatting broken"

## Prerequisites

1. Hermes Agent with Telegram gateway running
2. python-telegram-bot version 22.6 or newer (needs `do_api_request` method)
3. Bot API 10.1+ on Telegram side (endpoint exists if test request returns 400 "non-empty"; 404 means too old)
4. User must run restart command from external shell (agent cannot self-restart)

To verify python-telegram-bot version, ask the user to run their Hermes venv python with `import telegram; print(telegram.__version__)` from a terminal.

## Setup Procedure (3 steps + restart)

### Step 1: Enable rich_messages in the correct config path

The single most common mistake: setting `rich_messages` at the wrong location.

The runtime reads from this YAML path:

```yaml
gateway:
  platforms:
    telegram:
      extra:
        rich_messages: true
```

Setting it at the top level (`telegram.rich_messages`) does NOT work — there is no config bridge for that key. Always confirm the path is under `gateway.platforms.telegram.extra`.

To update the config, the agent (or user) should:

1. Open the Hermes config YAML file (typical location: under the user's home `.hermes` directory)
2. Navigate to `gateway.platforms.telegram.extra` (create the nested keys if missing)
3. Set `rich_messages: true`
4. Save the file

A small Python helper can do this idempotently. Provide it to the user, asking them to substitute their own config path:

```text
# Run from a shell. Replace CONFIG_PATH with the location of your Hermes config.yaml.
python3 -c "
import yaml
with open(CONFIG_PATH) as f: cfg = yaml.safe_load(f)
cfg.setdefault('gateway',{}).setdefault('platforms',{}).setdefault('telegram',{}).setdefault('extra',{})['rich_messages'] = True
with open(CONFIG_PATH,'w') as f: yaml.safe_dump(cfg, f, sort_keys=False, default_flow_style=False)
print('OK')
"
```

Also ask the user to look through their config for any stray top-level `rich_messages: false` entry that could mask the correct setting. Any duplicate at the root level should be removed.

### Step 2: Disable Telegram streaming

Critical: streaming sets `expect_edits: True` on every chunk, which makes the rich-attempt gate (`_should_attempt_rich`) return False. Rich messages will NEVER fire while streaming is active.

The target YAML location:

```yaml
display:
  platforms:
    telegram:
      streaming: false
```

Same Python helper pattern as Step 1, this time setting the streaming flag to False. Ask the user to substitute their config path and run from a shell:

```text
python3 -c "
import yaml
with open(CONFIG_PATH) as f: cfg = yaml.safe_load(f)
cfg.setdefault('display',{}).setdefault('platforms',{}).setdefault('telegram',{})['streaming'] = False
with open(CONFIG_PATH,'w') as f: yaml.safe_dump(cfg, f, sort_keys=False, default_flow_style=False)
"
```

Trade-off to communicate to the user: no more per-chunk typing animation; response appears all at once when the agent finishes thinking. Worth it for rich rendering.

Alternative (keep streaming active): set `streaming.fresh_final_after_seconds: 3.0` so the gateway streams legacy MarkdownV2 first, then replaces the final message with a rich one.

### Step 3: Restart the gateway from an external shell

The agent CANNOT restart itself (anti-loop guard blocks this). The user must run the restart from a separate SSH session or terminal window. Typical commands:

- Under systemd user services: `systemctl --user restart hermes-gateway`
- Direct CLI: `hermes gateway restart`

After restart, ask the user to verify a new gateway process is running. The `ps` command with `lstart` showing a recent timestamp confirms the process picked up the new config.

### Step 4: Verify with a NORMAL agent reply (not via send_message tool)

Have the agent produce any reply that uses rich formatting (a table, a checklist, a details block). Then check the agent log for which Bot API endpoint was actually called:

- A log line mentioning `sendRichMessage` confirms the rich path is live
- A log line mentioning `sendMessage` with `parse_mode=MARKDOWN_V2` means the fallback path is still being used

To make this visible if DEBUG logging is off, temporarily raise the log level to DEBUG via the same YAML-edit helper pattern (set `logging.level: DEBUG`), restart, and re-test.

## Agent Behavior — How to Use Rich Once Enabled

Once rich is confirmed working, the agent should actively use these formats in EVERY Telegram reply where they improve clarity.

### 1. Tables for any structured comparison or status

Use a pipe-table for status snapshots, option comparisons, before/after, config maps, command references, pricing/limits, or anything with 2+ structured rows.

Example pattern (treat as documentation, not a script):

```text
| Item | Owner | Status |
|---|---|---|
| Driver App release | Alex | Done |
| Portal QA | Sam | In progress |
| Route optimizer | Luke | Blocked |
```

### 2. Task lists for checklists and progress

Use GitHub-style task list syntax (bracket-space and bracket-x) for deployment checklists, TODO lists, completion tracking, multi-step procedures.

### 3. Collapsible details for supplementary info

Wrap risk callouts, debug data, optional context, or anything that would clutter the main reply inside a `<details>` block with a meaningful `<summary>`. Nested lists, tables, bold/italic, and inline code all work inside.

### 4. Headers for section breaks

Use level-2 headers (`##`) for primary sections, level-3 (`###`) for subsections. Avoid level-1 — too large in the Telegram client.

### 5. Combine for executive summaries

A perfect rich Telegram reply often follows this pattern:

```text
## Heading with optional emoji

One-line summary so user gets the gist immediately.

| Key | Value |
|---|---|
| ... | ... |

**Next steps:**

- [x] Already done
- [ ] To do
- [ ] To do later

<details>
<summary>Risks / Notes</summary>

Optional deeper context.

</details>
```

### Bot API 10.1 Limits

| Limit | Value |
|---|---|
| Max chars | 32,768 UTF-8 |
| Max blocks | 500 |
| Max nesting | 16 levels |
| Max table columns | 20 |

Exceeding any limit causes a BadRequest and automatic fallback to MarkdownV2 (still works, just no rich rendering).

## How It Works (Code Path Reference)

The relevant code is in the Hermes Agent source under `gateway/platforms/telegram.py`:

1. `_coerce_bool_extra("rich_messages", False)` reads `self.config.extra` (mapped from `gateway.platforms.telegram.extra` in YAML)
2. `_should_attempt_rich(content, metadata)` gates on:
   - `_rich_messages_enabled` is True
   - `_rich_send_disabled` is False (not latched off)
   - NOT `metadata.get("expect_edits")` — this is why streaming breaks rich
   - Content fits Bot API 10.1 limits
   - `_bot_supports_rich()` — python-telegram-bot has async `do_api_request`
3. `_try_send_rich(...)` calls `bot.do_api_request("sendRichMessage", api_kwargs={"chat_id", "rich_message": {"markdown": RAW_MARKDOWN}})`
4. On capability error (HTTP 404) → permanently latches `_rich_send_disabled` for the process lifetime
5. On per-message BadRequest → falls back to MarkdownV2 for that message only

Key source locations (approximate line numbers in telegram.py):
- `_coerce_bool_extra`: around line 902
- `_should_attempt_rich`: around line 982
- `_try_send_rich`: around line 1129
- Latch set on capability error: around line 1172
- `adapter.send()` rich-path entry point: around line 2209

## Pitfalls (in order of frequency)

1. **Wrong config path** — top-level `telegram.rich_messages` does nothing. Must be under `gateway.platforms.telegram.extra.rich_messages`.
2. **Streaming still on** — `display.platforms.telegram.streaming: true` silently blocks rich path via the `expect_edits` flag.
3. **Latch is permanent per process** — `_rich_send_disabled` never auto-recovers. Restart the gateway after fixing config.
4. **Duplicate config keys** — stray `rich_messages: false` at the root of config can mask the correct setting. Always have the user grep for duplicates.
5. **`send_message` tool BYPASSES rich path** — the agent's `send_message` tool calls `bot.send_message(parse_mode=HTML)` directly, NOT `adapter.send()`. To verify rich works, the agent must send a NORMAL reply, not a tool call.
6. **Tool-progress previews DO use rich** — seeing `sendRichMessage` in DEBUG logs from tool previews proves the rich path is wired, but does not prove the main reply path works.
7. **Details + math crashes Telegram Desktop** — the adapter auto-skips rich for `<details>` blocks containing LaTeX/math. Avoid math expressions inside collapsibles.
8. **Rich drafts (streaming) = DMs only** — `sendRichMessageDraft` only works in private chats. Group chats always use the legacy path.
9. **Never MarkdownV2-escape rich payloads** — `sendRichMessage` takes RAW markdown. Pre-escaping would destroy table pipes.
10. **Gateway restart from inside agent = blocked** — anti-loop guard. Must be done from external terminal/SSH.
11. **`patch` tool refuses to edit config** — agent self-modification guard. Use a shell-side YAML helper with user approval instead.
12. **python-telegram-bot version too old** — pre-22.6 lacks `do_api_request`. Rich latches off immediately on first attempt.

## Diagnosis Flowchart

When the user reports "rich messages don't work":

1. Have the user grep for `rich_messages` in their Hermes config — confirm only one entry, under the correct nested path.
2. Have the user check `display.platforms.telegram.streaming` — confirm False.
3. Compare the gateway process start time against when the config was last modified — if config changed AFTER process start, restart is needed.
4. Run a direct Telegram Bot API test (user-side curl, with their token, to `sendRichMessage`) — confirms Telegram + bot token + endpoint work end-to-end, bypassing Hermes entirely.
5. Enable DEBUG logging (Step 2 helper pattern, set `logging.level: DEBUG`), restart, send a normal reply, and grep the agent log for `Calling Bot API endpoint`. Look for `sendRichMessage` (rich works) vs `sendMessage` + MARKDOWN_V2 (fallback active).
6. Have the user visually confirm rendering in their Telegram client — the agent cannot inspect client-side rendering directly.

## Verification Test Snippet

Standard rich-render test for the agent to send after setup completes:

```text
## Rich Test

| Feature | Status |
|---|---|
| Tables | OK |
| Checkboxes | OK |
| Details | OK |

- [x] Done item
- [ ] Pending item

<details>
<summary>Click to expand</summary>

Hidden until clicked. Nested bold, italic, inline code all work.

</details>
```

Expected render on Telegram client (mobile + desktop):
- Heading: bold, larger font
- Table: native grid with borders
- Checkboxes: visual icons (filled and empty)
- "Click to expand": clickable, reveals hidden content on tap

If ANY of these four show as raw markdown text, rich path is not active — re-run the diagnosis flowchart.

## Key Commitments After Enabling

Once rich messages are enabled for a user:

- **Default to rich formatting in every Telegram reply** — don't wait to be asked
- **Use tables proactively** — any 2+ rows of structured data is a table opportunity
- **Use task lists for any list of actions/items** — checked when done, unchecked when pending
- **Wrap risks / debug info / optional context in `<details>`** — keeps the main reply clean
- **Save user preference to agent memory** — once a user confirms rendering works, that's a permanent style preference
- **Don't downgrade to plain bullets** — if the user said "perfect" or "rapi banget", lock that style in
