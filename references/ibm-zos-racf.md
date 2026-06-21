# PV Recipe: IBM z/OS RACF (over SSH + TSO)

For providers/tools that manage RACF security objects on z/OS — certificates,
keyrings, profiles, users — the system of record is RACF itself. There's no REST
truth to read; you SSH to the mainframe and run TSO commands (`RACDCERT`,
`LISTUSER`, `RLIST`) as an operator would. That SSH+TSO path is fully
independent of the provider's z/OSMF SDK, which is exactly what makes it a valid
last-mile.

## Connection: SSH + tsocmd

```bash
run_tso() {
  ssh -i "$SSH_KEY" -p "${SSH_PORT:-22}" \
      -o StrictHostKeyChecking=accept-new -o ConnectTimeout=10 \
      "${SSH_USER}@${SSH_HOST}" "tsocmd \"$1\"" 2>&1
}
# Connectivity probe before doing work:
run_tso "TIME" >/dev/null 2>&1 && log_pass "z/OS reachable" || { log_fail "SSH/TSO unreachable"; exit 1; }
```

Resolve `SSH_HOST` from the IBM credentials profile (`~/.ibmz/credentials`,
`zosmf_host`) when not given explicitly — see the credential-resolution pattern
in `harness-conventions.md`. Default user `ibmuser`, port `22`, both overridable.

## Last-mile commands by object type

Parse `terraform.tfstate` for the expected attributes, then confirm against RACF:

```bash
# Certificate (type drives the command)
case "$type" in
  CERTAUTH) cmd="RACDCERT CERTAUTH LIST(LABEL('$label'))" ;;
  USER)     cmd="RACDCERT ID($owner) LIST(LABEL('$label'))" ;;
  SITE)     cmd="RACDCERT SITE LIST(LABEL('$label'))" ;;
esac
out="$(run_tso "$cmd")"

# Keyring and its connected certs
out="$(run_tso "RACDCERT ID($owner) LISTRING($ring_name)")"

# General resource profile
out="$(run_tso "RLIST FACILITY IRR.DIGTCERT.GENREQ AUTHUSER")"
```

Then grep the output for each expected field — label, `Status: TRUST/NOTRUST`,
`End Date`, key usage, subject DN (`CN=...`), alt names (`dNSName=`,
`iPAddress=`), and ring associations (`Ring Owner:`). Compare to the values
parsed from state; `PASS` on match, `WARN` on cosmetic format differences (RACF
renders dates as `YYYY/MM/DD`), `FAIL` on a real mismatch or absence.

## RACF message codes — know what failure looks like

Grep for these to distinguish "really absent" from "present":

| Code | Meaning |
|---|---|
| `IRRD107I` / "no digital certificate" | certificate not found |
| `IRRD119I` / "no ring" | keyring not found |
| `IRRD101I` | **not authorized** to issue the RACDCERT command — often a missing FACILITY profile, not a true negative |
| `ICH408I` | access denied to a resource |

`IRRD101I` is a frequent first-run failure on a host where a needed FACILITY
profile (e.g. `IRR.DIGTCERT.GENREQ`, `IRR.DIGTCERT.ROLLOVER`) was never
`RDEFINE`d. Record it in FINDINGS as a setup/docs gap — field teams upgrading
will hit the same wall, and the fix is a documented `RDEFINE`/`PERMIT`/
`SETROPTS RACLIST(FACILITY) REFRESH` plus a setup-script update.

## Verify the lifecycle

- **Create** — every certificate attribute, every keyring, and each
  certificate↔keyring **connection** (usage `PERSONAL`/`CERTAUTH`/`SITE`,
  `DEFAULT YES/NO`, cert owner). Connections are where subtle bugs hide.
- **Idempotency** — after a clean apply, `terraform plan -detailed-exitcode`
  must return `0`. RACF-backed providers are prone to `Read()`-causes-drift bugs
  (e.g. a Read that re-issues `GENREQ` or rerolls a generation token); the plan
  check catches them.
- **Renewal** — bump the version; confirm RACF shows the re-finalized cert.
- **Destroy** — confirm every object is gone (no ghost certs, rings, or
  connections). Re-`LIST` and expect the not-found codes above.

## Gotchas

- **Quoting.** `RACDCERT` labels are case-sensitive and often contain dots and
  hyphens; keep the exact quoting (`LABEL('TF-E2E-ROOTCA')`) through the SSH
  layer — double-quote the `tsocmd` argument and single-quote the label inside.
- **Always destroy on exit.** RACF orphans (a leftover keyring or cert) corrupt
  the next run and linger on a shared system — use the cleanup trap.
- **Output is fixed-format text, not JSON.** Parse defensively with `grep`/`sed`
  and tolerate spacing; log the raw output under `--verbose` as evidence.
- **Permissions for the test user.** The SSH/TSO user needs authority to issue
  the LIST commands; a `IRRD101I` on *read* means your last-mile user is
  under-permissioned, distinct from the provider's own permission needs.
