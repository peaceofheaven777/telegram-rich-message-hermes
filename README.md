# 🦉 Telegram Rich Messages by Hoot

Enable **Telegram Bot API 10.1 Rich Messages** (`sendRichMessage`) in [Hermes Agent](https://github.com/NousResearch/hermes-agent) — render **native tables**, **real checkboxes**, and **expandable `<details>` sections** instead of flattened plain text.

This repo is a single, battle-tested Hermes skill — drop it into your agent and every Telegram reply renders like a proper rich document.

---

## ✨ What You Get

| Before | After |
|---|---|
| Tables show as raw `\|---\|---\|` pipes | Native grid with borders |
| Checklists show literal `- [x]` | Real checkbox icons ☑ / ☐ |
| `<details>` shows as plain text | Clickable, expandable sections |
| Walls of bullet text | Structured, scannable replies |

Plus, the skill teaches the agent **when and how to use rich formatting actively** — not just enables the capability.

---

## 🚀 Quick Install

### Option A — Hermes CLI (recommended)

```bash
hermes skills install \
  https://raw.githubusercontent.com/peaceofheaven777/telegram-rich-message-hermes/main/skills/telegram-rich-messages/SKILL.md \
  --name telegram-rich-messages
```

Then in your Hermes session:
```
/reload-skills
```

### Option B — Manual drop

```bash
git clone https://github.com/peaceofheaven777/telegram-rich-message-hermes.git
mkdir -p ~/.hermes/skills/autonomous-ai-agents/telegram-rich-messages
cp telegram-rich-message-hermes/skills/telegram-rich-messages/SKILL.md \
   ~/.hermes/skills/autonomous-ai-agents/telegram-rich-messages/SKILL.md
```

Then `/reload-skills` in Hermes.

### Option C — Paste to any agent

Open [`SKILL.md`](./skills/telegram-rich-messages/SKILL.md), copy contents, tell your agent:

> "Save this as a skill called `telegram-rich-messages`."

Works with any Hermes-compatible agent.

---

## 🛠 What the Skill Covers

| Section | Content |
|---|---|
| **Setup procedure** | 3-step config fix + restart (the only path that actually works) |
| **12 pitfalls** | Every gotcha discovered during live debugging — wrong config path, streaming bypass, latch behavior, `send_message` tool bypass, etc. |
| **Agent behavior** | When to use tables, checklists, `<details>`, headers — with patterns and examples |
| **Diagnosis flowchart** | 6-step decision tree for "rich messages don't work" |
| **Code path references** | Line numbers in `telegram.py` for deep debugging |
| **Verification test** | Standard rich-render test snippet + expected results |

---

## 📋 Setup TL;DR

If you don't want to read the whole skill, here's the minimum:

```bash
# 1. Enable rich messages in the CORRECT config path
python3 -c "
import yaml
p = '$HOME/.hermes/config.yaml'
with open(p) as f: c = yaml.safe_load(f)
c.setdefault('gateway',{}).setdefault('platforms',{}).setdefault('telegram',{}).setdefault('extra',{})['rich_messages'] = True
with open(p,'w') as f: yaml.safe_dump(c, f, sort_keys=False, default_flow_style=False)
"

# 2. Disable Telegram streaming (streaming sets expect_edits=True, blocking rich)
python3 -c "
import yaml
p = '$HOME/.hermes/config.yaml'
with open(p) as f: c = yaml.safe_load(f)
c.setdefault('display',{}).setdefault('platforms',{}).setdefault('telegram',{})['streaming'] = False
with open(p,'w') as f: yaml.safe_dump(c, f, sort_keys=False, default_flow_style=False)
"

# 3. Restart gateway (from EXTERNAL shell — agent cannot restart itself)
systemctl --user restart hermes-gateway
```

Then send any reply with a table — should render natively. If not, load the full skill and follow the diagnosis flowchart.

---

## ⚠️ Critical Gotchas (Top 3)

1. **Wrong config path** — `telegram.rich_messages` at root does NOTHING. Must be `gateway.platforms.telegram.extra.rich_messages`.
2. **Streaming silently blocks rich** — `display.platforms.telegram.streaming: true` makes `expect_edits=True`, which makes `_should_attempt_rich()` return False.
3. **`send_message` tool BYPASSES rich path** — Sends via legacy `bot.send_message(parse_mode=HTML)`. Verify with normal agent replies, not tool calls.

The full skill has 9 more.

---

## 🎁 Bonus: Agent Behavior Guide

After enabling, the skill instructs the agent to **default to rich formatting in every Telegram reply**:

```markdown
## 🚀 Heading

One-line summary.

| Key | Value |
|---|---|
| Status | ✅ |

**Next steps:**
- [x] Done
- [ ] Pending

<details>
<summary>⚠️ Risks</summary>
Optional deeper context.
</details>
```

This is the pattern that makes Hermes replies look like a polished status report instead of a chat dump.

---

## 🦉 About Hoot

This skill was extracted from a live debugging session in a Hermes Agent instance named **Hoot** — a cyber owl persona (deep navy + teal, glowing cyan eyes, circuit-board feathers).

- **Built on:** [Hermes Agent](https://github.com/NousResearch/hermes-agent) by Nous Research
- **Inspired by:** [Luke The Dev's tweet](https://x.com/iamlukethedev/status/2066197968701513998) on enabling Telegram Rich Messages
- **Style philosophy:** terse, action-first, rich formatting always

If you build on this or fix something, PRs welcome.

---

## 📜 License

MIT — see [`SKILL.md`](./skills/telegram-rich-messages/SKILL.md) frontmatter.

---

## 🔗 Links

| | |
|---|---|
| **Hermes Agent** | https://github.com/NousResearch/hermes-agent |
| **Hermes Docs** | https://hermes-agent.nousresearch.com/docs |
| **Telegram Bot API 10.1** | https://core.telegram.org/bots/api |
| **Skill authoring guide** | https://hermes-agent.nousresearch.com/docs/user-guide/features/skills |
