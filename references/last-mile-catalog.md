# Last-Mile Verification: Decision Framework & Catalog

The last-mile step — going to the real system of record and reading the truth —
is the judgment-heavy part of PV. Choose well and the green result means
something; choose badly and you get a confident result that verifies nothing.

## What makes a good last-mile path

A trustworthy verification path is:

- **Independent** — it does not reuse the system-under-test's own code,
  client library, or SDK to read the value back. If your REST API and your
  verification both go through the same ORM, a bug in the ORM hides from both.
  Use a different tool: the raw `psql` client, the cloud's own CLI, the
  backend's native CLI.
- **Authoritative** — it reads the actual system of record, not a cache, a
  read replica that may lag, a search index, or the system's own status
  endpoint. Ask "if these two disagreed, which one is *the truth*?" and verify
  against that one.
- **Field-level inspectable** — you can extract the specific attributes you
  need to compare, not just a "200 OK". A path that only tells you "something
  exists" is weak; one that lets you read every attribute you set is strong.

## The decision procedure

1. **Name the system of record.** Where does the effect *actually* persist?
   (A Postgres table, a RACF profile, a Vault path, an Okta user object, an S3
   bucket, a Kafka topic, a DNS zone.)
2. **Find the native, independent reader for it.** Usually the backend's own
   CLI or a low-level client — the thing an operator would use to inspect it by
   hand, bypassing your application entirely.
3. **Confirm it's authoritative and not a mirror.** Beware status endpoints,
   caches, and replicas. Prefer strongly-consistent reads.
4. **Check you have credentials and access.** Last-mile often needs separate
   creds (SSH key, DB password, cloud profile, API token). Resolve them via the
   conventions in `harness-conventions.md`.
5. **If any of 1–4 is unclear, ASK THE USER.** This is not a failure — it's the
   correct move. The user knows their environment, what "the truth" means, and
   which credentials exist. Present the options you can see and let them confirm
   or redirect. A wrong last-mile source produces a green run that lies, which
   is worse than pausing to ask.

### How to ask well

When last-mile is ambiguous, offer concrete options rather than an open
question. For example:

> The provider writes to <backend>. To verify the effect independently I can
> check the truth via:
> (a) `<native CLI>` over SSH — needs an SSH key for `<host>`
> (b) the `<cloud>` CLI with profile `<x>` — needs that profile configured
> (c) a direct DB query — needs the connection string
> Which reflects the real source of truth in your environment, and which
> credentials should the harness use?

## Catalog of known mappings

| System under test | System of record | Independent last-mile reader |
|---|---|---|
| REST/gRPC API → Postgres | Postgres table | `psql -c "SELECT ..."` |
| REST/gRPC API → MySQL | MySQL table | `mysql -e "SELECT ..."` |
| REST/gRPC API → MongoDB | Mongo collection | `mongosh --eval 'db.coll.find(...)'` |
| REST API → Redis | Redis keyspace | `redis-cli GET/HGETALL ...` |
| Terraform provider (AWS) | AWS resource | `aws <service> describe-* / get-*` |
| Terraform provider (GCP) | GCP resource | `gcloud ... describe` |
| Terraform provider (Azure) | Azure resource | `az ... show` |
| Terraform provider (IBM z/OS RACF) | RACF profile/keyring | SSH + `tsocmd "RACDCERT ... LIST"` |
| Terraform provider (Vault) | Vault path | `vault read` / `vault kv get` / `vault list` |
| Terraform provider (Kubernetes) | cluster object | `kubectl get -o yaml` |
| Vault secrets engine | secret path | `vault read <path>` / `vault list <path>` |
| Okta integration | Okta org objects | Okta REST API (`/api/v1/users/...`) or Okta CLI |
| IAM policy change | cloud IAM | `aws iam get-* ` / `gcloud projects get-iam-policy` |
| Object-storage writer | bucket | `aws s3api head-object` / `gsutil stat` |
| DNS provider | zone records | `dig @<authoritative-ns> <name>` |
| Message producer | queue/topic | native consumer CLI (`kafka-console-consumer`, `aws sqs receive-message`) |
| Email/SMS sender | provider sink | provider API / a test inbox (e.g. Mailpit, message log) |

When the row you need isn't here, follow the decision procedure — the pattern
generalizes: find the operator's native inspection tool for the system of
record, confirm it's authoritative, and use it.

## Anti-patterns to avoid

- **Verifying through the system's own read endpoint.** `GET` after `POST`
  using the same service proves the service is self-consistent, not that the
  data persisted. Read the database directly.
- **Trusting the state file alone.** `terraform.tfstate` is the system's
  self-report. It's your *expected*, never your *actual*. Always cross-check the
  real cloud/backend.
- **Verifying against a cache or replica** that may lag or diverge from the
  primary.
- **Reusing the SUT's client library** to read back — a shared bug stays
  invisible.
