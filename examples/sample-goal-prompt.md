# Goal Prompt — Harbor

_2026-08-13 · Owner: human operator · CoS: **Harbor CoS** (Grok Bot) ·
Workers: Grok CLI grok-4.6 on the **worker pool** (CoS assigns host at dispatch)_

Standing instruction: read this file, find the next incomplete phase, execute
it end-to-end, do not skip verification. Report AWAITING GATE with branch, PR,
head SHA, and COMMENT review URL.

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
- D4. Review: `gpt-5.6-sol` / high COMMENT. Never Approve from an agent.
- D5. Production, paid DNS, and real customer data are out of scope for workers.
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
mutation check on the conflict rule. Draft PR, COMMENT review, `AWAITING GATE`.

### Phase 2 — Operator UI

List slips, create a reservation, show conflicts. Exercise the real API, not a
fixture-only page.

### Phase 3 — Staging

Merge order: API before UI if they land separately. Deploy staging. Live-verify
health and one reservation round-trip. Production remains human.

## Worker pool

| Host | SSH | Fit | Max parallel tracks |
| --- | --- | --- | --- |
| `dev-1` | `user@host` | General coding | 1 |
| `dev-2` | `user@host` | UI track when files do not overlap Phase 1 | 1 |

CoS assigns the host at dispatch. A laptop-only pool may drop `dev-2`.

## Autonomy

`--always-approve --permission-mode bypassPermissions` for in-scope writes in
the Harbor worktree. Human-only: production, secrets, DNS, destructive data,
spend.

## Model policy

- `grok-4.6` writes code and ADRs.
- `gpt-5.6-sol` / high reviews via `codex review`.
- `gh pr review --comment` on this head. Never Approve.
- Never review the PR in the same Grok session that wrote it.

## Workflow (per PR)

Dedicated git worktree. Conventional commits. Lint, test, build. Mutation-test
new tests. Draft PR. Reviewer COMMENT. `AWAITING GATE`. CoS serializes merge.

## Verification (every phase)

- Repo gates green on this SHA.
- COMMENT review URL posted.
- After staging exists: live health check and one real caller path.

## Definition of done (per phase)

- Phase 0: ADRs merged or gated, architecture in-tree.
- Phase 1: API PR gated, conflict rule mutation-tested.
- Phase 2: UI PR gated, real API exercised.
- Phase 3: staging live-verified. No production.

## Progress log

| Phase | Status | PRs / ADRs | Notes |
| --- | --- | --- | --- |
| 0 | not started | | docs |
| 1 | not started | | api |
| 2 | not started | | ui |
| 3 | not started | | staging |
