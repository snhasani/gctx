---
slug: use-shell
parent: gctx-v1
use_case_ref: gctx-v1 / main scenario step 3
type: behavioral
status: ready
blocked_by: [emit-block]
blocks: []
related: []
migration: none
tracker_ids: {github: 4}
---

# gctx use — flip the current shell's context

## Goal
As a developer in an ad-hoc shell, I want `gctx use <name>` to switch that shell's context so that one-off commands hit the right project without cd'ing into a direnv repo.

## Context
A child process cannot mutate its parent shell's environment, so `use` cannot live in the Rust binary. The binary's hidden `_export` prints eval-able export lines; a zsh function evals them in-shell.

## Acceptance Criteria
- **Given** a zsh shell with the gctx function sourced **When** `gctx use dev` runs **Then** that shell's environment contains dev's full var union and no files were modified.
- **Given** a second shell **When** `gctx use stage` runs there **Then** the first shell's environment is unchanged.
- **Given** any other subcommand invoked through the function **Then** it is delegated to the binary unchanged.

## Orientation
- The zsh function shadows the binary: `use` is handled in-function via `eval "$(command gctx _export <name>)"`, everything else delegates (architecture section, plans/plan.md).
- Destined install location is `~/.config/zsh/gctx.zsh`, sourced from zshrc; install wiring and docs belong to the docs-man story.

## Out of scope
- direnv precedence handling or documentation (docs-man verifies + documents the rule).
- Install automation.
