# PV Recipe: REST/gRPC APIs that write to a Database

The classic last-mile case. The API returns `201 Created` and an entity body —
that's its self-report. The truth is the row it claims to have written. PV
drives the API as a black box, then reads the database directly through an
independent client to confirm the row is really there with the right values.

## Generate → Operate → Last-mile

### 1. Generate inputs
Build a set of request payloads covering the fields that matter — required and
optional, valid edge values, and a few that should be rejected. Keep them as
JSON files or heredocs so the harness is readable and re-runnable.

### 2. Operate the API as a black box
Drive the real HTTP surface with `curl` (or `grpcurl`). Capture the status code
and response body as the self-report:

```bash
resp="$(curl -sS -w '\n%{http_code}' -X POST "$BASE_URL/widgets" \
  -H 'Content-Type: application/json' \
  -d @payloads/widget-1.json)"
body="$(sed '$d' <<<"$resp")"; code="$(tail -n1 <<<"$resp")"
[ "$code" = "201" ] && log_pass "POST /widgets → 201" || log_fail "expected 201, got $code"
id="$(jq -r '.id' <<<"$body")"            # the API's claimed identity
```

### 3. Last-mile verify against the database (actual)
Use the database's **own** client — never the API's ORM or service code — so a
persistence bug can't hide behind shared code. Read the row the API claims to
have written and compare field by field.

```bash
# Postgres
row="$(psql "$DATABASE_URL" -tAF '|' -c \
  "SELECT name, status, owner_id FROM widgets WHERE id = '$id'")"
[ -n "$row" ] || log_fail "widget $id absent in DB — API claimed 201"
IFS='|' read -r db_name db_status db_owner <<<"$row"
[ "$db_name" = "$expected_name" ] && log_pass "name persisted" \
                                  || log_fail "name: API='$expected_name' DB='$db_name'"
```

Equivalents for other stores:
- MySQL: `mysql -N -e "SELECT ... FROM widgets WHERE id='$id'"`
- MongoDB: `mongosh --quiet --eval "db.widgets.findOne({_id:'$id'})"`
- Redis: `redis-cli HGETALL "widget:$id"`

Prefer a strongly-consistent read against the primary — not a replica or a
search index that may lag.

## Probe the full lifecycle, both directions

The create check is just the start. The valuable bugs are at the edges:

- **Update** — `PUT/PATCH` via the API, then re-`SELECT` and confirm the row
  changed to match (and that *only* the intended columns changed).
- **Delete (reverse check)** — `DELETE` via the API, then confirm the row is
  actually gone from the DB. A soft-delete that the API reports as "deleted"
  but leaves in the table is exactly the kind of lie PV exists to catch — assert
  on the real table state, and on the `deleted_at`/tombstone semantics if that's
  the design.
- **Constraints & side effects** — if a write should also touch related tables
  (audit log, join rows, counters), verify those too. APIs frequently report
  success while skipping a secondary write.
- **Rejection paths** — send a payload that *should* be rejected; confirm the
  API refuses it **and** that no partial row leaked into the DB.
- **Idempotency / retries** — if the endpoint claims idempotency (e.g. an
  idempotency key), POST twice and confirm exactly one row exists.

## Gotchas

- **Async writes.** If the API enqueues and writes later, a naive immediate
  `SELECT` races. Poll the DB with a bounded timeout, and treat "never appeared"
  as FAIL — but make the wait explicit in the harness so a reader sees it.
- **Connection identity.** Make sure your `psql`/`mysql` connects to the *same*
  database the API writes to (same host, db, schema) — verifying the wrong
  database yields a false FAIL or, worse, a false PASS against stale data.
- **Transactions left open / not committed.** Reading in a separate session is
  actually a feature here: it only sees committed data, so it catches "returned
  200 but never committed" bugs.
- **Type/serialization drift.** Numbers stored as strings, timestamps in the
  wrong zone, JSON columns with reordered keys — compare semantically, WARN on
  cosmetic differences, FAIL on real ones.
