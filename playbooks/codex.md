# Codex controller playbook

Use this when **Codex** is the Chief of Staff. Implementation is Codex CLI on a fleet node, or a recorded Codex CoS session ([ADR 0009](../docs/adr/0009-coding-locus-and-review-gate.md)). Review always runs on a fleet node, in a fresh session, on a different and at-least-as-capable model, preferably a different family (Claude or Grok when available). Same-family Codex review is last-resort only, with the reason on the PR.

## Model contract

| Role | Required model |
| --- | --- |
| Implement | `<coder-model>` (pin a current Codex model; high effort) |
| Authoritative review | `<reviewer-model>` on a fleet node, different family preferred |
| High-risk re-review | Fresh fleet review with a qualifying model and a risk-specific prompt |

Pin the explicit model names at dispatch. Do not float a generic alias. See the example table as of 2026-11 in [general.md](general.md#model-pins). Prefer a different family (Claude or Grok) when one is available and at least as capable. A same-family Codex reviewer is allowed only when no other family is on the fleet, and the PR states that reason.

## Loop

```text
dispatch precise task/goal
  → <coder-model> implements and verifies
  → CI-equivalent check in the worktree; red = do not push
  → scripts/pr-size-check; over cap → AWAITING SPLIT (do not open a megadiff)
  → draft PR; this-SHA CI green; then fleet COMMENT, AWAITING GATE
  → CoS runs a fresh fleet review with a qualifying model
  → REVISE or WAITING ON YOU: merge
  → human merges → staging → live-verify
```

## Dispatch

Steerable:

```sh
codex -m <coder-model> \
  -c model_reasoning_effort=high \
  --no-alt-screen \
  --dangerously-bypass-approvals-and-sandbox \
  -C <absolute-worktree-path> \
  "Read <absolute-goal-prompt-path> and execute the next incomplete worker phase end-to-end. Skip staging. In the assigned worktree, run the product repo CI-equivalent check (make check, or the exact commands from that repo CI). Push only if green. Run scripts/pr-size-check before opening a draft. Over cap: AWAITING SPLIT, do not open a megadiff. COMMENT only after this-SHA CI is green. See playbooks/general.md#cicd."
```

Unattended:

```sh
codex exec -m <coder-model> \
  -c model_reasoning_effort=high \
  --dangerously-bypass-approvals-and-sandbox \
  -C <absolute-worktree-path> \
  "Read <absolute-goal-prompt-path> and execute the next incomplete worker phase end-to-end. Skip staging. In the assigned worktree, run the product repo CI-equivalent check (make check, or the exact commands from that repo CI). Push only if green. Run scripts/pr-size-check before opening a draft. Over cap: AWAITING SPLIT, do not open a megadiff. COMMENT only after this-SHA CI is green. See playbooks/general.md#cicd."
```

Run in a named tmux session. Verify `codex --version` and `codex login status` before first use on a host. Never implement in the shared default-branch checkout.

For a small task on an existing persistent session: send one precise outcome, press Enter, and verify the pane started. Confirm the session is still the pinned `<coder-model>`.

## Review

From a fresh session on a **fleet node** and a clean worktree against the current PR head. Prefer a different-family reviewer CLI when one is available; the snippet below is the Codex-reviewer form used only when that last-resort applies, or when the chosen `<reviewer-model>` is a Codex model. Follow the SHA-pin protocol in [general.md](general.md): tee to `$HOME/.cortex/reviews/<repo>-pr<n>-<sha>.md`, check exit status, pin `commit_id` to the exact head, abort if the live head moved, then `gh pr review --comment --body-file`. Never interpolate the verdict into `--body`.

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

Never Approve. Read the verdict and check material findings against the actual head.

## Worker must

Same handoff as [general.md](general.md): real caller path, product repo CI-equivalent check in the worktree before push ([CI/CD](general.md#cicd)), mutation tests, `scripts/pr-size-check` before the draft (over cap → `AWAITING SPLIT`, do not open a megadiff), draft PR, COMMENT only after this-SHA CI is green, `AWAITING GATE`, no `gh pr ready`, no self-merge, no production, no staging dispatch.

## Gate and merge

Rebase, clobber-check. If rebase moved the SHA, wait for this-SHA CI (or record that no current-head run exists), then post a fresh COMMENT. Confirm COMMENT covers **this** head. CI green on this SHA, or the report that no current-head run exists. Mark **only the next PR in merge order** ready, then emit `WAITING ON YOU: merge PR #N`. Leave later PRs draft. Do not merge while waiting. After the human merges, rebase and re-gate the next PR, then watch default-branch CI and staging. A green PR is not proof that the default branch deployed. Live-verify the endpoint, data effect, or UI.

## Heartbeat

Ten-minute cadence while tracks run. Every CoS-visible message starts with WORKING, WAITING ON YOU, or BLOCKED; see [general.md](general.md#heartbeat). Do not go quiet at GATE. Watchdog may inspect and repeat an authorized steer. It may not redefine scope, invent gates, merge production, or retry an unresolved escalation indefinitely. Three attempts, then escalate.
