# Harness Conventions

Reusable bash scaffolding for a PV harness. These patterns are battle-tested —
match them so your harness behaves predictably and a reviewer recognizes the
shape immediately. Adapt freely, but keep the invariants: **exit non-zero on
any failure, always clean up, always capture evidence.**

## Table of contents
- [Script skeleton](#script-skeleton)
- [PASS/FAIL/WARN logging with counters](#passfailwarn-logging-with-counters)
- [Argument parsing](#argument-parsing)
- [Credential resolution](#credential-resolution)
- [Prerequisite & connectivity checks](#prerequisite--connectivity-checks)
- [Parsing self-reported state with jq](#parsing-self-reported-state-with-jq)
- [The compare step](#the-compare-step)
- [Phased runners](#phased-runners)
- [Cross-phase fact capture](#cross-phase-fact-capture)
- [Cleanup traps](#cleanup-traps)
- [Exit codes & summary](#exit-codes--summary)

## Script skeleton

```bash
#!/usr/bin/env bash
# PV harness for <feature>. Generates inputs, drives the system as a black box,
# and last-mile-verifies the effect against <real backend>.
set -euo pipefail
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
cd "$SCRIPT_DIR"
LOG="$SCRIPT_DIR/run.log"; : > "$LOG"
```

`set -euo pipefail` is non-negotiable: it turns silent failures into loud ones.
Tee important output to a log so the evidence survives the run.

## PASS/FAIL/WARN logging with counters

The whole harness is a stream of checks. Give each a verdict and keep a tally.

```bash
PASS_COUNT=0; FAIL_COUNT=0; WARN_COUNT=0
STRICT=${STRICT:-false}

log_step() { printf '\n\033[1;36m==> %s\033[0m\n' "$*" | tee -a "$LOG"; }
log_pass() { printf '    \033[0;32m[PASS]\033[0m %s\n' "$*" | tee -a "$LOG"; PASS_COUNT=$((PASS_COUNT+1)); }
log_warn() { printf '    \033[0;33m[WARN]\033[0m %s\n' "$*" | tee -a "$LOG"; WARN_COUNT=$((WARN_COUNT+1)); }
log_info() { printf '    \033[0;90m[INFO]\033[0m %s\n' "$*" | tee -a "$LOG"; }
log_fail() {
  printf '    \033[0;31m[FAIL]\033[0m %s\n' "$*" | tee -a "$LOG"
  FAIL_COUNT=$((FAIL_COUNT+1))
  [ "$STRICT" = "true" ] && exit 1 || true
}
```

Use `WARN` (not `FAIL`) for differences that are probably benign — a date format
that differs cosmetically, an attribute the backend renders differently than the
state stores it. WARN keeps the run green but flags the thing for human eyes.
`--strict` mode turns the first FAIL into an immediate exit, which CI often wants.

## Argument parsing

Provide sensible defaults so the common case is zero-config, but make everything
overridable. Typical flags from the z/OSMF harness:

```bash
SKIP_APPLY=false; TF_DIR="."; STATE_FILE="terraform.tfstate"; VERBOSE=false
while [ $# -gt 0 ]; do
  case "$1" in
    --skip-apply) SKIP_APPLY=true; shift ;;
    --tf-dir)     TF_DIR="$2"; shift 2 ;;
    --strict)     STRICT=true; shift ;;
    --verbose)    VERBOSE=true; shift ;;
    -h|--help)    usage; exit 0 ;;
    *) echo "Unknown option: $1" >&2; usage; exit 1 ;;
  esac
done
```

A `--skip-apply` / `--skip-setup` flag is valuable: it lets you re-run only the
verification half against an already-provisioned environment while iterating.

## Credential resolution

Resolve in priority order — **explicit flags > credentials file > environment
variables** — and fail with an actionable message when a required credential is
missing. Never hardcode secrets; read them from the environment or a creds file.

```bash
resolve_host() {
  [ -n "${HOST:-}" ] && return                 # 1. flag already set it
  HOST="$(yq -r ".profiles.$PROFILE.host // \"\"" "$CREDS_FILE" 2>/dev/null || true)"  # 2. creds file
  [ -z "$HOST" ] && HOST="${BACKEND_HOST:-}"    # 3. env var
  [ -z "$HOST" ] && { echo "Error: host not set (use --host, creds file, or BACKEND_HOST)" >&2; exit 1; }
}
```

## Prerequisite & connectivity checks

Fail fast and clearly *before* doing work. Check that every CLI you depend on
exists, then prove you can reach the backend.

```bash
for bin in jq terraform ssh; do
  command -v "$bin" >/dev/null || { echo "Error: $bin required but not installed" >&2; exit 1; }
done

log_step "Testing connectivity"
if probe_backend >/dev/null 2>&1; then        # e.g. a cheap read / 'TIME' / SELECT 1
  log_pass "Backend reachable"
else
  log_fail "Cannot reach backend — check host, credentials, network"; exit 1
fi
```

A connectivity check that prints troubleshooting hints on failure (which key,
which host, the exact manual command to try) saves enormous debugging time.

## Parsing self-reported state with jq

Extract the system's claimed "expected" values into structured records. For
Terraform, parse `terraform.tfstate`; for a REST API, parse the response body.

```bash
parse_resources() {
  jq -c '.resources[]
    | select(.type == "myprovider_thing")
    | .instances[].attributes
    | {id, name, owner, status}' "$TF_DIR/$STATE_FILE"
}

while IFS= read -r rec; do
  [ -z "$rec" ] && continue
  validate_thing "$rec"
done < <(parse_resources)
```

Emitting compact JSON (`-c`) per record and looping keeps each validation
function simple: it receives one record and checks it.

## The compare step

The verification function's job: pull the expected value from the record, query
the backend for the actual value via the independent path, and compare.

```bash
validate_thing() {
  local rec="$1"
  local id name expected_status actual_status
  id="$(jq -r '.id' <<<"$rec")"
  name="$(jq -r '.name' <<<"$rec")"
  expected_status="$(jq -r '.status' <<<"$rec")"

  log_step "Thing: $name ($id)"
  local out; out="$(query_backend_for "$id" 2>&1)" || true   # independent path

  if grep -qi "not found\|does not exist" <<<"$out"; then
    log_fail "Thing $id absent in backend — system claimed it exists"; return
  fi
  log_pass "Thing exists in backend"

  actual_status="$(extract_status <<<"$out")"
  if [ "$actual_status" = "$expected_status" ]; then
    log_pass "status=$expected_status"
  else
    log_fail "status mismatch: state says '$expected_status', backend says '$actual_status'"
  fi
  [ "$VERBOSE" = "true" ] && { log_info "backend output:"; sed 's/^/      /' <<<"$out"; }
}
```

Always log the raw backend output under `--verbose` — it is the evidence.

## Phased runners

For lifecycle verification, structure the run as numbered phases with a filter so
you can run one in isolation while iterating.

```bash
PHASE_FILTER="${1:-all}"; [ "$PHASE_FILTER" = "--phase" ] && PHASE_FILTER="$2"
run_phase() { [ "$PHASE_FILTER" = "all" ] || [ "$PHASE_FILTER" = "$1" ]; }

if run_phase 1; then log_step "Phase 1: create";  : ; fi
if run_phase 2; then log_step "Phase 2: idempotency (expect no drift)"; : ; fi
if run_phase 3; then log_step "Phase 3: update/renew"; : ; fi
if run_phase 4; then log_step "Phase 4: destroy + orphan check"; : ; fi
```

Typical phases: **create → idempotency → update/renew → destroy**. The
idempotency and destroy phases catch the bugs create-only testing misses.

## Cross-phase fact capture

When a later phase must assert against a value produced earlier, persist it to a
dotfile so phases stay independently runnable.

```bash
printf '%s\n' "$FINAL_SERIAL" > .final-serial      # phase 1 writes
[ "$(cat .final-serial)" = "$NOW_SERIAL" ] || log_fail "serial drift"   # phase 3 reads
```

## Cleanup traps

The most important safety property: **a failed run must still tear down what it
created.** A trap guarantees it.

```bash
cleanup() {
  local rc=$?
  log_step "Cleanup: tearing down created resources (best-effort)"
  terraform destroy -auto-approve >/dev/null 2>&1 || true
  docker rm -f "$CONTAINER_NAME" >/dev/null 2>&1 || true
  exit $rc
}
trap cleanup EXIT INT TERM
```

Orphaned resources from a half-run harness corrupt the next run and pollute the
real environment — the trap is what keeps PV safe to run repeatedly.

## Exit codes & summary

End with a summary block and an exit code that *is* the verdict.

```bash
log_step "Summary"
printf '  Passed:  %s\n  Warnings: %s\n  Failed:  %s\n' "$PASS_COUNT" "$WARN_COUNT" "$FAIL_COUNT" | tee -a "$LOG"
if [ "$FAIL_COUNT" -gt 0 ]; then
  printf '\n  \033[0;31m✗ Verification FAILED\033[0m\n' | tee -a "$LOG"; exit 1
fi
printf '\n  \033[0;32m✓ All verifications passed\033[0m\n' | tee -a "$LOG"; exit 0
```

For Terraform idempotency specifically, lean on `terraform plan
-detailed-exitcode`: exit `0` = no drift (good), `2` = drift pending (a bug to
report), anything else = the plan itself errored.
