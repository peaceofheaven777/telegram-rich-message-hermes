# 🦉 Hermes Skills by Hoot

Reusable [Hermes Agent](https://github.com/NousResearch/hermes-agent) skills — procedural knowledge that any Hermes-compatible agent can load and follow.

> Skills are how Hermes "learns" — each skill is a self-contained Markdown file with frontmatter that tells the agent **when to use it** and **how to do it**. Drop one into `~/.hermes/skills/` and the agent picks it up automatically.

---

## 📦 Skills in This Repo

| Skill | What It Does |
|---|---|
| [`telegram-rich-messages`](./skills/telegram-rich-messages/SKILL.md) | Enable Telegram Bot API 10.1 `sendRichMessage` in Hermes — native tables, real checkboxes, expandable `<details>`. Also defines agent behavior for actively using rich formatting once enabled. |

---

## 🚀 Quick Install

### Option A — Drop into local skills folder

```bash
git clone https://github.com/peaceofheaven777/hermes-skills.git
cp -r hermes-skills/skills/* ~/.hermes/skills/
# In Hermes CLI:
/reload-skills
```

### Option B — Single skill via Hermes CLI

```bash
hermes skills install \
  https://raw.githubusercontent.com/peaceofheaven777/hermes-skills/main/skills/telegram-rich-messages/SKILL.md \
  --name telegram-rich-messages
```

### Option C — Manual paste

Open the SKILL.md, copy contents, and tell your Hermes agent:

> "Save this as a skill called `<skill-name>`"

---

## 🎯 Why These Exist

Skills capture lessons learned the hard way — config gotchas, hidden code paths, behavior preferences — so the next agent (or the next session of the same agent) doesn't have to rediscover them.

The `telegram-rich-messages` skill, for example, encodes 12 pitfalls discovered during a live debugging session:
- Wrong config key path (top-level vs `gateway.platforms.telegram.extra`)
- Streaming silently blocking rich path via `expect_edits=True`
- `send_message` tool bypassing rich entirely
- Latch state being permanent per process
- Restart blocked from inside agent (anti-loop guard)

Without the skill, every new session would burn 30+ minutes re-debugging the same issues.

---

## 🛠 Skill Format (For Contributors)

Each skill is a Markdown file with YAML frontmatter:

```markdown
---
name: skill-name-here
description: "One-line summary of what the skill does."
version: 1.0.0
author: Your Name
license: MIT
platforms: [linux, macos]
metadata:
  hermes:
    tags: [tag1, tag2]
---

# Skill Title

## Trigger
When to use this skill.

## Prerequisites
What needs to be set up first.

## Procedure
Step-by-step instructions.

## Pitfalls
Things that go wrong, ranked by frequency.

## Verification
How to confirm it worked.
```

See the [Hermes skill authoring docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills) for the full schema.

---

## 🤝 Contributing

PRs welcome. Each new skill should:

1. Live in `skills/<skill-name>/SKILL.md`
2. Have frontmatter with `name`, `description`, `version`
3. Document trigger conditions, steps, pitfalls
4. Be tested end-to-end before submission

---

## 📜 License

MIT — see individual skills for their license headers.

---

## 🦉 About Hoot

Hoot is the agent identity used in the Hermes instance that generated these skills.
- **Visual:** cyber owl, deep navy + teal, glowing cyan eyes, circuit-board feathers
- **Built on:** [Hermes Agent](https://github.com/NousResearch/hermes-agent) by Nous Research
- **Style:** terse, action-first, rich formatting always

If you save a skill from your own Hermes session, you're contributing to the same hivemind 🌐
