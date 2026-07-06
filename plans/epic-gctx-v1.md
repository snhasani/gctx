---
slug: gctx-v1
status: ready
blocked_by: []
blocks: []
related: []
migration: none
tracker_ids: {github_milestone: 1}
---

# gctx v1 — gcloud context switcher

## Problem

Working across many GCP orgs/projects means hand-maintaining a gcloud env block in every repo's `.envrc`. The three dials (wallet, ADC, configuration) drift apart silently: gcloud works while Terraform hits the wrong project, client libs fail with "Cannot sign data without client_email", and org policy blocks the SA-key workaround. Standing up a new context is a fiddly, undocumented multi-step ritual; nothing verifies the dials agree.

## Outcome / Definition of Done

- `gctx new` stands up a complete context: gcloud, `terraform plan`, and a client-lib call resolve the same project + credentials with zero manual env editing (opt-in E2E smoke passes).
- Both switching mechanisms verified: cd into repo (direnv) and `gctx use` (same shell).
- `gctx doctor` names cause + fix for every fault in the troubleshooting catalogue (each simulated fault → its documented message).
- The monorepo's hand-written gcloud block is replaced by a managed block (one-time manual `gctx emit` migration executed).
- Repo self-contained + public: README manual, `concepts.md`, `troubleshooting.md`, man page; CI green; binary installed at `~/.local/bin/gctx`.

## Use-Case Narrative

- **Actor:** k1 — developer working across many GCP orgs/projects.
- **Goal:** stand up a GCP context once; switch contexts per repo/shell with the three dials always agreeing.
- **Preconditions:** gcloud + direnv installed and hooked; browser available once (base ADC); org blocks SA key downloads.
- **Main scenario:**
  1. k1 runs `gctx new <name>` — picks account/project/region/zone + ADC mode; gctx creates the config, logs in what's missing, writes sidecar + `.envrc` block.
  2. k1 cds into the repo — direnv applies the block; gcloud, Terraform, client libs all read the same context.
  3. In an ad-hoc shell, `gctx use <name>` flips that shell only.
  4. `gctx ls` shows every context × ADC mode at a glance.
  5. Something misbehaves — `gctx doctor <name>` names the broken dial and the fix.
  6. Context evolves or dies — `gctx emit` refreshes the block; `gctx rm` deletes config + sidecar + ADC file.
- **Variations / extensions:**
  - V1 impersonate mode: ADC file wraps base ADC for an SA (no key, signs via IAM).
  - V2 missing sidecar (raw-gcloud config): treated as implicit user mode.
  - V3 drift/dangling block: doctor flags; emit repairs.

## Story Index

| Story slug | Intent | Use-case step | Migration | Status |
|---|---|---|---|---|
| emit-block | `gctx emit` + `_export`: env union from config+sidecar, idempotent marker block | 2, 6 | none | todo |
| use-shell | `gctx use` zsh shim flips current shell via `_export` | 3 | none | todo |
| new-bootstrap | `gctx new` flag-driven bootstrap, user mode, prompts for missing | 1 | none | todo |
| impersonate-adc | impersonate mode across new/emit: constructed ADC file, extra vars | V1 | none | todo |
| ls-contexts | `gctx ls`: configs × sidecars join, orphans + implicit user visible | 4 | none | todo |
| doctor-checks | `gctx doctor`: full check catalogue incl. drift + dangling + token | 5, V3 | none | todo |
| rm-context | `gctx rm`: delete config + sidecar + ADC file | 6 | none | todo |
| docs-man | README manual, concepts.md (incl. verified use×direnv rule), troubleshooting.md, man page, install | all | none | todo |

## Sequencing

emit-block is the tracer-equivalent: thinnest end-to-end path through the core (env union → block). Relations:

- use-shell, new-bootstrap, ls-contexts — `blocked_by: emit-block`
- impersonate-adc, rm-context — `blocked_by: new-bootstrap`
- doctor-checks — `blocked_by: impersonate-adc`
- docs-man — `blocked_by: doctor-checks, rm-context`

## Risks / Open questions

- direnv reload semantics unverified — must be verified during docs-man (concepts.md precedence rule).
- gcloud CLI output stability — typed gcloud.rs seam isolates parsing; stub keeps tests offline.

> GAP: docs-man has no direct user-goal step — technical-outcome story (public self-contained repo).
