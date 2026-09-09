# Intent — Harbor berth scheduling

_2026-08-13 · Originator: operator team · Status: originator review pending_

This is a **fictional** intent used to show the shape of a pre-dispatch artifact. Swap Harbor for your product. Replace placeholder dates and teams with your context.

## Problem

Berth operators schedule slip reservations using a shared spreadsheet. Two operators can assign the same slip to different vessels at overlapping times. Conflicts are discovered only when a vessel arrives and finds the slip occupied. Manual conflict resolution is expensive (move the vessel, update the board, notify the captain) and damages operator reputation.

Operators cannot see which slips are free, which are reserved, or which reservations conflict. The spreadsheet does not enforce constraints. Operators refresh the sheet manually and hope nobody else edited it between their read and their write.

## Why now

The Harbor fleet is adding ten slips next month. The spreadsheet conflict rate will grow non-linearly. One conflict costs ~$500 in labor and goodwill. At current volume (5 conflicts/month), that is $2500/month lost. Adding slips without conflict detection will push the rate to ~15 conflicts/month = $7500/month.

The operator team asked for this in June. They raised it again last week when a high-value client threatened to move berths. Fixing it now saves $60k/year and keeps the client.

## Non-goals

- Multi-tenant / marketplace: Harbor is one operator org in v1. Other orgs are out of scope.
- Billing / payments: reservations are free in v1. Paid slips are a later product decision.
- Public mobile app: the arrivals board is public (read-only), but the reservation UI is operator-only.
- Real-time sync: operators will refresh the page. WebSockets and live updates are deferred.
- Historical analytics: "which slips are busiest?" is interesting but not blocking. v1 stores reservations; reports are later.

## Success signals

- Operators can list slips and see which are free at a given time.
- Operators can create a reservation. The system rejects overlapping reservations for the same slip.
- Operators see a conflict warning **before** they commit the reservation, not after.
- The public arrivals board shows today's reservations (slip, vessel, arrival time). No login required.
- Zero conflicts in the first month (current baseline: 5/month).

## Open questions

- Who owns the slip inventory (adding/removing slips)? Is that an operator action or an admin action?
- Do operators need to edit or cancel a reservation, or only create?
- What time granularity? Current spreadsheet is "day." Do we need hour-level reservations, or is "August 13 all day" enough?
- Is there a maximum reservation duration (e.g., 7 days), or can a vessel reserve a slip for a year?
- If a conflict is detected, should the system suggest an alternative slip, or just reject and let the operator pick?

---

**Next step:** originator review. After approval, present to product owner for sign-off and spend cap.
