# General controller playbook

Controller-agnostic rules. Variant playbooks add model names and launch flags; they do not relax these rules.

## Roles

See [ADR 0001](../docs/adr/0001-cos-worker-reviewer.md). The CoS coordinates and gates. The worker implements one phase. The reviewer is a different session and a different model family.

## Worker pool (fill in)

A worker is any host you can reach that has the implementer CLI installed and logged in. VPN, SSM, LAN SSH, or "this laptop" are all fine. Keep the live list in a **local** pool file (see [examples/worker-pool.example.tsv](../examples/worker-pool.example.tsv)); do not commit real hosts.

| Host | SSH | Fit | Max parallel tracks |
| --- | --- | --- | --- |
| `dev-1` | `user@host` | General coding | 1 |
| `dev-2` | `user@host` | General coding | 1 |
| `ci-box` | `user@host` | Heavy jobs; do not starve CI | 1 |

Rules:

- Before every dispatch, run `scripts/fleet-preflight` on the local pool file. Resolve the pool live. Do not cache host IPs.
- Do not double-dispatch a busy host. Leftover tmux panes without a live implementer process are not busy.
- One git worktree per track on the host that owns the track.
- Login (implementer models + `gh`) is a per-host dispatch input. A logged-out host is AWAITING LOGIN, not down, and does not block the wave. Re-pick the least-loaded logged-in host and write the new owner into the graph. An SSH timeout on one host is not "no hosts" — finish the table.
- If no logged-in host fits, escalate dispatch. The CoS does not become the implementer.
- A laptop-only pool is valid. Isolation still comes from worktrees.

## Planning

Size is the backstop. Grouping is the method.

The CoS writes phases **before dispatch**, in the living graph and the goal prompt. Each phase is one graph node, one PR, and one mergeable outcome a reviewer can hold in their head.

**Mapping:** phase N → graph node → one PR → one outcome.

A worker implements this phase only. Do not add a second outcome to the same PR. If the change set is "the whole feature," it is already too big: split into phases before anyone codes.

Use a short phased ADR only when the *shape* of the system changes (new runtime, new auth boundary, new deploy path). The ADR lists the phases so later tracks cannot collapse them. Do not ADR a rename, a lockfile bump, or a docs-only fix.

## Loop

```text
CoS writes phases (graph + goal prompt) → dispatch each unblocked track
  → worker implements this phase's one outcome
  → second outcome or over size cap: AWAITING SPLIT (CoS updates graph, re-dispatch)
  → draft PR, then COMMENT, AWAITING GATE
  → CoS confirms review on this head, clobber-checks, exact-head CI
  → high-risk: fresh reviewer COMMENT
  → REVISE or GATE → CoS marks ready → serialize merge → CoS staging → live-verify
```

Workers stop at `AWAITING GATE` or `AWAITING SPLIT`. Staging / last integration is CoS-only. Production stays a human gate.

## Dispatch checklist

1. Read the goal progress log. Name the next incomplete **worker** phase (one graph node, one PR, one outcome). Do not dispatch staging or last integration.
2. Confirm the track is unblocked in the dependency graph.
3. Confirm blast radius is one subsystem / one outcome. Two subsystems or two outcomes is already two tracks — write the split in the graph first.
4. Run `scripts/fleet-preflight` against the **local** pool file (argv or `FLEET_POOL`). Resolve live; do not reuse a remembered IP. Pick the `recommended:` free logged-in host.
5. `login-needed` is AWAITING LOGIN, not down — skip that host, re-pick, and write the new owner into the graph. A timeout is not "no hosts"; finish the table. Leftover tmux panes without a live implementer process are not busy.
6. `git fetch` and `git worktree add` from the current default-branch tip.
7. Launch the worker against the **absolute path** of the goal prompt.
8. Confirm the process is alive. Record host, worktree, session, branch.

If the controller cannot paste into tmux, write the instruction to a file and send a short Read of that file.

A missing tmux pane is not a dead track. Infer `AWAITING GATE` from git/PR: draft PR + COMMENT on this head + SHA. Do not re-dispatch.

If dispatch fails, escalate dispatch. The CoS does not implement.

## Worker must

- Implement only this phase's one outcome. Do not smuggle a second outcome into the same PR. If the phase is already two outcomes, report `AWAITING SPLIT` with a proposed split.
- Implement only in its worktree.
- Exercise the real caller path, not a mock of the mock.
- Run the repo's lint, test, typecheck, and build.
- Mutation-test new tests: revert the fix, the test must fail.
- Before opening a draft PR, run `scripts/pr-size-check` (see [PR size](#pr-size)). If it exits non-zero / prints `AWAITING SPLIT`, do not open the PR. Report `AWAITING SPLIT` with the file list grouped by subsystem, the product-line count, and a proposed split.
- Open a **draft** PR only when it is still one outcome and under the hard cap. Never `gh pr ready`.
- Run the reviewer CLI from a clean tree. Tee the verdict outside the repo. Post `gh pr review --comment --body-file <verdict.md>` on the pinned head. Do not COMMENT-review a megadiff.
- Report `AWAITING GATE` with branch, PR, head SHA, COMMENT URL, and risks. Never report `AWAITING GATE` on an over-cap PR or a two-outcome PR.
- Stop. Never self-merge. Never ship production. Never start staging.

Do not revert a correct fix to satisfy a stale test. Update the test and prove it fails without the fix.

## PR size

Product files = `git diff` vs the default branch, excluding lockfiles, generated output, and snapshots: `package-lock.json`, `go.sum`, `*.lock`, `dist/`, `generated/`, `**/__snapshots__/**`, and `vendor/` (already vendored).

**Mechanical check.** Workers and CoS run `scripts/pr-size-check` before `AWAITING GATE`. No network. It counts added product lines vs `origin/HEAD` (or this repo's default branch) after the exclusions above. Exit 0 if under 800. Exit 2 at ≥800 and prints `AWAITING SPLIT` plus counts. File count is still a smell, not a fail.

```sh
scripts/pr-size-check
```

**Grouping (graph, before dispatch).** One node = one outcome. The CoS writes phases in the living graph and the goal prompt before anyone codes. "The whole feature" is already two or more phases. A worker must not add a second outcome to the same PR.

**Soft (subsystems).** If a track would touch two or more subsystems (for example compose/runtime vs IAM vs alarms vs app vs CI lockfile), it is already two tracks. Write that split in the living graph before anyone codes.

**Hard (repo gate).** Size is the backstop. A PR with ≥800 added product lines cannot report `AWAITING GATE`. The worker stops and reports `AWAITING SPLIT` with the file list grouped by subsystem, the product-line count, and a proposed split (one track per subsystem / one outcome). The CoS updates the graph and re-dispatches. There is no "justify the megadiff." Over-cap is not a 3-attempt loop. File count alone does not fail GATE.

**File count is a graph smell, not a cap.** High file count with a small product diff (renames, import paths) is still reviewable. Low file count with a huge rewrite is not. If the file list spans two or more subsystems, split before dispatch regardless of loc.

**Worker self-check.** Confirm one outcome. Then run `scripts/pr-size-check`. If this PR has a second outcome or the check exits non-zero, do not open a megadiff PR; split first or report `AWAITING SPLIT`.

## Review

Resolve `<default>` from this repo; do not assume `main`. Tee the verdict outside the repo (`$HOME/.cortex/reviews/<repo>-pr<n>-<sha>.md` or `$HOME/.grok/reviews/...`). Check the reviewer exit status. Pin `commit_id` to the exact head. Abort if the live head is not that SHA. Never interpolate the verdict into `--body`. Never Approve.

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
<reviewer-cli> --base "$DEFAULT" | tee "$VERDICT"
status=$?
if [ "$status" -ne 0 ]; then
  echo "abort: reviewer exit $status"
  exit "$status"
fi
LIVE=$(gh pr view "$PR" --json headRefOid --jq .headRefOid)
if [ "$LIVE" != "$PINNED" ]; then
  echo "abort: live head $LIVE != pinned $PINNED"
  exit 1
fi
gh pr review "$PR" --comment --body-file "$VERDICT"
```

## CoS gate

1. Outcome, then size: this PR is still one graph node / one outcome. Then run `scripts/pr-size-check`. A second outcome, or exit 2 / `AWAITING SPLIT`, is REVISE to split. Do not COMMENT-review a +11k PR hoping the reviewer will save it. Do not GATE. File count alone does not fail GATE.
2. Clobber-check against other live tracks. Rebase on the latest default branch.
3. If rebase moved the SHA, repeat the size check on the new head, then run a fresh COMMENT review on the new head.
4. Confirm a COMMENT review exists for **this** head.
5. CI green on this SHA.
6. Read the verdict. Do not merge on a grep for "no blocking."
7. After GATE, mark the draft ready. Workers never run this. GitHub will not merge a draft. Do not run this before COMMENT and exact-head CI are green.

```sh
gh pr ready <PR#>
```

Then serialize merge.

Escalate to a fresh CoS-run review for schema, auth, migration, security, data-write, or a thin pre-review.

After merge: watch default-branch CI, deploy staging, live-verify the merged SHA. Green pre-merge CI is not a staging check. Staging is CoS-only.

## Escalate

Same blocker on one track:

1. Capture evidence (command, SHA, log, failure).
2. One REVISE with file, failure, and required proof.
3. Stop the track and escalate. There is no attempt 4.

Independent tracks continue. If a same-class P1 was already fixed once in this SHA lineage, accept the residual or escalate — do not open another REVISE loop.

## Heartbeat

Implementing or CI: every 10 minutes, dual timestamps. Re-run `scripts/fleet-preflight` on that cadence (live resolve; do not cache IPs). `AWAITING GATE` or `AWAITING SPLIT`: one immediate notice, then quiet until the CoS issues REVISE, GATE, or a graph update and re-dispatch. Stuck or steer still fire immediately. Quiet when idle.

```text
Thu Aug 13, 2026, 6:45:00 AM PT
2026-08-13 06:45:00 PDT
dev-1 still working track harbor-api (repo#12, SHA abc1234). CI in progress.
Will check in again in 10 minutes.
```

## Human-stop

Production, secrets, paid resources, DNS, destructive data, and locked-decision changes. See [docs/design.md](../docs/design.md).

DNS is the registrar or dashboard. An in-repo `CNAME` file is not DNS. `wrangler dns` does not exist.
