# ADR 0006: A meta-repo is a map, not a product

- Status: Accepted
- Date: 2026-08-13

## Context

A shop with several product repos needs a place for cross-cutting method, inventory, and "start here" notes. Dumping that into one product repo hides it from the others. Dumping private host tables into a public method repo leaks inventory.

## Decision

- Keep a **meta-repo** as a map: what products exist, which playbook applies, where standing goals live.
- Keep **product ADRs and code** in product repos.
- Keep **operator inventory** (real hosts, IPs, secrets, personal paths) out of any public tree. A `.private/` directory never publishes.
- This playbook repository publishes method ADRs only.

See [docs/meta-repo.md](../meta-repo.md).

## Consequences

- A CoS session starts in the map, then implements in a product worktree.
- Public readers can adopt the method without inheriting a private mesh.
- Copy-paste from a private runbook is a defect, not a shortcut.
