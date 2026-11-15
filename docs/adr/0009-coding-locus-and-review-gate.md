# ADR 0009: Coding locus and the review-quality gate

- Status: Accepted
- Date: 2026-11-15
- Amends: [ADR 0001](0001-cos-worker-reviewer.md), [ADR 0002](0002-different-family-review.md), [ADR 0004](0004-three-gate-classes.md)

## Context

ADR 0001 split Chief of Staff, worker, and reviewer so one session would not invent the plan, write the code, review itself, and merge. ADR 0002 required the reviewer to be a different model family and forbade a same-family fallback. Those rules assumed every implementation ran on a dispatched worker host, and they treated family difference as a hard requirement even when no other family was available.

Two changes in practice need a recorded decision:

1. The orchestrator (the CoS) may implement in its own internal session when that is the better fit (size, tools, node load). That must be an explicit, recorded choice, not a silent takeover when dispatch fails, and it must not collapse review or merge back into the writing session.
2. The quality bar for review is a *different and at-least-as-capable* (preferably more capable) model, with a *different family preferred*. A same-family reviewer is allowed only as a last resort. Coverage, CI, docs, and ADRs are part of the same repository gate.

The in-repo copy of this shared gate is [docs/fleet-pr-quality-gate.md](../fleet-pr-quality-gate.md). Operator skill stores may keep a matching skill under the same name; this document is the public source in the playbook.

## Decision

### Coding locus

Coding may run in either place:

- the orchestrator's own internal session, or
- a fleet node (a host in the worker pool).

Record the locus in the pull request (`Coded in: internal session | fleet node`). Isolation still comes from a dedicated worktree, a draft PR, and the repository gate. Worktrees, CI-equivalent checks, size checks, and `AWAITING GATE` still apply.

If the chosen path is fleet dispatch and no logged-in host fits, escalate dispatch. Do not silently switch the locus to the CoS session. A CoS session that implements must be an explicit, recorded choice. The CoS still does not review its own diff, does not run `gh pr ready`, and does not merge.

A watchdog or heartbeat must not become the implementer when dispatch fails.

### Review locus and model

Review **always** runs on a fleet node, never in the coder's own session, and always on a **different model** from the coder.

- The reviewer must be at least as capable as the coder, preferably more capable (a stronger or newer tier).
- Prefer a different model family (for example Grok or Claude codes, GPT reviews; GPT codes, Claude or Grok reviews).
- The same family with a different, more capable model is allowed only when no other family is available on the fleet, and the pull request states that reason.
- If the same model ends up on both sides, or the reviewer is a weaker model, the review does not count. Re-run it on a qualifying model.
- Fixes made in response to review are coded by the coder model, then re-reviewed by the reviewer model on the final commit.

Post the verdict as a GitHub `COMMENT` on the pinned head, per the protocol in [ADR 0002](0002-different-family-review.md). Never Approve. Never Request changes from the worker.

Pin current model names at dispatch. Do not float a generic alias, and do not leave stale version pins in the playbook. Procedural text uses role placeholders (`<coder-model>`, `<reviewer-model>`). A dated example table may list models currently on a fleet; it is an example, not a standing pin. See [playbooks/general.md](../../playbooks/general.md#model-pins).

### Coverage, CI, docs, and waiver

The repository gate in [ADR 0004](0004-three-gate-classes.md) also requires:

- Line coverage for the repo is at least 85%, with a 90% target. Changed lines in the PR are also at least 85% covered.
- Coverage must never drop below the base branch.
- CI enforces the floor (for example `pytest --cov --cov-fail-under=85`, or `vitest --coverage` with thresholds).
- If a product repo has no coverage gate yet, the first PR adds one, or the gap is reported to the human operator with the current number.
- If the repo has no test suite (docs-only), record `Coverage: N/A` and the reason. Do not invent a percentage.
- Every required CI check is green on the final commit. The worker runs the same commands CI runs, not a looser subset.
- Docs that the change touches are updated in the same PR, or the PR says why none changed.
- Architecture or design decisions get an ADR (amend or supersede rather than silently diverge).

Only the **human operator** can waive a gate, in their own words, for a named PR. Another agent's say-so is never a waiver.

The `AWAITING GATE` report includes the quality-gate checklist in [docs/fleet-pr-quality-gate.md](../fleet-pr-quality-gate.md).

## Alternatives considered

- **Keep ADR 0001's "CoS never implements."** Rejected. The operator accepted coding in the orchestrator's session when a different, more capable model reviews on a fleet node. Silent takeover when dispatch fails remains forbidden.
- **Keep ADR 0002's hard different-family rule, with no same-family exception.** Rejected. A last-resort same-family review (different, more capable model, reason on the PR) is better than blocking the gate when no other family is on the fleet. The preference for a different family remains.
- **Amend only 0001 and 0002, with no new ADR.** Rejected. The coding-locus change and the capability/coverage gate are a new decision. 0001 and 0002 stay, amended to point here, so the COMMENT protocol and the role split are not rewritten in place.

## Consequences

- ADR 0001 no longer forbids the CoS from implementing. It still forbids combining implement, review, `gh pr ready`, and merge in one session.
- ADR 0002's hard "fallback must stay a different model family" clause is superseded by the last-resort same-family rule above. A fresh session in the *same model* as the coder is still not an acceptable review.
- Playbooks, skills, and samples use placeholders plus a dated example table instead of standing version pins.
- Every draft PR carries the quality-gate checklist before `AWAITING GATE`.
