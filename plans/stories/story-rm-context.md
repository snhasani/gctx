---
slug: rm-context
parent: gctx-v1
use_case_ref: gctx-v1 / main scenario step 6
type: behavioral
status: ready
blocked_by: [new-bootstrap]
blocks: [docs-man]
related: []
migration: none
tracker_ids: {github: 9}
---

# gctx rm — delete a context completely

## Goal
As a developer ending a client engagement, I want `gctx rm <name>` to remove the context completely so that stale credentials and configs don't accumulate on my machine.

## Context
Deletion spans the gcloud configuration, the gctx sidecar, and the per-context impersonated ADC file. Repo `.envrc` blocks cannot be enumerated (any repo may hold one), so they are out of rm's reach by design.

## Acceptance Criteria
- **Given** a context with config + sidecar + impersonated ADC file **When** `gctx rm <name>` runs **Then** all three are gone and repo `.envrc` blocks are left untouched.
- **Given** partial remnants (e.g. config already deleted via raw gcloud, sidecar remains) **When** rm runs **Then** it removes what exists and succeeds.

## Orientation
- Dangling repo blocks are doctor's concern (flagged), never rm's — rm cannot find every repo.
- The opt-in E2E smoke test's teardown relies on rm, which is why partial-remnant tolerance matters.

## Out of scope
- Bulk operations (`rm --all`, prune).
- Touching any `.envrc` file.
