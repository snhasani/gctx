# Constructed impersonated-ADC file

Status: accepted

gctx builds each impersonate-mode context's ADC file itself — serde-wrapping the base user ADC into the documented `impersonated_service_account` shape — instead of the official `gcloud auth application-default login --impersonate-service-account` flow. The official flow costs a browser round-trip per context; org policy blocks SA key downloads, so a constructed file is the only no-browser, no-key credential that can sign (via IAM Credentials `generateAccessToken`). The shape is documented and read by all Google auth libraries; `gctx doctor` verifies the credential can actually mint a token, so a shape regression surfaces immediately.
