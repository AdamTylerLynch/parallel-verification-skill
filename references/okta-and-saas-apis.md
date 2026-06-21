# PV Recipe: Okta & SaaS-API Backends

When the system of record is a SaaS platform (Okta, Auth0, GitHub, Salesforce,
Stripe, Datadog…), the truth lives behind that platform's API. The system under
test — often a Terraform provider or an integration service — claims it created
a user, group, app, policy, or webhook. PV drives that system, then reads the
object back through the SaaS platform's **own** API/CLI, which is independent of
your integration code.

## Generate → Operate → Last-mile

### 1. Generate inputs
Define the objects to create with attributes that matter: for Okta, a user's
profile fields, group memberships, app assignments, policy rules. Output any
server-assigned IDs you'll need for the read-back.

### 2. Operate as a black box
Drive the integration through its real interface — `terraform apply` for a
provider, or the service's API. Capture the claimed object IDs from the
self-report (state file or response body).

### 3. Last-mile verify via the platform's own API
Use the platform API directly. The independence is what counts: your integration
might use a buggy client, but a raw API call sees the real object.

```bash
# Okta: confirm a user really exists with the expected status & profile
user="$(curl -sS -H "Authorization: SSWS $OKTA_API_TOKEN" \
  "https://$OKTA_ORG.okta.com/api/v1/users/$USER_ID")"
status="$(jq -r '.status' <<<"$user")"
[ "$status" = "ACTIVE" ] && log_pass "user ACTIVE" || log_fail "status=$status"
[ "$(jq -r '.profile.email' <<<"$user")" = "$expected_email" ] \
  && log_pass "email matches" || log_fail "email drift"

# Group membership (a relationship, not just existence)
curl -sS -H "Authorization: SSWS $OKTA_API_TOKEN" \
  "https://$OKTA_ORG.okta.com/api/v1/groups/$GROUP_ID/users" \
  | jq -e --arg id "$USER_ID" 'any(.id == $id)' >/dev/null \
  && log_pass "user in group" || log_fail "membership missing"
```

The same shape applies to any SaaS backend — swap in that platform's API and
auth header (GitHub `Authorization: Bearer`, Stripe `-u sk_...:`, etc.). Many
also ship a CLI (`okta`, `gh`, `stripe`) that's an equally valid independent
reader.

## Verify relationships and lifecycle, not just existence

SaaS bugs cluster in relationships and state transitions:

- **Membership & assignment** — user-in-group, app-assigned-to-user,
  role-bound-to-policy. Confirm the *relationship* via the platform, not just
  that both endpoints exist.
- **Status transitions** — activate/suspend/deactivate a user; confirm the
  platform reflects the new status. A provider that reports `SUSPENDED` while
  Okta still shows `ACTIVE` is a security-relevant lie.
- **Deletion** — delete via the integration, then confirm the platform returns
  `404`/deprovisioned. Lingering access after a claimed delete is a serious
  finding.
- **Policy/rule effect** — where possible, verify the rule's *effect*, not just
  its presence (e.g. a sign-on policy actually applies to the right group).

## Gotchas

- **Rate limits & eventual consistency.** SaaS APIs throttle and may take a
  moment to reflect writes. Poll with a bounded timeout and back off on `429`;
  make the wait explicit so it's not mistaken for a flaky FAIL.
- **Credential scope.** The read-back token needs read scope on the object type.
  Resolve it via the credential conventions; fail with a clear message if it's
  absent. Never hardcode tokens — read from env (`OKTA_API_TOKEN`, etc.).
- **Test org / sandbox.** Run against a sandbox or dedicated test org, never
  production — PV creates and destroys real objects. Confirm the target org with
  the user if there's any ambiguity.
- **Pagination.** List endpoints paginate; a membership check that only reads
  page one can falsely report "missing". Follow `next` links or filter
  server-side by ID.
