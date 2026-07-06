# gctx — gcloud context switcher

Bootstrap + switch GCP contexts (account · project · region · zone · ADC) across many orgs/projects. One command to stand up a context; direnv + a manual command to switch. Replaces the hand-maintained `.envrc` gcloud block.

## Locked decisions

- Name: `gctx`.
- Scope: bootstrap a context **and** emit a matching `.envrc` block.
- Switching: direnv (auto per-repo) **+** manual `gctx use` (in-shell).
- ADC: login included; **user** or **impersonated** SA (no downloaded keys — org blocks them).
- ADC-mode state: sidecar `~/.config/gctx/<name>.json` = `{adcMode, serviceAccount}`.
- Impersonated ADC file: constructed in Rust (serde) from the base ADC — no browser per context (ADR 0001); persisted at `~/.config/gcloud/adc/<name>.json`, `chmod 600`.
- Env-var names: research-driven union so gcloud + Terraform + client libs all work unmodified (below).
- `emit` is offline: the block derives fully from config + sidecar — nothing network-fetched lands in it.
- Workhorse: **Rust** (single binary). `use` stays a **zsh** function — a binary can't mutate the parent shell.
- Testable via seams: pure logic units + relocatable `CLOUDSDK_CONFIG` + relocatable `GCTX_CONFIG_DIR` (sidecar dir) + a `GCTX_GCLOUD` exec override. `new` is flag-driven so it runs non-interactively in tests; prompts stay thin — read a line, feed the same pure validators, nothing else.

## The three dials (why the var set is what it is)

Independent surfaces; each tool reads different names → emit the union.

| Concept | gcloud reads | Terraform reads (1st) | Client libs read | ⇒ gctx emits |
| --- | --- | --- | --- | --- |
| active config | `CLOUDSDK_ACTIVE_CONFIG_NAME` | — | — | `CLOUDSDK_ACTIVE_CONFIG_NAME` |
| project | `CLOUDSDK_CORE_PROJECT`¹ | `GOOGLE_PROJECT` | `GOOGLE_CLOUD_PROJECT` | `GOOGLE_PROJECT` + `GOOGLE_CLOUD_PROJECT` |
| region | `CLOUDSDK_COMPUTE_REGION`¹ | `GOOGLE_REGION` | — | `GOOGLE_REGION` + `CLOUDSDK_COMPUTE_REGION` |
| zone | `CLOUDSDK_COMPUTE_ZONE`¹ | `GOOGLE_ZONE` | — | `GOOGLE_ZONE` + `CLOUDSDK_COMPUTE_ZONE` |
| ADC (creds) | — | falls back to `GOOGLE_APPLICATION_CREDENTIALS` | `GOOGLE_APPLICATION_CREDENTIALS` | `GOOGLE_APPLICATION_CREDENTIALS` |
| impersonation | `CLOUDSDK_AUTH_IMPERSONATE_SERVICE_ACCOUNT` | `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT` | via impersonated ADC file | both vars + impersonated ADC file |

¹ Also carried by the active config, so redundant for gcloud but set for tools that read `CLOUDSDK_*` directly.

> `CLOUDSDK_ACTIVE_CONFIG_NAME` is the special-cased name for active config (verified: Managing gcloud CLI configurations). The generic `CLOUDSDK_SECTION_PROPERTY` pattern does **not** apply to it.

Dropped from the old `.envrc`: `GCP_ACCOUNT`, `COMPUTE_DEFAULT_REGION`, `COMPUTE_DEFAULT_ZONE`, `GOOGLE_PROJECT_NUMBER` — no tool reads them, and project number isn't derivable offline from config + sidecar.

Sources: [How ADC works](https://cloud.google.com/docs/authentication/application-default-credentials) · [Managing gcloud CLI configurations](https://cloud.google.com/sdk/docs/configurations) · [gcloud properties](https://cloud.google.com/sdk/docs/properties) · [Terraform google provider reference](https://registry.terraform.io/providers/hashicorp/google/latest/docs/guides/provider_reference).

## Architecture

Manual switch must mutate the *current* shell → shell function. Everything else → Rust binary. Split:

- `~/.local/bin/gctx` — Rust binary: `new`, `emit`, `ls`, `rm`, `doctor`, hidden `_export`.
- `~/.config/zsh/gctx.zsh` — zsh function shadowing it; handles `use` in-shell, delegates the rest:
  ```zsh
  gctx() {
    case "$1" in
      use) shift; eval "$(command gctx _export "$1")" ;;
      *)   command gctx "$@" ;;
    esac
  }
  ```

Rust module layout (seams isolate the untestable boundary from the logic):

| Module | Responsibility | Tested? |
| --- | --- | --- |
| `envrc.rs` | `env_for(ctx) -> Vec<(Key, Value)>` — single source for block **and** `_export` (drift impossible); `render_block()`, `upsert_envrc()` — marker-bounded, idempotent, preserves other lines | unit (pure) |
| `adc.rs` | build `impersonated_service_account` JSON from base ADC + SA | unit (pure) |
| `sidecar.rs` | `$GCTX_CONFIG_DIR/<name>.json` (de)serialize (serde) | unit (tempdir) |
| `gcloud.rs` | typed ops (`list_configurations`, `describe_project`, `set_property`, `login`, `print_access_token`) — output parsing lives here, callers get typed values. Private `run()` is the internal seam honoring `GCTX_GCLOUD` | faked in tests |
| `context.rs` | orchestration: `new`/`emit`/`ls`/`rm`/`doctor` compose the above | integration |
| `main.rs` | clap CLI, arg parse, dispatch | — |

## State / files

| Path | Role |
| --- | --- |
| `~/.config/gcloud/configurations/config_<name>` | dial 3 (gcloud-owned): account, project, region, zone |
| `~/.config/gctx/<name>.json` (`GCTX_CONFIG_DIR` override) | `{adcMode:"user"\|"impersonate", serviceAccount?}` — the one bit gcloud can't store |
| `$CLOUDSDK_CONFIG/adc/<name>.json` (default `~/.config/gcloud/adc/`) | dial 2: impersonated ADC file, `chmod 600`; relocates with `CLOUDSDK_CONFIG` |
| `<repo>/.envrc` | generated block between markers; rest of file untouched |

Source of truth = the gcloud configuration + the sidecar. The `.envrc` block is a regenerable cache. Missing sidecar ⇒ implicit `user` mode (`ls` shows `user (implicit)`).

## Commands

- `gctx new <name>` — bootstrap. Validates `<name>` first (inherits gcloud config-name rule: lowercase letters/digits/hyphens). Fails if the context exists; `--reuse` opts into reconfiguring. Flag-driven (`--account --project --region --zone --run-region --adc user|impersonate --sa <email> --reuse`); prompts only for what's missing (so it's scriptable *and* interactive). Steps:
  1. `gcloud config configurations create <name>` (`--reuse` adopts an existing one).
  2. account: pick; `gcloud auth login <acct>` if absent from the wallet.
  3. project: list → pick; `gcloud config set` project + compute/region + compute/zone + run/region.
  4. base ADC: well-known file missing → run `gcloud auth application-default login` (once per machine).
  5. ADC mode prompt: **user** or **impersonate <SA>** → build ADC file (below) + write sidecar.
  6. write/refresh `.envrc` block in CWD; remind to `direnv allow`.
- `gctx use <name>` — flip **this shell** now (function): eval `_export`. No files touched. Holds until direnv's next reload — exact rule verified + documented in `concepts.md`.
- `gctx emit [name]` — (re)write `.envrc` block in CWD from config + sidecar. No creation, no network.
- `gctx rm <name>` — delete config + sidecar + impersonated ADC file. Repo blocks stay; `doctor` flags dangling ones.
- `gctx ls` — configs × ADC mode table (join `gcloud config configurations list` + sidecars).
- `gctx doctor [name]` — verify all three dials agree + ADC can actually sign.
- `gctx help [command]` — usage + subcommand list; `gctx` with no args and unknown commands print it too. clap auto-generates per-command `--help`; this is the friendly top-level entry (the zsh shim forwards it to the binary).
- `gctx _export <name>` (hidden) — print `export …` lines for `use`/`.envrc`.

## ADC file construction (no repeated browser, no key)

One-time base user ADC: `gcloud auth application-default login` → well-known file.

Per context:
- **user mode** → `GOOGLE_APPLICATION_CREDENTIALS` = base ADC path.
- **impersonate mode** → build `$CLOUDSDK_CONFIG/adc/<name>.json` by wrapping the base ADC (serde) into the documented `impersonated_service_account` shape:
  ```json
  { "type": "impersonated_service_account",
    "service_account_impersonation_url":
      "https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/<SA>:generateAccessToken",
    "source_credentials": { <base user ADC> },
    "delegates": [] }
  ```
  This credential **signs** via IAM (fixes "Cannot sign without client_email"), no key file. Requires `roles/iam.serviceAccountTokenCreator` on `<SA>`.

## Generated .envrc block

User mode:
```bash
# >>> gctx:<name> >>>  (managed by gctx — regenerate with `gctx emit`)
export CLOUDSDK_ACTIVE_CONFIG_NAME=<name>
export GOOGLE_APPLICATION_CREDENTIALS="$HOME/.config/gcloud/application_default_credentials.json"
export GOOGLE_PROJECT=<pid>
export GOOGLE_CLOUD_PROJECT=<pid>
export GOOGLE_REGION=<region>
export CLOUDSDK_COMPUTE_REGION=<region>
export GOOGLE_ZONE=<zone>
export CLOUDSDK_COMPUTE_ZONE=<zone>
# <<< gctx:<name> <<<
```

Impersonate mode adds:
```bash
export GOOGLE_APPLICATION_CREDENTIALS="$HOME/.config/gcloud/adc/<name>.json"   # overrides base
export GOOGLE_IMPERSONATE_SERVICE_ACCOUNT=<SA>
export CLOUDSDK_AUTH_IMPERSONATE_SERVICE_ACCOUNT=<SA>
```

Only text between markers is owned; `gctx emit` replaces just that. Hand-added lines (NODE_OPTIONS, POSTHOG…) survive.

## doctor checks (encodes today's bugs)

- config exists; account is logged in (in the wallet).
- project resolves (`gcloud projects describe` succeeds as this context).
- ADC file exists + readable; its `type` ∈ {service_account, impersonated_service_account} when signing is expected (serde) → warn on `authorized_user`.
- impersonate mode: `gcloud auth print-access-token --impersonate-service-account=<SA>` succeeds (token-creator present) — else explain the missing role.
- `.envrc` block project == config project (drift check).
- block names a context that no longer exists (dangling after `rm`) → suggest re-`emit` or manual removal.

## Testing (seams, not language)

Three tiers; tiers 1–2 need no network and touch nothing real.

1. **Unit (pure logic — the risky 30%)** — `cargo test`:
   - `env_for`: `.envrc` block and `_export` emit the identical var set.
   - `render_block`: same inputs → byte-identical; second `upsert` is a no-op (idempotent); hand-added `.envrc` lines survive; two contexts don't clobber each other.
   - `build_impersonated_adc`: correct `type`, `service_account_impersonation_url`, embedded `source_credentials`.
   - `sidecar`: round-trips in a tempdir.
2. **Integration (real gcloud, throwaway dir)** — set `CLOUDSDK_CONFIG=$(mktemp -d)` + `GCTX_CONFIG_DIR=$(mktemp -d)`; run real `gcloud config create/set/list` offline; assert config + sidecar + exactly one `.envrc` block. The `GCTX_GCLOUD` stub is a *delegating* fake: passes `config *` through to real gcloud, fakes only network calls (`projects describe`, `print-access-token`).
3. **E2E (opt-in, real project, manual)** — `gctx new gctx-smoke --project $REAL … && gctx doctor gctx-smoke`; teardown: `gctx rm gctx-smoke` + delete temp `.envrc`. Reversible.

`gcloud.rs::run()` is the only impurity → override with `GCTX_GCLOUD` to inject a stub binary. Everything above it is deterministic.

## Install

- `cargo build --release`; copy `target/release/gctx` → `~/.local/bin/gctx`.
- `mkdir -p ~/.config/gctx ~/.config/gcloud/adc`.
- Source `~/.config/zsh/gctx.zsh` from zshrc.
- Ensure `~/.local/bin` on PATH; direnv hook already present.
- Monorepo migration is manual: run `gctx emit` there once; delete the legacy hand-written gcloud lines in the same edit.

## Deliverables

1. Rust crate `gctx` (binary) at `lab/gctx/`.
2. `gctx.zsh` — the `use` shell function.
3. `man` page `gctx.1`, generated from the clap command via `clap_mangen` (stays in sync with `--help`), installed to `~/.local/share/man/man1/`.
4. Docs: `README.md` (the manual — install + quickstart + overview), `docs/concepts.md`, `docs/troubleshooting.md`, `docs/adr/`.

## Docs

Repo is public, notes are private → the repo is the single manual. `README.md` carries it (install + quickstart + overview); `docs/` + `man` are the OSS-standard homes for the deep material; the vault keeps only a pointer note.

- **`gctx help` / `--help`** — clap-generated, per subcommand.
- **`man gctx`** — `gctx.1` (roff) generated by `clap_mangen` at build time from the same clap `Command`, so help and man never drift. A `gctx man` subcommand prints/installs it.
- **`docs/concepts.md`** — durable explainers, mirrored from the `gcp-learning` notes so the repo is self-contained:
  - the three dials (`gcloud auth login` vs ADC vs configurations) + `CLOUDSDK_ACTIVE_CONFIG_NAME`.
  - ADC search order and the three ways to fill it (user / key / impersonated) — which can sign.
  - service-account impersonation: why, the Token Creator role, the impersonated-ADC file.
  - the env-var contract table (which names gcloud / Terraform / client libs read).
  - `use` × direnv precedence: the verified reload rule — which wins, until when.
- **`docs/troubleshooting.md`** — failure catalogue, each symptom → cause → fix:
  - `Cannot sign data without client_email` → ADC is `authorized_user` → impersonate or key.
  - `PERMISSION_DENIED iam.serviceAccountKeys.create` → org blocks keys → use impersonation.
  - impersonation token denied → missing `roles/iam.serviceAccountTokenCreator` on the SA.
  - `gcloud` works but app is denied → dial 2 (ADC) ≠ dial 3 (config); `.envrc` didn't set `GOOGLE_APPLICATION_CREDENTIALS`.
  - direnv not applying → `direnv allow` not run after `gctx emit`.
  - `.envrc` block drift → re-run `gctx emit`; `gctx doctor` flags it.

## Build order

1. **Repo + git via `gh` (first step)** — from `lab/gctx/` (already has `mise.toml`, `plan.md`): add `.gitignore` (`/target`), `git init`, `gh repo create gctx --private --source=. --remote=origin --push`.
2. `cargo init --name gctx`; commit scaffold bare — each dep arrives with the first slice that uses it (serde/serde_json → sidecar + adc, clap/clap_mangen → CLI, anyhow → orchestration). Tooling mirrors agup: lefthook pre-commit (`cargo fmt --check` · `clippy -D warnings` · `cargo test` — clippy subsumes typecheck), mise pins rust + lefthook, GH Actions CI runs the same checks on PR.
3. TDD `envrc.rs` (env_for + render_block + idempotent upsert) — red → green first.
4. `adc.rs` (impersonated JSON) + `sidecar.rs` (serde) — unit-tested.
5. `gcloud.rs` typed ops over private `run()` (`GCTX_GCLOUD` override) + `context.rs` orchestration.
6. `main.rs` clap CLI + `clap_mangen`; `gctx.zsh` shim.
7. Integration tests (`CLOUDSDK_CONFIG` tmpdir + stub) + docs (`concepts.md`, `troubleshooting.md`, man).
8. Install: `cargo build --release` → `~/.local/bin`; source zsh shim; install man page.

## Unresolved

1. direnv reload semantics unverified — when exactly does direnv re-assert the block over a manual `gctx use` (cd? every prompt? `.envrc` mtime)? Verify before writing the `concepts.md` precedence rule.

Resolved 2026-07-06 (interview): `eject`→`emit` · `new` fails-if-exists + `--reuse` · `rm` in v1 · missing sidecar ⇒ implicit user · drop `GOOGLE_PROJECT_NUMBER` · constructed ADC confirmed (ADR 0001) · manual per-repo migration · README is the manual · `new` runs base ADC login when missing · crate lives in its own repo (`snhasani/gctx`).
