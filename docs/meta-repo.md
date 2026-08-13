# Meta-repo

A **meta-repo** is a map of how your products fit together. It is not a product, and it is not a dump of every private note you have ever written.

## Why a map repo exists

Product repos accumulate local truth: this service's ADRs, this app's deploy, this schema. Cross-cutting questions — which controller playbook applies, which goal is standing, which repo owns auth — have no single product home.

The meta-repo answers those questions without becoming a monorepo:

- Index of products and their public purpose
- Method ADRs and playbooks (or links to this repository)
- Standing goal-prompt locations
- "Start here" for a new operator or a new CoS session

It does **not** replace product history. Implementation PRs still land in the product repo.

## What belongs where

| Belongs in the meta-repo | Belongs in a product repo | Belongs in `.private/` (never published) |
| --- | --- | --- |
| Method ADRs (CoS, gates, worktrees) | Product ADRs (schema, tenancy, stack) | Host tables with real machines |
| Playbook index and controller choice | App README, architecture, runbooks | IPs, SSH targets, VPN inventory |
| Links to standing goals | The goal prompt for that product, if you keep it next to the code | Personal names, mailboxes, chat handles |
| Fictional or placeholder examples | CI, deploy, and code | Secrets, account IDs, customer names |

If a document needs a real hostname to be useful, it is not a public method document. Keep it out of the public tree.

## `.private/` never publishes

Use a `.private/` directory (any name in that class) for operator-only notes. Do not commit it to a public remote. Do not link to it from Pages. Do not copy filenames from a private tree into a public README.

This repository on purpose contains **no** `.private/` directory and **no** real private filenames. The rule is the pattern: local operator state stays local.

## How the CoS uses the map

1. Open the meta-repo to learn which product repos exist and which playbook applies.
2. Open the product repo worktree to implement.
3. Record method changes here or in this playbook repo.
4. Record product decisions as product ADRs.

Do not vendor another public product (including a runtime) into the playbook. Link it.

## Publishing a method

When you extract a playbook for others, rewrite. Do not paste a private runbook and delete a few lines. The public artifact should read as if the private shop never existed: placeholders for workers, documentation-range IPs only, no colleague names, no home paths.
