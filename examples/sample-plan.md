# Plan — Harbor berth scheduling

_2026-08-14 · Feeds goal prompt · Self-contained for a fresh CoS session_

This is a **fictional** plan used to show the shape of a pre-dispatch artifact. Swap Harbor for your product. This plan is written **after the spec is locked** and feeds the goal prompt. A fresh CoS session should be able to write a goal from this plan alone.

## Phases

### Phase 0 — Docs and ADRs

Write architecture doc, stack ADR (framework choice, auth mechanism), tenancy ADR (single-org, no multi-tenant), and a local-dev README. No application code. Record the chosen stack (e.g., Next.js + API routes, or Express + React) and auth approach (e.g., session cookies, JWT, or basic auth for v1).

### Phase 1 — Slip and reservation API

Implement CRUD for slips and reservations. Data model: `Slip` (id, name), `Reservation` (id, slip_id, vessel, start_date, end_date). Conflict detection: reject `POST /reservations` if the requested `(slip_id, start_date, end_date)` overlaps an existing reservation. Overlap means any shared date (day-level granularity). Mutation-test the conflict rule: a test that passes when the conflict check is removed should fail.

API endpoints:
- `GET /slips` — list all slips
- `GET /slips/:id/availability?start=YYYY-MM-DD&end=YYYY-MM-DD` — free/busy
- `POST /reservations` — create (reject if conflict)
- `GET /reservations` — list (operator-only)
- `DELETE /reservations/:id` — cancel

Auth: operator endpoints require login per the Phase 0 ADR. Tests include a conflict-detection proof (overlap is rejected) and a mutation check (removing the conflict rule fails the test).

### Phase 2 — Operator UI

List slips, show which are free on a date range, create a reservation with conflict warning, cancel a reservation. Exercise the **real API** from Phase 1 (not a fixture-only page). The conflict warning appears before the operator commits the reservation. Use the `/slips/:id/availability` endpoint to check before `POST /reservations`.

Screens (low-fidelity examples from the spec):
- Slip list with availability status
- Create reservation form (slip dropdown, vessel name, start/end dates, "Check availability" button, "Create" button)
- Conflict warning if overlap detected
- Reservation list with cancel action

Tests include: create a reservation, verify it appears in the list, attempt a conflicting reservation, verify the conflict warning, cancel a reservation, verify the slip is available again.

### Phase 3 — Public arrivals board

A public (no-auth) page showing today's reservations: slip, vessel, arrival date. Query reservations where `start_date <= today` and `end_date >= today`. No historical view, no future view beyond today. The page is a read-only `GET /public/arrivals` endpoint or a server-rendered page.

Tests include: create a reservation with today's date, verify it appears on the public board, verify a reservation starting tomorrow does not appear.

### Phase 4 — Staging (CoS-only)

Workers stop at `AWAITING GATE`. Do not treat this as the next incomplete phase. After Phases 1–3 merge, the CoS serializes merge order, deploys staging, and live-verifies: health check, create a reservation via the UI, verify conflict detection (attempt an overlapping reservation), cancel the reservation, verify the public board shows today's reservations. Production remains human-gated.

## Risks

- **Flaky CI:** If the product repo CI is flaky, workers will block at the DoD check. Mitigation: verify CI health on the default branch before dispatch. If CI is red or missing, record `ci: none` in the status file and rely on local `make check`.
- **Auth mechanism unclear:** Phase 0 must lock the auth approach (session, JWT, basic). If the ADR leaves it ambiguous, Phase 1 will stall on "how do operators log in?" Mitigation: the Phase 0 ADR must include a concrete auth example (e.g., "basic auth with hardcoded credentials for v1 staging").
- **Conflict-detection edge case:** Day-level overlap is simple, but "does August 13–15 overlap August 15–17?" needs a clear rule. Mitigation: define overlap as "any shared date" (August 15 is shared, so it overlaps). Test the boundary in Phase 1.
- **Public board query performance:** If there are 1000 slips and 5000 reservations, filtering "today's reservations" may be slow. Mitigation: v1 staging is low-volume (10 slips, <50 reservations). Defer optimization. Record the query shape in Phase 3 so a later phase can add an index if needed.
- **Worker pool logged out:** If `dev-1` or `dev-2` is logged out (`grok` or `gh`), dispatch will fail. Mitigation: CoS re-picks a logged-in host and updates the graph. Login is a per-host input.

## Dependencies

- **Repo access:** The `example-app` repo must be created (or use an existing empty repo). The CoS and workers need push access.
- **Worker pool:** At least one logged-in host (`dev-1` or local). `grok` CLI and `gh` CLI authenticated.
- **CI or equivalent:** The product repo must have a `make check` target (or equivalent commands that workers run before push). If there is no CI, workers will record `ci: none` and rely on local checks.
- **Staging environment:** A way to deploy the merged code (e.g., a VPS, a PaaS, or a local container). The CoS will deploy after merge. Production is a later human gate.
- **Secrets:** If auth requires a secret (e.g., a JWT signing key), it must be created before Phase 1. If the secret is deferred, record it as a blocker and escalate.

## Definition-of-done hints (per phase)

These are hints for the CoS to refine into exact commands in the goal prompt. Workers will run these checks before `AWAITING GATE`.

- **Phase 0:** `make check` (or the exact commands from `example-app` CI) exit 0. ADRs are in-tree (stack, tenancy, local-dev README).
- **Phase 1:** `make check` exit 0. Conflict-detection test passes. Mutation test fails when the conflict rule is removed (proves the test is real). All API endpoints are exercised in tests.
- **Phase 2:** `make check` exit 0. UI exercises the **real API** from Phase 1 (not a fixture-only page). Conflict warning appears when an overlapping reservation is attempted.
- **Phase 3:** `make check` exit 0. Public board shows today's reservations. A reservation starting tomorrow does not appear.
- **Phase 4:** CoS live-verified staging (not a worker check). Health check returns 200. Create a reservation via the UI, verify the conflict warning, cancel the reservation, verify the public board updates.

---

**Next step:** The CoS writes a goal prompt per [`skills/goal-prompt/SKILL.md`](../skills/goal-prompt/SKILL.md). The goal prompt includes:

- Header with owner, CoS identity, worker pool
- Standing instruction (read, find next incomplete worker phase, execute, verify, draft PR, COMMENT, AWAITING GATE)
- Goal (Harbor berth scheduling)
- Decisions already made (D1–D10 from the spec)
- Context (verified against default branch)
- Phases (copied from this plan, expanded with exact DoD commands)
- Worker pool table (hosts, fit, max parallel tracks)
- Autonomy (in-scope work, human-stop list)
- Model policy (writer CLI, reviewer family)
- Workflow (worktree, CI-equivalent check, `scripts/pr-size-check`, draft PR, COMMENT, AWAITING GATE)
- Verification (per phase)
- Definition of done (exact commands from this plan's hints)
- Progress log (Phase | Status | PRs / ADRs | Notes)

After the goal is written, the CoS draws the dependency graph (Phases 1–2 can run in parallel if files do not overlap; Phase 3 depends on Phase 1 merge), dispatches workers, implements, reviews, gates, and the human merges. Staging is CoS-only. Production is human-gated.
