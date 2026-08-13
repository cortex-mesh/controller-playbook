# ADR 0001: CoS, worker, and reviewer are separate roles

- Status: Accepted
- Date: 2026-08-13

## Context

One long-lived CLI session is asked to invent the plan, write the code, review itself, merge, and deploy. That collapses authority. Session death loses the plan. Self-review hides bugs the writer is invested in. Merge order becomes whoever finished typing.

## Decision

Split three roles:

1. **Chief of Staff** — owns the goal, the dependency graph, dispatch, the gate, merge order, and staging live-verify.
2. **Worker** — implements one phase in one worktree and stops at `AWAITING GATE`.
3. **Reviewer** — a different session (and a different model family; see [ADR 0002](0002-different-family-review.md)) that comments on the pull request.

These are steps, not extra bot identities. One CoS per standing product goal.

## Consequences

- Implementation does not run inside the CoS chat as a silent takeover.
- Workers never self-merge and never ship production.
- A crashed worker session can be resumed against the same goal prompt.
- The CoS can run many workers without becoming the author of their diffs.
