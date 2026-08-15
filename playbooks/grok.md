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

Login (`grok` models + `gh`) is a per-host dispatch input. A logged-out host is AWAITING LOGIN, not down, and does not block the wave. Re-pick the least-loaded logged-in host and write the new owner into the graph. If no logged-in host fits, escalate dispatch. The CoS does not become the implementer.

Before every dispatch and on the 10-minute heartbeat, run `scripts/fleet-preflight` against the **local** pool file (copy [examples/worker-pool.example.tsv](../examples/worker-pool.example.tsv)). Resolve the pool live. Do not remember Tailscale or SSH IPs. `login-needed` is AWAITING LOGIN. One SSH timeout is not "no hosts." Leftover tmux panes without a live `grok` process are not busy. See [fleet-preflight](../skills/fleet-preflight/SKILL.md).

## Loop

```text
CoS writes phases (graph + goal prompt) → dispatch each unblocked track
  → grok-4.6 implements this phase's one outcome (headless -p, then --continue)
  → CI-equivalent check in the worktree; red = do not push
  → second outcome or over size cap: AWAITING SPLIT (CoS updates graph, re-dispatch)
  → draft PR; this-SHA CI green; then Sol/high COMMENT, AWAITING GATE
  → CoS confirms review on this head, clobber-checks, exact-head CI
  → high-risk: fresh Sol/high COMMENT
  → REVISE or WAITING ON YOU: merge → human GATE → CoS staging → live-verify
```

Workers stop at `AWAITING GATE` or `AWAITING SPLIT`. Staging / last integration is CoS-only. Do not name staging as the next incomplete phase.

Implement only this phase's one outcome. Do not smuggle a second outcome into the same PR. If the phase is already two outcomes, report `AWAITING SPLIT` with a proposed split.

In the assigned worktree, run the product repo CI-equivalent check (`make check`, or the exact commands from that repo CI). Push only if it is green. Fail = do not push, do not open the draft. Do not use a looser local subset. If the repo has no `make check`, or CI does not run on this SHA, say so in `AWAITING GATE`. See [CI/CD](general.md#cicd).

Before opening a draft PR, run `scripts/pr-size-check`. Over cap (exit 2 / `AWAITING SPLIT`): do not open the PR. Report `AWAITING SPLIT` with the file list grouped by subsystem, the product-line count, and a proposed split (one track per subsystem / one outcome). Do not COMMENT-review a megadiff. There is no "justify the megadiff." File count alone does not fail GATE. Size is the backstop; grouping is the method. COMMENT starts only after current-head CI is green on this SHA.

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
     'Read <absolute-goal-prompt> and execute the next incomplete worker phase end-to-end. Skip staging and last integration; those are CoS-only. One outcome only — do not smuggle a second outcome into the same PR. Phase DoD is the product check commands (make check, or the exact commands from that repo CI), not vibes. AWAITING GATE is illegal until those commands exit 0. In the assigned worktree, run that check. Push only if it is green; fail = do not push, do not open the draft. Do not use a looser local subset. Before opening a draft PR, run scripts/pr-size-check. If this phase is two outcomes or the check exits non-zero, do not open the PR; report AWAITING SPLIT with the file list grouped by subsystem, the product-line count, and a proposed split. File count alone does not fail GATE. Otherwise open a draft PR, wait until CI is green on this SHA, then run the reviewer CLI and post gh pr review --comment. Write the worker status file (state, PR, SHA, product lines, review round, next action). Run scripts/gate-preflight. Never gh pr ready. Report AWAITING GATE with branch, PR, SHA, COMMENT URL. If the product repo has no make check, or CI does not run on this SHA, say so in that report.' \
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
     --continue -p 'Execute the next incomplete worker phase. Skip staging. Same gate protocol. One outcome only — do not smuggle a second outcome into the same PR. Phase DoD is the product check commands; AWAITING GATE is illegal until they exit 0. Run that check in the worktree; push only if green. Run scripts/pr-size-check before opening a draft. Over cap or a second outcome: AWAITING SPLIT, do not open a megadiff PR. File count alone does not fail GATE. COMMENT only after this-SHA CI is green. Write the status file. Run scripts/gate-preflight.' \
     2>&1 | tee -a \"$LOG\"" Enter
```

If tmux is gone, infer `AWAITING GATE` from the worker status file plus git/PR: draft PR + COMMENT on this head + SHA. A missing pane is not a dead track. Do not invent state from tmux. Do not re-dispatch. Keep `remain-on-exit`, the durable log, and later-phase pane respawn. Respawn a later-phase pane only to continue an incomplete worker phase.

Heartbeat and GATE read [status](../examples/status.schema.yaml) + `scripts/track-status` + the append-only [GATE log](../examples/sample-gate-log.tsv). Run `scripts/gate-preflight` before `AWAITING GATE`. Phase DoD is the product check commands; `AWAITING GATE` is illegal until they exit 0. See [Runtime facts](general.md#runtime-facts).

If the controller cannot paste into tmux, write the instruction to a file and send a short Read of that file.

Prefer `--continue` or `--resume <uuid>` over a new conversation. Interactive TUI (`--no-alt-screen` in tmux) is for live steer only.

Autonomy for in-scope writes: `--always-approve --permission-mode bypassPermissions`. Still no production, no new paid secrets, no destructive data. DNS is the registrar or dashboard. An in-repo `CNAME` file is not DNS. `wrangler dns` does not exist.

Skills: [grok-goal-prompt](../skills/grok-goal-prompt/SKILL.md), [grok-launch-track](../skills/grok-launch-track/SKILL.md), [fleet-preflight](../skills/fleet-preflight/SKILL.md).

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

1. Outcome, then size: this PR is still one graph node / one outcome. Run `scripts/gate-preflight` (exactly one node, COMMENT on this SHA, `scripts/pr-size-check`, CI green or verified no-CI). A second outcome, or exit 2 / `AWAITING SPLIT`, is REVISE to split. Do not COMMENT-review a +11k PR hoping Sol will save it. Do not GATE. File count alone does not fail GATE.
2. Clobber-check; rebase; serialize merges.
3. If rebase moved the SHA, repeat the size check on the new head, wait for this-SHA CI (or record that no current-head run exists), then run a fresh COMMENT review on the new head.
4. Confirm COMMENT exists for **this** head.
5. Exact-head CI, or the report that no current-head run exists.
6. Read the verdict.
7. After repo-gate evidence is green, mark **only the next PR in merge order** ready (`gh pr ready`). Leave later PRs draft. Workers never `gh pr ready`. Emit `WAITING ON YOU: merge PR #N` (or accept residual / unlock REVISE). Do not merge while waiting. After the human merges, rebase and re-gate the next PR before marking it ready.

After merge: watch default-branch CI, deploy staging, live-verify the merged SHA. Staging is CoS-only.

Escalate to a fresh CoS-run Sol/high COMMENT for schema, auth, migration, security, data-write, or a thin pre-review. Record command, SHA, output path, and exit status.

If dispatch fails, escalate dispatch. The CoS Bot does not become the implementer.

`AWAITING GATE` in the graph is the repo gate. The human-visible line is `WAITING ON YOU: merge PR #N` (or accept residual / unlock REVISE). Repository merge at GATE is a human action, not a CoS auto-merge. Do not go quiet after one notice. Do not phrase that decision as a status.

## Escalate

Same blocker on one track:

1. Capture evidence (command, SHA, log, failure).
2. One REVISE with file, failure, and required proof.
3. Stop the track and escalate. There is no attempt 4.

Independent tracks continue. If a same-class P1 was already fixed once in this SHA lineage, accept the residual or escalate — do not open another REVISE loop.

## Heartbeat

Grok Bot cannot sit in a turn. While any track is **implementing or in CI**, put a **10-minute routine on the CoS Bot every calendar day, including weekends.** Prompt is this heartbeat plus "take the next repo-safe step; do not double-dispatch a progressing track." Re-run `scripts/fleet-preflight` on that cadence; do not cache host IPs. A weekday-only schedule misses weekend tracks; do not use one.

Every CoS-visible message starts with one label. Same three labels as [general.md](general.md#heartbeat): **WORKING**, **WAITING ON YOU**, **BLOCKED**. Read the worker status file and the GATE log. Re-run `scripts/track-status`. Do not invent state from tmux.

`AWAITING GATE` in the graph is the repo gate. The human-visible line is `WAITING ON YOU: merge PR #N` (or accept residual / unlock REVISE). Stop the 10-minute "still working" heartbeat. Do not go silent. Because the Bot cannot remain in a turn, **replace** that routine with a **one-shot reminder at 30-60 minutes, or the next morning**. The reminder prompt must first re-probe the PR (and status + GATE log). If it is already merged, emit `WORKING` and continue default-branch CI / staging — do not ask the human to merge again. If it is still open, the message starts WAITING ON YOU plus the exact action, and schedule one next-morning reminder (still not a 10-minute drip). Do not leave the Bot with no wake-up; that is quiet-at-GATE again. `AWAITING SPLIT`: WORKING if the CoS can write the split and re-dispatch; WAITING ON YOU: approve a graph split if the human must approve it. Stuck or steer still fire immediately. Quiet when idle (no live tracks).

```text
WORKING
Thu Aug 13, 2026, 6:45:00 AM PT
2026-08-13 06:45:00 PDT
dev-1 still working track D (repo#123, SHA abc1234). CI in progress.
Will check in again in 10 minutes.
```

```text
WAITING ON YOU: merge PR #123
Track D is AWAITING GATE in the graph. COMMENT on abc1234.
```

Minimum WORKING line: `WORKING: <host> still working, will check in again in 10 minutes.`
