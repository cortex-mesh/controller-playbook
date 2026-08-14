# Codex controller playbook

Use this when **Codex** is the Chief of Staff. Implementation is Codex CLI. Review is a **fresh** `gpt-5.6-sol` / high session — not the implementing session.

## Model contract

| Role | Required model |
| --- | --- |
| Implement | `gpt-5.6-terra`, `model_reasoning_effort=high` |
| Authoritative review | `gpt-5.6-sol`, `model_reasoning_effort=high` |
| High-risk re-review | Fresh sol/high with a risk-specific prompt |

Pin the explicit model names. Do not float a generic alias, and do not use Opus as the authoritative reviewer on this path.

## Loop

```text
dispatch precise task/goal
  → Terra/high implements and verifies
  → scripts/pr-size-check; over cap → AWAITING SPLIT (do not open a megadiff)
  → draft PR, AWAITING GATE
  → CoS runs fresh Sol/high review
  → REVISE or GATE
  → serialize merge → staging → live-verify
```

## Dispatch

Steerable:

```sh
codex -m gpt-5.6-terra \
  -c model_reasoning_effort=high \
  --no-alt-screen \
  --dangerously-bypass-approvals-and-sandbox \
  -C <absolute-worktree-path> \
  "Read <absolute-goal-prompt-path> and execute the next incomplete worker phase end-to-end. Skip staging. Run scripts/pr-size-check before opening a draft. Over cap: AWAITING SPLIT, do not open a megadiff."
```

Unattended:

```sh
codex exec -m gpt-5.6-terra \
  -c model_reasoning_effort=high \
  --dangerously-bypass-approvals-and-sandbox \
  -C <absolute-worktree-path> \
  "Read <absolute-goal-prompt-path> and execute the next incomplete worker phase end-to-end. Skip staging. Run scripts/pr-size-check before opening a draft. Over cap: AWAITING SPLIT, do not open a megadiff."
```

Run in a named tmux session. Verify `codex --version` and `codex login status` before first use on a host. Never implement in the shared default-branch checkout.

For a small task on an existing persistent session: send one precise outcome, press Enter, and verify the pane started. Confirm the session is still Terra/high.

## Review

From a fresh session and a clean worktree against the current PR head. Follow the SHA-pin protocol in [general.md](general.md): tee to `$HOME/.cortex/reviews/<repo>-pr<n>-<sha>.md`, check exit status, pin `commit_id` to the exact head, abort if the live head moved, then `gh pr review --comment --body-file`. Never interpolate the verdict into `--body`.

Resolve `<default>` from this repo; do not assume `main`.

```sh
DEFAULT=$(git symbolic-ref --quiet --short refs/remotes/origin/HEAD \
  || echo origin/<default>)
REPO=$(basename "$(git rev-parse --show-toplevel)")
PR=<PR#>
PINNED=$(gh pr view "$PR" --json headRefOid --jq .headRefOid)
REVDIR="$HOME/.cortex/reviews"
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

Never Approve. Read the verdict and check material findings against the actual head.

## Worker must

Same handoff as [general.md](general.md): real caller path, repo gates, mutation tests, `scripts/pr-size-check` before the draft (over cap → `AWAITING SPLIT`, do not open a megadiff), draft PR, then COMMENT, `AWAITING GATE`, no `gh pr ready`, no self-merge, no production, no staging dispatch.

## Gate and merge

Rebase, clobber-check, then confirm COMMENT covers **this** head (fresh COMMENT if rebase moved the SHA). CI green on this SHA. Serialize merges. After merge, watch default-branch CI and staging. A green PR is not proof that the default branch deployed. Live-verify the endpoint, data effect, or UI.

## Heartbeat

Ten-minute cadence while tracks run. Watchdog may inspect and repeat an authorized steer. It may not redefine scope, invent gates, merge production, or retry an unresolved escalation indefinitely. Three attempts, then escalate.
