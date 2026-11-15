# Codex as controller

Use this setup when **Codex** is the Chief of Staff. Shared rules are in [general.md](general.md). This page is the Codex setup: how it dispatches workers, when it may code in its own session, which fleet reviewers pair with it, and its limits.

Review always runs on a fleet node, in a fresh session, on a different and at-least-as-capable model, preferably a different family (Claude Code or Grok when available), posted as a GitHub `COMMENT`. Never Approve. Same-family Codex review is last-resort only, with the reason on the PR.

## Dispatching workers

Verify `codex --version` and `codex login status` before first use on a host. Create a dedicated worktree. Never implement in the shared default-branch checkout. Name the next incomplete **worker** phase. Do not dispatch staging. Run `scripts/fleet-preflight` against the local pool file; see [general.md](general.md#worker-pool-fill-in). If the chosen path is fleet dispatch and no logged-in host fits, escalate dispatch. Do not silently switch the locus to the controller session.

This setup commonly dispatches a **Codex** worker. It may also dispatch a Grok CLI or Claude Code worker, or implement in a recorded Codex session. Launch snippets are in [Worker CLIs](#worker-clis). Run the worker in a named tmux session.

```text
dispatch precise task/goal
  → <coder-model> implements and verifies (or a recorded internal session)
  → CI-equivalent check in the worktree; red = do not push
  → scripts/pr-size-check; over cap → AWAITING SPLIT (do not open a megadiff)
  → draft PR; this-SHA CI green; then fleet COMMENT, AWAITING GATE
  → CoS runs a fresh fleet review with a qualifying model
  → REVISE or WAITING ON YOU: merge
  → human merges → staging → live-verify
```

Same worker handoff as [general.md](general.md): real caller path, product repo CI-equivalent check in the worktree before push ([CI/CD](general.md#cicd)), mutation tests, `scripts/pr-size-check` before the draft, draft PR, COMMENT only after this-SHA CI is green, `AWAITING GATE`, no `gh pr ready`, no self-merge, no production, no staging dispatch.

## Internal-session coding

Codex may implement in its own session only as an explicit, recorded choice ([ADR 0009](../docs/adr/0009-coding-locus-and-review-gate.md)). Isolation still comes from a dedicated worktree, a draft PR, and the repository gate. The session that writes the change does not review it, does not run `gh pr ready`, and does not merge.

When the session exposes a model selector, pin `<coder-model>` (high effort). Do not float a generic alias. If the in-session model cannot be pinned or named, the PR records:

```
Coded in: internal session
Coded by: controller internal session / Codex / <model not pinnable>
```

In that case the fleet reviewer is the strongest available qualifying tier (see [Fleet reviewer pairings](#fleet-reviewer-pairings)).

For a small task on an existing persistent session: send one precise outcome, press Enter, and verify the pane started. Confirm the session is still the pinned `<coder-model>` when a pin is available.

## Fleet reviewer pairings

Prefer a **different model family** from the coder, on a fleet node, at least as capable and preferably more capable:

| Coder | Preferred fleet reviewer |
| --- | --- |
| A Codex worker, or a Codex internal session | Claude Code or Grok, strongest available qualifying tier |
| A Grok CLI worker dispatched by this controller | Claude Code or Codex, strongest available qualifying tier |
| A Claude Code worker dispatched by this controller | Codex or Grok, as long as the reviewer is a different family from the coder |

Pin the explicit model names at dispatch and at review. See the example table as of 2026-11 in [general.md](general.md#model-pins). A same-family Codex reviewer is allowed only when no other family is on the fleet, must be more capable, and the PR states that reason.

Never review in the coder's session. From a fresh session on a **fleet node** and a clean worktree against the current PR head. Prefer a different-family reviewer CLI when one is available. For a Claude Code reviewer, use the review command in [claude.md](claude.md#fleet-reviewer-pairings). The snippet below is the Codex-reviewer form, used when the chosen `<reviewer-model>` is a Codex model. That is last-resort same-family only when the coder was also Codex.

Follow the SHA-pin protocol in [general.md](general.md#review): tee to `$HOME/.cortex/reviews/<repo>-pr<n>-<sha>.md`, check exit status, pin `commit_id` to the exact head, abort if the live head moved, then `gh pr review --comment --body-file`. Never interpolate the verdict into `--body`.

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

Never Approve. Read the verdict and check material findings against the actual head. High-risk diffs get a fresh fleet review with a qualifying model and a risk-specific prompt.

## Limits

- Do not float a generic model alias. Pin `<coder-model>` and `<reviewer-model>` when the CLI exposes a selector.
- If the in-session model cannot be pinned or named, record `<model not pinnable>` and pick the strongest available fleet reviewer tier.
- Same-family Codex review is last-resort only. Prefer Claude Code or Grok when either is available and at least as capable.
- Shared method rules, coverage, the quality-gate checklist, and the human-only waiver are in [general.md](general.md). The Cursor IDE is not a controller setup; see [general.md](general.md#cursor-ide-is-operator-homework).

## Worker CLIs

### Worker: Codex

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

### Worker: Grok CLI

A Codex controller may dispatch a Grok CLI worker on a fleet node. Use the Grok CLI launch commands in [grok.md](grok.md#worker-grok-cli), with a pinned `<coder-model>` and a dedicated worktree. Review then prefers Claude Code (different family).

### Worker: Claude Code

A Codex controller may dispatch a Claude Code worker on a fleet node. Use the Claude Code worker notes in [claude.md](claude.md#worker-claude-code), with a pinned `<coder-model>`. Review then prefers Grok or Codex (both a different family from the Claude Code coder).

## Gate and merge

Rebase, clobber-check. If rebase moved the SHA, wait for this-SHA CI (or record that no current-head run exists), then post a fresh COMMENT. Confirm COMMENT covers **this** head. CI green on this SHA, or the report that no current-head run exists. Mark **only the next PR in merge order** ready, then emit `WAITING ON YOU: merge PR #N`. Leave later PRs draft. Do not merge while waiting. After the human merges, rebase and re-gate the next PR, then watch default-branch CI and staging. A green PR is not proof that the default branch deployed. Live-verify the endpoint, data effect, or UI.

## Heartbeat

Ten-minute cadence while tracks run. Every CoS-visible message starts with WORKING, WAITING ON YOU, or BLOCKED; see [general.md](general.md#heartbeat). Do not go quiet at GATE. Watchdog may inspect and repeat an authorized steer. It may not redefine scope, invent gates, merge production, or retry an unresolved escalation indefinitely. Three attempts, then escalate.
