# ADR 0007: A 10-minute heartbeat watches running tracks

- Status: Accepted
- Date: 2026-08-13

## Context

Controller chats end. Headless workers continue. Without an external wake-up, nobody notices a stuck tmux pane, a red default-branch CI, or a PR that has been `AWAITING GATE` for an hour.

## Decision

While any track is **implementing or in CI**, the CoS emits a **dual-timestamped heartbeat at least every 10 minutes**, including "still working." `AWAITING GATE` fires once immediately, then stays quiet until the CoS issues REVISE or GATE. Stuck and steer still fire immediately. Quiet when idle.

A watchdog (a scheduled CoS routine, or a tmux loop that only repeats an authorized step) may:

- inspect tmux, git, PR head, and CI
- infer `AWAITING GATE` from git/PR when the pane is gone (draft PR + COMMENT on this head + SHA). A missing pane is not a dead track. Do not re-dispatch.
- take the next already-authorized repo-safe step
- refuse to double-dispatch a progressing track

It may not redefine scope, invent gates, merge production, become the implementer, or retry an unresolved escalation forever.

Escalation for the same blocker: (1) capture evidence, (2) one REVISE, (3) stop the track and escalate. There is no attempt 4. Independent tracks continue. If a same-class P1 was already fixed once in this SHA lineage, accept the residual or escalate.

Every implementing/CI heartbeat opens with local time and `YYYY-MM-DD HH:MM:SS TZ`.

## Consequences

- Grok Bot CoS uses a 10-minute routine every calendar day, including weekends, while any track is implementing or in CI. Claude-style controllers poll on the same cadence.
- "Still working on `dev-1`" is a valid heartbeat. Silence after one `AWAITING GATE` notice is expected until REVISE or GATE.
- Operators see progress without opening every machine.
