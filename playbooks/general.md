# General controller playbook

Controller-agnostic rules. Variant playbooks add model names and launch flags; they do not relax these rules.

## Roles

See [ADR 0001](../docs/adr/0001-cos-worker-reviewer.md). The CoS coordinates and gates. The worker implements one phase. The reviewer is a different session and a different model family.

## Worker pool (fill in)

A worker is any host you can reach that has the implementer CLI installed and logged in. VPN, SSM, LAN SSH, or "this laptop" are all fine.

| Host | SSH | Fit | Max parallel tracks |
| --- | --- | --- | --- |
| `dev-1` | `user@host` | General coding | 1 |
| `dev-2` | `user@host` | General coding | 1 |
| `ci-box` | `user@host` | Heavy jobs; do not starve CI | 1 |

Rules:

- Before dispatch, list tmux and implementer sessions. Do not double-dispatch a busy host.
- One git worktree per track on the host that owns the track.
- Login (implementer models + `gh`) is a per-host dispatch input. A logged-out host does not block the wave. Re-pick the least-loaded logged-in host and write the new owner into the graph.
- If no logged-in host fits, escalate dispatch. The CoS does not become the implementer.
- A laptop-only pool is valid. Isolation still comes from worktrees.

## Loop

```text
CoS writes graph → dispatch each unblocked track to a free worker
  → worker implements this phase
  → draft PR, then COMMENT, AWAITING GATE
  → CoS confirms review on this head, clobber-checks, exact-head CI
  → high-risk: fresh reviewer COMMENT
  → REVISE or GATE → CoS marks ready → serialize merge → CoS staging → live-verify
```

Workers stop at `AWAITING GATE`. Staging / last integration is CoS-only. Production stays a human gate.

## Dispatch checklist

1. Read the goal progress log. Name the next incomplete **worker** phase. Do not dispatch staging or last integration.
2. Confirm the track is unblocked in the dependency graph.
3. Pick the least-loaded host that fits and is logged in (implementer + `gh`).
4. If the assigned host is logged out, re-pick and write the new owner into the graph.
5. `git fetch` and `git worktree add` from the current default-branch tip.
6. Launch the worker against the **absolute path** of the goal prompt.
7. Confirm the process is alive. Record host, worktree, session, branch.

If the controller cannot paste into tmux, write the instruction to a file and send a short Read of that file.

A missing tmux pane is not a dead track. Infer `AWAITING GATE` from git/PR: draft PR + COMMENT on this head + SHA. Do not re-dispatch.

If dispatch fails, escalate dispatch. The CoS does not implement.

## Worker must

- Implement only in its worktree.
- Exercise the real caller path, not a mock of the mock.
- Run the repo's lint, test, typecheck, and build.
- Mutation-test new tests: revert the fix, the test must fail.
- Open a **draft** PR. Never `gh pr ready`.
- Run the reviewer CLI from a clean tree. Tee the verdict outside the repo. Post `gh pr review --comment --body-file <verdict.md>` on the pinned head.
- Report `AWAITING GATE` with branch, PR, head SHA, COMMENT URL, and risks.
- Stop. Never self-merge. Never ship production. Never start staging.

Do not revert a correct fix to satisfy a stale test. Update the test and prove it fails without the fix.

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

1. Clobber-check against other live tracks. Rebase on the latest default branch.
2. If rebase moved the SHA, run a fresh COMMENT review on the new head.
3. Confirm a COMMENT review exists for **this** head.
4. CI green on this SHA.
5. Read the verdict. Do not merge on a grep for "no blocking."
6. After GATE, mark the draft ready. Workers never run this. GitHub will not merge a draft. Do not run this before COMMENT and exact-head CI are green.

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

Implementing or CI: every 10 minutes, dual timestamps. `AWAITING GATE`: one immediate notice, then quiet until the CoS issues REVISE or GATE. Stuck or steer still fire immediately. Quiet when idle.

```text
Thu Aug 13, 2026, 6:45:00 AM PT
2026-08-13 06:45:00 PDT
dev-1 still working track harbor-api (repo#12, SHA abc1234). CI in progress.
Will check in again in 10 minutes.
```

## Human-stop

Production, secrets, paid resources, DNS, destructive data, and locked-decision changes. See [docs/design.md](../docs/design.md).

DNS is the registrar or dashboard. An in-repo `CNAME` file is not DNS. `wrangler dns` does not exist.
