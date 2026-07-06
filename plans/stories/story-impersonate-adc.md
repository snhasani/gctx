---
slug: impersonate-adc
parent: gctx-v1
use_case_ref: gctx-v1 / variation V1
type: behavioral
status: ready
blocked_by: [new-bootstrap]
blocks: [doctor-checks]
related: []
migration: none
tracker_ids: {github: 6}
---

# Impersonate mode — keyless SA credentials per context

## Goal
As a developer on an org that blocks SA key downloads, I want impersonate-mode contexts so that Terraform and client libs can sign as the service account with no key file and no per-context browser login.

## Context
Org policy blocks SA key creation; the base user ADC exists from a one-time login. Impersonation wraps it into the documented `impersonated_service_account` JSON, persisted per context with mode 600.

## Acceptance Criteria
- **Given** `gctx new svc --adc impersonate --sa <email>` **When** it completes **Then** the per-context ADC file has type `impersonated_service_account`, the correct `generateAccessToken` URL for the SA, the base ADC embedded as `source_credentials`, file mode 600, and the sidecar records the mode + SA.
- **Given** an impersonate-mode context **When** `gctx emit` / `_export` run **Then** the output adds `GOOGLE_APPLICATION_CREDENTIALS` → per-context file plus both impersonation vars (`GOOGLE_IMPERSONATE_SERVICE_ACCOUNT`, `CLOUDSDK_AUTH_IMPERSONATE_SERVICE_ACCOUNT`).
- **Given** the ADC construction in tests **Then** no browser flow and no network call occur — it is a pure JSON transform of the base ADC.

## Orientation
- docs/adr/0001 records why constructed-JSON beats the official per-context `--impersonate-service-account` login flow — don't "fix" toward the official flow.
- Runtime signing requires `roles/iam.serviceAccountTokenCreator` on the SA; verifying that is doctor's job, not new's.
- The impersonated ADC path derives from `CLOUDSDK_CONFIG` (default `~/.config/gcloud/adc/<name>.json`) so tests can relocate it.

## Out of scope
- Token-mint verification (doctor-checks story).
