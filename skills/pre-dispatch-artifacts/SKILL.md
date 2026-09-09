---
name: pre-dispatch-artifacts
description: Author and advance intent, spec, and plan artifacts before a CoS writes a goal prompt. Use when starting new autonomous work that requires human sign-off checkpoints before dispatch.
---

# Pre-dispatch artifacts

Before a Chief of Staff writes a goal prompt and dispatches workers, advance a **durable document chain** with human checkpoints: `intent.md` → originator review → PO sign-off → `spec.md` → `plan.md` → **then** the goal prompt.

This is not a replacement for the goal prompt, dependency graph, or gates. It is a **pre-dispatch stage** that produces a locked plan the CoS can turn into a goal without re-reading chat.

Inspired by Anthropic's AI-native SDLC (Boris / Claude Code playbook): each stage starts from durable documents, not one long chat. Humans and agents hand off via artifacts before the CoS ever writes a goal prompt.

## When to use

- New product, feature, or initiative where problem framing and scope must be explicit before implementation starts.
- When a human originator (the person who raised the intent) and a product owner (who owns scope/spend) need checkpoints before a CoS commits compute.
- When the intent will outlive one chat and resume in future sessions.

Skip this chain for small, already-scoped work where the goal prompt is the natural starting document.

## Artifact chain

| Artifact | Owned by | Checkpoint | Next step |
| --- | --- | --- | --- |
| `intent.md` | Originator or agent | **Originator review** | Product-owner sign-off |
| `spec.md` | Agent after PO sign-off | **Product-owner sign-off** | Write plan |
| `plan.md` | Agent after spec locked | None (feeds goal prompt) | CoS authors goal prompt per [goal-prompt](../goal-prompt/SKILL.md) |

After `plan.md` is complete, the **Chief of Staff** writes a goal prompt (per [`goal-prompt`](../goal-prompt/SKILL.md)), draws the dependency graph, dispatches workers, implements, runs different-family COMMENT review, gates, and human merge — the same loop as always.

## 1. Intent (`intent.md`)

Start here. A short document (1–2 pages) that names the problem, not the solution.

### Sections (all required)

- **Problem:** What is broken, missing, or expensive today? Who feels the pain?
- **Why now:** Why is this urgent? What is the cost of waiting?
- **Non-goals:** What is explicitly out of scope?
- **Success signals:** How will we know the problem is solved? User-facing outcomes, not tasks.
- **Open questions:** What is unknown? Dependencies, risks, or unresolved product choices.

### Style

- Terse. No fluff. No solution yet.
- Use placeholder names (`Harbor`, `dev-1`, `user@host`) if examples are needed. No personal names, IPs, or fleet inventory.
- Keep the problem space front. "Users cannot see conflicts" is clearer than "we should build a conflict view."

### Checkpoint: originator review

After drafting, the **originator** (the human who raised the intent) reviews and approves. If the originator is unavailable, escalate. Do not advance to PO sign-off without originator confirmation.

## 2. Product-owner sign-off

After the originator confirms the intent, the **product owner** (who owns scope and spend) reviews:

- Is this the right problem?
- Is now the right time?
- What is the spend cap?
- What is locked out of scope?

PO approval unlocks the spec. Do not write a spec without PO sign-off.

## 3. Spec (`spec.md`)

Written **after PO sign-off**. This is the product/behavior contract.

### Sections (all required)

- **In scope:** What ships in this effort. Be concrete.
- **Out of scope:** What is deferred or rejected. List the locked descopes from PO sign-off.
- **UX or API surface:** Screens, flows, endpoints, or CLI commands. Low-fidelity wireframes or examples are fine. No final polish.
- **Acceptance criteria:** User-facing tests. "Operator can create a reservation and see conflicts" is testable; "improve UX" is not.
- **Locked decisions:** Number them (`D1`, `D2`, …). These are not reopened during implementation.

### Style

- Terse, concrete, testable.
- No implementation detail yet. "Conflict detection" is in-scope; "use a spatial index" is a plan concern.
- Use examples liberally. Placeholder data (`Harbor`, `dev-1`, `2026-08-13`) is better than prose.

### When the spec is approved

The spec is locked. Scope changes after this point require re-approval.

## 4. Plan (`plan.md`)

Written **after the spec is locked**. This is a self-contained implementation plan that a CoS can turn into a goal prompt without re-reading chat.

### Sections (all required)

- **Phases:** Ordered work packages. Phase 0 is always docs/ADRs. Later phases are code. Name inputs and dependencies.
- **Risks:** What might block or slow the work? Missing logins, flaky CI, unreviewed libraries, ambiguous requirements.
- **Dependencies:** External teams, credentials, DNS, paid resources, or other gates.
- **Definition-of-done hints:** Per-phase verification (CI-equivalent check, mutation tests, staging live-verify). The CoS will refine these into exact commands in the goal prompt.

### Style

- Self-contained. A fresh CoS session should be able to write a goal prompt from this plan alone.
- Terse. Phase 0 is one line ("Write stack ADR, tenancy ADR, local-dev README"). Code phases are one paragraph each.
- No chat. No "we should" or "let's." Imperative: "Implement CRUD for slips. Conflict detection. Mutation-test the conflict rule."

### After the plan is complete

The plan feeds the **goal prompt**. The CoS still authors a goal per [`goal-prompt`](../goal-prompt/SKILL.md), then:

1. Dependency graph before a parallel wave
2. Dispatch to workers
3. Implement, draft PR, different-family COMMENT review
4. GATE, human merge, staging verify, production (human)

The plan is not a substitute for the goal prompt, graph, or gates. It is the durable input the goal consumes.

## Relationship to the shipping loop

The pre-dispatch chain is a **prefix** to the existing loop. Nothing changes after the goal prompt is written:

1. **Pre-dispatch** (this skill): `intent.md` → originator review → PO sign-off → `spec.md` → `plan.md`
2. **Then** the CoS writes a goal prompt (per [`goal-prompt`](../goal-prompt/SKILL.md))
3. **Then** dependency graph, dispatch, implement, COMMENT review, GATE, human merge, staging verify, production (the same loop as before)

The CoS is still the orchestrator. Workers still implement phases. Different-family reviewers still post COMMENT. Humans still merge and gate production.

## Hard rules

- Do not skip checkpoints. Originator review and PO sign-off are mandatory.
- Do not write a spec without PO sign-off.
- Do not write a plan without a locked spec.
- Do not write a goal prompt without a locked plan (unless the work is small enough to skip this chain).
- Placeholders only. No personal names, IPs, fleet inventory, Slack channels, AWS account IDs, or home paths. Use `dev-1`, `ci-box`, `user@host`, Harbor.

## Examples

See [`examples/sample-intent.md`](../../examples/sample-intent.md), [`examples/sample-spec.md`](../../examples/sample-spec.md), and [`examples/sample-plan.md`](../../examples/sample-plan.md).
