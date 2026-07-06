---
slug: emit-block
parent: gctx-v1
use_case_ref: gctx-v1 / main scenario steps 2, 6
type: behavioral
status: ready
blocked_by: []
blocks: [use-shell, new-bootstrap, ls-contexts]
related: []
migration: none
tracker_ids: {github: 3}
---

# gctx emit — managed .envrc block from an existing context

## Goal
As a developer juggling many GCP projects, I want `gctx emit <name>` to (re)generate the managed env block in the current repo's `.envrc` so that gcloud, Terraform, and client libs all read the same context without hand-maintained env lines.

## Context
The crate is a bare scaffold (no modules yet). gcloud configurations already exist on the machine (creatable via raw gcloud). The marker-bounded `.envrc` block is the product's core artifact; every other command composes around it.

## Acceptance Criteria
- **Given** a gcloud configuration `dev` and an `.envrc` containing hand-written lines **When** `gctx emit dev` runs twice **Then** the file contains exactly one marker-bounded block with the documented user-mode var union, byte-identical across runs, with hand-written lines intact.
- **Given** no sidecar exists for `dev` **When** `gctx emit dev` runs **Then** the block is user-mode (missing sidecar means implicit user mode).
- **Given** `gctx _export dev` output eval'd in a shell **Then** the exported variable set equals the block's variable set exactly.

## Orientation
- `env_for(ctx) -> Vec<(Key, Value)>` is the single pure source both renderings (marker block, `_export` lines) consume — locked in plans/plan.md so block/`use` drift is impossible.
- The var union per mode is specified in the plan's "three dials" table and "Generated .envrc block" section; `GOOGLE_PROJECT_NUMBER` is deliberately absent (not derivable offline) — don't re-add it.
- The only exec boundary is gcloud.rs's private `run()` honoring the `GCTX_GCLOUD` override; the sidecar dir honors `GCTX_CONFIG_DIR`; integration tests relocate both plus `CLOUDSDK_CONFIG`.
- `emit` is offline by locked decision: exec'ing gcloud for local config reads is acceptable, network subcommands are not.

## Out of scope
- The `use` zsh shim, `new` bootstrap, and impersonate-mode vars (own stories).
- Any clap surface beyond `emit` and hidden `_export`.
