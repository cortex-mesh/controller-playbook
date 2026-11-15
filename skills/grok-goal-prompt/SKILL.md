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
| Implementer | Grok CLI `<coder-model>` on the host the CoS assigns, or a recorded CoS internal session |
| Model policy | `<coder-model>` writes; `<reviewer-model>` reviews on a fleet node (different family preferred); COMMENT on the PR |
| Launch | Headless `$HOME/.grok/bin/grok -p` then `--continue` / `--resume`. Tmux wraps the process. TUI only to steer. |
| Autonomy | `--always-approve --permission-mode bypassPermissions` for in-scope writes. No production, no new paid secrets, no destructive data. |

Use the absolute binary path. Non-interactive PATH often omits `$HOME/.grok/bin`.

## Header example

```markdown
_2026-08-13 · Owner: human operator · CoS: **<Bot name>** (Grok Bot) ·
Workers: Grok CLI `<coder-model>` on the **worker pool**, or a recorded CoS internal session_
```

## Extra hard rules for this path

- Never tell the worker to review the PR in the same session that wrote it. A fresh session of the same model is not an acceptable reviewer. Review always runs on a fleet node.
- Pin the current Grok coding model as `<coder-model>`. Do not float `grok`. See the example table as of 2026-11 in [playbooks/general.md](../../playbooks/general.md#model-pins).
- One CoS Bot per standing product goal.
- Heartbeat is a 10-minute Bot routine while implementing or in CI, not a 15- or 30-minute quiet poll. Every CoS-visible message starts with WORKING, WAITING ON YOU, or BLOCKED. `AWAITING GATE` in the graph is `WAITING ON YOU: merge PR #N` only after `gh pr ready`; until then WORKING. Not one notice then quiet.
- Do not dispatch staging as the next incomplete phase. Workers never `gh pr ready`.
- In the assigned worktree, run the product repo CI-equivalent check (`make check`, or the exact commands from that repo CI). Push only if it is green. COMMENT only after this-SHA CI is green. Phase DoD is those commands; `AWAITING GATE` is illegal until they exit 0. If the repo has no `make check`, or CI does not run on this SHA, say so in `AWAITING GATE`. See [playbooks/general.md](../../playbooks/general.md#cicd).
- Run `scripts/pr-size-check` before opening a draft PR. Over cap → `AWAITING SPLIT`, do not open a megadiff, do not COMMENT-review it. File count is a smell, not a fail. Write the [status file](../../examples/status.schema.yaml). Run `scripts/gate-preflight`.
- If the chosen path is fleet dispatch and dispatch fails, escalate dispatch. Do not silently switch the locus to the CoS session. A CoS session that implements must be an explicit, recorded choice ([ADR 0009](../../docs/adr/0009-coding-locus-and-review-gate.md)).
