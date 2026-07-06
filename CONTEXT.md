# gctx

Bootstraps and switches complete GCP working state (account · project · region · zone · credentials) per shell and per repo, across many orgs/projects.

## Language

**Context**:
A named, complete GCP working state — gcloud configuration + sidecar (+ impersonated ADC file when mode is impersonate). 1:1 with a gcloud configuration name.
_Avoid_: profile, environment

**Configuration**:
gcloud's own named property set (account, project, region, zone) at `config_<name>`; dial 3. gctx reads it, never duplicates it.
_Avoid_: config (ambiguous with gctx's own state)

**Sidecar**:
gctx's per-context JSON holding the one bit gcloud can't store: `{adcMode, serviceAccount?}`.

**Wallet**:
The set of accounts logged in via `gcloud auth login` (`gcloud auth list`); dial 1.
_Avoid_: credentials store

**ADC**:
Application Default Credentials — the credential file client libraries and Terraform read; dial 2. Independent of the wallet.

**Base ADC**:
The well-known user ADC produced once by `gcloud auth application-default login`; source credential for impersonation.

**Impersonated ADC**:
Per-context `impersonated_service_account` JSON wrapping the base ADC; signs via IAM Credentials, no key file.

**ADC mode**:
`user` | `impersonate` — which credential a context points `GOOGLE_APPLICATION_CREDENTIALS` at.

**Dial**:
One of the three independent auth/config surfaces: 1 = wallet, 2 = ADC, 3 = configuration. A context is healthy only when all three agree.

**Block**:
The marker-bounded, gctx-owned region of a repo's `.envrc`; a regenerable cache of context state. Lines outside it are never touched.
_Avoid_: snippet, section
