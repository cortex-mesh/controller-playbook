# Grok Bot as controller

Use this setup when a **Grok Bot** is the Chief of Staff. Shared rules are in [general.md](general.md). This page is the Grok Bot setup: how it dispatches workers, when it may code in its own session, which fleet reviewers pair with it, and its limits.

Review always runs on a fleet node, on a different and at-least-as-capable model, preferably a different family, posted as a GitHub `COMMENT`. Never Approve.

## Dispatching workers

Fill the worker-pool table. Examples only:

| Host | SSH | Fit | Max parallel Grok tracks |
| --- | --- | --- | --- |
| `dev-1` | `user@host` | General coding | 1 |
| `ci-box` | `user@host` | Heavy jobs; do not starve CI | 1 |

Before every dispatch and on the 10-minute heartbeat, run `scripts/fleet-preflight` against the **local** pool file (copy [examples/worker-pool.example.tsv](../examples/worker-pool.example.tsv)). Resolve the pool live. Do not remember VPN or SSH IPs. `login-needed` is AWAITING LOGIN. One SSH timeout is not "no hosts." Leftover tmux panes without a live worker process are not busy. See [fleet-preflight](../skills/fleet-preflight/SKILL.md).

Login (implementer models + `gh`) is a per-host dispatch input. A logged-out host is AWAITING LOGIN, not down, and does not block the wave. Re-pick the least-loaded logged-in host and write the new owner into the graph. If the chosen path is fleet dispatch and no logged-in host fits, escalate dispatch. Do not silently switch the locus to the controller session.

Create a dedicated git worktree. Name the next incomplete **worker** phase. Do not dispatch staging. Launch snippets for each worker CLI are in [Worker CLIs](#worker-clis).

```text
CoS writes phases (graph + goal prompt) → dispatch each unblocked track
  → implement this phase's one outcome (fleet worker CLI, or a recorded internal session)
  → CI-equivalent check in the worktree; red = do not push
  → second outcome or over size cap: AWAITING SPLIT (CoS updates graph, re-dispatch)
  → draft PR; this-SHA CI green; then fleet COMMENT (different family preferred), AWAITING GATE
  → CoS confirms review on this head, clobber-checks, exact-head CI
  → high-risk: fresh fleet COMMENT
  → REVISE or WAITING ON YOU: merge → human GATE → CoS staging → live-verify
```

Workers stop at `AWAITING GATE` or `AWAITING SPLIT`. Staging / last integration is CoS-only. Do not name staging as the next incomplete phase.

Implement only this phase's one outcome. In the assigned worktree, run the product repo CI-equivalent check; push only if it is green. Run `scripts/pr-size-check` before a draft. COMMENT starts only after current-head CI is green on this SHA. See [CI/CD](general.md#cicd).

Skills: [grok-goal-prompt](../skills/grok-goal-prompt/SKILL.md), [grok-launch-track](../skills/grok-launch-track/SKILL.md), [fleet-preflight](../skills/fleet-preflight/SKILL.md).

## Internal-session coding

A Grok Bot may implement in its own internal session only as an explicit, recorded choice ([ADR 0009](../docs/adr/0009-coding-locus-and-review-gate.md)). Isolation still comes from a dedicated worktree, a draft PR, and the repository gate. The session that writes the change does not review it, does not run `gh pr ready`, and does not merge.

The coding model inside a Grok Bot session cannot be pinned or named. The PR records:

```
Coded in: internal session
Coded by: controller internal session / Grok Bot / <model not pinnable>
```

Do not invent a Grok version name to fill the model slot. The fleet reviewer is the strongest available qualifying tier (see [Fleet reviewer pairings](#fleet-reviewer-pairings)).

## Fleet reviewer pairings

Prefer a **different model family** from the coder, on a fleet node, at least as capable and preferably more capable:

| Coder | Preferred fleet reviewer |
| --- | --- |
| Grok Bot internal session, or a Grok CLI worker | Codex or Claude Code, strongest available qualifying tier |
| A Codex worker dispatched by this controller | Claude Code, or a Grok reviewer if that is the strongest available different-family option |
| A Claude Code worker dispatched by this controller | Codex or Grok, strongest available qualifying tier |

A fresh Grok session of the same model that wrote the change is not an acceptable reviewer. Same-family review is last-resort only, must be more capable, and the PR states why no other family was available. See the example model table as of 2026-11 in [general.md](general.md#model-pins).

Never review in the coder's session. Tee the verdict outside the repo. Pin `commit_id` to the exact head. Abort if the live head moved. Never interpolate the verdict into `--body`. Re-review the final commit after any fix.

Resolve `<default>` from this repo; do not assume `main`. The snippet below is the Codex-reviewer form. For a Claude Code reviewer, use the review command in [claude.md](claude.md#fleet-reviewer-pairings).

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
  -c 'model="<reviewer-model>"' \
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

High-risk diffs (schema, auth, migration, security, data-write) or a thin pre-review get a fresh fleet COMMENT, still with a qualifying model. Record command, SHA, output path, and exit status.

## Limits

- The Grok Bot in-session model cannot be pinned or named. Record `<model not pinnable>` and pick the strongest available fleet reviewer tier.
- The Bot cannot sit in a turn. Heartbeat is a scheduled routine, not a long-lived chat (see [Heartbeat](#heartbeat)).
- A silent switch from a failed fleet dispatch into the Bot session is forbidden.
- Grok CLI is not `codex exec` and not Claude Code. `grok -p` is one shot. Chain phases with `--continue` / `--resume`.
- Shared method rules, coverage, the quality-gate checklist, and the human-only waiver are in [general.md](general.md). The Cursor IDE is not a controller setup; see [general.md](general.md#cursor-ide-is-operator-homework).

## Worker CLIs

### Worker: Grok CLI

`grok` often lives at `$HOME/.grok/bin/grok` and is **missing from non-interactive SSH PATH**. Always call the absolute path.

```sh
GROK="$HOME/.grok/bin/grok"
$GROK --version
```

`grok --worktree` does not create a worktree in headless `-p` mode. You create the git worktree. Pin `<coder-model>` on a fleet Grok CLI worker; do not float `grok`. (This pin applies to the CLI worker, not to a Grok Bot internal session.)

Default tmux `remain-on-exit` is off. Set it on and tee to `$HOME/.grok/logs/<track>.log` so a finished `grok -p` does not erase `AWAITING GATE` or failures. Capture from the held pane or that log.

First phase:

```sh
GROK="$HOME/.grok/bin/grok"
LOG="$HOME/.grok/logs/<track>.log"
mkdir -p "$HOME/.grok/logs"
tmux new-session -d -s <track>
tmux set-option -t <track> remain-on-exit on
tmux send-keys -t <track> \
  "$GROK --no-auto-update --always-approve --permission-mode bypassPermissions \
     -m <coder-model> --cwd <worktree> --verbatim -p \
     'Read <absolute-goal-prompt> and execute the next incomplete worker phase end-to-end. Skip staging and last integration; those are CoS-only. One outcome only — do not smuggle a second outcome into the same PR. Phase DoD is the product check commands (make check, or the exact commands from that repo CI), not vibes. AWAITING GATE is illegal until those commands exit 0. In the assigned worktree, run that check. Push only if it is green; fail = do not push, do not open the draft. Do not use a looser local subset. Before opening a draft PR, run scripts/pr-size-check. If this phase is two outcomes or the check exits non-zero, do not open the PR; report AWAITING SPLIT with the file list grouped by subsystem, the product-line count, and a proposed split. File count alone does not fail GATE. Otherwise open a draft PR, wait until CI is green on this SHA, then run the reviewer CLI and post gh pr review --comment. Write the worker status file (state, PR, SHA, product lines, review round, next action). Run scripts/gate-preflight. Never gh pr ready. Report AWAITING GATE with branch, PR, SHA, COMMENT URL, and the quality-gate checklist (playbooks/general.md#awaiting-gate-report). If the product repo has no make check, or CI does not run on this SHA, say so in that report.' \
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
  "$GROK --no-auto-update --always-approve -m <coder-model> --cwd <worktree> \
     --continue -p 'Execute the next incomplete worker phase. Skip staging. Same gate protocol. One outcome only — do not smuggle a second outcome into the same PR. Phase DoD is the product check commands; AWAITING GATE is illegal until they exit 0. Run that check in the worktree; push only if green. Run scripts/pr-size-check before opening a draft. Over cap or a second outcome: AWAITING SPLIT, do not open a megadiff PR. File count alone does not fail GATE. COMMENT only after this-SHA CI is green. Write the status file. Run scripts/gate-preflight.' \
     2>&1 | tee -a \"$LOG\"" Enter
```

If tmux is gone, infer `AWAITING GATE` from the worker status file plus git/PR: draft PR + COMMENT on this head + SHA. A missing pane is not a dead track. Do not invent state from tmux. Do not re-dispatch. Prefer `--continue` or `--resume <uuid>` over a new conversation. Interactive TUI (`--no-alt-screen` in tmux) is for live steer only.

If the controller cannot paste into tmux, write the instruction to a file and send a short Read of that file.

Autonomy for in-scope writes: `--always-approve --permission-mode bypassPermissions`. Still no production, no new paid secrets, no destructive data. DNS is the registrar or dashboard. An in-repo `CNAME` file is not DNS. `wrangler dns` does not exist.

Heartbeat and GATE read [status](../examples/status.schema.yaml) + `scripts/track-status` + the append-only [GATE log](../examples/sample-gate-log.tsv). Run `scripts/gate-preflight` before `AWAITING GATE`. See [Runtime facts](general.md#runtime-facts).

### Worker: Codex

A Grok Bot controller may dispatch a Codex worker on a fleet node when that CLI is the better fit. Use the Codex launch commands in [codex.md](codex.md#worker-codex) (steerable `codex` or unattended `codex exec`), with a pinned `<coder-model>` and a dedicated worktree. Review then prefers Claude Code on a fleet node (different family).

### Worker: Claude Code

A Grok Bot controller may dispatch a Claude Code worker on a fleet node when that CLI is the better fit. Use the Claude Code worker notes in [claude.md](claude.md#worker-claude-code), with a pinned `<coder-model>`. Review then prefers Codex on a fleet node (different family).

## CoS gate

Agent-side fleet COMMENT is already the independent-model check. Default CoS gate is **not** a second review pass on every low-risk diff:

1. Outcome, then size: this PR is still one graph node / one outcome. Run `scripts/gate-preflight` (exactly one node, COMMENT on this SHA, `scripts/pr-size-check`, CI green or verified no-CI). A second outcome, or exit 2 / `AWAITING SPLIT`, is REVISE to split. Do not COMMENT-review a +11k PR hoping the reviewer will save it. Do not GATE. File count alone does not fail GATE.
2. Clobber-check; rebase; serialize merges.
3. If rebase moved the SHA, repeat the size check on the new head, wait for this-SHA CI (or record that no current-head run exists), then run a fresh COMMENT review on the new head.
4. Confirm COMMENT exists for **this** head.
5. Exact-head CI, or the report that no current-head run exists.
6. Read the verdict.
7. After repo-gate evidence is green, mark **only the next PR in merge order** ready (`gh pr ready`). Leave later PRs draft. Workers never `gh pr ready`. Emit `WAITING ON YOU: merge PR #N` (or accept residual / unlock REVISE). Do not merge while waiting. After the human merges, rebase and re-gate the next PR before marking it ready.

After merge: watch default-branch CI, deploy staging, live-verify the merged SHA. Staging is CoS-only.

If the chosen path is fleet dispatch and dispatch fails, escalate dispatch. Do not silently switch the locus to the controller session.

`AWAITING GATE` in the graph is the repo gate. The human-visible line is `WAITING ON YOU: merge PR #N` (or accept residual / unlock REVISE). Repository merge at GATE is a human action, not a CoS auto-merge. Do not go quiet after one notice. Do not phrase that decision as a status.

## Escalate

Same blocker on one track:

1. Capture evidence (command, SHA, log, failure).
2. One REVISE with file, failure, and required proof.
3. Stop the track and escalate. There is no attempt 4.

Independent tracks continue. If a same-class P1 was already fixed once in this SHA lineage, accept the residual or escalate — do not open another REVISE loop.

## Heartbeat

Grok Bot cannot sit in a turn. While any track is **implementing or in CI**, put a **10-minute routine on the controller every calendar day, including weekends.** Prompt is this heartbeat plus "take the next repo-safe step; do not double-dispatch a progressing track." Re-run `scripts/fleet-preflight` on that cadence; do not cache host IPs. A weekday-only schedule misses weekend tracks; do not use one.

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
