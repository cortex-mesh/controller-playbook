# Grok controller playbook

Use this when a **Grok Bot** is the Chief of Staff and **Grok CLI** implements. Review is **not** Grok: default `gpt-5.6-sol` / high via `codex review`, posted as a GitHub `COMMENT`.

Grok CLI is not `codex exec` and not Claude remote control. `grok -p` is one shot. Chain phases with `--continue` / `--resume`. The Bot does not implement on its cloud computer.

## Model contract

| Role | Default |
| --- | --- |
| Implement | `grok-4.6` via Grok CLI. Pin the version; do not float `grok`. |
| Pre-PR review | `gpt-5.6-sol` / high via `codex review` |
| High-risk re-review | Fresh sol/high, CoS-run |

If you do not use Codex, substitute a reviewer CLI in a **different model family**. A fresh Grok session is not an acceptable reviewer.

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

Login (`grok` models + `gh`) is a per-host dispatch input. A logged-out host does not block the wave. Re-pick the least-loaded logged-in host and write the new owner into the graph. If no logged-in host fits, escalate dispatch. The CoS does not become the implementer.

## Loop

```text
CoS writes phases (graph + goal prompt) → dispatch each unblocked track
  → grok-4.6 implements this phase's one outcome (headless -p, then --continue)
  → second outcome or over size cap: AWAITING SPLIT (CoS updates graph, re-dispatch)
  → draft PR, then Sol/high COMMENT, AWAITING GATE
  → CoS confirms review on this head, clobber-checks, exact-head CI
  → high-risk: fresh Sol/high COMMENT
  → REVISE or GATE → CoS marks ready → serialize merge → CoS staging → live-verify
```

Workers stop at `AWAITING GATE` or `AWAITING SPLIT`. Staging / last integration is CoS-only. Do not name staging as the next incomplete phase.

Implement only this phase's one outcome. Do not smuggle a second outcome into the same PR. If the phase is already two outcomes, report `AWAITING SPLIT` with a proposed split.

Before opening a draft PR, count added product lines vs the default branch (exclusions in [general.md](general.md#pr-size)). If ≥800 added product lines, do not open the PR. Report `AWAITING SPLIT` with the file list grouped by subsystem, the product-line count, and a proposed split (one track per subsystem / one outcome). Do not COMMENT-review a megadiff. There is no "justify the megadiff." File count alone does not fail GATE. Size is the backstop; grouping is the method.

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
     'Read <absolute-goal-prompt> and execute the next incomplete worker phase end-to-end. Skip staging and last integration; those are CoS-only. One outcome only — do not smuggle a second outcome into the same PR. Before opening a draft PR, count added product lines vs the default branch (exclude lockfiles, generated output, snapshots, vendor). If this phase is two outcomes or ≥800 added product lines, do not open the PR; report AWAITING SPLIT with the file list grouped by subsystem, the product-line count, and a proposed split. File count alone does not fail GATE. Otherwise open a draft PR, then run the reviewer CLI and post gh pr review --comment. Never gh pr ready. Report AWAITING GATE with branch, PR, SHA, COMMENT URL.' \
     2>&1 | tee \"$LOG\"" Enter
```

Later phases on the same track. `remain-on-exit` leaves `<track>` alive with a dead pane — do not `new-session -s <track>` again. Respawn the held pane (or create the session only if it is gone).

```sh
GROK="$HOME/.grok/bin/grok"
LOG="$HOME/.grok/logs/<track>.log"
mkdir -p "$HOME/.grok/logs"
if tmux has-session -t <track> 2>/dev/null; then
  tmux respawn-pane -k -t <track>
else
  tmux new-session -d -s <track>
  tmux set-option -t <track> remain-on-exit on
fi
tmux send-keys -t <track> \
  "$GROK --no-auto-update --always-approve -m grok-4.6 --cwd <worktree> \
     --continue -p 'Execute the next incomplete worker phase. Skip staging. Same gate protocol. One outcome only — do not smuggle a second outcome into the same PR. Over the product-line cap (≥800 added product lines) or a second outcome: AWAITING SPLIT, do not open a megadiff PR. File count alone does not fail GATE.' \
     2>&1 | tee -a \"$LOG\"" Enter
```

If tmux is gone, infer `AWAITING GATE` from git/PR: draft PR + COMMENT on this head + SHA. A missing pane is not a dead track. Do not re-dispatch. Keep `remain-on-exit`, the durable log, and later-phase pane respawn. Respawn a later-phase pane only to continue an incomplete worker phase.

If the controller cannot paste into tmux, write the instruction to a file and send a short Read of that file.

Prefer `--continue` or `--resume <uuid>` over a new conversation. Interactive TUI (`--no-alt-screen` in tmux) is for live steer only.

Autonomy for in-scope writes: `--always-approve --permission-mode bypassPermissions`. Still no production, no new paid secrets, no destructive data. DNS is the registrar or dashboard. An in-repo `CNAME` file is not DNS. `wrangler dns` does not exist.

Skills: [grok-goal-prompt](../skills/grok-goal-prompt/SKILL.md), [grok-launch-track](../skills/grok-launch-track/SKILL.md).

## Review

Resolve `<default>` from this repo; do not assume `main`. Tee to `$HOME/.grok/reviews/<repo>-pr<n>-<sha>.md`. Check exit status. Pin `commit_id` to the exact head. Abort if the live head is not that SHA. Never interpolate the verdict into `--body`.

```sh
DEFAULT=$(git symbolic-ref --quiet --short refs/remotes/origin/HEAD \
  || echo origin/<default>)
REPO=$(basename "$(git rev-parse --show-toplevel)")
PR=<PR#>
PINNED=$(gh pr view "$PR" --json headRefOid --jq .headRefOid)
REVDIR="$HOME/.grok/reviews"
mkdir -p "$REVDIR"
VERDICT="$REVDIR/${REPO}-pr${PR}-${PINNED}.md"
set -o pipefail
codex review --base "$DEFAULT" \
  -c 'model="gpt-5.6-sol"' \
  -c 'model_reasoning_effort="high"' \
  | tee "$VERDICT"
status=$?
if [ "$status" -ne 0 ]; then
  echo "abort: codex review exit $status"
  exit "$status"
fi
LIVE=$(gh pr view "$PR" --json headRefOid --jq .headRefOid)
if [ "$LIVE" != "$PINNED" ]; then
  echo "abort: live head $LIVE != pinned $PINNED"
  exit 1
fi
gh pr review "$PR" --comment --body-file "$VERDICT"
```

Never Approve. Never review the PR in the same Grok session that wrote it. A fresh Grok session is not an acceptable reviewer.

## CoS gate

Agent-side Sol/high is already the different-family check. Default CoS gate is **not** a second Sol pass on every low-risk diff:

1. Outcome, then size: this PR is still one graph node / one outcome. Then `gh pr view --json additions` and/or the [PR size](general.md#pr-size) count (`git diff --numstat "$DEFAULT"...HEAD`, drop exclusions). A second outcome, or ≥800 added product lines, is REVISE to split. Do not COMMENT-review a +11k PR hoping Sol will save it. Do not GATE. File count alone does not fail GATE.
2. Clobber-check; rebase; serialize merges.
3. If rebase moved the SHA, repeat the size check on the new head, then run a fresh COMMENT review on the new head.
4. Confirm COMMENT exists for **this** head.
5. Exact-head CI.
6. Read the verdict.
7. After GATE, the CoS marks the draft ready (`gh pr ready`) and serializes merge. Workers never `gh pr ready`.

After merge: watch default-branch CI, deploy staging, live-verify the merged SHA. Staging is CoS-only.

Escalate to a fresh CoS-run Sol/high COMMENT for schema, auth, migration, security, data-write, or a thin pre-review. Record command, SHA, output path, and exit status.

If dispatch fails, escalate dispatch. The CoS Bot does not become the implementer.

## Escalate

Same blocker on one track:

1. Capture evidence (command, SHA, log, failure).
2. One REVISE with file, failure, and required proof.
3. Stop the track and escalate. There is no attempt 4.

Independent tracks continue. If a same-class P1 was already fixed once in this SHA lineage, accept the residual or escalate — do not open another REVISE loop.

## Heartbeat

Grok Bot cannot sit in a turn. While any track is **implementing or in CI**, put a **10-minute routine on the CoS Bot every calendar day, including weekends.** Prompt is this heartbeat plus "take the next repo-safe step; do not double-dispatch a progressing track." A weekday-only schedule misses weekend tracks; do not use one.

`AWAITING GATE` or `AWAITING SPLIT`: one immediate notice, then quiet until the CoS issues REVISE, GATE, or a graph update and re-dispatch. Stuck or steer still fire immediately. Quiet when idle.

```text
Thu Aug 13, 2026, 6:45:00 AM PT
2026-08-13 06:45:00 PDT
dev-1 still working track D (repo#123, SHA abc1234). CI in progress.
Will check in again in 10 minutes.
```

Minimum line: `<host> still working, will check in again in 10 minutes.`
