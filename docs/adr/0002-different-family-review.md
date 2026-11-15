# ADR 0002: Review is a different, more capable model, posted as COMMENT

- Status: Accepted (amended 2026-11-15 by [ADR 0009](0009-coding-locus-and-review-gate.md))
- Date: 2026-08-13

## Context

Same-model self-review is the failure mode this split exists to prevent. A writer asked to "also review" will defend the design it just produced. GitHub Approve from an agent also collapses the human merge decision.

The original decision required a different model family in every case and named standing version pins. [ADR 0009](0009-coding-locus-and-review-gate.md) keeps the COMMENT protocol and the ban on same-model review, and amends the family rule and the pins.

## Decision

- Review always runs on a fleet node, never in the coder's session (see [ADR 0009](0009-coding-locus-and-review-gate.md)).
- The reviewer is a **different model** from the implementer, and at least as capable, preferably more capable.
- Prefer a **different model family**. Same family with a different, more capable model is allowed only when no other family is available on the fleet, and the pull request states that reason.
- A fresh session of the *same model* that wrote the change is not an acceptable reviewer.
- Procedural text uses role placeholders (`<coder-model>`, `<reviewer-model>`). Pin the current names at dispatch. A dated example table may list models currently on a fleet; it is not a standing pin. See [playbooks/general.md](../../playbooks/general.md#model-pins).
- Post the verdict with `gh pr review --comment --body-file <verdict.md>`. Tee the file outside the repo. Check exit status. Pin `commit_id` to the exact head and abort if the live head moved. Never interpolate model output into a shell `--body`. Never Approve. Never Request changes from the worker.
- Review the **current head**. A verdict on a previous SHA does not gate a new SHA.
- High-risk diffs (schema, auth, migration, security, data-write) get a fresh CoS-run review, not a reread of the worker's summary. That review still runs on a fleet node with a qualifying model.

## Consequences

- Every draft PR carries an independent COMMENT before `AWAITING GATE`.
- The CoS reads the verdict; it does not grep for "no blocking."
- Substituting another reviewer CLI is allowed only if the model is different from the coder, at least as capable, and preferably a different family. A same-family fallback must be justified on the PR.
- Agent-side review is the default; the CoS does not re-review every low-risk docs or UI PR.
- The hard "fallback must stay a different model family" rule from the 2026-08-13 text is superseded by [ADR 0009](0009-coding-locus-and-review-gate.md).
