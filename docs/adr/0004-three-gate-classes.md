# ADR 0004: Repository, staging, and production gates stay separate

- Status: Accepted (amended 2026-11-15 by [ADR 0009](0009-coding-locus-and-review-gate.md))
- Date: 2026-08-13

## Context

Teams collapse "tests passed," "it is on staging," and "customers can use it" into one word: done. Then a repository PR waits on production DNS, or a production push rides a green unit test with no live check.

## Decision

Three gate classes, kept separate:

1. **Repository** — the product repo's CI-equivalent check (`make check`, or the exact commands from that repo's CI; not a looser local subset), mutation proof, COMMENT review after current-head CI is green, CI on this SHA. Line coverage is at least 85% (target 90%); changed lines are at least 85% covered; coverage must not drop below the base branch; CI enforces the floor. Docs-only repos with no test suite record `Coverage: N/A` and the reason. A PR with ≥800 added product lines (excluding lockfiles, generated output, snapshots, and `vendor/`) cannot complete this gate. File count alone does not fail this gate. Only the human operator can waive a gate, in their own words, for a named PR. See [ADR 0009](0009-coding-locus-and-review-gate.md) and [docs/fleet-pr-quality-gate.md](../fleet-pr-quality-gate.md).
2. **Staging / integration** — merge order, staging deploy, live verification of the deployed revision.
3. **Production** — secrets, paid resources, destructive migrations, traffic. Human only.

Production-only evidence must not block a repository PR merely because production does not exist yet. Production evidence remains mandatory before traffic.

Workers may complete a repository gate and then stop at `AWAITING GATE`. A PR over the product-line hard cap reports `AWAITING SPLIT` instead: the CoS updates the living graph and re-dispatches. There is no "justify the megadiff." `AWAITING SPLIT` is a stop state, not a fourth gate class. Staging / last integration is CoS-only. Goal prompts and launch skills must not dispatch staging as the next incomplete phase. Humans own production.

## Consequences

- A draft PR reports repository evidence, not a claim that production is live. Merge does not deploy. Staging is CoS-only after default-branch CI. Production is human.
- The product repo owns the check command. GitHub Actions (or the product CI) must call that same target, including the coverage floor when the repo has tests. Workers run it in the assigned worktree and push only if it is green. Git hooks are optional extra, not the control. If the repo has no `make check`, or CI does not run on this SHA, say so in `AWAITING GATE` instead of inventing a subset. If there is no coverage gate yet, the first PR adds one, or the gap is reported to the human operator with the current number. A worker that runs only `make lint`, a formatter, or one test file has not satisfied the repository gate.
- Auto-deploy from default-branch CI is a staging concern. A red default-branch run can skip deploy with no PR failure; watch it.
- Goal prompts list which actions the CoS may take without another human decision.
- Sample goals and launch skills skip staging when naming the next incomplete worker phase.
- Before GATE, the CoS confirms one outcome, then size-checks (`gh pr view --json additions` and/or `git diff --numstat "$DEFAULT"...HEAD` after the product-file exclusions). REVISE-to-split if there is a second outcome or the product-line count is ≥800. Do not COMMENT-review a +11k PR hoping the reviewer will save it. File count alone does not fail GATE. Size is the backstop; grouping is the method ([ADR 0003](0003-worktrees-and-parallel-tracks.md)).
