# ADR 0002: Review is a different model family, posted as COMMENT

- Status: Accepted
- Date: 2026-08-13

## Context

Same-model self-review is the failure mode this split exists to prevent. A writer asked to "also review" will defend the design it just produced. GitHub Approve from an agent also collapses the human merge decision.

## Decision

- The reviewer is a **different model family** from the implementer.
- Default Grok-implement path: `gpt-5.6-sol` / high via `codex review`.
- Default Claude-controller path: Opus.
- Default Codex-controller path: `gpt-5.6-sol` / high, in a fresh session, not the implementing session.
- Post the verdict with `gh pr review --comment --body-file <verdict.md>`. Never interpolate model output into a shell `--body`. Never Approve. Never Request changes from the worker.
- Review the **current head**. A verdict on a previous SHA does not gate a new SHA.
- High-risk diffs (schema, auth, migration, security, data-write) get a fresh CoS-run review, not a reread of the worker's summary.

## Consequences

- Every draft PR carries an independent COMMENT before `AWAITING GATE`.
- The CoS reads the verdict; it does not grep for "no blocking."
- Substituting another reviewer CLI is allowed if it is not the implementing session.
- Agent-side review is the default; the CoS does not re-review every low-risk docs or UI PR.
