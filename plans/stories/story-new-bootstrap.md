---
slug: new-bootstrap
parent: gctx-v1
use_case_ref: gctx-v1 / main scenario step 1
type: behavioral
status: ready
blocked_by: [emit-block]
blocks: [impersonate-adc, rm-context]
related: []
migration: none
tracker_ids: {github: 5}
---

# gctx new — one-command user-mode context bootstrap

## Goal
As a developer starting with a new client/org, I want `gctx new <name>` to stand up a complete user-mode context in one command so that a fresh project is usable by every tool in minutes instead of a manual multi-step ritual.

## Context
Flag-driven (`--account --project --region --zone --run-region --adc --sa --reuse`) with interactive prompts only for missing values, so tests run it non-interactively with all flags passed. User-mode only; impersonate mode is its own story.

## Acceptance Criteria
- **Given** account/project/region/zone flags **When** `gctx new dev` runs **Then** the gcloud config exists with those properties, a sidecar records `adcMode=user`, the CWD `.envrc` gains the managed block, and a `direnv allow` reminder is printed.
- **Given** the context already exists **When** `gctx new dev` runs **Then** it fails with a clear error; with `--reuse` it reconfigures instead.
- **Given** no base ADC on the machine **When** `gctx new` runs **Then** it triggers the one-time `gcloud auth application-default login`; when present, no browser flow occurs.

## Orientation
- Context names inherit gcloud's config-name rule (lowercase letters/digits/hyphens) — validated before any mutation (commands section, plans/plan.md).
- Wallet membership decides whether `gcloud auth login <account>` runs. In tests every gcloud call goes through the `GCTX_GCLOUD` stub — a delegating fake: local `config *` commands pass through to real gcloud, network calls are faked.
- Prompts stay thin: read a line, feed the same pure validators the flags use.

## Out of scope
- Impersonate mode (own story).
- Fetching the project number — deliberately dropped so `emit` stays offline.
