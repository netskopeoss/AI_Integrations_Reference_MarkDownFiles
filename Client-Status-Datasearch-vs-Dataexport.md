# Client Status: datasearch vs. dataexport Integration Guide

Netskope exposes Netskope Client status and health data two different ways: an ad-hoc query API (`datasearch`) and a streaming bulk-export API (`dataexport`'s Client Status Iterator). They are **not interchangeable** — each returns a different field set in a different format. Use this guide to query client status, retrieve Endpoint DLP (`epdlp`) driver/service health telemetry, and pick the right API for your integration.

---

## Overview

Use this guide to:
- Query real-time client status for one or more devices (OS, version, location, policy state)
- Retrieve Endpoint DLP (`epdlp`) health telemetry — CPU/memory usage, driver and service versions, error counts, running vs. enabled state
- Set up continuous bulk export of client status events for SIEM/data-lake ingestion
- Choose between `datasearch` and `dataexport` based on what your integration actually needs

**Prerequisites:**
- NG-SWG, CASB, or equivalent subscription with the Netskope Client deployed to endpoints
- API scope enabled for the Events (Datasearch/Dataexport) functional area
- Include a `User-Agent` header formatted as `<Vendor-Product-Version>` on every call

---

## 1. Query Client Status (`datasearch`)

Ad-hoc query for client status records within any historical time window.

**Endpoint:** `GET /api/v2/events/datasearch/clientstatus`

**Parameters:**
- `starttime` / `endtime` — Unix timestamps for the query window
- `fields` — comma-separated list of fields to return (supports dot notation for nested fields, e.g. `epdlp.cpu` — see workflow 2)
- `query` — optional Skope IT filter expression, e.g. `device_id eq '<id>'`
- `limit` — max records to return

```bash
curl -X GET "https://<tenant>.goskope.com/api/v2/events/datasearch/clientstatus?starttime=1773101400&endtime=1773187800&fields=nsdeviceuid,hostname,os,client_version" \
  -H "Authorization: Bearer <token>" \
  -H "User-Agent: Partner-RiskAssessment-1.0"
```

**Response:**
```json
{
  "result": [
    {
      "_id": "019888fb-f0eb-4d4f-806e-ac0f066201eb",
      "client_version": "135.1.10.2611",
      "nsdeviceuid": "416012FE-E4CD-860D-0DAE-A6432CB21A76",
      "hostname": "nskp-w11-joe-3",
      "os": "Windows"
    }
  ],
  "status": {
    "count": 10,
    "execution": "SUCCESS",
    "message": "Executed Successfully",
    "status_code": 200
  }
}
```

**Use case:** Look up current client status for a specific device or set of devices as part of an investigation or enrichment workflow, without needing to run a continuous export pipeline.

---

## 2. Retrieve Endpoint DLP (`epdlp`) Health Telemetry

The `clientstatus` dataset also exposes Endpoint DLP driver/service health under a nested `epdlp` object — useful for partners building device health or posture integrations. **Confirmed live against a tenant.**

Request the sub-fields individually with dot notation — requesting the bare `epdlp` field name returned an HTTP 500 error on the tenant we tested against:

```bash
curl -X GET "https://<tenant>.goskope.com/api/v2/events/datasearch/clientstatus?starttime=1773101400&endtime=1773187800&fields=device_id,hostname,epdlp.runningStatus,epdlp.enabledStatus,epdlp.criticalErrorsCount,epdlp.infoErrorsCount,epdlp.driverVersion,epdlp.serviceVersion,epdlp.cpu,epdlp.memoryInBytes,epdlp.policyConfigVersion,epdlp.timestampInUnix" \
  -H "Authorization: Bearer <token>" \
  -H "User-Agent: Partner-RiskAssessment-1.0"
```

**Response (confirmed shape, real values from a populated device):**
```json
{
  "device_id": "C8gcDwPElrB735s4jRxi_EE1A590B-D1D7-AA2B-6A06-22700F9B3A2D",
  "hostname": "API-Win-Client",
  "epdlp": {
    "cpu": 0,
    "criticalErrorsCount": 0,
    "driverVersion": "123.0.0.4377",
    "enabledStatus": "Enabled",
    "infoErrorsCount": 1417,
    "memoryInBytes": 41893888,
    "policyConfigVersion": "unknown",
    "runningStatus": "Disabled",
    "serviceVersion": "123.0.0.4378",
    "timestampInUnix": 1742498664
  }
}
```

**Field reference:**

| Field | Description |
|-------|-------------|
| `runningStatus` | Whether the Endpoint DLP driver/service is currently running |
| `enabledStatus` | Whether Endpoint DLP is enabled by policy for this device |
| `criticalErrorsCount` | Count of critical-severity errors logged by the driver/service |
| `infoErrorsCount` | Count of informational-severity errors logged |
| `driverVersion` | Installed Endpoint DLP driver version |
| `serviceVersion` | Installed Endpoint DLP service version |
| `cpu` | CPU usage reported by the Endpoint DLP process |
| `memoryInBytes` | Memory usage reported by the Endpoint DLP process, in bytes |
| `policyConfigVersion` | Version identifier of the policy configuration currently applied |
| `timestampInUnix` | Unix timestamp of when this health snapshot was captured |

**Note:** These values are commonly empty strings on devices where Endpoint DLP isn't active — that's expected, not an error. In the example above, `enabledStatus: Enabled` with `runningStatus: Disabled` shows Endpoint DLP is turned on by policy but the local driver/service isn't currently running — exactly the kind of health discrepancy this field set is designed to surface.

**Use case:** Monitor Endpoint DLP health across your fleet to catch devices where protection is configured but not actually running.

---

## 3. Set Up the Dataexport Client Status Iterator

For continuous bulk ingestion (e.g., into a SIEM), use the Client Status Iterator rather than polling `datasearch` repeatedly.

**Step 1 — Create the iterator:**

**Endpoint:** `POST /api/v2/events/dataexport/iterator/{index}?eventtype=clientstatus`

```bash
curl -X POST "https://<tenant>.goskope.com/api/v2/events/dataexport/iterator/my_integration_index?eventtype=clientstatus" \
  -H "Authorization: Bearer <token>" \
  -H "User-Agent: Partner-RiskAssessment-1.0"
```

**Response:**
```json
{"message": "Creation of the iterator, my_integration_index_<generated-suffix> is in progress. Please use the iterator status API to check the status of the iterator."}
```

**Important:** Netskope appends a generated suffix to your index name to guarantee uniqueness — use the full returned name (e.g. `my_integration_index_<suffix>`) in all subsequent calls, not the name you requested.

**Step 2 — Check iterator status:**

**Endpoint:** `GET /api/v2/events/dataexport/iterator/{index}`

```bash
curl -X GET "https://<tenant>.goskope.com/api/v2/events/dataexport/iterator/my_integration_index_<suffix>" \
  -H "Authorization: Bearer <token>"
```

**Response:**
```json
{"status": "Ready"}
```

**Step 3 — Fetch events:**

**Endpoint:** `GET /api/v2/events/dataexport/iterator/{index}/events?operation=next`

```bash
curl -X GET "https://<tenant>.goskope.com/api/v2/events/dataexport/iterator/my_integration_index_<suffix>/events?operation=next" \
  -H "Authorization: Bearer <token>"
```

**Response:** CSV, with response headers `Record_count`, `Schema_version`, and `Wait_time` (seconds to wait before the next `operation=next` call). Example header row (schema v32, confirmed live):
```
_id,as,asn,city,client_install_time,client_version,count,continent,country,device_hash,device_id,enriched,guid,heart_beat,host_info.client_version,host_info.device_make,host_info.device_model,host_info.hostname,host_info.last_update_timestamp,host_info.mac_addresses,host_info.managementID,host_info.nsdeviceuid,host_info.old_nsdeviceuid,host_info.os,host_info.os_version,host_info.serial_number,host_info.steering_config,isp,last_connected_from_private_ip,last_connected_from_public_ip,last_event_timestamp,last_seen_device_event.actor,last_seen_device_event.event,last_seen_device_event.event_details,last_seen_device_event.npa_status,last_seen_device_event.service_name,last_seen_device_event.status,last_seen_device_event.status_v2,last_seen_device_event.timestamp,latitude,longitude,organization,postal_code,pop_name,pop_pinned,region,timestamp,user_info.device_classification_status,user_info.orgkey,user_info.userkey,user_info.username,zipcode,record_type
```

**Note:** This CSV schema does **not** include `epdlp` fields — see the comparison below.

**Doc correction (confirmed live):** Netskope's public documentation lists the iterator path as `/api/v2/dataexport/iterator/...`, which 404s. The working path is `/api/v2/events/dataexport/iterator/...`, nested under `events` like every other dataexport endpoint.

**Use case:** Stream client status events continuously into a SIEM or data lake without re-querying the same records.

---

## `datasearch` vs. `dataexport`: Which Should You Use?

Client status data is available both ways, but they are **not equivalent** — pick based on what you need.

| | `datasearch` (ad-hoc query) | `dataexport` Client Status Iterator |
|---|---|---|
| **Endpoint** | `GET /api/v2/events/datasearch/clientstatus` | Create: `POST /api/v2/events/dataexport/iterator/{index}?eventtype=clientstatus`<br>Fetch: `GET /api/v2/events/dataexport/iterator/{index}/events?operation=next` |
| **Response format** | JSON | CSV |
| **Query model** | Ad-hoc — filter by any time range, any device, any field | Streaming/sequential — consume from a tracked index, oldest-to-newest |
| **Includes `epdlp` (Endpoint DLP health) fields** | ✅ Yes — confirmed live | ❌ No — confirmed absent from the schema (tested at schema version v32) |
| **History available** | Arbitrary historical window via `starttime`/`endtime` | Only the last **7 days** — older events are dropped if not consumed in time |
| **Concurrency** | Any number of independent queries | **One iterator per tenant, per event type** — creating a second fails with `"Only one iterator is allowed per event type"` until the existing one is deleted |
| **Best for** | One-off lookups / enrichment at investigation time (e.g., "get this device's current state right now") | Continuous bulk ingestion into a SIEM or data lake |

**Bottom line:** If you need Endpoint DLP health telemetry, use `datasearch` — it's the only one of the two that returns it. If you need continuous, high-volume ingestion and don't need `epdlp`, the iterator is the better fit.

---

## Best Practices

- Use `datasearch` for investigation-time lookups and `dataexport` for continuous bulk ingestion — don't poll `datasearch` on a tight loop as a substitute for the iterator
- Request `epdlp` sub-fields individually with dot notation (`epdlp.cpu`, `epdlp.runningStatus`, etc.) — never the bare `epdlp` field name
- Store the full generated iterator name (including the appended suffix) returned at creation time — you'll need it for every subsequent status/fetch call
- Consume iterator events promptly — the 7-day retention window means unconsumed events are dropped, not queued indefinitely
- Before creating a new iterator, consider whether one already exists for your event type — the platform only allows one per tenant per event type, and creating a duplicate will fail

---

## Integration Tips

### Error Handling
- Querying `clientstatus` with `fields=...,epdlp` (the bare parent field) returns HTTP 500 — always use dotted sub-field selectors instead
- Creating a second iterator for an event type that already has one returns HTTP 400 with the existing iterator's full name — use that instead of retrying with a different index name
- An invalid `eventtype` on iterator creation returns HTTP 400 with `"Invalid event type"`

### Idempotency
- The iterator index name you request gets a generated suffix appended — always read the actual name back from the create/list response rather than assuming it matches what you sent
- `operation=resend` re-delivers the last page without advancing the index, useful if your consumer failed to process a batch

### Timing
- Respect the `Wait_time` value returned with each iterator fetch before calling `operation=next` again
- `epdlp` values are commonly empty strings when Endpoint DLP isn't active on a device — don't treat this as a query error

### Testing
- Start by querying `datasearch` for a single known device before building a full iterator pipeline
- Verify your iterator reaches `"status": "Ready"` before attempting to fetch events

---

## Common Patterns

### Endpoint DLP Health Monitoring
```
1. Query datasearch/clientstatus with the epdlp.* dotted fields for your device fleet
2. Flag devices where enabledStatus is "Enabled" but runningStatus is "Disabled"
3. Flag devices with a climbing criticalErrorsCount over time
4. Route flagged devices to your EDR/helpdesk workflow for remediation
```

### SIEM Ingestion Pipeline
```
1. Create a Client Status Iterator (one-time setup, workflow 3)
2. Poll the status endpoint until status is "Ready"
3. Loop: fetch events with operation=next, process the CSV batch, wait Wait_time seconds, repeat
4. If a batch fails to process, retry it once with operation=resend before advancing
```

---

## Troubleshooting

**Problem:** Querying `clientstatus` with `fields=...,epdlp` returns an HTTP 500 error
- **Solution:** Request the `epdlp` sub-fields individually with dot notation (`epdlp.cpu`, `epdlp.runningStatus`, etc.) instead of the bare parent field name.

**Problem:** `epdlp` fields are missing when pulling client status via the dataexport iterator
- **Solution:** Expected — the dataexport Client Status Iterator schema does not include `epdlp` fields. Use `datasearch` (workflow 1–2) for Endpoint DLP health telemetry.

**Problem:** Creating an iterator fails with `"Only one iterator is allowed per event type"`
- **Solution:** An iterator for that event type already exists (the error message includes its name). Use the existing iterator, or delete it first if you need to reset its position.

**Problem:** Iterator creation returns `"Invalid event type"`
- **Solution:** Double-check the `eventtype` value — only specific Netskope-defined event types are accepted. `clientstatus` is confirmed valid for this guide's purposes.

**Problem:** `operation=<epoch timestamp>` returns `"Invalid iterator operation. Please use next or resend."` on the Client Status Iterator
- **Solution:** Epoch-timestamp seeding is not supported on this iterator's events endpoint — only `next` and `resend` are valid. A freshly created iterator starts streaming from its creation time forward.

---

## API Reference

| Workflow | Endpoint | Method | Purpose |
|----------|----------|--------|---------|
| 1 | `/api/v2/events/datasearch/clientstatus` | GET | Query client status (ad-hoc) |
| 2 | `/api/v2/events/datasearch/clientstatus` (with `epdlp.*` fields) | GET | Retrieve Endpoint DLP health telemetry |
| 3a | `/api/v2/events/dataexport/iterator/{index}?eventtype=clientstatus` | POST | Create a Client Status Iterator |
| 3b | `/api/v2/events/dataexport/iterator/{index}` | GET | Check iterator status |
| 3c | `/api/v2/events/dataexport/iterator/{index}/events` | GET | Fetch the next batch of events (CSV) |

**Note:** The dataexport iterator path is nested under `/events/`, not directly under `/api/v2/` as Netskope's public documentation states — use the path shown above.

---

## Support & Additional Resources

- **REST API v2 overview:** https://docs.netskope.com/en/netskope-help/admin-console/rest-api/rest-api-v2-overview-312207/
- **Related guide:** [Device-Tags-for-Classification.md](Device-Tags-for-Classification.md) — uses `datasearch/clientstatus` to resolve `nsdeviceuid` for device tagging
- **Questions?** Contact your Netskope account team for API support.
