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

Default tmux `remain-on-exit` is off. Set it on and tee to `$HOME/.grok/logs/<track>.log` so a finished `grok -p` does not erase `AWAITING GATE` or failures. Capture from the held pane or that log.

```sh
GROK="$HOME/.grok/bin/grok"
LOG="$HOME/.grok/logs/<track>.log"
mkdir -p "$HOME/.grok/logs"
tmux new-session -d -s <track>
tmux set-option -t <track> remain-on-exit on
tmux send-keys -t <track> \
  "$GROK --no-auto-update --always-approve --permission-mode bypassPermissions \
     -m grok-4.6 --cwd <worktree> --verbatim -p \
     'Read <absolute-goal-prompt> and execute the next incomplete phase end-to-end. Open a draft PR, then run the reviewer CLI and post gh pr review --comment. Report AWAITING GATE with branch, PR, SHA, COMMENT URL.' \
     2>&1 | tee \"$LOG\"" Enter
```

Later phases on the same track:

```sh
LOG="$HOME/.grok/logs/<track>.log"
mkdir -p "$HOME/.grok/logs"
tmux new-session -d -s <track>
tmux set-option -t <track> remain-on-exit on
tmux send-keys -t <track> \
  "$GROK --no-auto-update --always-approve -m grok-4.6 --cwd <worktree> \
     --continue -p 'Execute the next incomplete phase. Same gate protocol.' \
     2>&1 | tee \"$LOG\"" Enter
```

Prefer `--continue` or `--resume <uuid>` over a new conversation. Interactive TUI (`--no-alt-screen` in tmux) is for live steer only. Send "read this file" rather than pasting a huge prompt.

Autonomy for in-scope writes: `--always-approve --permission-mode bypassPermissions`. Still no production, no new paid secrets, no destructive data.

Skills: [grok-goal-prompt](../skills/grok-goal-prompt/SKILL.md), [grok-launch-track](../skills/grok-launch-track/SKILL.md).

## Review

Resolve `<default>` from this repo; do not assume `main`.

```sh
DEFAULT=$(git symbolic-ref --quiet --short refs/remotes/origin/HEAD \
  || echo origin/<default>)
codex review --base "$DEFAULT" \
  -c 'model="gpt-5.6-sol"' \
  -c 'model_reasoning_effort="high"'
gh pr review <PR#> --comment --body-file <verdict.md>
```

Never Approve. Never review the PR in the same Grok session that wrote it.

## CoS gate

Agent-side Sol/high is already the different-family check. Default CoS gate is **not** a second Sol pass on every low-risk diff:

1. Clobber-check; rebase; serialize merges.
2. If rebase moved the SHA, run a fresh COMMENT review on the new head.
3. Confirm COMMENT exists for **this** head.
4. Exact-head CI.
5. Read the verdict.

After merge: watch default-branch CI, deploy staging, live-verify the merged SHA.

Escalate to a fresh CoS-run Sol/high COMMENT for schema, auth, migration, security, data-write, or a thin pre-review. Record command, SHA, output path, and exit status.

## Heartbeat

Grok Bot cannot sit in a turn. While any track is running, put a **10-minute routine on the CoS Bot every calendar day, including weekends.** Prompt is this heartbeat plus "take the next repo-safe step; do not double-dispatch a progressing track." A weekday-only schedule misses weekend tracks; do not use one.

```text
Thu Aug 13, 2026, 6:45:00 AM PT
2026-08-13 06:45:00 PDT
dev-1 still working track D (repo#123, SHA abc1234). CI in progress.
Will check in again in 10 minutes.
```

Minimum line: `<host> still working, will check in again in 10 minutes.`
