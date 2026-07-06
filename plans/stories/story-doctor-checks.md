---
slug: doctor-checks
parent: gctx-v1
use_case_ref: gctx-v1 / main scenario step 5, variation V3
type: behavioral
status: ready
blocked_by: [impersonate-adc]
blocks: [docs-man]
related: []
migration: none
tracker_ids: {github: 8}
---

# gctx doctor — name the broken dial and its fix

## Goal
As a developer facing a mysteriously failing tool, I want `gctx doctor <name>` to check all three dials and name the broken one with its fix so that credential-drift debugging takes seconds instead of hours.

## Context
Doctor encodes the real failure catalogue that motivated gctx (the plan's "doctor checks" and troubleshooting sections). Checks span the wallet, the ADC file, the gcloud configuration, and the repo's `.envrc` block.

## Acceptance Criteria
- **Given** a healthy context **When** doctor runs **Then** every check passes: config exists, account in wallet, project resolves, ADC file readable with the right type, block project matches config.
- **Given** each catalogued fault (account missing from wallet, `authorized_user` ADC where signing is expected, missing token-creator role, block drift, dangling block) **When** doctor runs **Then** it reports that specific cause with its documented fix.
- **Given** an impersonate-mode context **When** doctor runs **Then** it proves a token can actually be minted for the SA.

## Orientation
- Network-touching checks (`projects describe`, `print-access-token`) flow through the `GCTX_GCLOUD` exec seam — integration tests fake them; check logic stays pure above the seam.
- "Warn on `authorized_user`" applies only where signing is expected — a user-mode context's ADC is legitimately `authorized_user`.

## Out of scope
- Auto-fixing — doctor diagnoses; `emit`/`new`/`rm` repair.
