# PV Recipe: Vault Engines & Secret Stores

A Vault secrets engine (PKI, KV, database, transit, an SDK-backed custom engine)
claims to store or issue something. The self-report is the API/Terraform
response; the truth is what `vault read`/`vault list` shows at the path. PV
drives the engine through its real interface, then reads the path back with the
independent `vault` CLI.

## Generate → Operate → Last-mile

### 1. Generate inputs
Decide how the engine is exercised:
- **Directly** via the `vault` CLI / API (writing secrets, issuing certs,
  configuring roles), or
- **Through the Terraform Vault provider** (`vault_pki_secret_backend_*`,
  `vault_kv_secret_v2`, etc.) — combine this with the Terraform recipe.

Write realistic configs/role parameters and outputs for any issued serial,
lease ID, or path you'll assert on.

### 2. Stand up a real Vault (for local PV)
A dev-mode container is the standard isolated backend — real Vault, disposable:

```bash
docker run -d --name "$CONTAINER" -p 8200:8200 --cap-add=IPC_LOCK \
  -e VAULT_DEV_ROOT_TOKEN_ID=root -e VAULT_DEV_LISTEN_ADDRESS=0.0.0.0:8200 \
  hashicorp/vault:latest
export VAULT_ADDR=http://127.0.0.1:8200 VAULT_TOKEN=root
./bootstrap-vault.sh        # enable engines, configure PKI roots/roles, etc.
```

**Always tear the container down in a cleanup trap** (see
`harness-conventions.md`) — and run `terraform destroy` *before* killing the
container so partial state on a downstream backend (e.g. RACF) doesn't leak.

### Mixing dev-overrides with a registry provider
If your config uses both a locally-built provider (dev override) and
`hashicorp/vault` from the registry, `terraform init` conflicts with the dev
override. Init the registry provider in a throwaway dir, then copy its cache in:

```bash
EMPTY_TFRC="$(mktemp)"; printf 'provider_installation { direct {} }\n' > "$EMPTY_TFRC"
TMP="$(mktemp -d)"; cat > "$TMP/versions.tf" <<'EOF'
terraform { required_providers { vault = { source = "hashicorp/vault", version = "~> 4.0" } } }
EOF
(cd "$TMP" && TF_CLI_CONFIG_FILE="$EMPTY_TFRC" terraform init) 
rm -rf ./.terraform; cp -R "$TMP/.terraform" ./.terraform
cp "$TMP/.terraform.lock.hcl" ./.terraform.lock.hcl
```

### 3. Operate, then last-mile verify with the `vault` CLI
The `vault` CLI is your independent reader — it talks straight to Vault's API,
not through your engine's Go code.

```bash
# PKI: confirm an issued cert really exists at its serial
vault read -format=json "pki/cert/$SERIAL" | jq -e '.data.certificate' >/dev/null \
  && log_pass "cert $SERIAL present in Vault" || log_fail "cert $SERIAL missing"

# KV v2: confirm a written secret and its value
val="$(vault kv get -field=password secret/app/db 2>/dev/null)"
[ "$val" = "$expected" ] && log_pass "secret value matches" || log_fail "secret mismatch"

# List a path to confirm membership / count
vault list -format=json pki/certs | jq 'length'
```

## Things to verify beyond "it exists"

- **Round-trip integrity** — what you wrote is what reads back, byte-for-byte
  for secrets, attribute-for-attribute for issued certs (CN, SANs, TTL, issuer).
- **Lease & revocation** — if the engine issues leased credentials, verify the
  lease exists (`vault lease lookup`), then revoke and confirm it's gone /
  unusable. A revoke that the engine reports but doesn't enforce is a real bug.
- **Renewal / rotation** — re-issue or rotate; confirm Vault shows a fresh
  serial/version (Vault PKI serials are non-deterministic, so "changed" is the
  right assertion, unlike a deterministic local CA).
- **Cross-system finalize** — when a Vault-issued artifact is re-imported
  elsewhere (e.g. signed cert back into RACF), last-mile-verify *both* ends:
  present in Vault, and present/active in the destination.
- **Tolerance quirks** — Vault PKI accepts some non-standard inputs other
  signers reject (e.g. a `NEW CERTIFICATE REQUEST` PEM header). If you observe
  that, record it in FINDINGS with the precise compatibility boundary — it's
  exactly the kind of cross-tool discovery PV is meant to surface.

## Gotchas

- Set `VAULT_ADDR`/`VAULT_TOKEN` in the harness env; a wrong address silently
  reads a different Vault.
- Dev-mode Vault is in-memory — restarting the container wipes it; don't split
  verification across a restart.
- For non-dev Vault, last-mile reads may need a token with a policy granting
  read on the path; resolve it via the credential conventions and fail clearly
  if it's missing.
