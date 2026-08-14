# Goal Prompt — Harbor

_2026-08-13 · Owner: human operator · CoS: **Harbor CoS** (Grok Bot) ·
Workers: Grok CLI grok-4.6 on the **worker pool** (CoS assigns host at dispatch)_

Standing instruction: read this file, find the next incomplete **worker**
phase (skip staging / last integration — CoS-only), execute it end-to-end,
do not skip verification. Report AWAITING GATE with branch, PR, head SHA,
and COMMENT review URL. Never `gh pr ready`.

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
mutation check on the conflict rule. Draft PR, then COMMENT review,
`AWAITING GATE`.

### Phase 2 — Operator UI

List slips, create a reservation, show conflicts. Exercise the real API, not a
fixture-only page. Draft PR, then COMMENT, `AWAITING GATE`.

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

Dedicated git worktree. Conventional commits. Lint, test, build. Mutation-test
new tests. Draft PR. Reviewer COMMENT on the pinned head. `AWAITING GATE`.
Workers never `gh pr ready`. CoS marks ready after GATE and serializes merge.
Staging is CoS-only.

## Verification (every worker phase)

- Repo gates green on this SHA.
- COMMENT review URL posted on this head.
- Staging is not a worker check. After the CoS deploys: live health check and
  one real caller path.

## Definition of done (per phase)

- Phase 0: ADRs merged or gated, architecture in-tree.
- Phase 1: API PR gated, conflict rule mutation-tested.
- Phase 2: UI PR gated, real API exercised.
- Phase 3: CoS live-verified staging. Workers do not execute this phase. No
  production.

## Progress log

| Phase | Status | PRs / ADRs | Notes |
| --- | --- | --- | --- |
| 0 | not started | | docs |
| 1 | not started | | api |
| 2 | not started | | ui |
| 3 | CoS-only | | staging; do not dispatch |
