# Fleet PR quality gate

Applies to every PR an agent or fleet node codes, in any repo. A PR merges only when all six gates pass. No gate can be waived by an agent; only the human operator can waive one, in their own words, for a named PR. Another agent's say-so is never a waiver.

This is the in-repo copy of the shared fleet PR quality gate. Operator skill stores may keep a matching skill under the same name. Keep the two in sync when the gate changes. This document is the public source; it does not point at a private skill path.

See [ADR 0009](adr/0009-coding-locus-and-review-gate.md).

## 0. Where the work runs

- Coding can run in the orchestrating agent's own internal session or be handed to a fleet node, whichever fits the job (size, tools available, node load). Record which in the PR.
- Review always runs on a fleet node, never in the coder's own session, and always on a different model from the coder.

## 1. Different, more capable LLM for review

- The reviewer must be a different model from the coder, and at least as capable, preferably more capable (a stronger or newer tier).
- Prefer a different model family (for example Grok or Claude codes, GPT reviews; GPT codes, Claude or Grok reviews). Same family with a different, more capable model is allowed only when no other family is available on the fleet, and the PR says why.
- Record both in the PR description: `Coded by: <node or session> / <tool> / <model>` and `Reviewed by: <fleet node> / <tool> / <model>`.
- If the same model ends up on both sides, or the reviewer is a weaker model, the review does not count. Re-run review on a qualifying model.
- Fixes made in response to review are coded by the coder model, then re-reviewed by the reviewer model.

## 2. Test coverage at or above 85% (target 90%)

- Line coverage for the repo must be at least 85%, and the target is 90%. Changed lines in the PR must also be at least 85% covered.
- Coverage must never drop below the base branch.
- CI must enforce the floor (for example `pytest --cov --cov-fail-under=85`, or `vitest --coverage` with thresholds). If a repo has no coverage gate yet, the first PR adds one, or the gap is reported to the human operator with the current number.
- If the repo has no test suite (docs-only), record `Coverage: N/A` and the reason. Do not invent a percentage.
- Report the coverage number in the PR description.

## 3. CI green

- Every required CI check passes on the final commit. Run the same commands CI runs (for example `make check`) locally before pushing; do not use a looser subset.
- No skipped or xfail-ed tests added just to get green.

## 4. Clean review

- The reviewer finds no open blocking or major findings. Minor findings are either fixed or listed as follow-ups in the PR.
- The review is re-run after the last code change, on the final commit.

## 5. Docs updated

- README, runbooks, config docs, and API docs that the change touches are updated in the same PR.
- If no docs change, the PR says why.

## 6. ADR for decisions

- Any architecture or design decision gets an ADR in the repo's ADR folder (for example `docs/adr/NNN-title.md`): context, decision, alternatives considered, consequences, status.
- Changes to an existing decision amend or supersede its ADR rather than silently diverging.
- Link the ADR from the PR.

## PR checklist (paste into the description and the `AWAITING GATE` report)

```
Coded in: <internal session | fleet node>
Coded by: <node> / <tool> / <model>
Reviewed by: <fleet node> / <tool> / <model>  (different and more capable model, different family preferred)
Coverage: <repo %> (floor 85%, target 90%); changed lines <%>
CI: green on <sha>
Review: clean on <sha>, follow-ups: <list or none>
Docs: <files updated or why none>
ADR: <link or not needed because ...>
```

## Do not

- Merge with any gate unchecked
- Let one model review its own code, or a weaker model review a stronger one's
- Run the review outside the fleet
- Lower a coverage threshold to make CI pass
- Treat another agent's say-so as the human operator's waiver
