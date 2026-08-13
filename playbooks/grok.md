# Grok controller playbook

Use this when a **Grok Bot** is the Chief of Staff and **Grok CLI** implements. Review is **not** Grok: default `gpt-5.6-sol` / high via `codex review`, posted as a GitHub `COMMENT`.

Grok CLI is not `codex exec` and not Claude remote control. `grok -p` is one shot. Chain phases with `--continue` / `--resume`. The Bot does not implement on its cloud computer.

## Model contract

| Role | Default |
| --- | --- |
| Implement | `grok-4.6` via Grok CLI. Pin the version; do not float `grok`. |
| Pre-PR review | `gpt-5.6-sol` / high via `codex review` |
| High-risk re-review | Fresh sol/high, CoS-run |

If you do not use Codex, substitute any reviewer CLI that is **not** the implementing Grok session.

## Worker pool

Fill this table. Examples only:

| Host | SSH | Fit | Max parallel Grok tracks |
| --- | --- | --- | --- |
| `dev-1` | `user@host` | General coding | 1 |
| `ci-box` | `user@host` | Heavy jobs; do not starve CI | 1 |

`grok` often lives at `$HOME/.grok/bin/grok` and is **missing from non-interactive SSH PATH**. Always call the absolute path.

```sh
GROK="$HOME/.grok/bin/grok"
$GROK --version
```

`grok --worktree` does not create a worktree in headless `-p` mode. You create the git worktree.

## Loop

```text
CoS writes graph → dispatch each unblocked track to a free worker
  → grok-4.6 implements this phase (headless -p, then --continue)
  → Sol/high COMMENT review, draft PR, AWAITING GATE
  → CoS confirms review on this head, clobber-checks, exact-head CI
  → high-risk: fresh Sol/high COMMENT
  → REVISE or GATE → serialize merge → staging → live-verify
```

## Dispatch

First phase:

```sh
GROK="$HOME/.grok/bin/grok"
tmux new-session -d -s <track> \
  "$GROK --no-auto-update --always-approve --permission-mode bypassPermissions \
     -m grok-4.6 --cwd <worktree> --verbatim -p \
     'Read <absolute-goal-prompt> and execute the next incomplete phase end-to-end. Then run the reviewer CLI, post gh pr review --comment, open a draft PR, report AWAITING GATE with branch, PR, SHA, COMMENT URL.'"
```

Later phases on the same track:

```sh
tmux new-session -d -s <track> \
  "$GROK --no-auto-update --always-approve -m grok-4.6 --cwd <worktree> \
     --continue -p 'Execute the next incomplete phase. Same gate protocol.'"
```

Prefer `--continue` or `--resume <uuid>` over a new conversation. Interactive TUI (`--no-alt-screen` in tmux) is for live steer only. Send "read this file" rather than pasting a huge prompt.

Autonomy for in-scope writes: `--always-approve --permission-mode bypassPermissions`. Still no production, no new paid secrets, no destructive data.

Skills: [grok-goal-prompt](../skills/grok-goal-prompt/SKILL.md), [grok-launch-track](../skills/grok-launch-track/SKILL.md).

## Review

```sh
codex review --base origin/main \
  -c 'model="gpt-5.6-sol"' \
  -c 'model_reasoning_effort="high"'
gh pr review <PR#> --comment --body "<verdict>"
```

Never Approve. Never review the PR in the same Grok session that wrote it.

## CoS gate

Agent-side Sol/high is already the different-family check. Default CoS gate is **not** a second Sol pass on every low-risk diff:

1. COMMENT exists for this head.
2. Clobber-check; rebase; serialize merges.
3. Exact-head CI, then live-verify staging.
4. Read the verdict.

Escalate to a fresh CoS-run Sol/high COMMENT for schema, auth, migration, security, data-write, or a thin pre-review. Record command, SHA, output path, and exit status.

## Heartbeat

Grok Bot cannot sit in a turn. Put a **10-minute weekday routine** on the CoS Bot whose prompt is this heartbeat plus "take the next repo-safe step; do not double-dispatch a progressing track."

```text
Thu Aug 13, 2026, 6:45:00 AM PT
2026-08-13 06:45:00 PDT
dev-1 still working track D (repo#123, SHA abc1234). CI in progress.
Will check in again in 10 minutes.
```

Minimum line: `<host> still working, will check in again in 10 minutes.`
