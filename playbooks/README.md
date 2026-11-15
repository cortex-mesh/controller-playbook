---
permalink: /playbooks/
---

# Playbooks

Choose a playbook by **controller setup** — Grok Bot, Claude Code, or Codex — not by which laptop happens to be awake. Shared rules live in [general.md](general.md). Each setup page covers how that controller dispatches workers, when it may code in its own internal session, which fleet reviewers pair with it for a different model family, and its limits.

The Cursor IDE is not a controller setup. See [general.md](general.md#cursor-ide-is-operator-homework).

| Controller setup | Read | Typical implementer | Preferred fleet reviewer |
| --- | --- | --- | --- |
| Shared rules | [general.md](general.md) | Internal session or fleet CLI | Different, at-least-as-capable model on a fleet node (different family preferred) |
| Grok Bot as controller | [grok.md](grok.md) | Grok CLI `<coder-model>`, or a recorded internal session (`<model not pinnable>`) | Codex or Claude Code, strongest available qualifying tier |
| Claude Code as controller | [claude.md](claude.md) | Often a Codex worker; or a recorded Claude Code session | Different family from the coder (Codex or Grok when Claude Code wrote the change; Claude Code when a different-family worker wrote it) |
| Codex as controller | [codex.md](codex.md) | Codex CLI `<coder-model>`, or a recorded Codex session | Claude Code or Grok, strongest available qualifying tier |

Do not combine reviewer defaults from two setups on one goal. The controller setup picks the authoritative reviewer pairing. Launch mechanics (tmux, worktree, host checks) stay in the chosen setup. Worker CLI notes (Grok CLI, Codex, Claude Code) are sub-sections of each setup.

Start with [general.md](general.md) if you are adopting the method. Then open the setup that matches the controller.

Related:

- [Design](../docs/design.md)
- [Architecture](../docs/architecture.md)
- [Fleet PR quality gate](../docs/fleet-pr-quality-gate.md)
- [Goal-prompt skill](../skills/goal-prompt/SKILL.md)
