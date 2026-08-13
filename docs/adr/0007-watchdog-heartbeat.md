# ADR 0007: A 10-minute heartbeat watches running tracks

- Status: Accepted
- Date: 2026-08-13

## Context

Controller chats end. Headless workers continue. Without an external wake-up, nobody notices a stuck tmux pane, a red default-branch CI, or a PR that has been `AWAITING GATE` for an hour.

## Decision

While any track is running, the CoS emits a **timestamped heartbeat at least every 10 minutes**, including "still working." Quiet only when the pool is idle. Stuck, steer, blocked, and `AWAITING GATE` fire immediately.

A watchdog (a scheduled CoS routine, or a tmux loop that only repeats an authorized step) may:

- inspect tmux, git, PR head, and CI
- take the next already-authorized repo-safe step
- refuse to double-dispatch a progressing track

It may not redefine scope, invent gates, merge production, or retry an unresolved escalation forever.

Escalation for the same blocker: (1) capture evidence, (2) one authorized recovery, (3) stop the track and escalate. There is no attempt 4.

Every heartbeat opens with local time and `YYYY-MM-DD HH:MM:SS TZ`.

## Consequences

- Grok Bot CoS uses a 10-minute routine every calendar day, including weekends, while any track is running. Claude-style controllers poll on the same cadence.
- "Still working on `dev-1`" is a valid heartbeat. Silence is not.
- Operators see progress without opening every machine.
