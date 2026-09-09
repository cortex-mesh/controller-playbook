# Spec — Harbor berth scheduling

_2026-08-14 · Product owner: human operator · Status: locked after PO sign-off · Spend cap: $200 compute_

This is a **fictional** spec used to show the shape of a pre-dispatch artifact. Swap Harbor for your product.

## In scope

- **Slip inventory:** A fixed list of slips (A1, A2, B1, …). Adding/removing slips is an operator action via API or direct data edit (no admin UI in v1).
- **Reservation CRUD:** Operators can create, view, and cancel reservations. Editing a reservation (changing the slip or dates) is out of scope; cancel and re-create.
- **Conflict detection:** The system rejects a reservation if it overlaps an existing reservation for the same slip. "Overlap" means any shared date (day-level granularity; hours are out of scope).
- **Operator UI:** List slips, show which are free on a date range, create a reservation, see conflicts before committing, cancel a reservation.
- **Public arrivals board:** A read-only page showing today's reservations (slip, vessel, arrival date). No login required. No historical view (only today).
- **Staging deploy:** One staging environment. Operators can test the full flow (create, conflict, cancel, public board). Production is a later human gate.

## Out of scope (locked descopes)

- Multi-tenant: Harbor is one operator org. Other orgs are deferred.
- Billing / payments: reservations are free. Paid slips are a later product decision.
- Mobile app: the arrivals board is web-only. Native mobile is deferred.
- Real-time sync: operators refresh the page. WebSockets are deferred.
- Reservation editing: cancel and re-create instead. Editing the slip or dates in place is out of scope.
- Hour-level granularity: reservations are day-level ("August 13 all day"). Hours are deferred.
- Historical analytics: "which slips are busiest?" is deferred. v1 stores reservations; reports are later.
- Suggested alternatives: if a conflict is detected, the system rejects and the operator picks a different slip. Auto-suggest is deferred.
- Admin UI for slip inventory: operators edit slips via API or direct data. A UI for adding/removing slips is deferred.

## UX or API surface

### Operator UI (examples, not final design)

**List slips:**

```
Slips
─────────────────────────────────
Slip    Status on 2026-08-13
A1      Available
A2      Reserved (Vessel: Osprey)
B1      Available
```

**Create reservation:**

```
Create Reservation
─────────────────────────────────
Slip:      [A1 ▼]
Vessel:    [Osprey]
Start:     [2026-08-13]
End:       [2026-08-15]

[Check availability]  [Create]
```

If a conflict is detected:

```
⚠️  Conflict: Slip A1 is already reserved 2026-08-13 to 2026-08-14 (Vessel: Merlin).
Please choose a different slip or date.
```

**Public arrivals board:**

```
Harbor Arrivals — August 13, 2026
─────────────────────────────────
Slip    Vessel       Arrival
A2      Osprey       2026-08-13
C1      Falcon       2026-08-13
```

### API endpoints (example shape)

- `GET /slips` — list all slips
- `GET /slips/:id/availability?start=YYYY-MM-DD&end=YYYY-MM-DD` — check if a slip is free
- `POST /reservations` — create a reservation (rejects if conflict)
- `GET /reservations` — list reservations (operator-only)
- `DELETE /reservations/:id` — cancel a reservation
- `GET /public/arrivals` — today's reservations (public, no auth)

Auth: operator UI and reservation endpoints require login (mechanism TBD in Phase 0 ADR). Public board is unauthenticated.

## Acceptance criteria

1. Operator can list slips and see which are free on a given date range.
2. Operator can create a reservation. If the reservation overlaps an existing reservation for the same slip, the system rejects it with a conflict message.
3. Operator can cancel a reservation. The slip becomes available again.
4. Public arrivals board shows today's reservations (slip, vessel, arrival date). No login required.
5. A conflict is detected **before** the reservation is committed (not after).
6. Two operators cannot create overlapping reservations for the same slip (even if they both check availability before the other commits).
7. Staging deploy is live. Operators can exercise the full flow end-to-end.

## Locked decisions

- **D1.** Product name: Harbor (fictional). Repo: `example-app`.
- **D2.** Stack: TypeScript web app + HTTP API. Exact framework (Next.js, Express, etc.) is an accepted ADR in Phase 0.
- **D3.** Tenancy: single operator org in v1. No multi-tenant.
- **D4.** Reservation granularity: day-level (not hour-level).
- **D5.** Reservation editing: cancel and re-create. No in-place edit.
- **D6.** Conflict detection: reject with a message. No auto-suggest alternative slips.
- **D7.** Public board: today only (no historical view, no future view beyond today).
- **D8.** Spend cap: $200 compute for staging. Production is a later human gate.
- **D9.** Review: different-family COMMENT. Never Approve from an agent.
- **D10.** Public docs use placeholders only (`dev-1`, `user@host`). No personal names, IPs, or fleet inventory.

---

**Next step:** write `plan.md` with phases, risks, dependencies, and per-phase DoD hints. After the plan is locked, the CoS writes a goal prompt.
