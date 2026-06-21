# PV Recipe: Terraform Providers

Terraform providers are the canonical PV target: the provider claims an effect,
records it in `terraform.tfstate` (its self-report), and the real effect lands
in some external system (the last-mile truth). PV is the diff between the two.

## Generate → Operate → Last-mile

### 1. Generate inputs
Write a small `main.tf` that exercises the resources under test with realistic,
varied attributes — not just one happy-path resource, but the optional fields,
ownership/identity variations, and defaults that tend to harbor bugs. Add
`output` blocks for any value a later phase must assert on (IDs, serials,
computed flags); the harness reads them with `terraform output -raw <name>`.

### 2. Point at the locally built provider
You're verifying *your* provider build, not a published one. Use dev overrides so
`terraform` runs your local binary without `init` pulling from a registry:

```hcl
# ~/.terraformrc
provider_installation {
  dev_overrides { "hcf/terraform-z/ibm-zosmf" = "/Users/you/.terraform.d/plugins/.../darwin_arm64" }
  direct {}
}
```

With dev overrides, **skip `terraform init`** for your provider (init will warn).
If the config *also* needs a real registry provider (e.g. `hashicorp/tls` or
`hashicorp/vault`), lock just that one — see the Vault recipe for the
init-in-a-temp-dir trick that avoids dev-override conflicts.

### 3. Operate as a black box
Drive the real `terraform` binary:

```bash
terraform apply -auto-approve -var='version=1' 2>&1 | tee -a "$LOG"
```

### 4. Capture self-report (expected)
Parse `terraform.tfstate` with jq to extract what the provider *claims* it
created. This is your expected set — never your proof.

```bash
jq -c '.resources[] | select(.type=="myprov_thing") | .instances[].attributes
       | {id, label, owner, status, not_after}' terraform.tfstate
```

### 5. Last-mile verify (actual)
Branch by what the provider targets. Read the real backend through its own CLI:

| Provider targets | Last-mile reader |
|---|---|
| AWS | `aws <svc> describe-*/get-*` |
| GCP | `gcloud ... describe` |
| Azure | `az ... show` |
| IBM z/OS RACF | SSH + `tsocmd "RACDCERT ... LIST"` — see `ibm-zos-racf.md` |
| Vault | `vault read/kv get/list` — see `vault-and-secrets.md` |
| Kubernetes | `kubectl get -o yaml` |
| Okta | Okta REST API / CLI — see `okta-and-saas-apis.md` |

Compare each captured attribute against what the backend actually holds.

## The lifecycle phase pattern

Create-only verification misses most provider bugs. Run these phases (see
`harness-conventions.md` for the runner skeleton):

1. **apply (v=1)** — create; last-mile confirm every attribute landed.
2. **plan (idempotency)** — `terraform plan -detailed-exitcode` with unchanged
   inputs. Exit `0` = clean (good); `2` = drift pending — a bug. Drift after a
   no-op apply usually means `Read()` is mutating state or computing values
   non-deterministically.
3. **apply (v=2 / changed input)** — update/renew; last-mile confirm the backend
   changed to match, and re-check idempotency afterward.
4. **destroy** — tear down; last-mile confirm the backend is actually clean.
   **Check for orphans/ghosts** — resources the provider forgot to delete are a
   classic, dangerous bug a passing destroy can hide.

Capture cross-phase values (e.g. the post-apply serial) to dotfiles so later
phases can assert against them.

## Gotchas worth checking explicitly

- **`Read()` purity.** A correct `Read` reports drift; it must never *cause*
  drift by mutating the backend or rerolling computed values. The idempotency
  phase is precisely this test — if `plan` is dirty after a clean apply, `Read`
  is suspect.
- **Ownership / handoff transitions.** Resources that hand a backing object
  between two resources (e.g. a request → certificate handoff) can legitimately
  show a one-time post-apply diff. Verify it converges (a second apply absorbs
  it, then plan is clean) rather than oscillating forever.
- **Required backend permissions.** A first run on a fresh environment often
  fails on a missing permission/profile the provider needs. Capture the exact
  error code (e.g. RACF `IRRD101I`) in FINDINGS — it's an actionable docs/setup
  gap for field teams, not just a local hiccup.
- **Format mismatches at the boundary.** Values can be semantically correct but
  syntactically off (a non-standard PEM header, a date rendered differently).
  Flag as WARN, note which downstream consumers accept vs. reject it (test both
  if you can), and recommend the fix layer (usually the SDK, so every caller
  benefits).
- **Determinism of computed values.** If the backend assigns IDs/serials
  non-deterministically, you can assert "changed on renewal"; if deterministic,
  assert "unchanged" — know which before writing the check.
