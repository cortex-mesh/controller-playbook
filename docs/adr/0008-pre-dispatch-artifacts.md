# ADR 0008: Pre-dispatch artifacts precede the goal prompt

- Status: Accepted
- Date: 2026-09-09

## Context

Chat history is not the system of record before a Chief of Staff writes a goal prompt and dispatches workers. A human originator raises a problem. A product owner must approve scope and spend. A CoS needs a locked plan to turn into a goal without re-reading one long chat.

Without durable pre-dispatch artifacts, three failure modes recur:

1. The CoS writes a goal prompt before the problem is framed or the product owner has approved scope, then discovers the work is out-of-scope mid-implementation.
2. A human originator and a product owner are never explicitly consulted. The CoS invents scope from an ambiguous request.
3. The plan lives only in chat. A fresh CoS session cannot resume without re-reading 50 turns.

Inspired by Anthropic's AI-native SDLC (Boris / Claude Code playbook): each stage starts from durable documents, not one long chat. Humans and agents hand off via artifacts.

## Decision

Before a Chief of Staff writes a goal prompt, advance a **pre-dispatch artifact chain**:

1. **`intent.md`** — problem, why now, non-goals, success signals, open questions.
2. **Originator review** checkpoint (human who raised the intent).
3. **Product-owner sign-off** checkpoint (human who owns scope/spend).
4. **`spec.md`** — product/behavior contract after PO sign-off (in/out of scope, UX or API surface, acceptance criteria, locked decisions).
5. **`plan.md`** — self-contained implementation plan (phases, risks, dependencies, DoD hints) that a CoS can turn into a goal prompt without re-reading chat.

After `plan.md` is locked, the CoS **still** authors a **goal prompt** per [`skills/goal-prompt/SKILL.md`](../../skills/goal-prompt/SKILL.md). Then dependency graph, dispatch, implement, different-family COMMENT review, GATE, human merge, staging verify, production (human) — the same loop as before.

`plan.md` feeds the goal; it is not a substitute for the goal prompt, graph, or gates.

Authoring notes live in [`skills/pre-dispatch-artifacts/SKILL.md`](../../skills/pre-dispatch-artifacts/SKILL.md). Examples: [`examples/sample-intent.md`](../../examples/sample-intent.md), [`examples/sample-spec.md`](../../examples/sample-spec.md), [`examples/sample-plan.md`](../../examples/sample-plan.md).

## Consequences

- Chat is not the system of record before dispatch. Durable documents are.
- A human originator and a product owner have explicit checkpoints before a CoS commits compute.
- A fresh CoS session can resume from `plan.md` without re-reading 50 chat turns.
- The goal prompt still owns the exact DoD commands, the dependency graph, the dispatch loop, and the progress log. The plan is an input, not a replacement.
- Small, already-scoped work can skip this chain and start directly at the goal prompt. The pre-dispatch chain is for new products, features, or initiatives where problem framing and scope must be explicit.
