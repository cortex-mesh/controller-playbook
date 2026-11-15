---
permalink: /playbooks/
---

# Playbooks

Choose the playbook by **controller identity**, not by which laptop happens to be awake.

| If the CoS is… | Read | Implementer | Reviewer |
| --- | --- | --- | --- |
| Any / mixed | [general.md](general.md) | Internal session or fleet CLI | Different, more capable model on a fleet node (different family preferred) |
| Claude | [claude.md](claude.md) | `<coder-model>` (often Codex) | `<reviewer-model>` on a fleet node |
| Codex | [codex.md](codex.md) | Codex CLI `<coder-model>` | `<reviewer-model>` on a fleet node, different family preferred |
| Grok Bot | [grok.md](grok.md) | Grok CLI `<coder-model>` or a recorded internal session | `<reviewer-model>` on a fleet node, different family preferred |

Do not combine reviewer defaults from two playbooks on one goal. The controller identity picks the authoritative reviewer. Launch mechanics (tmux, worktree, host checks) stay in the chosen playbook.

Start with [general.md](general.md) if you are adopting the method. Switch to a variant when you pin models and CLI flags.

Related:

- [Design](../docs/design.md)
- [Architecture](../docs/architecture.md)
- [Goal-prompt skill](../skills/goal-prompt/SKILL.md)
