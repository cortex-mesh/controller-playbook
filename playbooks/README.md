---
permalink: /playbooks/
---

# Playbooks

Choose the playbook by **controller identity**, not by which laptop happens to be awake.

| If the CoS is… | Read | Implementer | Reviewer |
| --- | --- | --- | --- |
| Any / mixed | [general.md](general.md) | Isolated worker CLI | Different family from the writer |
| Claude | [claude.md](claude.md) | Worker CLI (often Codex) | Opus |
| Codex | [codex.md](codex.md) | Codex CLI | `gpt-5.6-sol` / high |
| Grok Bot | [grok.md](grok.md) | Grok CLI `grok-4.6` | `gpt-5.6-sol` / high COMMENT |

Do not combine reviewer defaults from two playbooks on one goal. The controller identity picks the authoritative reviewer. Launch mechanics (tmux, worktree, host checks) stay in the chosen playbook.

Start with [general.md](general.md) if you are adopting the method. Switch to a variant when you pin models and CLI flags.

Related:

- [Design](../docs/design.md)
- [Architecture](../docs/architecture.md)
- [Goal-prompt skill](../skills/goal-prompt/SKILL.md)
