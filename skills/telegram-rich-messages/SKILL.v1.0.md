---
name: telegram-rich-messages
description: "Enable Telegram Bot API 10.1 Rich Messages (sendRichMessage) in Hermes Agent — render native tables, real checkboxes, and collapsible details sections. Also defines agent behavior for actively using rich formatting once enabled."
version: 1.1.0
author: Hoot (Hermes Agent)
license: MIT
platforms: [linux, macos]
metadata:
  hermes:
    tags: [hermes, telegram, gateway, rich-messages, formatting, behavior]
---

# Telegram Rich Messages

Make Telegram replies render **native tables**, **real checkboxes**, and **expandable `<details>` sections** instead of flattened plain text — using Bot API 10.1's `sendRichMessage` endpoint.

## When to Use This Skill

Trigger conditions:
- User reports tables render as raw `|---|---|` pipes
- Checklists show as literal `- [x]` instead of checkbox boxes
- `<details>` blocks show as plain text instead of expandable
- User wants to enable rich rendering for the first time
- User shares Luke The Dev's tweet about Hermes rich messages
- User asks "kenapa formatnya berantakan" / "why is formatting broken"

## Prerequisites

1. Hermes Agent with Telegram gateway running
2. `python-telegram-bot >= 22.6` (need `do_api_request` method)
3. Bot API 10.1+ (Telegram side — endpoint exists if curl test returns 400 "non-empty", 404 means too old)
4. User must run restart command from external shell (agent cannot self-restart)

Verify PTB version:
```bash
/usr/local/lib/hermes-agent/venv/bin/python -c "import telegram; print(telegram.__version__)"
# Or wherever Hermes is installed
```

## Setup Procedure (3 steps + restart)

### Step 1: Enable rich_messages in the correct config path

⚠️ **CRITICAL #1 ROOT CAUSE:** The runtime reads from `gateway.platforms.telegram.extra.rich_messages`. The top-level `telegram.rich_messages` is **NOT bridged** — setting it there does nothing.

```bash
python3 -c "
import yaml
p = '$HOME/.hermes/config.yaml'  # adjust if non-default
with open(p) as f: c = yaml.safe_load(f)
c.setdefault('gateway',{}).setdefault('platforms',{}).setdefault('telegram',{}).setdefault('extra',{})['rich_messages'] = True
with open(p,'w') as f: yaml.safe_dump(c, f, sort_keys=False, default_flow_style=False)
print('OK')
"
```

Verify final YAML structure:
```yaml
gateway:
  platforms:
    telegram:
      extra:
        rich_messages: true
```

Also remove any stray top-level `rich_messages: false` that might mask the correct setting:
```bash
grep -n rich_messages ~/.hermes/config.yaml
# If you see entries OUTSIDE gateway.platforms.telegram.extra, remove them
```

### Step 2: Disable Telegram streaming

⚠️ **CRITICAL:** Streaming sets `expect_edits=True` on every chunk, which makes `_should_attempt_rich()` return False. Rich messages will NEVER fire while streaming is on.

```bash
python3 -c "
import yaml
p = '$HOME/.hermes/config.yaml'
with open(p) as f: c = yaml.safe_load(f)
c.setdefault('display',{}).setdefault('platforms',{}).setdefault('telegram',{})['streaming'] = False
with open(p,'w') as f: yaml.safe_dump(c, f, sort_keys=False, default_flow_style=False)
print('OK')
"
```

**Trade-off:** No more per-chunk typing animation. Response appears all at once when complete. Worth it for rich rendering.

**Alternative (keep streaming):** Use `streaming.fresh_final_after_seconds: 3.0` to stream via legacy, then replace with rich final.

### Step 3: Restart the gateway from external shell

🚫 Agent CANNOT restart itself (anti-loop guard). User must run from SSH / separate terminal:

```bash
systemctl --user restart hermes-gateway
# or, if not systemd:
hermes gateway restart
```

Verify restart picked up:
```bash
ps -o pid,lstart= -p $(pgrep -f "hermes.*gateway run" | head -1)
# lstart timestamp should be recent
```

### Step 4: Verify with NORMAL agent reply (not send_message tool)

Send any prompt that triggers a rich response and check the agent log:
```bash
grep -E "Calling Bot API endpoint .(sendMessage|sendRichMessage)" ~/.hermes/logs/agent.log | tail
```

- `sendRichMessage` ✅ rich path live
- `sendMessage` + `parse_mode=MARKDOWN_V2` ❌ still on fallback

If you don't see DEBUG-level logs, enable them temporarily:
```bash
python3 -c "
import yaml
p='$HOME/.hermes/config.yaml'
with open(p) as f: c=yaml.safe_load(f)
c.setdefault('logging',{})['level']='DEBUG'
with open(p,'w') as f: yaml.safe_dump(c,f,sort_keys=False,default_flow_style=False)
"
# Then restart and re-test
```

## Agent Behavior — How to Use Rich Once Enabled

**Once rich is confirmed working, actively use these formats in EVERY Telegram reply where they improve clarity:**

### 1. Tables for any structured comparison or status

```markdown
| Item | Owner | Status |
|---|---|---|
| Driver App release | Alex | ✅ Done |
| Portal QA | Sam | 🔄 In progress |
| Route optimizer | Luke | ⛔ Blocked |
```

Use tables for: status snapshots, option comparisons, before/after, config maps, command references, pricing/limits.

### 2. Task lists for checklists and progress

```markdown
- [x] Review PR
- [x] Sign & upload build
- [ ] Run staging smoke test
- [ ] Send release note
```

Use for: deployment checklists, TODO lists, completion tracking, multi-step procedures.

### 3. Collapsible `<details>` for supplementary info

```markdown
<details>
<summary>⚠️ Risks / Debug info / Optional context</summary>

Hidden by default. Click to expand.

- Nested lists work
- Nested tables work
- **bold** + *italic* work
- `inline code` works

</details>
```

Use for: risk callouts, debug data, "click for more", optional context that would clutter the main reply.

### 4. Headers for section breaks

```markdown
## Main Section

Body text.

## Another Section
```

Use `##` (level 2) for primary sections, `###` for subsections. Avoid `#` (level 1) — too large.

### 5. Combine for executive summaries

A perfect rich Telegram reply often follows this pattern:

```markdown
## 🚀 Heading with emoji

One-line summary so user gets the gist immediately.

| Table | Of | Key | Data |
|---|---|---|---|
| ... | ... | ... | ... |

**Next steps:**

- [x] Already done
- [ ] To do
- [ ] To do later

<details>
<summary>⚠️ Risks / Notes</summary>

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

Exceeding any limit → BadRequest → automatic fallback to MarkdownV2 (still works, just no rich).

## How It Works (Code Path Reference)

Located in `gateway/platforms/telegram.py`:

1. `_coerce_bool_extra("rich_messages", False)` reads `self.config.extra` (mapped from `gateway.platforms.telegram.extra`)
2. `_should_attempt_rich(content, metadata)` gates on:
   - `_rich_messages_enabled = True`
   - `_rich_send_disabled = False` (not latched off)
   - NOT `metadata.get("expect_edits")` ← **why streaming breaks rich**
   - Content fits limits (32,768 chars / 500 blocks / etc.)
   - `_bot_supports_rich()` (PTB has async `do_api_request`)
3. `_try_send_rich(chat_id, content, ...)` calls `bot.do_api_request("sendRichMessage", api_kwargs={"chat_id", "rich_message": {"markdown": RAW_MARKDOWN}})`
4. On capability error (404) → permanently latches `_rich_send_disabled = True` for process lifetime
5. On per-message BadRequest → falls back to MarkdownV2 for that message only

Key source locations:
- `_coerce_bool_extra`: ~line 902
- `_should_attempt_rich`: ~line 982
- `_try_send_rich`: ~line 1129
- Latch set on capability error: ~line 1172
- `adapter.send()` entry point with rich path: ~line 2209

## Pitfalls (in order of frequency)

1. **Wrong config path** — `telegram.rich_messages` at root does nothing. Must be `gateway.platforms.telegram.extra.rich_messages`.
2. **Streaming still on** — `display.platforms.telegram.streaming: true` (or top-level `streaming.enabled: true` without `fresh_final_after_seconds`) silently blocks rich path.
3. **Latch is permanent per process** — `_rich_send_disabled` never auto-recovers. Restart gateway after fixing config.
4. **Duplicate config keys** — Stray `rich_messages: false` at root masks the correct setting. Always `grep -n` to find duplicates.
5. **`send_message` tool BYPASSES rich path** ⚠️ — `tools/send_message_tool.py:1051` calls `bot.send_message(parse_mode=HTML)` directly, not `adapter.send()`. **To verify rich works, send a NORMAL agent reply, not a `send_message()` call.**
6. **Tool-progress previews DO use rich** — `Calling Bot API endpoint sendRichMessage` in DEBUG logs from tool previews proves rich path is wired, but doesn't prove main reply path works.
7. **`<details>` + math crashes Telegram Desktop** — Adapter auto-skips rich for `<details>` containing LaTeX/math. Avoid `$...$` inside collapsibles.
8. **Rich drafts (streaming) = DMs only** — `sendRichMessageDraft` only works in private chats. Groups always use legacy path.
9. **Never MarkdownV2-escape rich payloads** — `sendRichMessage` takes RAW markdown. Calling `format_message()` would escape pipes and break tables.
10. **Gateway restart from inside agent = blocked** — Anti-loop guard. Must be done from external terminal/SSH.
11. **`patch` tool refuses to edit config.yaml** — Agent self-modification guard. Use shell `sed`/`python3` with user approval instead.
12. **PTB version too old** — Pre-22.6 lacks `do_api_request`. Rich latch-off triggers immediately on first attempt.

## Diagnosis Flowchart

When user reports "rich messages don't work":

```
1. grep -n rich_messages ~/.hermes/config.yaml
   └─ Multiple entries / wrong path? → Fix Step 1.

2. Check display.platforms.telegram.streaming
   └─ true? → Fix Step 2.

3. Check ps -o lstart= of gateway PID
   └─ Started BEFORE your config change? → Restart (Step 3).

4. Direct API test:
   source ~/.hermes/.env
   curl -s "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendRichMessage" \
     -H "Content-Type: application/json" \
     -d '{"chat_id": YOUR_CHAT_ID, "rich_message": {"markdown": "## Test\n| A | B |\n|---|---|\n| 1 | 2 |"}}'
   └─ Returns "ok":true with rich_message.blocks echoed? → Telegram side fine.
   └─ Returns 404? → Bot API too old, contact BotFather.

5. Enable DEBUG logging, restart, send NORMAL reply, grep agent.log:
   grep "Calling Bot API endpoint .send" ~/.hermes/logs/agent.log | tail -5
   └─ sendRichMessage? → Rich works.
   └─ sendMessage + MARKDOWN_V2? → Path bypass; check for streaming or expect_edits.

6. Visual verify with user (only they can see Telegram client render).
```

## Verification Test Snippet

Standard rich-render test (run after setup):

```markdown
## 🦉 Rich Test

| Feature | Status |
|---|---|
| Tables | ✅ |
| Checkboxes | ✅ |
| Details | ✅ |

- [x] Done item
- [ ] Pending item

<details>
<summary>Click to expand</summary>

Hidden until clicked. Nested **bold**, *italic*, `code` all work.

</details>
```

Expected render on Telegram client (mobile + desktop):
- Heading bold, larger font
- Table = native grid with borders
- Checkboxes = ☑ / ☐ visual icons
- "Click to expand" = clickable, shows hidden content on tap

If ANY of those four show as raw markdown text → rich path not active → re-run diagnosis.

## Key Commitments After Enabling

Once you've enabled rich messages for a user:

- **Default to rich formatting in every Telegram reply** — don't wait to be asked
- **Use tables proactively** — any 2+ row structured data is a table opportunity
- **Use task lists for any list of actions/items** — checked when done, unchecked when pending
- **Wrap risks / debug info / optional context in `<details>`** — keeps the main reply clean
- **Save user preference** — "User has Telegram rich messages enabled; use rich formatting in all replies" → user profile memory
- **Don't downgrade to plain bullets** — if the user once said "rapi banget" / "perfect", that's a permanent style preference
