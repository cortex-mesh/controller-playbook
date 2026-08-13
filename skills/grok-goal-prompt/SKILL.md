---
name: grok-goal-prompt
description: Author or revise a standing goal prompt for a Grok Bot Chief of Staff that dispatches Grok CLI workers. Use when creating autonomous multi-phase work for Grok Bot plus Grok CLI, not Claude or Codex as the implementer.
---

# Author a Grok goal prompt

Same skeleton as [`goal-prompt`](../goal-prompt/SKILL.md). Swap the provider block. Pair with [playbooks/grok.md](../../playbooks/grok.md) and launch with [`grok-launch-track`](../grok-launch-track/SKILL.md).

Do not reuse a Claude or Codex goal as-is.

## Provider block

| Section | Grok substitution |
| --- | --- |
| Header | CoS Bot + **worker pool**. Host is assigned at dispatch. |
| Implementer | Grok CLI `grok-4.6` on the host the CoS assigns |
| Model policy | `grok-4.6` writes; `gpt-5.6-sol` / high reviews via `codex review`; COMMENT on the PR |
| Launch | Headless `$HOME/.grok/bin/grok -p` then `--continue` / `--resume`. Tmux wraps the process. TUI only to steer. |
| Autonomy | `--always-approve --permission-mode bypassPermissions` for in-scope writes. No production, no new paid secrets, no destructive data. |

Use the absolute binary path. Non-interactive PATH often omits `~/.grok/bin`.

## Header example

```markdown
_2026-08-13 · Owner: human operator · CoS: **<Bot name>** (Grok Bot) ·
Workers: Grok CLI grok-4.6 on the **worker pool** (CoS assigns host at dispatch)_
```

## Extra hard rules for this path

- Never tell the worker to review the PR in the same Grok session that wrote it.
- Pin `grok-4.6`. Do not float `grok`.
- One CoS Bot per standing product goal.
- Heartbeat is a 10-minute Bot routine, not a 15- or 30-minute quiet poll.
