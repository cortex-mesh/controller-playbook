# ADR 0001: CoS, worker, and reviewer are separate roles

- Status: Accepted (amended 2026-11-15 by [ADR 0009](0009-coding-locus-and-review-gate.md))
- Date: 2026-08-13

## Context

One long-lived CLI session is asked to invent the plan, write the code, review itself, merge, and deploy. That collapses authority. Session death loses the plan. Self-review hides bugs the writer is invested in. Merge order becomes whoever finished typing.

## Decision

Split three roles:

1. **Chief of Staff** — owns the goal, the dependency graph, dispatch, the gate, merge order, and staging live-verify. May implement only when that locus is an explicit, recorded choice (see [ADR 0009](0009-coding-locus-and-review-gate.md)).
2. **Worker** — implements one phase in one worktree and stops at `AWAITING GATE`. The worker may be a fleet-node CLI or the orchestrator's internal session.
3. **Reviewer** — a different session on a fleet node, using a different and at-least-as-capable model (different family preferred; see [ADR 0002](0002-different-family-review.md) and [ADR 0009](0009-coding-locus-and-review-gate.md)) that comments on the pull request.

These are steps, not extra bot identities. One CoS per standing product goal.

## Consequences

- Implementation does not run inside the CoS chat as a silent takeover. When the CoS implements, that locus is recorded on the PR, and review plus merge still stay outside that session.
- If the chosen path is fleet dispatch and dispatch fails, escalate dispatch. Do not silently switch the locus to the CoS session.
- Workers never self-merge, never `gh pr ready`, and never ship production. A CoS session that implements follows the same rule.
- A crashed worker session can be resumed against the same goal prompt.
- The CoS can run many workers without becoming the author of their diffs. When it does author a diff, a fleet reviewer still comments on that head.
