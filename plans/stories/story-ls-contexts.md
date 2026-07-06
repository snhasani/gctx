---
slug: ls-contexts
parent: gctx-v1
use_case_ref: gctx-v1 / main scenario step 4
type: behavioral
status: ready
blocked_by: [emit-block]
blocks: []
related: []
migration: none
tracker_ids: {github: 7}
---

# gctx ls — contexts × ADC mode at a glance

## Goal
As a developer juggling many contexts, I want `gctx ls` to show every context with its ADC mode so that I can see what exists and spot orphans at a glance.

## Context
A context is a gcloud configuration joined with a gctx sidecar; either side can exist alone (config created via raw gcloud, or a sidecar left behind after a config was deleted outside gctx).

## Acceptance Criteria
- **Given** configs with and without sidecars **When** `gctx ls` runs **Then** each config shows its mode: `user`, `impersonate (<SA>)`, or `user (implicit)` when no sidecar exists.
- **Given** a sidecar whose config was deleted via raw gcloud **When** `gctx ls` runs **Then** that row is flagged as an orphan.

## Orientation
- The join is full-outer by design: config without sidecar renders `user (implicit)`; sidecar without config is an orphan row (resolution recorded in plans/plan.md).

## Out of scope
- Repair actions — adopting an orphan is `new --reuse`; deletion is `rm`.
