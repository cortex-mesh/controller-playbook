# General controller playbook

Controller-agnostic rules. Variant playbooks add model names and launch flags; they do not relax these rules.

## Roles

See [ADR 0001](../docs/adr/0001-cos-worker-reviewer.md). The CoS coordinates and gates. The worker implements one phase. The reviewer is a different session.

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
- If a host is missing the CLI or logged out, skip it and note it.
- A laptop-only pool is valid. Isolation still comes from worktrees.

## Loop

```text
CoS writes graph → dispatch each unblocked track to a free worker
  → worker implements this phase
  → different-family COMMENT review, draft PR, AWAITING GATE
  → CoS confirms review on this head, clobber-checks, exact-head CI
  → high-risk: fresh reviewer COMMENT
  → REVISE or GATE → serialize merge → staging → live-verify
```

Production stays a human gate.

## Dispatch checklist

1. Read the goal progress log. Name the next incomplete phase.
2. Confirm the track is unblocked in the dependency graph.
3. Pick the least-loaded host that fits.
4. `git fetch` and `git worktree add` from the current default-branch tip.
5. Launch the worker against the **absolute path** of the goal prompt.
6. Confirm the process is alive. Record host, worktree, session, branch.

## Worker must

- Implement only in its worktree.
- Exercise the real caller path, not a mock of the mock.
- Run the repo's lint, test, typecheck, and build.
- Mutation-test new tests: revert the fix, the test must fail.
- Open a **draft** PR.
- Run the reviewer CLI from a clean tree and post `gh pr review --comment`.
- Report `AWAITING GATE` with branch, PR, head SHA, COMMENT URL, and risks.
- Never self-merge. Never ship production.

Do not revert a correct fix to satisfy a stale test. Update the test and prove it fails without the fix.

## CoS gate

1. Clobber-check against other live tracks. Rebase on the latest default branch.
2. If rebase moved the SHA, run a fresh COMMENT review on the new head.
3. Confirm a COMMENT review exists for **this** head.
4. CI green on this SHA. Then live-verify staging. Green CI is not enough.
5. Read the verdict. Do not merge on a grep for "no blocking."

Escalate to a fresh CoS-run review for schema, auth, migration, security, data-write, or a thin pre-review.

## Heartbeat

Every 10 minutes while any track is running, including "still working." Immediate message on stuck, steer, block, or `AWAITING GATE`. Quiet only when idle.

```text
2026-08-13 06:45:00 PDT
dev-1 still working track harbor-api (repo#12, SHA abc1234). CI in progress.
Will check in again in 10 minutes.
```

## Human-stop

Production, secrets, paid resources, DNS, destructive data, and locked-decision changes. See [docs/design.md](../docs/design.md).
