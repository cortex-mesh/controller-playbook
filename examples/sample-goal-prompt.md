# Goal Prompt — Harbor

_2026-08-13 · Owner: human operator · CoS: **Harbor CoS** (Grok Bot) ·
Workers: Grok CLI grok-4.6 on the **worker pool** (CoS assigns host at dispatch)_

Standing instruction: read this file, find the next incomplete **worker**
phase (skip staging / last integration — CoS-only), execute it end-to-end,
do not skip verification. In the assigned worktree, run the product repo
CI-equivalent check (`make check`, or the exact commands from that repo
CI). Push only if it is green. Fail = do not push, do not open the draft.
Run `scripts/pr-size-check` before opening a draft PR. Over cap →
`AWAITING SPLIT` (do not open a megadiff). Under cap: draft PR, wait until
CI is green on this SHA, then COMMENT. Phase DoD is those check commands.
`AWAITING GATE` is illegal until they exit 0. Write the worker status
file. Run `scripts/gate-preflight`. Report AWAITING GATE with branch,
PR, head SHA, and COMMENT review URL. Never `gh pr ready`. If the product
repo has no `make check`, or CI does not run on this SHA, say so in that report.

This is a **fictional** product used to show the shape of a goal. Swap Harbor
for your app. Replace `dev-1` / `dev-2` with your hosts. Do not copy a private
inventory into the public tree.

## Goal

Ship Harbor, a small berth-scheduling app: operators reserve a slip, see
conflicts, and publish a public "arrivals" board. v1 is one web app, one API,
one staging deploy. Production is a later human gate.

## Decisions already made (human, 2026-08-13)

- D1. Product name: Harbor (fictional). Repo: `example-app`.
- D2. Stack: TypeScript web app + HTTP API. Exact framework is an accepted ADR in Phase 0.
- D3. Tenancy: single operator org in v1. No marketplace, no multi-vendor billing.
- D4. Review: `gpt-5.6-sol` / high COMMENT. Never Approve from an agent. A fresh Grok session is not an acceptable reviewer.
- D5. Production, paid DNS, and real customer data are out of scope for workers. DNS is the registrar or dashboard. An in-repo `CNAME` file is not DNS. `wrangler dns` does not exist.
- D6. Public docs use placeholders only (`dev-1`, `user@host`).

## Context (verified 2026-08-13 — re-validate against current default branch)

- Empty product repo except this playbook's examples.
- No product ADRs yet. Phase 0 writes them.
- Staging URL will be recorded after the first deploy; do not invent one.

## Phases

### Phase 0 — Docs and ADRs

Write architecture, stack ADR, tenancy ADR, and a local-dev README. No
application code.

### Phase 1 — Berth API

CRUD for slips and reservations. Conflict detection. Tests including a
mutation check on the conflict rule. Run the product repo CI-equivalent
check; push only if green. Run `scripts/pr-size-check`. Over
cap → `AWAITING SPLIT`. Else draft PR, wait for this-SHA CI, then COMMENT
review, `AWAITING GATE`.

### Phase 2 — Operator UI

List slips, create a reservation, show conflicts. Exercise the real API, not a
fixture-only page. Run the product repo CI-equivalent check; push only if
green. Run `scripts/pr-size-check`. Over cap → `AWAITING SPLIT`.
Else draft PR, wait for this-SHA CI, then COMMENT, `AWAITING GATE`.

### Phase 3 — Staging (CoS-only)

Workers stop at `AWAITING GATE`. Do not treat this as the next incomplete
phase and do not dispatch it to a worker. After both repository PRs merge, the
CoS serializes merge order, deploys staging, and live-verifies health plus one
reservation round-trip. Production remains human.

## Worker pool

| Host | SSH | Fit | Max parallel tracks |
| --- | --- | --- | --- |
| `dev-1` | `user@host` | General coding | 1 |
| `dev-2` | `user@host` | UI track when files do not overlap Phase 1 | 1 |

CoS assigns the host at dispatch. Login (`grok` models + `gh`) is a dispatch
input. A logged-out host does not block the wave: re-pick the least-loaded
logged-in host and write the new owner into the graph. A laptop-only pool may
drop `dev-2`.

## Autonomy

`--always-approve --permission-mode bypassPermissions` for in-scope writes in
the Harbor worktree. Human-only: production, secrets, DNS, destructive data,
spend.

## Model policy

- `grok-4.6` writes code and ADRs.
- `gpt-5.6-sol` / high reviews via `codex review`.
- `gh pr review --comment` on this head. Never Approve.
- Never review the PR in the same Grok session that wrote it. A fresh Grok
  session is not an acceptable reviewer.

## Workflow (per PR)

Dedicated git worktree. Conventional commits. Product repo CI-equivalent
check in the worktree (`make check`, or the exact commands from that repo
CI — not a looser local subset). Push only if green. Mutation-test
new tests. Run `scripts/pr-size-check` before the draft. Over cap →
`AWAITING SPLIT`, do not open a megadiff. File count is a smell, not a fail.
Else draft PR. Reviewer COMMENT on the pinned head after this-SHA CI is
green. `AWAITING GATE`. Workers never `gh pr ready`. CoS marks ready and
emits WAITING ON YOU: merge. Staging is CoS-only.

## Verification (every worker phase)

- `scripts/pr-size-check` exit 0 on this head (over cap is `AWAITING SPLIT`, not GATE).
- Product repo CI-equivalent check green in the worktree before push.
- Repo gates / current-head CI green on this SHA.
- COMMENT review URL posted on this head.
- Staging is not a worker check. After the CoS deploys: live health check and
  one real caller path.

## Definition of done (per phase)

Each phase lists the exact commands. `AWAITING GATE` is illegal until
they exit 0. If Harbor has no `make check` and no current-head CI, record
`ci: none` in the status file.

- Phase 0: `make check` (or the exact commands from example-app CI) exit 0;
  ADRs in-tree.
- Phase 1: `make check` exit 0; conflict-rule mutation test fails without
  the fix.
- Phase 2: `make check` exit 0; real API exercised, not a fixture-only page.
- Phase 3: CoS live-verified staging. Workers do not execute this phase. No
  production.

## Progress log

| Phase | Status | PRs / ADRs | Notes |
| --- | --- | --- | --- |
| 0 | not started | | docs |
| 1 | not started | | api |
| 2 | not started | | ui |
| 3 | CoS-only | | staging; do not dispatch |
