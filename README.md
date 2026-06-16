# 🦉 Telegram Rich Messages by Hoot

Enable **Telegram Bot API 10.1 Rich Messages** (`sendRichMessage`) in [Hermes Agent](https://github.com/NousResearch/hermes-agent) — render **native tables**, **real checkboxes**, and **expandable details sections** instead of flattened plain text.

This repo is a single, battle-tested Hermes skill — drop it into your agent and every Telegram reply renders like a proper rich document.

---

## ✨ What You Get

| Before | After |
|---|---|
| Tables show as raw pipe characters | Native grid with borders |
| Checklists show literal `- [x]` | Real checkbox icons ☑ / ☐ |
| Details blocks show as plain text | Clickable, expandable sections |
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

### Option B — Manual drop (bypasses scanner)

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

## 🛡️ If the Security Scanner Blocks Install

Hermes scans community-source skills before installing. If you see:

```
Verdict: DANGEROUS — Decision: BLOCKED
```

That's a **false positive** caused by detection rules. This skill:

- Modifies your Hermes config (by design — that's how it enables rich messages)
- References `~/.hermes/config.yaml` and the gateway service (normal config-edit work)

Some scanner versions flag any skill that touches Hermes internals. The skill does NOT:

- ❌ Send your data to any external server
- ❌ Read your `.env` or credentials
- ❌ Modify anything outside your Hermes config

### Verify the skill yourself

1. Open the [raw SKILL.md](https://raw.githubusercontent.com/peaceofheaven777/telegram-rich-message-hermes/main/skills/telegram-rich-messages/SKILL.md)
2. Read through it — there's no executable code, just documentation and YAML/Python helper snippets the agent will offer the user
3. All file paths stay under your home `.hermes/` directory

### Install with the trust flag

Once you've verified:

```bash
hermes skills install \
  https://raw.githubusercontent.com/peaceofheaven777/telegram-rich-message-hermes/main/skills/telegram-rich-messages/SKILL.md \
  --name telegram-rich-messages \
  --trust
```

The `--trust` flag bypasses the scan for this single install.

Or just use Option B (manual git clone + cp) — local files aren't scanned.

---

## 🛠 What the Skill Covers

| Section | Content |
|---|---|
| **Setup procedure** | 3-step config fix + restart (the only path that actually works) |
| **12 pitfalls** | Every gotcha discovered during live debugging — wrong config path, streaming bypass, latch behavior, `send_message` tool bypass, etc. |
| **Agent behavior** | When to use tables, checklists, details, headers — with patterns and examples |
| **Diagnosis flowchart** | 6-step decision tree for "rich messages don't work" |
| **Code path references** | Line numbers in `telegram.py` for deep debugging |
| **Verification test** | Standard rich-render test snippet + expected results |

---

## 📋 Setup TL;DR

The minimum 3 steps (full skill has more context + diagnosis):

1. Edit your Hermes `config.yaml` and add under `gateway.platforms.telegram.extra`:
   ```yaml
   rich_messages: true
   ```

2. In the same `config.yaml`, set under `display.platforms.telegram`:
   ```yaml
   streaming: false
   ```

3. From an external shell (NOT inside the agent):
   ```bash
   systemctl --user restart hermes-gateway
   ```

Then send any agent reply with a table — should render natively. If not, install the full skill and follow its diagnosis flowchart.

---

## ⚠️ Critical Gotchas (Top 3)

1. **Wrong config path** — `telegram.rich_messages` at root does NOTHING. Must be `gateway.platforms.telegram.extra.rich_messages`.
2. **Streaming silently blocks rich** — `display.platforms.telegram.streaming: true` makes the rich-attempt gate return False every time.
3. **`send_message` tool BYPASSES rich path** — uses legacy MarkdownV2 directly. Verify rich with NORMAL agent replies, not tool calls.

The full skill has 9 more.

---

## 🎁 Bonus: Agent Behavior Guide

After enabling, the skill instructs the agent to **default to rich formatting in every Telegram reply**:

```text
## 🚀 Heading

One-line summary.

| Key | Value |
|---|---|
| Status | OK |

**Next steps:**
- [x] Done
- [ ] Pending

<details>
<summary>Risks</summary>
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
