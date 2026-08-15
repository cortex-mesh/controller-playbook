# ADR 0005: Goal prompts are the standing instruction

- Status: Accepted
- Date: 2026-08-13

## Context

Chat history is not a project plan. A new session, a resumed session, and a CoS heartbeat all need the same source of truth: what is already decided, what phase is next, and what "done" means.

## Decision

A **goal prompt** is a long, versioned document that every session re-reads:

- Header names the CoS and a worker **pool**, not a single immortal machine.
- Locked decisions are numbered and not reopened by workers.
- Phases start with docs/ADRs. Each phase has a definition of done: the exact product check commands (`make check`, or the exact commands from that repo CI), not a prose vibe. `AWAITING GATE` is illegal until those commands exit 0.
- A progress log is updated in place. History is not silently rewritten.
- Model policy and launch shape match the chosen controller playbook.
- Autonomy is explicit. The human-stop list is explicit.

Amend the goal through a PR. Do not point a goal written for one controller at another without rewriting the provider block.

Authoring notes live in [skills/goal-prompt/SKILL.md](../../skills/goal-prompt/SKILL.md).

## Consequences

- Fresh workers can finish a phase with no prior chat.
- The CoS can answer "what is next?" from the progress log.
- Short goals are treated as incomplete, not clever.
