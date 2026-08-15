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

The living graph is YAML ([schema](../examples/living-graph.schema.yaml), [Harbor example](../examples/sample-dependency-graph.yaml)). The markdown table is the human snapshot. Before a wave, run `scripts/graph-lint` on the YAML. Lint fails on a missing required field, two outcomes on one node, overlapping in-flight file globs, REVISE rounds above 3, or an over-cap / `AWAITING SPLIT` node marked `AWAITING GATE`. Graph facts stay there: `scripts/graph-lint` and `scripts/track-status` are the source. Do not copy them into chat.

**Mapping:** phase N → graph node → one PR → one outcome.

A worker implements this phase only. Do not add a second outcome to the same PR. If the change set is "the whole feature," it is already too big: split into phases before anyone codes.

Use a short phased ADR only when the *shape* of the system changes (new runtime, new auth boundary, new deploy path). The ADR lists the phases so later tracks cannot collapse them. Do not ADR a rename, a lockfile bump, or a docs-only fix.

## Loop

```text
CoS writes phases (graph + goal prompt) → dispatch each unblocked track
  → worker implements this phase's one outcome
  → CI-equivalent check in the worktree; red = do not push
  → second outcome or over size cap: AWAITING SPLIT (CoS updates graph, re-dispatch)
  → draft PR; this-SHA CI green; then COMMENT, AWAITING GATE
  → CoS confirms review on this head, clobber-checks, exact-head CI
  → high-risk: fresh reviewer COMMENT
  → REVISE or WAITING ON YOU: merge → human GATE → CoS staging → live-verify
```

Workers stop at `AWAITING GATE` or `AWAITING SPLIT`. Staging / last integration is CoS-only. Production stays a human gate.

## Runtime facts

Heartbeat and GATE read files, not chat. The CoS does not invent state, PR, SHA, product lines, review round, or next action from tmux output.

- **Graph.** Living YAML via `scripts/graph-lint` and `scripts/track-status`.
- **Worker status.** [schema](../examples/status.schema.yaml), [Harbor example](../examples/sample-status.yaml). The worker writes; the CoS reads. Required: `state`, `pr`, `head_sha`, `added_product_lines`, `review_round`, `next_action`. Live file is local (`STATUS=` or next to the graph).
- **GATE log.** Append-only ([example](../examples/sample-gate-log.tsv)). Events: `DISPATCH`, `REVISE 1`, `REVISE 2`, `REVISE 3`, `SPLIT`, `GATE`, `HUMAN`. Survives a new chat. 3-cap and merge order come from this log plus the graph YAML. There is no `REVISE 4`.
- **Preflight.** Before `AWAITING GATE` / before GATE, run `scripts/gate-preflight`. It fails unless this PR maps to exactly one graph node, a COMMENT review is on this SHA, `scripts/pr-size-check` is green, and current-head CI is green or verified no-CI.

A missing pane is still not DEAD. Infer from status + `scripts/track-status`, not from poetry in the pane.

## Dispatch checklist

1. Read the goal progress log. Name the next incomplete **worker** phase (one graph node, one PR, one outcome). Do not dispatch staging or last integration.
2. Confirm the track is unblocked in the dependency graph. Run `scripts/graph-lint` on the YAML before the wave.
3. Confirm blast radius is one subsystem / one outcome. Two subsystems or two outcomes is already two tracks — write the split in the graph first.
4. Run `scripts/fleet-preflight` against the **local** pool file (argv or `FLEET_POOL`). Resolve live; do not reuse a remembered IP. Pick the `recommended:` free logged-in host.
5. `login-needed` is AWAITING LOGIN, not down — skip that host, re-pick, and write the new owner into the graph. A timeout is not "no hosts"; finish the table. Leftover tmux panes without a live implementer process are not busy.
6. `git fetch` and `git worktree add` from the current default-branch tip.
7. Launch the worker against the **absolute path** of the goal prompt.
8. Confirm the process is alive. Record host, worktree, session, branch.

If the controller cannot paste into tmux, write the instruction to a file and send a short Read of that file.

A missing tmux pane is not a dead track. Run `scripts/track-status` and infer `DISPATCHED` / `AWAITING GATE` / `DEAD` from git, PR, and tmux facts. Read the worker status file. Do not invent those fields from tmux. A missing pane is not a re-dispatch if a draft PR exists. Infer `AWAITING GATE` from draft PR + COMMENT on this head + SHA.

If dispatch fails, escalate dispatch. The CoS does not implement.

## Worker must

- Implement only this phase's one outcome. Do not smuggle a second outcome into the same PR. If the phase is already two outcomes, report `AWAITING SPLIT` with a proposed split.
- Implement only in its worktree.
- Exercise the real caller path, not a mock of the mock.
- Run the product repo's CI-equivalent check in this worktree (see [CI/CD](#cicd)). Push and open a draft only if it is green. Fail = do not push, do not open the draft.
- Mutation-test new tests: revert the fix, the test must fail.
- Before opening a draft PR, run `scripts/pr-size-check` (see [PR size](#pr-size)). If it exits non-zero / prints `AWAITING SPLIT`, do not open the PR. Report `AWAITING SPLIT` with the file list grouped by subsystem, the product-line count, and a proposed split.
- Open a **draft** PR only when it is still one outcome and under the hard cap. Never `gh pr ready`.
- After current-head CI is green on this SHA, run the reviewer CLI from a clean tree. Tee the verdict outside the repo. Post `gh pr review --comment --body-file <verdict.md>` on the pinned head. Do not COMMENT-review a megadiff.
- Write the worker status file (schema above). Phase DoD is the product check commands (`make check`, or the exact commands from that repo CI). `AWAITING GATE` is illegal until those commands exit 0. If there is no `make check` and no current-head CI, record `ci: none` and say so.
- Run `scripts/gate-preflight` before reporting `AWAITING GATE`.
- Report `AWAITING GATE` with branch, PR, head SHA, COMMENT URL, and risks. Never report `AWAITING GATE` on an over-cap PR or a two-outcome PR.
- Stop. Never self-merge. Never ship production. Never start staging.

Do not revert a correct fix to satisfy a stale test. Update the test and prove it fails without the fix.

## CI/CD

The **product repo** owns the check command that CI runs. Prefer `make check`. If there is no such target, use the exact commands from that repo's CI workflow. GitHub Actions (or the product CI) must call that same target so the lists cannot drift. Do not copy a product Makefile into this playbook repo.

**Worker.** Run that command **in the assigned worktree**. Push only if it is green. Fail = do not push, do not open the draft. Do not substitute a looser local subset (`make lint` alone, a single formatter, one test file).

Git hooks are optional extra, not the control. Unattended workers may skip hooks (`git push --no-verify` or a permission bypass). The worker must invoke the gate itself.

If the product repo has no `make check`, or has no check that runs on this SHA (no CI, or CI that only fires on the default branch), say so in the `AWAITING GATE` report. Do not invent a subset.

**Review.** COMMENT starts only after current-head CI is green on this SHA. GATE already requires exact-head CI. If there is no current-head CI run to wait for, say so in `AWAITING GATE` and continue to COMMENT.

**Merge is not deploy.** Staging is CoS-only after default-branch CI. Production is human.

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

Start COMMENT only after current-head CI is green on this SHA (see [CI/CD](#cicd)). Resolve `<default>` from this repo; do not assume `main`. Tee the verdict outside the repo (`$HOME/.cortex/reviews/<repo>-pr<n>-<sha>.md` or `$HOME/.grok/reviews/...`). Check the reviewer exit status. Pin `commit_id` to the exact head. Abort if the live head is not that SHA. Never interpolate the verdict into `--body`. Never Approve.

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

1. Outcome, then size: this PR is still one graph node / one outcome. Run `scripts/gate-preflight` (exactly one node, COMMENT on this SHA, `scripts/pr-size-check`, CI green or verified no-CI). A second outcome, or exit 2 / `AWAITING SPLIT`, is REVISE to split. Do not COMMENT-review a +11k PR hoping the reviewer will save it. Do not GATE. File count alone does not fail GATE.
2. Clobber-check against other live tracks. Rebase on the latest default branch.
3. If rebase moved the SHA, repeat the size check on the new head, wait for this-SHA CI (or record that no current-head run exists), then run a fresh COMMENT review on the new head.
4. Confirm a COMMENT review exists for **this** head.
5. CI green on this SHA, or the report that no current-head run exists.
6. Read the verdict. Do not merge on a grep for "no blocking."
7. After repo-gate evidence is green, mark the draft ready so the human can merge. Workers never run this. GitHub will not merge a draft. Do not run this before COMMENT and exact-head CI are green.

```sh
gh pr ready <PR#>
```

Then emit `WAITING ON YOU: merge PR #N` (or accept residual / unlock REVISE). Do not merge while waiting. Do not go quiet. After the human GATEs, serialize any remaining merge order and watch default-branch CI.

Escalate to a fresh CoS-run review for schema, auth, migration, security, data-write, or a thin pre-review.

After merge: watch default-branch CI, deploy staging, live-verify the merged SHA. Green pre-merge CI is not a staging check. Staging is CoS-only.

`AWAITING GATE` in the graph is the repo gate. The human-visible line is `WAITING ON YOU: merge PR #N` (or accept residual / unlock REVISE). Repository merge at GATE is a human action, not a CoS auto-merge. Do not phrase that decision as a status ("merge as-is, or I send the worker back"). If the CoS is still running gate-preflight, rebase, or COMMENT confirm, the line is `WORKING`.

## Escalate

Same blocker on one track:

1. Capture evidence (command, SHA, log, failure).
2. One REVISE with file, failure, and required proof.
3. Stop the track and escalate. There is no attempt 4.

Independent tracks continue. If a same-class P1 was already fixed once in this SHA lineage, accept the residual or escalate — do not open another REVISE loop.

## Heartbeat

Every CoS-visible message starts with one label:

- **WORKING** — CoS/worker still going. No action from the human. 10-minute heartbeat continues ("will check in again in 10 minutes").
- **WAITING ON YOU** — first line, then the exact action (merge PR #N, authorize device login, approve a graph split). Stop the 10-minute "still working" heartbeat. Do not go silent. If still blocked on the human after about 30-60 minutes (or the next morning), one reminder that still starts WAITING ON YOU, not a 10-minute drip.
- **BLOCKED** — not the human (implementer logged out, CI, SSH). Name what is blocked and whether the CoS is unblocking it.

Do not phrase a human decision as a status ("merge as-is, or I send the worker back"). That is WAITING ON YOU.

`AWAITING GATE` in the graph still means the repo gate. The human-visible line is `WAITING ON YOU: merge PR #N` (or accept residual / unlock REVISE). Quiet-at-GATE / one notice then silent is not allowed. Stop the 10-minute WORKING drip, but keep a wake-up: one reminder after about 30-60 minutes or the next morning. `AWAITING SPLIT`: `WORKING` if the CoS can write the split and re-dispatch; `WAITING ON YOU: approve a graph split` if the human must approve it.

Implementing or CI: every 10 minutes, dual timestamps, label `WORKING` first. Re-run `scripts/fleet-preflight` on that cadence (live resolve; do not cache IPs). Re-run `scripts/track-status` on the same cadence so a vanished pane is not mistaken for a dead track. Read the worker status file and the GATE log. Do not invent state, PR, SHA, or next action from tmux or chat. Stuck or steer still fire immediately. Quiet when idle (no live tracks).

```text
WORKING
Thu Aug 13, 2026, 6:45:00 AM PT
2026-08-13 06:45:00 PDT
dev-1 still working track harbor-api (repo#12, SHA abc1234). CI in progress.
Will check in again in 10 minutes.
```

```text
WAITING ON YOU: merge PR #12
Track harbor-berth-api is AWAITING GATE in the graph. COMMENT on abc1234.
```

```text
BLOCKED: implementer logged out on dev-1. CoS is re-picking a logged-in host; not waiting on you.
```

## Human-stop

Production, secrets, paid resources, DNS, destructive data, and locked-decision changes. See [docs/design.md](../docs/design.md).

DNS is the registrar or dashboard. An in-repo `CNAME` file is not DNS. `wrangler dns` does not exist.
