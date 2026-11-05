# Destination Profiles Integration Guide

Destination Profiles are the modern replacement for Custom URL Lists in Netskope. They define sets of destinations — domains, hosts, IPs, CIDRs, IP ranges, or regex patterns — that Real-time Protection policies match against, with a staged change workflow for safer production updates.

> **Verification status:** The base path, both read (`GET`) endpoints, create, update, deploy, revert, and delete were all exercised live against a test tenant using disposable test profiles (created and fully cleaned up afterward). Two things remain unconfirmed: a dedicated "applied version" lookup (workflow 3) and the URL List migration endpoint (workflow 11) — see the notes on each below.

---

## Overview

Use these endpoints to:
- List and search destination profiles
- Retrieve a specific profile
- Create, update, and delete profiles
- Stage changes safely, then deploy or revert them
- Migrate a legacy Custom URL List to a Destination Profile

**Prerequisites:**
- NG-SWG or CASB subscription with Real-time Protection policies (Destination Profiles are configured under **Policies > Destination** in the Netskope UI)
- The relevant API scope enabled for your token — confirm the exact scope name in your tenant's Swagger UI, consistent with how other Netskope v2 endpoints are scoped by functional area
- Include a `User-Agent` header formatted as `<Vendor-Product-Version>` on every call, consistent with other Netskope REST API v2 integrations

**Limits:**
- Max **1,000** destination profiles per tenant (default)
- Max **100,000** destinations per profile
- Max profile size **10 MB**, including comments (comments themselves have no count limit, but count toward this size cap)
- Max **300,000** destinations per tenant in total, excluding comments — raising this requires a license. This is the same tenant-wide ceiling used by legacy Custom URL Lists (see [Custom-URL-Lists.md](Custom-URL-Lists.md))
- Profile name: max 100 characters; description: max 200 characters; change note: max 100 characters

---

## Key Concepts

### Profile Types
The `type` field on a profile (confirmed via live tenant data) takes one of:
- `insensitive` — case-insensitive exact matching (most common use case)
- `sensitive` — case-sensitive exact matching
- `regex` — regular expression pattern matching

The Netskope UI presents the equivalent options as **Exact** (with a Case Sensitive/Case Insensitive toggle) and **RegEx** under Policies > Destination.

### Supported Destination Formats
- **Domains:** fully qualified (FQDN) and partially qualified (PQDN), e.g. `example.com`
- **Wildcards:** `*.example.com`
- **IP addresses:** `1.1.1.1`
- **CIDR blocks:** `10.0.0.0/8`
- **IP ranges**
- **URLs** (with an IP or domain), optionally including a port, path, or query
- **Regex patterns** — only when `type` is `regex`

**Important:** IP-based entries are also evaluated against resolved domains. Path and query matching is **only supported for decrypted (SSL-inspected) web traffic** — a path/query added to a destination on non-decrypted traffic will not match.

### Profile Status Values
Confirmed live by creating, staging, deploying, and reverting a test profile:
- `applied` — currently active in production
- `pending-create` — new profile awaiting deployment
- `pending-update` — modified profile awaiting deployment
- `pending-delete` — marked for deletion, awaiting deployment (inferred by symmetry with the other two; not independently tested)

### Staged Change Workflow (Interactive Mode)

**Confirmed:** `interactive` is a **query parameter**, not a body field (passing it in the JSON body has no effect — the write still applies immediately). Add `?interactive=true` to a create or update call to stage the change instead of applying it immediately:

```
POST /api/v2/profiles/destinations?interactive=true        → stages as pending-create, returns HTTP 202
PATCH /api/v2/profiles/destinations/{id}?interactive=true   → stages as pending-update, returns HTTP 202
```

A non-interactive call to the same endpoints returns **HTTP 201** (create) or **HTTP 200** (update) and applies immediately — confirmed live.

Review the staged state with workflow 2 (`get` — it returns the pending data, since there's no separate applied-only view confirmed), then either deploy (workflow 8) or revert (workflow 9).

---

## 1. List Destination Profiles

List and filter destination profiles in the tenant. **Confirmed live**, including the `filter` syntax below.

**Endpoint:** `GET /api/v2/profiles/destinations`

```bash
curl -X GET \
  'https://<customer-tenant>.goskope.com/api/v2/profiles/destinations?limit=50' \
  -H 'accept: application/json' \
  -H 'Netskope-Api-Token: <token>' \
  -H 'User-Agent: <Vendor-Product-Version>'
```

**Response (confirmed shape):**
```json
{
  "elements": [
    {
      "id": "5c0174ed-fb8e-4c8e-897d-4b8fc2cf51b9",
      "name": "example-threat-feed",
      "description": "Example destination profile",
      "type": "insensitive",
      "values": ["malware.example.com"],
      "values_count": 1,
      "status": "applied",
      "create_by": "integration@example.com",
      "create_time": "2026-06-17T12:30:43.888Z",
      "modify_by": "integration@example.com",
      "modify_time": "2026-06-17T12:30:52.125Z",
      "label_ids": []
    }
  ],
  "total_count": 6
}
```

**Parameters:**
- `filter` — filter expression in the form `attribute operator "value"`, joined with `and`
  - Operators: `eq` (equals), `co` (contains), `in` (one of a list)
  - Filterable attributes: `id`, `name`, `status`, `values`, `description`, `label_ids`
  - Values must be double-quoted inside the filter string
  - **Confirmed live:** `filter=name co "<substring>"` correctly returned only the matching profile
- `limit` — max results per page (confirmed working)

**Example filters:**
```
filter=name co "engineering"
filter=status in ("pending-update","pending-delete")
filter=values eq "www.netskope.com"
filter=name co "eng" and status eq "applied"
filter=values co "github"
```

**Use case:** Find profiles by name, status, or whether they contain a specific destination value — e.g., locating every profile referencing a domain before changing or removing it.

---

## 2. Get a Destination Profile

Retrieve a specific profile by ID. **Confirmed live** — returns the same object shape as a single element in workflow 1's list, not wrapped in `elements`. If the profile has a pending change, this returns the pending data (see workflow 3 for the caveat on getting the applied-only view).

**Endpoint:** `GET /api/v2/profiles/destinations/{id}`

```bash
curl -X GET \
  'https://<customer-tenant>.goskope.com/api/v2/profiles/destinations/5c0174ed-fb8e-4c8e-897d-4b8fc2cf51b9' \
  -H 'accept: application/json' \
  -H 'Netskope-Api-Token: <token>' \
  -H 'User-Agent: <Vendor-Product-Version>'
```

**Response (confirmed shape):**
```json
{
  "id": "5c0174ed-fb8e-4c8e-897d-4b8fc2cf51b9",
  "name": "example-threat-feed",
  "description": "Example destination profile",
  "type": "insensitive",
  "values": ["malware.example.com"],
  "values_count": 1,
  "status": "applied",
  "create_by": "integration@example.com",
  "create_time": "2026-06-17T12:30:43.888Z",
  "modify_by": "integration@example.com",
  "modify_time": "2026-06-17T12:30:52.125Z",
  "label_ids": []
}
```

**Use case:** Inspect a profile's current (or pending, if interactive) state before deciding whether to update, deploy, or revert it.

---

## 3. Get Applied (Live) Version

**Not confirmed.** We tested this against a profile with a genuinely different pending vs. applied state (a `pending-update` with a changed description) and could not find a working endpoint:
- `GET /{id}/applied` and `GET /{id}/live` both returned **HTTP 403 "Forbidden"**
- Query parameters `?version=applied`, `?state=applied`, and `?applied=true` were silently ignored — all returned the current (pending) state unchanged, not the prior applied state

It's possible this requires a permission our test token didn't have, uses a path we didn't guess, or doesn't exist as a separate call. **Verify in your tenant's Swagger before building against this.** In practice, you can track the applied state yourself by storing a profile's values right before staging an `interactive=true` change.

**Use case (if it exists):** Compare a profile's pending changes against what's actually live in production before deploying.

---

## 4. Create a Destination Profile

Create a new destination profile. **Confirmed live**, including both the immediate and staged (`interactive=true`) variants.

**Endpoint:** `POST /api/v2/profiles/destinations`

```bash
curl -X POST \
  'https://<customer-tenant>.goskope.com/api/v2/profiles/destinations' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Netskope-Api-Token: <token>' \
  -H 'User-Agent: <Vendor-Product-Version>' \
  -d '{
    "name": "example-threat-feed",
    "type": "insensitive",
    "values": ["malware.example.com"],
    "description": "Example destination profile",
    "label_ids": []
  }'
```

**Response — applies immediately (confirmed: HTTP 201):**
```json
{
  "id": "a974311f-7523-4d10-9289-f373722958ba",
  "name": "example-threat-feed",
  "description": "Example destination profile",
  "type": "insensitive",
  "values": ["malware.example.com"],
  "values_count": 1,
  "status": "applied",
  "create_by": "integration@example.com",
  "create_time": "2026-11-05T20:53:50.894Z",
  "modify_by": "integration@example.com",
  "modify_time": "2026-11-05T20:53:50.894Z",
  "label_ids": []
}
```

**Staged variant — add `?interactive=true` to the URL (confirmed: HTTP 202, status `pending-create`):**
```bash
curl -X POST \
  'https://<customer-tenant>.goskope.com/api/v2/profiles/destinations?interactive=true' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Netskope-Api-Token: <token>' \
  -H 'User-Agent: <Vendor-Product-Version>' \
  -d '{
    "name": "example-threat-feed-staged",
    "type": "insensitive",
    "values": ["malware.example.com"],
    "description": "Example staged destination profile"
  }'
```

**Parameters:**
- `name` — required
- `type` — required (`insensitive`, `sensitive`, or `regex`)
- `values` — required; array of destinations
- `description` — optional
- `label_ids` — optional
- `interactive=true` — **query parameter**, not a body field; stages the change instead of applying it immediately

**Use case:** Provision a new destination profile for a threat feed, malware/C2 list, or migrated URL List content.

---

## 5. Update a Destination Profile

Partially update an existing profile — only send the fields you want to change. **Confirmed live**, including the staged variant and the exact validation rule below.

**Endpoint:** `PATCH /api/v2/profiles/destinations/{id}`

```bash
curl -X PATCH \
  'https://<customer-tenant>.goskope.com/api/v2/profiles/destinations/a974311f-7523-4d10-9289-f373722958ba' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Netskope-Api-Token: <token>' \
  -H 'User-Agent: <Vendor-Product-Version>' \
  -d '{
    "description": "Updated description"
  }'
```

**Staged variant (confirmed: HTTP 202, status `pending-update`):**
```bash
curl -X PATCH \
  'https://<customer-tenant>.goskope.com/api/v2/profiles/destinations/a974311f-7523-4d10-9289-f373722958ba?interactive=true' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Netskope-Api-Token: <token>' \
  -d '{"description": "Testing a staged update"}'
```

**Parameters:**
- At least one field to update (`name`, `description`, `values`, `type`, `label_ids`)
- `interactive=true` — query parameter; stages the change as `pending-update`

**Important — confirmed by live testing, and stricter than you might expect:** Whenever `values` is included in the request body, `type` **must also be included**, even if you aren't changing the match type. Omitting `type` while sending `values` returns:
```json
{
  "err_code": "50101",
  "message": "invalid format: validation error occurred in the request",
  "validation_errors": [
    {"field": "values[0]", "error": "values[0] is not a valid destination profile value"},
    {"field": "type", "error": "type is a required field"}
  ]
}
```
Fields that don't touch `values` (e.g., `description` alone) can be sent without `type` — confirmed working.

**Use case:** Rename a profile, update its description, or replace its destination list.

---

## 6. Adding Values Incrementally

**Not a separate endpoint — this is a client-side pattern, not a dedicated API call.** We tested `POST .../{id}/values` (404), `POST .../{id}/append` (404), `PATCH .../{id}/append` (403), and `PATCH .../{id}?action=append` (silently behaved as a full replace, not an incremental append — it overwrote the existing values rather than adding to them). None of these added to the list without replacing it.

To add destinations without losing existing ones:
```
1. GET the profile (workflow 2) to retrieve its current `values` array
2. Append your new value(s) to that array client-side
3. PATCH the profile (workflow 5) with the full combined `values` array and `type`
```

**Use case:** Add newly discovered malicious domains/IPs to an existing threat-feed profile.

---

## 7. Removing Values

Same situation as workflow 6 — **no dedicated remove endpoint was found**. Use the same read-modify-write pattern:
```
1. GET the profile (workflow 2) to retrieve its current `values` array
2. Remove the value(s) you want to drop client-side
3. PATCH the profile (workflow 5) with the full remaining `values` array and `type`
```

**Use case:** Clean up stale or expired entries (e.g., a threat indicator that's aged out) from a profile.

---

## 8. Deploy Staged Changes

Apply one or more profiles' pending (staged) changes to production. **Confirmed live** — this is how we promoted a `pending-create` test profile to `applied`.

**Endpoint:** `POST /api/v2/profiles/destinations/deploy`

```bash
curl -X POST \
  'https://<customer-tenant>.goskope.com/api/v2/profiles/destinations/deploy' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Netskope-Api-Token: <token>' \
  -H 'User-Agent: <Vendor-Product-Version>' \
  -d '{
    "ids": ["c5dee54a-177c-4346-8505-43deb8f76da5"],
    "change_note": "Deploying staged threat feed update"
  }'
```

**Response (confirmed shape):**
```json
{
  "applied": ["c5dee54a-177c-4346-8505-43deb8f76da5"]
}
```

**Parameters:**
- `ids` — array of profile IDs to deploy, max 50
- `change_note` — optional, max 100 characters; recorded for audit trail

**Not tested:** `deploy_all` (deploying every pending profile at once) — the `ids`-based call above is confirmed; `deploy_all` is a reasonable assumption based on common Netskope API conventions but wasn't exercised live.

**Use case:** Promote reviewed, staged changes into live policy enforcement, with an audit trail of why the change was made.

---

## 9. Revert Pending Changes

Discard a profile's staged changes and return it to its last applied (live) version. **Confirmed live** — reverted a `pending-update` test profile and its description correctly rolled back to the original applied value.

**Endpoint:** `POST /api/v2/profiles/destinations/{id}/revert`

```bash
curl -X POST \
  'https://<customer-tenant>.goskope.com/api/v2/profiles/destinations/a974311f-7523-4d10-9289-f373722958ba/revert' \
  -H 'accept: application/json' \
  -H 'Netskope-Api-Token: <token>' \
  -H 'User-Agent: <Vendor-Product-Version>'
```

**Response:** the profile object, with `status` back to `applied` and the pending field changes discarded.

**Use case:** Back out a staged change that failed review, without having touched production.

---

## 10. Delete a Destination Profile

Delete a profile entirely. **Confirmed live.**

**Endpoint:** `DELETE /api/v2/profiles/destinations/{id}`

```bash
curl -X DELETE \
  'https://<customer-tenant>.goskope.com/api/v2/profiles/destinations/a974311f-7523-4d10-9289-f373722958ba' \
  -H 'accept: application/json' \
  -H 'Netskope-Api-Token: <token>' \
  -H 'User-Agent: <Vendor-Product-Version>'
```

**Response (confirmed):**
```json
{"status": "success"}
```

**Important (not independently tested, but standard Netskope platform behavior — see [Custom-URL-Lists.md](Custom-URL-Lists.md) and other guides in this repo):** Expect this to fail with HTTP 409 if the profile is referenced by an active policy. Update or remove the policy reference first.

**Use case:** Clean up unused or decommissioned profiles.

---

## 11. Migrate a URL List to a Destination Profile

Convert a legacy Custom URL List into a Destination Profile. **Not tested** — we didn't want to create or disturb a legacy URL List as part of this verification pass.

**Endpoint:** `POST /api/v2/profiles/destinations/migrate` *(sub-path inferred from the confirmed base resource and REST convention — verify in Swagger)*

**Parameters:**
- `url_list_id` — required; the ID of the legacy URL List (see [Custom-URL-Lists.md](Custom-URL-Lists.md) workflow 1 to find it)

**Expected behavior (unverified):** The migrated profile is created in `pending` status and requires an explicit deploy (workflow 8) before it's live.

**Use case:** Move an existing threat feed or blocklist off the legacy Custom URL List API onto Destination Profiles without manually re-entering every value.

---

## Best Practices

- Treat these as **live production security policy changes** — a bad deploy can open up or block the wrong traffic immediately
- Use `?interactive=true` for any production change so you can review before it goes live
- Add a `change_note` to every deploy for audit trail purposes
- Always send `type` alongside `values` on any update — omitting it when `values` is present will fail validation
- Use the read-modify-write pattern (workflows 6/7) for incremental edits, and always fetch the current values first to avoid accidentally dropping existing entries
- Check for policy references before deleting a profile (expect a 409 if it's still in use — update the policy first)
- Test profile changes against a non-production profile before rolling out broadly

---

## Integration Tips

### Error Handling
- Sending `values` without `type` in a create or update call returns HTTP 400 with `"type is a required field"` — always send both together whenever you're touching values
- Deleting a profile referenced by an active policy is expected to return HTTP 409 — resolve the policy reference first
- `interactive` has no effect as a body field — it must be a query parameter, or your "staged" change will apply immediately without warning

### Idempotency
- Store the profile `id` returned at creation for all future update/deploy/delete calls
- Use `filter` on the list endpoint (e.g., `filter=values co "github"`) to re-discover a profile's ID if it wasn't stored locally
- Since there's no confirmed incremental append/remove endpoint, always re-GET the current `values` array immediately before modifying it, to avoid racing another writer

### Timing
- Non-interactive changes apply immediately — confirmed via HTTP 200/201 responses with `status: applied`
- Staged (`pending-*`) changes have no effect on policy until explicitly deployed (workflow 8) — confirmed via the revert test, which showed the applied policy was untouched while a change was pending

### Testing
- Start with a disposable test profile (clearly named, e.g. prefixed `TEST-` or `ZZZ-`) before writing automation against production profiles
- Exercise the full interactive review cycle (create with `?interactive=true` → get → deploy or revert) at least once before relying on direct, non-interactive writes
- Delete your test profiles when done — nothing cleans them up automatically

---

## Common Patterns

### Creating a New Profile (Staged)
```
1. POST .../profiles/destinations?interactive=true to create
2. GET the profile to review its pending state
3. Confirm the destinations look correct
4. POST .../deploy with the profile's id and a change_note
```

### Incremental Threat Feed Updates (Read-Modify-Write)
```
1. GET the profile to retrieve its current values array
2. Add or remove destinations in that array client-side
3. PATCH the profile with the full updated values array plus type
```

### Reviewing Before a Risky Change
```
1. PATCH the profile with ?interactive=true (stages as pending-update)
2. GET the profile to review what will change
3. If it looks right: POST .../deploy
4. If not: POST .../{id}/revert to discard the staged change with zero production impact
```

---

## Troubleshooting

**Problem:** Update or create returns `"type is a required field"` even though you're only changing `values`
- **Solution:** Always include `type` in the same request body as `values` — this is required on every call that touches `values`, not just when the match type itself is changing.

**Problem:** A value you sent is rejected with `"is not a valid destination profile value"`
- **Solution:** This error also appears (somewhat misleadingly) when `type` is missing from the request — fix that first and re-check whether the value itself is still rejected.

**Problem:** Sent `interactive: true` in the JSON body but the change still applied immediately
- **Solution:** `interactive` must be a query parameter (`?interactive=true`), not a body field. The body field is silently ignored.

**Problem:** Tried to append a single value and it replaced the whole list instead
- **Solution:** There's no confirmed incremental append endpoint. `PATCH` always replaces the `values` array — fetch the current values first, add your new value client-side, then PATCH the combined array.

**Problem:** Delete fails with HTTP 409
- **Solution:** The profile is likely still referenced by an active policy. Update the policy to stop referencing it, then delete.

**Problem:** A path/query condition on a destination never matches traffic
- **Solution:** Path and query matching only applies to decrypted (SSL-inspected) web traffic. If the traffic isn't being SSL-decrypted by policy, path/query portions of the destination are ignored.

---

## API Reference

| Workflow | Endpoint | Method | Verified? |
|----------|----------|--------|-----------|
| 1 | `/api/v2/profiles/destinations` | GET | ✅ Confirmed live |
| 2 | `/api/v2/profiles/destinations/{id}` | GET | ✅ Confirmed live |
| 3 | Applied-only view of a profile | GET | ❌ Not found (tried `/applied`, `/live`, and 3 query params) |
| 4 | `/api/v2/profiles/destinations` (`?interactive=true` to stage) | POST | ✅ Confirmed live, both variants |
| 5 | `/api/v2/profiles/destinations/{id}` (`?interactive=true` to stage) | PATCH | ✅ Confirmed live, both variants |
| 6 | *(no endpoint — read-modify-write via workflow 2 + 5)* | — | ✅ Confirmed no dedicated endpoint exists at the paths tried |
| 7 | *(no endpoint — read-modify-write via workflow 2 + 5)* | — | ✅ Confirmed no dedicated endpoint exists at the paths tried |
| 8 | `/api/v2/profiles/destinations/deploy` | POST | ✅ Confirmed live |
| 9 | `/api/v2/profiles/destinations/{id}/revert` | POST | ✅ Confirmed live |
| 10 | `/api/v2/profiles/destinations/{id}` | DELETE | ✅ Confirmed live |
| 11 | `/api/v2/profiles/destinations/migrate` | POST | ⚠️ Not tested — path inferred |

---

## Related APIs

- **Custom URL Lists (legacy):** see [Custom-URL-Lists.md](Custom-URL-Lists.md) — Destination Profiles are the replacement for this API
- **Real-time Protection Policies:** reference destination profiles for traffic matching
- **DLP Policies:** can target destinations defined in a profile

---

## Support & Additional Resources

- **Netskope Destination Profile documentation:** https://docs.netskope.com/en/destination-profile
- **REST API v2 overview:** https://docs.netskope.com/en/netskope-help/admin-console/rest-api/rest-api-v2-overview-312207/
- **Swagger API documentation:** available in your tenant at Settings > Tools > REST API v2 > API Documentation
- **Questions?** Contact your Netskope account team for API support.
