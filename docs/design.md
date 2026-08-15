# Design

How a Chief of Staff, a worker pool, and a different-family reviewer ship a product without turning one chat into the system of record.

## Roles

| Role | Owns | Does not own |
| --- | --- | --- |
| **Human operator** | Goal, spend, secrets, DNS, production, unresolved product choices | Day-to-day implementation once decisions are locked |
| **Chief of Staff (CoS)** | Goal prompt, dependency graph, dispatch, gates, merge order, staging live-verify, heartbeat | Writing the feature in the same turn that gates it |
| **Worker** | One track, one worktree, one phase: implement, prove, draft PR, request review | Self-merge, `gh pr ready`, production, inventing new gates, staging dispatch |
| **Reviewer** | A fresh session in a **different model family** than the writer. Posts a GitHub `COMMENT`. A fresh Grok session is not an acceptable reviewer of a Grok implementation. | Approve, Request changes, or rubber-stamping the implementer |

Do not mint extra bots named dispatcher, coder, reviewer, or merger. Those are steps. One CoS per standing product goal. If two controllers must share a goal, use a shared thread, not a specialist swarm.

## Loop

1. **Lock decisions.** Number them (`D1`, `D2`, …). Workers do not re-litigate them.
2. **Write the goal long.** Short goals stall. Phase 0 is docs and ADRs. Later phases are code.
3. **Graph before a wave.** Each track names repo, assigned host (after failover), tmux or gone, log path, branch, base SHA, draft PR, head SHA, COMMENT URL, inputs, overlaps, gate class, CoS-may-do, and login/preflight.
4. **Dispatch to a free logged-in worker.** Check tmux and existing sessions first. Login is a per-host input. A logged-out host does not block the wave: re-pick and write the new owner into the graph. If dispatch fails, escalate; the CoS does not become the implementer.
5. **Implement this phase.** Dedicated worktree. Prove the real caller path. Run the product repo CI-equivalent check; push only if green. Mutation-test new tests. Skip staging — CoS-only.
6. **Draft PR, wait for this-SHA CI, then COMMENT review.** Different family from the writer. Never Approve. Never a fresh Grok session reviewing a Grok implementation.
7. **Write the status file.** Run `scripts/gate-preflight`. Report `AWAITING GATE` with branch, PR, head SHA, COMMENT URL, and risks. Workers never `gh pr ready`. Phase DoD is the product check commands; `AWAITING GATE` is illegal until they pass.
8. **CoS repo gate.** Read status + GATE log + graph. Run `scripts/gate-preflight`. Clobber-check and rebase. If the SHA moved, refresh the COMMENT. Confirm the COMMENT covers this head. Exact-head CI.
9. **REVISE or GATE.** After repo-gate evidence is green, mark the draft ready and emit `WAITING ON YOU: merge PR #N` (or accept residual / unlock REVISE). Do not merge while waiting. Watch default-branch CI after the human merges. Same blocker: evidence, one REVISE, then stop and escalate.
10. **Staging gate (CoS-only).** Deploy the merged SHA and live-verify. Green pre-merge CI is not a staging check.
11. **Human production.** Workers never dispatch production.

The CoS chat is authoritative. A watchdog may inspect, heartbeat, and take the next already-authorized repo-safe step. It may not invent gates, merge production, or retry an unresolved escalation forever.

## Three gate classes

Keep them separate. Evidence from a later class must not block an earlier class.

1. **Repository.** Product repo CI-equivalent check, mutation proof, COMMENT review after current-head CI is green, CI green on **this** SHA. See [playbooks/general.md](../playbooks/general.md#cicd).
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
6. One CoS per standing goal. Heartbeat every 10 minutes while implementing or in CI. Every CoS-visible message starts with WORKING, WAITING ON YOU, or BLOCKED. `AWAITING GATE` in the graph is `WAITING ON YOU: merge PR #N` (or accept residual / unlock REVISE), not one notice then quiet.

## Human-stop list

Stop and ask a human before any of the following:

- Production deploy, traffic enablement, or rollback against production data
- Creating secrets, paid cloud resources, or new credentials
- DNS and other domain changes. DNS is the registrar or dashboard. An in-repo `CNAME` file is not DNS. `wrangler dns` does not exist.
- Destructive data changes or irreversible migrations
- Changing locked goal decisions (`D1`…)
- Merging without a COMMENT review on **this** head
- Approving a PR from the same session that wrote it
- Retrying the same blocker after three documented attempts
- Publishing personal names, home paths, IPs, or a real worker-host inventory

Workers run in-scope repository work without asking. Everything on this list is out of scope until a human unlocks it in the goal prompt.

## Public-tree rule

Anything published from this method — docs, PRs, commit messages, Pages artifacts — stays free of personal fleet inventory: no names, no IPs, no private host tables, no shop-internal URLs. Use placeholders (`dev-1`, `ci-box`, `user@host`) and documentation-range examples (`192.0.2.1`) when you need a table.
