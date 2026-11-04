# Destination Profiles Integration Guide

Destination Profiles are the modern replacement for Custom URL Lists in Netskope. They define sets of destinations — domains, hosts, IPs, CIDRs, IP ranges, or regex patterns — that Real-time Protection policies match against, with a staged change workflow for safer production updates.

> **Endpoint paths in this guide are placeholders.** Netskope's public documentation does not publish the literal REST paths for the Destinations API — they're only available via Swagger inside your own tenant (**Settings > Tools > REST API v2 > API Documentation**). Confirm the exact `METHOD /api/v2/...` path and JSON schema for each operation below against your tenant's Swagger spec before shipping an integration against it.

---

## Overview

Use these endpoints to:
- List and search destination profiles
- Retrieve a specific profile, including its currently-applied (live) version
- Create, update, and delete profiles
- Incrementally append or remove destination values without rewriting the whole list
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
- Max **10** values per `append_values`/`remove_values` call
- Max **50** profile IDs per `deploy` call
- Profile name: max 100 characters; description: max 200 characters; change note: max 100 characters

---

## Key Concepts

### Profile Types
- `insensitive` — case-insensitive exact matching (most common use case)
- `sensitive` — case-sensitive exact matching
- `regex` — regular expression pattern matching

The Netskope UI presents the equivalent options as **Exact** (with a Case Sensitive/Case Insensitive toggle) and **RegEx** under Policies > Destination. The values above are the field values used by the API — verify the exact literal strings against your tenant's Swagger spec.

### Supported Destination Formats
- **Domains:** fully qualified (FQDN) and partially qualified (PQDN), e.g. `example.com`
- **Wildcards:** `*.example.com`
- **IP addresses:** `1.1.1.1`
- **CIDR blocks:** `10.0.0.0/8`
- **IP ranges**
- **URLs** (with an IP or domain), optionally including a port, path, or query
- **Regex patterns** — only when `profile_type` is `regex`

**Important:** IP-based entries are also evaluated against resolved domains. Path and query matching is **only supported for decrypted (SSL-inspected) web traffic** — a path/query added to a destination on non-decrypted traffic will not match.

### Profile Status Values
- `applied` — currently active in production
- `pending-create` — new profile awaiting deployment
- `pending-update` — modified profile awaiting deployment
- `pending-delete` — marked for deletion, awaiting deployment

### Staged Change Workflow (Interactive Mode)
Passing `interactive=True` on a create/update/delete call stages the change instead of applying it immediately — the profile's status becomes `pending-create`/`pending-update`/`pending-delete`. Review the staged state with workflow 2 (`get`), compare it against the live version with workflow 3 (`get_applied_version`), then either deploy (workflow 8) or revert (workflow 9).

**Without `interactive=True`, changes apply immediately** — there is no staging or review step.

---

## 1. List Destination Profiles

List and filter destination profiles in the tenant.

**Endpoint:** `GET /api/v2/<path-tbd>` *(confirm in Swagger — likely under a `policy` or `profiles` resource path)*

**Parameters:**
- `filter` — filter expression in the form `attribute operator "value"`, joined with `and`
  - Operators: `eq` (equals), `co` (contains), `in` (one of a list)
  - Filterable attributes: `id`, `name`, `status`, `values`, `description`, `label_ids`
  - Values must be double-quoted inside the filter string

**Example filters:**
```
filter='name co "engineering"'
filter='status in ("pending-update","pending-delete")'
filter='values eq "www.netskope.com"'
filter='name co "eng" and status eq "applied"'
filter='values co "github"'
```

**Use case:** Find profiles by name, status, or whether they contain a specific destination value — e.g., locating every profile referencing a domain before changing or removing it.

---

## 2. Get a Destination Profile

Retrieve a specific profile, including its full set of destination values.

**Endpoint:** `GET /api/v2/<path-tbd>/{profile_id}`

**Parameters:**
- `profile_id` — required
- `details` — optional; set to `False` to omit the `values` array (useful for large profiles where you only need metadata)

**Use case:** Inspect a profile's current (or pending, if interactive) state before deciding whether to update, deploy, or revert it.

---

## 3. Get Applied (Live) Version

Retrieve the currently-deployed version of a profile, independent of any pending changes.

**Endpoint:** `GET /api/v2/<path-tbd>/{profile_id}/applied` *(path shape unconfirmed — verify in Swagger)*

**Use case:** Compare a profile's pending changes (workflow 2) against what's actually live in production before deploying, so you know exactly what will change.

---

## 4. Create a Destination Profile

Create a new destination profile.

**Endpoint:** `POST /api/v2/<path-tbd>`

**Parameters:**
- `name` — required
- `profile_type` — required (`insensitive`, `sensitive`, or `regex`)
- `values` — required; array of destinations
- `description` — optional
- `label_ids` — optional
- `interactive` — optional; `True` stages the change as `pending-create` instead of applying immediately

**Use case:** Provision a new destination profile for a threat feed, malware/C2 list, or migrated URL List content.

---

## 5. Update a Destination Profile

Partially update an existing profile — only send the fields you want to change.

**Endpoint:** `PATCH /api/v2/<path-tbd>/{profile_id}`

**Parameters:**
- `profile_id` — required
- At least one field to update (`name`, `description`, `values`, `profile_type`, `label_ids`)
- `interactive` — optional; `True` stages the change as `pending-update`

**Important:** `profile_type` can only be changed together with `values` in the same call — changing the match type without resending the values returns an error.

**Use case:** Rename a profile, update its description, or change its destinations without a full replace.

---

## 6. Append Values

Add destinations to an existing profile incrementally, without resending the full list.

**Endpoint:** `POST /api/v2/<path-tbd>/{profile_id}/values` *(path shape unconfirmed — verify in Swagger)*

**Parameters:**
- `profile_id` — required
- `values` — up to 10 new destination values per call

**Use case:** Add newly discovered malicious domains/IPs to an existing threat-feed profile without re-reading and resending the entire current list.

---

## 7. Remove Values

Remove destinations from an existing profile, either by value or by position.

**Endpoint:** `DELETE /api/v2/<path-tbd>/{profile_id}/values` *(path shape unconfirmed — verify in Swagger)*

**Parameters:**
- `profile_id` — required
- Either `values` (exact match) **or** `indexes` (position-based) — these are mutually exclusive; sending both is invalid
- Up to 10 values/indexes per call

**Use case:** Clean up stale or expired entries (e.g., a threat indicator that's aged out) from a profile.

---

## 8. Deploy Staged Changes

Apply one or more profiles' pending (staged) changes to production.

**Endpoint:** `POST /api/v2/<path-tbd>/deploy` *(path shape unconfirmed — verify in Swagger)*

**Parameters:**
- `ids` — array of profile IDs to deploy, max 50 — **or**
- `deploy_all` — `True` to deploy every pending profile
- `change_note` — optional, max 100 characters; recorded for audit trail

**Use case:** Promote reviewed, staged changes (created with `interactive=True`) into live policy enforcement, with an audit trail of why the change was made.

---

## 9. Revert Pending Changes

Discard a profile's staged changes and return it to its last applied (live) version.

**Endpoint:** `POST /api/v2/<path-tbd>/{profile_id}/revert` *(path shape unconfirmed — verify in Swagger)*

**Use case:** Back out a staged change that failed review, without having touched production.

---

## 10. Delete a Destination Profile

Delete a profile entirely.

**Endpoint:** `DELETE /api/v2/<path-tbd>/{profile_id}`

**Important:** Fails with HTTP 409 if the profile is referenced by an active policy. Update or remove the policy reference first.

**Use case:** Clean up unused or decommissioned profiles.

---

## 11. Migrate a URL List to a Destination Profile

Convert a legacy Custom URL List into a Destination Profile.

**Endpoint:** `POST /api/v2/<path-tbd>/migrate` *(path shape unconfirmed — verify in Swagger)*

**Parameters:**
- `url_list_id` — required; the ID of the legacy URL List (see [Custom-URL-Lists.md](Custom-URL-Lists.md) workflow 1 to find it)

**Important:** The migrated profile is always created in `pending` status — it requires an explicit deploy (workflow 8) before it's live, even if you didn't pass `interactive=True`.

**Use case:** Move an existing threat feed or blocklist off the legacy Custom URL List API onto Destination Profiles without manually re-entering every value.

---

## Best Practices

- Treat these as **live production security policy changes** — a bad deploy can open up or block the wrong traffic immediately
- Use `interactive=True` for any production change so you can review before it goes live
- Always compare the pending state (workflow 2) against the applied state (workflow 3) before deploying
- Add a `change_note` to every deploy for audit trail purposes
- Use `append_values`/`remove_values` for incremental edits instead of a full `update` — faster, and avoids accidentally dropping existing entries
- Check for policy references before deleting a profile (a 409 means it's still in use — update the policy first)
- Test profile changes in a non-production tenant or policy before rolling out broadly

---

## Integration Tips

### Error Handling
- Changing `profile_type` without also sending `values` in the same call returns an error — always send both together
- Sending both `values` and `indexes` to `remove_values` is invalid — they're mutually exclusive
- Deleting a profile referenced by an active policy returns HTTP 409 — resolve the policy reference first
- Updating or deleting a profile that's already in a `pending-*` status without `interactive=True` returns a conflict — either deploy/revert the pending change first, or use `interactive=True` again

### Idempotency
- Store the profile `id` returned at creation for all future update/deploy/delete calls
- Use `filter` on the list endpoint (e.g., `filter='values co "github"'`) to re-discover a profile's ID if it wasn't stored locally

### Timing
- Changes made without `interactive=True` apply immediately — there's no staging delay to account for
- Staged (`pending-*`) changes have no effect on policy until explicitly deployed (workflow 8)

### Testing
- Start with a small number of test destinations and a non-critical policy before bulk-loading a production threat feed
- Always exercise the interactive review cycle (create → get → get_applied_version → deploy) at least once before relying on direct, non-interactive writes

---

## Common Patterns

### Creating a New Profile (Staged)
```
1. Create with interactive=True
2. get the profile to review its pending state
3. Confirm the destinations look correct
4. Deploy with a change_note describing why
```

### Migrating from Custom URL Lists
```
1. Find the legacy URL List's ID (Custom-URL-Lists.md, workflow 1)
2. Call migrate_url_list with that url_list_id
3. The new profile is created in pending status
4. Review the migrated profile's values (workflow 2)
5. Deploy when ready (workflow 8)
```

### Incremental Threat Feed Updates
```
1. append_values to add newly discovered malicious destinations
2. remove_values to drop expired/stale ones
3. Both are faster than a full update on a large profile, and lower-risk since they don't require resending the entire list
```

---

## Troubleshooting

**Problem:** `update` call returns an error when changing the match type
- **Solution:** `profile_type` can only be changed together with `values` in the same request — resend the full values array alongside the new `profile_type`.

**Problem:** `remove_values` call is rejected
- **Solution:** Use either `values` or `indexes`, never both in the same call.

**Problem:** `delete` fails with HTTP 409
- **Solution:** The profile is still referenced by an active policy. Update the policy to stop referencing it, then delete.

**Problem:** Update or delete on a profile fails with a conflict even though no policy references it
- **Solution:** The profile likely already has a pending change from a prior `interactive=True` call. Deploy or revert that pending change first, or set `interactive=True` on this call too.

**Problem:** A path/query condition on a destination never matches traffic
- **Solution:** Path and query matching only applies to decrypted (SSL-inspected) web traffic. If the traffic isn't being SSL-decrypted by policy, path/query portions of the destination are ignored.

**Problem:** Migrated URL List profile isn't enforcing after migration
- **Solution:** `migrate_url_list` always creates the new profile in `pending` status, regardless of `interactive`. It must be explicitly deployed (workflow 8) before it takes effect.

---

## API Reference

| Workflow | Endpoint (confirm in Swagger) | Method | Purpose |
|----------|-------------------------------|--------|---------|
| 1 | `/api/v2/<path-tbd>` | GET | List/search destination profiles |
| 2 | `/api/v2/<path-tbd>/{profile_id}` | GET | Retrieve a specific profile |
| 3 | `/api/v2/<path-tbd>/{profile_id}/applied` | GET | Get the currently-applied (live) version |
| 4 | `/api/v2/<path-tbd>` | POST | Create a new profile |
| 5 | `/api/v2/<path-tbd>/{profile_id}` | PATCH | Partially update a profile |
| 6 | `/api/v2/<path-tbd>/{profile_id}/values` | POST | Append up to 10 values |
| 7 | `/api/v2/<path-tbd>/{profile_id}/values` | DELETE | Remove up to 10 values |
| 8 | `/api/v2/<path-tbd>/deploy` | POST | Deploy staged changes |
| 9 | `/api/v2/<path-tbd>/{profile_id}/revert` | POST | Revert staged changes |
| 10 | `/api/v2/<path-tbd>/{profile_id}` | DELETE | Delete a profile |
| 11 | `/api/v2/<path-tbd>/migrate` | POST | Migrate a URL List to a destination profile |

**All paths above are placeholders** — confirm the exact endpoint, method, and payload shape in your tenant's Swagger UI (**Settings > Tools > REST API v2 > API Documentation**) before implementing against them.

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
