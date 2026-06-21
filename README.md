# Parallel Verification (PV) — Agent Skill

An AI agent skill that builds **out-of-band, last-mile integration harnesses** — checks that prove a system *actually* does what it claims by driving its real public interface and then confirming the effects against the real backend through an **independent path the code author cannot fake**.

> Unit tests and acceptance tests run inside the system's own worldview — they trust its mocks, its in-memory state, and its own assertions. A provider can return a state file saying a certificate exists, an API can return `201 Created`, and every test can pass while nothing real ever landed in the backend. **PV is how you catch that.**

## The core idea

PV rests on a **two-observer principle**:

- The **self-report** — what the system claims it did (API response, `terraform.tfstate`, a return value).
- The **ground truth** — what an *independent* CLI sees in the real system of record (`psql`, `RACDCERT`, the AWS CLI, the `vault` CLI).

**PV is the diff between those two observers. Any gap is a finding.** The code author can make the code lie; they cannot make `psql` lie about what is actually stored. The harness is a standalone, re-runnable executable that talks to real systems through real tools — so it cannot be talked out of the truth. **The harness *is* the firewall against self-deception.**

## What This Skill Does

When a user asks to "verify this end-to-end", "prove it really works", "did it actually write to the database/state/backend", or "build a verification harness", the agent:

1. **Asks where** to put the harness (defaults to `./scratchpads/`) and scopes an **atomic, self-contained** test directory.
2. **Generates realistic inputs** that exercise the real surface area — not just the happy path.
3. **Operates the system as a black box** through its real interface (`terraform apply`, `curl`, the CLI).
4. **Captures the self-report** (`terraform.tfstate`, API responses) as the *expected* values.
5. **Last-mile verifies** against ground truth through an independent reader, comparing field by field.
6. **Probes the full lifecycle** — idempotency, update/renewal, teardown + orphan checks, failure/rollback.
7. **Delivers a reusable harness + a `FINDINGS.md` report** with evidence, severities, and remediation.

## What You Get

Two artifacts, always:

- **A reusable harness** — executable scripts that generate inputs, drive the system, last-mile-verify, and exit non-zero on any mismatch. Kept in the repo, re-runnable by a human or CI. It is the proof, kept.
- **A findings report** (`FINDINGS.md`) — the verdict, an evidence table citing the real serials/rows/CLI output, lifecycle trajectory, and any discoveries that go beyond pass/fail.

## Supported Backends

The core method is backend-agnostic; only the last-mile step changes mechanics. Each backend has a dedicated recipe in `references/`.

| Backend | System of record | Independent last-mile reader | Recipe |
|---|---|---|---|
| REST/gRPC API → database | Postgres / MySQL / Mongo / Redis | `psql`, `mysql`, `mongosh`, `redis-cli` | `rest-api-database.md` |
| Terraform provider | AWS / GCP / Azure / K8s / … | `aws`, `gcloud`, `az`, `kubectl` | `terraform-providers.md` |
| Vault engine | Vault path | `vault read` / `kv get` / `list` | `vault-and-secrets.md` |
| Okta / SaaS API | Okta org objects | Okta REST API / CLI | `okta-and-saas-apis.md` |
| IBM z/OS RACF | RACF profile / keyring | SSH + `tsocmd "RACDCERT … LIST"` | `ibm-zos-racf.md` |

Backend not listed? The skill applies the core loop, uses the **last-mile decision framework** to choose a path (asking you when the source of truth is ambiguous), and follows the shared harness conventions.

## What's Inside

```
parallel-verification/
├── SKILL.md                         # The universal PV method + decision flow
└── references/
    ├── harness-conventions.md       # Reusable bash scaffolding (logging, traps, exit codes…)
    ├── last-mile-catalog.md         # Decision framework + backend→verification-path catalog
    ├── terraform-providers.md       # dev-overrides, tfstate parsing, plan -detailed-exitcode
    ├── rest-api-database.md         # curl black-box + direct SELECT verification
    ├── vault-and-secrets.md         # vault CLI round-trip, dockerized dev Vault
    ├── okta-and-saas-apis.md        # platform-API read-back, relationships & lifecycle
    └── ibm-zos-racf.md              # SSH + TSO RACDCERT, RACF message codes
```

`SKILL.md` is the load-bearing piece (always loaded); reference files load on demand based on the backend, keeping token usage lean.

## Installation

### skills.sh

```bash
skills install parallel-verification
```

### Claude Code

```bash
# System-wide (all projects)
git clone https://github.com/AdamTylerLynch/parallel-verification-skill.git
ln -s "$(pwd)/parallel-verification-skill" ~/.claude/skills/parallel-verification

# Or copy instead of symlink
cp -r parallel-verification-skill ~/.claude/skills/parallel-verification
```

### IBM Bob

```bash
ln -s "$(pwd)/parallel-verification-skill" ~/.bob/skills/parallel-verification
```

### Other Agent Ecosystems

The skill is pure markdown with no code dependencies. Include `SKILL.md` as system context; the agent loads the relevant `references/` file based on the backend under test. For context-constrained systems, include only `SKILL.md` initially — it points to the specific reference file to read.

## Usage Examples

```
> I just finished a REST endpoint that writes orders to Postgres. Verify
  end-to-end that it actually persists correctly — don't trust the API's
  own responses.

> Build a verification harness for my Terraform provider that proves the
  resources really land in AWS, and that destroy leaves no orphans.

> My Vault PKI engine signs CSRs. Prove the issued certs are actually
  readable back at the path, and that revocation really takes effect.

> Acceptance tests pass but I don't trust it — do a last-mile check that
  the RACF certificate and keyring connection exist on z/OS.

> This Okta integration claims it suspended the user. Verify against
  Okta's own API that the status actually changed.
```

## How It Works — The Core Loop

```
 ┌─────────────┐   generate    ┌───────────────┐   black-box   ┌──────────────┐
 │  Realistic  │──────────────▶│  System under │──────────────▶│ Self-report  │  ← "expected"
 │   inputs    │    inputs     │     test      │   operate     │ (tfstate/API)│
 └─────────────┘               └───────────────┘               └──────┬───────┘
                                                                       │ compare
                                       independent path                ▼  field-by-field
 ┌─────────────┐   last-mile   ┌───────────────┐               ┌──────────────┐
 │   Ground    │◀──────────────│  Real system  │◀──────────────│   Verdict +  │  ← "actual"
 │    truth    │   read-back   │  of record    │               │  FINDINGS.md │
 └─────────────┘               └───────────────┘               └──────────────┘
```

Then probe the rest of the lifecycle — **idempotency** (re-run = no drift), **update/renewal**, **teardown** (no orphans), and **failure/rollback** — because that's where the real bugs hide.

## Why "Parallel"

1. It runs *alongside and outside* the normal test pyramid — a parallel track doing full integration verification the unit/acc tests can't.
2. The harness executes **independently of the context that wrote the code**. A fresh session, a human, or CI can run it cold and get the truth.

## License

MIT
