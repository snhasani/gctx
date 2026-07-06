---
slug: docs-man
parent: gctx-v1
use_case_ref: gctx-v1 / all steps (self-serve manual)
type: technical
status: ready
blocked_by: [doctor-checks, rm-context]
blocks: []
related: []
migration: none
tracker_ids: {github: 10}
---

# Self-contained public manual — README, concepts, troubleshooting, man page

## Goal
As a public-repo visitor (and the future maintainer), I want a self-contained manual so that install, concepts, and failure recovery need no access to private notes.

## Context
The repo is public; the maintainer's vault is private and keeps only a pointer note. clap_mangen generates the man page from the same clap `Command` as `--help`, so the two cannot drift.

## Acceptance Criteria
- [ ] README is the manual (install incl. zsh shim + man page, quickstart, command overview); `docs/concepts.md` covers the three dials, ADC search order + which credentials can sign, impersonation + the Token Creator role, and the env-var contract table; `docs/troubleshooting.md` maps every catalogued symptom → cause → fix.
- [ ] `man gctx` works: `gctx.1` is generated from the clap `Command` via clap_mangen; a `gctx man` subcommand prints/installs it.
- [ ] direnv reload semantics are verified empirically and the `use` × direnv precedence rule is documented in `docs/concepts.md`.

## Orientation
- The use×direnv precedence rule must come from an actual experiment (what triggers direnv re-evaluation), not assumption — it is the plan's only remaining unresolved item.
- `docs/concepts.md` mirrors the durable explainers so the repo needs no private notes; the env-var contract table already exists in plans/plan.md to mirror from.

## Out of scope
- Publishing to crates.io / Homebrew; release automation.
