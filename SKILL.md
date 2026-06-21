---
name: parallel-verification
version: 1.0.1
description: >-
  Build a Parallel Verification (PV) harness — an out-of-band, last-mile
  integration check that proves a system actually does what it claims by
  driving its real public interface and then confirming the effects against
  the real backend through an INDEPENDENT path the code author cannot fake.
  Use this whenever you are building or reviewing something whose correctness
  ultimately lands in an external system of record: a REST API that writes to a
  database, a Terraform provider, a Vault secrets engine, an Okta integration,
  an IBM z/OS RACF resource, a message queue, an object store, an IAM policy,
  etc. Trigger on phrases like "verify this end-to-end", "prove it really
  works", "last-mile verification", "out-of-band check", "did it actually write
  to the database / state / backend", "black-box integration test", "build a
  verification harness", or any time unit tests and acceptance tests pass but
  you still don't trust that the real-world effect happened. Reach for PV
  precisely when you want a check that does not trust the code's own assertions.
---

# Parallel Verification (PV)

## What this is, and why it matters

Unit tests and acceptance tests run *inside* the system's own worldview. They
trust the code's mocks, its in-memory state, and its own assertions about what
happened. That trust is exactly the problem: a provider can return a state file
saying a certificate exists, an API can return `201 Created`, and the code can
pass every test — while nothing real ever landed in the backend.

**Parallel Verification is the practice of confirming an effect from the
outside.** You operate the system through its real public interface as a black
box, capture what the system *claims* it did, and then go to the actual system
of record through a **separate, independent path** and check whether the effect
is really there. The verification path must not share code, libraries, or trust
with the thing under test — that independence is the entire point. The code
author (human or agent) can make the code lie; they cannot make `psql`,
`RACDCERT`, the AWS CLI, or the Vault CLI lie about what is actually stored.

It's called **Parallel** for two reasons:

1. It runs *alongside and outside* the normal test pyramid — a parallel track
   to unit/acceptance tests, doing full integration verification that those
   can't.
2. The deliverable is a **standalone harness** that executes independently of
   the context that wrote the code. A human or CI job can re-run it cold. Because
   it talks to real systems through real CLIs, it cannot be talked out of the
   truth. The harness *is* the firewall against self-deception.

The two-observer principle is the heart of it: the system's **self-report**
(its API response, its state file, its return value) and the backend's
**ground truth** (what an independent CLI sees in the real system of record)
must agree. PV is the diff between those two observers. Any gap is a finding.

## What you deliver

Two artifacts, always:

1. **A reusable harness** — executable scripts (typically bash) that generate
   inputs, drive the system, perform last-mile verification, and exit non-zero
   on any mismatch. It lives in the repo (e.g. under `scratchpads/<feature>-e2e/`
   or a `verify/` directory) so anyone can re-run it. It is not a throwaway; it
   is the proof, kept.
2. **A findings report** (`FINDINGS.md`) — the human-readable verdict: what was
   verified, the evidence, and any discoveries that go beyond pass/fail
   (severity, root cause, remediation). See the structure below.

## The core PV loop

Apply this loop regardless of backend. The backend only changes step 5's
mechanics; the shape is universal.

1. **Identify the contract.** What does the system claim to do, and where does
   the truth ultimately live? Name the real system of record (the Postgres row,
   the RACF profile, the Vault path, the Okta user, the S3 object). If the
   effect doesn't land *somewhere external and inspectable*, PV may not apply —
   say so.

2. **Generate realistic inputs.** Produce payloads/configs that exercise the
   real surface area — not just the happy path, but the fields and combinations
   that are easy to get wrong (types, owners, defaults, optional attributes).
   In the z/OSMF examples this was a `main.tf`; for a REST API it's a set of
   request bodies.

3. **Operate the system as a black box.** Drive it through its real public
   interface — `terraform apply`, `curl`, the CLI, the SDK's public API — never
   by reaching into its internals. If you're testing a Terraform provider, point
   at the locally built provider via dev overrides and run the actual
   `terraform` binary. Black-box discipline is what makes the result credible.

4. **Capture the system's self-report ("expected").** Record what the system
   says it did: the API response body, `terraform.tfstate`, the CLI's output,
   the returned IDs/serials. Parse it into structured values (jq is your
   friend) — these become the expectations you'll check against reality.

5. **Last-mile verify against ground truth ("actual").** Go to the real system
   of record through an independent path and read what's actually there.
   - REST API → Postgres: `psql` and `SELECT` the rows the API claims to have written.
   - Terraform provider for AWS: the `aws` CLI to describe the real resource.
   - Terraform provider for IBM z/OS RACF: SSH + `tsocmd "RACDCERT ... LIST"`.
   - Vault engine: the `vault` CLI to read/list the secret path.
   - Okta engine: Okta's REST API or CLI to fetch the user/group/app.

   **The last-mile method is not always obvious. When it isn't, ask the user**
   rather than guessing — they know their environment, credentials, and what
   "the truth" means here. See `references/last-mile-catalog.md` for the
   decision framework and a backend→method catalog.

6. **Compare field by field.** Diff expected against actual for every attribute
   that matters. Emit `PASS` / `FAIL` / `WARN` per check (WARN for things like a
   format mismatch that may be benign). A single comparison is worth a hundred
   assertions the code makes about itself.

7. **Probe beyond create.** The interesting bugs live in the rest of the
   lifecycle. Exercise and verify:
   - **Idempotency** — re-run with no input change; there must be no drift.
     For Terraform, `plan -detailed-exitcode` returning `2` means drift (a bug);
     `0` means clean.
   - **Update / renewal** — change an input; confirm the backend changed to match.
   - **Teardown** — destroy/delete; confirm the backend is actually clean, with
     no orphaned or ghost entries left behind.
   - **Failure & rollback** — where feasible, force a failure mid-operation and
     confirm the system doesn't leave partial garbage.

8. **Report.** Produce the machine summary (counts, exit code) and the
   `FINDINGS.md` writeup. Surface not just pass/fail but anything you learned —
   a missing permission, a non-standard output format, a confusing-but-correct
   behavior. Those discoveries are often the most valuable output.

## Choosing the last-mile method

This is the judgment-heavy step, so give it real thought. A good last-mile path
is **independent** (doesn't reuse the system-under-test's own code to read
back), **authoritative** (reads the actual system of record, not a cache or a
mirror), and **inspectable at the field level** (you can extract the specific
values to compare).

When the path is genuinely unclear — an unusual backend, an internal system,
ambiguous "truth", or missing credentials — **ask the user**. Offer the options
you can see and let them pick or correct you. Guessing a last-mile method and
verifying against the wrong source of truth is worse than asking: it produces a
confident green result that means nothing.

`references/last-mile-catalog.md` holds the framework plus a catalog of known
backend → verification-path mappings.

## Harness conventions

Don't reinvent the scaffolding each time. `references/harness-conventions.md`
distills the reusable bash patterns — colored `PASS/FAIL/WARN` logging with
counters, argument parsing, credential resolution order (flags > creds file >
env), prerequisite and connectivity checks, jq-based state parsing, phased
runners (`--phase N`), cross-phase fact capture, cleanup traps that always tear
down, and meaningful exit codes. Read it before writing a harness so your output
matches the proven shape and a reviewer recognizes it instantly.

Key invariants worth stating up front:
- **Exit non-zero on any FAIL.** The harness's exit code is its verdict; CI
  depends on it.
- **Always clean up.** Use a trap so a mid-run failure still tears down created
  resources — orphans poison the next run and the real environment.
- **Capture evidence.** Log the actual backend output you compared against, so a
  reader can see *why* a check passed or failed, not just that it did.

## The findings report (`FINDINGS.md`)

Structure it so a reviewer gets the verdict in five seconds and the depth on
demand. Model it on this shape (drawn from the z/OSMF CSR verification):

```markdown
# <Feature> Verification — Findings (<env/host>)

**Result: PASS|FAIL.** <One-paragraph verdict: what works, what doesn't, the
single most important takeaway.>

## What was verified
| Check | Result | Evidence |
|---|---|---|
| <claim checked> | PASS | <the actual backend value / serial / row that proves it> |

## State trajectory (if lifecycle was exercised)
<table tracking key values across apply → plan → update → destroy phases>

## New findings discovered during testing
### FINDING N — <title> (<SEVERITY>, <merge impact>)
<what you observed, root cause, and concrete remediation / action items>

## Recommendation
<ship / ship-after-fix / block, with the reasoning>

## Reproducing
<exact commands + prerequisites to re-run the harness>
```

The `Evidence` column matters most: cite the real serial, row, or CLI output you
saw. Evidence is what makes the report trustworthy rather than another assertion.

## Backend-specific recipes

Read the one that matches the system under test — each has the concrete
generate→operate→last-mile recipe and the gotchas for that backend:

- `references/terraform-providers.md` — any Terraform provider: dev overrides /
  local provider, `apply`, parsing `terraform.tfstate`, `plan -detailed-exitcode`
  for idempotency, the apply/plan/renew/destroy phase pattern, and how last-mile
  branches by what the provider targets (AWS, IBM z/OS, Vault, Okta…).
- `references/rest-api-database.md` — REST/gRPC APIs whose effects land in a
  database: payload generation, black-box driving with `curl`, capturing
  responses, and last-mile `SELECT`s via `psql`/`mysql`/`mongosh`, including the
  reverse check (delete via API → confirm gone in DB).
- `references/vault-and-secrets.md` — Vault engines and secret stores: operating
  via the `vault` CLI or the Terraform Vault provider, then reading back the path.
- `references/okta-and-saas-apis.md` — Okta and SaaS-API backends: driving via
  API/CLI and last-mile via the provider's own REST API or CLI.
- `references/ibm-zos-racf.md` — IBM z/OS RACF over SSH + TSO (`tsocmd`,
  `RACDCERT`), including the message codes that signal real failures.

If the backend isn't covered, apply the core loop, use
`references/last-mile-catalog.md` to choose a path (asking the user when
unsure), and follow `references/harness-conventions.md` for the scaffolding.
