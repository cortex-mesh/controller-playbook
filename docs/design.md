# Design

How a Chief of Staff, a worker pool, and a different-family reviewer ship a product without turning one chat into the system of record.

## Roles

| Role | Owns | Does not own |
| --- | --- | --- |
| **Human operator** | Goal, spend, secrets, DNS, production, unresolved product choices | Day-to-day implementation once decisions are locked |
| **Chief of Staff (CoS)** | Goal prompt, dependency graph, dispatch, gates, merge order, staging live-verify, heartbeat | Writing the feature in the same turn that gates it |
| **Worker** | One track, one worktree, one phase: implement, prove, draft PR, request review | Self-merge, production, inventing new gates |
| **Reviewer** | A fresh session in a **different model family** than the writer. Posts a GitHub `COMMENT` | Approve, Request changes, or rubber-stamping the implementer |

Do not mint extra bots named dispatcher, coder, reviewer, or merger. Those are steps. One CoS per standing product goal. If two controllers must share a goal, use a shared thread, not a specialist swarm.

## Loop

1. **Lock decisions.** Number them (`D1`, `D2`, …). Workers do not re-litigate them.
2. **Write the goal long.** Short goals stall. Phase 0 is docs and ADRs. Later phases are code.
3. **Graph before a wave.** Each track names repo, host, branch, base SHA, inputs, overlaps, and gate class.
4. **Dispatch to a free worker.** Check tmux and existing sessions first. Skip a busy or logged-out host.
5. **Implement this phase.** Dedicated worktree. Prove the real caller path. Mutation-test new tests.
6. **Draft PR + COMMENT review.** Different family from the writer. Never Approve.
7. **Report `AWAITING GATE`** with branch, PR, head SHA, COMMENT URL, and risks.
8. **CoS gate.** Clobber-check and rebase. If the SHA moved, refresh the COMMENT. Confirm the COMMENT covers this head. Exact-head CI. Live-verify staging.
9. **REVISE or merge.** Serialize overlapping merges. Watch default-branch CI after merge.
10. **Human production.** Workers never dispatch production.

The CoS chat is authoritative. A watchdog may inspect, heartbeat, and take the next already-authorized repo-safe step. It may not invent gates, merge production, or retry an unresolved escalation forever.

## Three gate classes

Keep them separate. Evidence from a later class must not block an earlier class.

1. **Repository.** Lint, tests, typecheck, build, mutation proof, COMMENT review, CI green on **this** SHA.
2. **Staging / integration.** Merge order, staging deploy, live checks against the deployed revision.
3. **Production.** Secrets, paid resources, destructive migrations, traffic. Human only.

A repository PR is allowed to merge when its repository gate is green even if production resources do not exist yet. Production evidence is still mandatory before traffic.

See [ADR 0004](adr/0004-three-gate-classes.md).

## Parallel tracks

Parallelize only when files, interfaces, and merge slots do not overlap. Prefer spreading tracks across hosts over stacking many sessions on one box.

On one laptop, serial tracks in separate worktrees are still the right shape. The worktree is the isolation boundary; the second machine is optional.

See [ADR 0003](adr/0003-worktrees-and-parallel-tracks.md).

## Anti-churn

1. Spread work across the pool. Do not pile every track onto the strongest host.
2. Parallelize independent tracks only. Serialize merges.
3. The worker posts the different-family COMMENT. The CoS re-reviews high-risk diffs only (schema, auth, migration, security, data-write, or a thin pre-review).
4. Live-verify staging. Green CI is not enough.
5. Phase the worker with resume/continue. Do not pretend one shot is an immortal session.
6. One CoS per standing goal. Heartbeat every 10 minutes while any track is running.

## Human-stop list

Stop and ask a human before any of the following:

- Production deploy, traffic enablement, or rollback against production data
- Creating secrets, paid cloud resources, or new credentials
- DNS and other domain changes
- Destructive data changes or irreversible migrations
- Changing locked goal decisions (`D1`…)
- Merging without a COMMENT review on **this** head
- Approving a PR from the same session that wrote it
- Retrying the same blocker after three documented attempts
- Publishing personal names, home paths, IPs, or a real worker-host inventory

Workers run in-scope repository work without asking. Everything on this list is out of scope until a human unlocks it in the goal prompt.

## Public-tree rule

Anything published from this method — docs, PRs, commit messages, Pages artifacts — stays free of personal fleet inventory: no names, no IPs, no private host tables, no shop-internal URLs. Use placeholders (`dev-1`, `ci-box`, `user@host`) and documentation-range examples (`192.0.2.1`) when you need a table.
