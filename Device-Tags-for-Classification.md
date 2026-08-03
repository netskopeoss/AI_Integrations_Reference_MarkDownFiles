# Device Tags for Classification Integration Guide

Programmatically manage device tags and device classifications within your Netskope environment. Use these workflows to organize devices by custom risk assessment, device type, department, or any other business classification criteria, then drive real-time policy enforcement off those tags.

---

## Overview

Use these endpoints to:
- View existing device tags and classification rules
- Create new device tags and classification tags/rules
- Look up devices and retrieve their current tags
- Apply tags to one or more devices in bulk
- Manage tag lifecycle (update, remove) over time

**Prerequisites:**
- **API Access:** The Device Classification APIs are currently in **beta**. Contact your Netskope account team to request access.
- **API Token:** Generate a token with the following functional area permissions:
  - `Access Control > NS Client` — set both **Device Classification** and **Devices** to **Manage**
  - `CASB > CCI` — set to **View**
- **User-Agent header:** Include a user-agent string formatted as `<Vendor-Product-Version>` in all API requests.

---

## Device Classifications vs. Device Tags

These are two related but distinct systems — most integrations use both together.

### Device Classifications
- **Purpose:** Automatic, rule-based classification
- **How it works:** You create rules that automatically apply a classification label to devices based on conditions (OS type, presence of other tags, AV status, etc.)
- **Applied by:** The rules engine (automatic)
- **Use case:** Static classification based on device characteristics
- **Example IDs:** `18006` ("low risk"), `16341` ("medium risk"), `7841` ("Risky Device")

### Device Tags
- **Purpose:** Manual, explicit tagging for direct device assignment
- **How it works:** You directly assign tags to specific devices via API calls
- **Applied by:** Direct API calls (manual)
- **Use case:** Dynamic tagging based on real-time risk assessment from third-party tools
- **Example IDs:** `1137` ("low risk"), `912` ("Medium Risk")

**Typical pattern:** Your integration writes a **Device Tag** to a device based on a real-time finding, then a pre-configured **Device Classification Rule** checks for that tag (`device_tag_check`) and promotes it into a classification that Netskope policy can match on.

---

## Things to Note

### Batch Processing & Performance
- Devices are processed in batches of **100** (`TAG_DEVICE_BATCH_SIZE`)
- Larger device sets are split into multiple sequential batches
- Each batch triggers a separate Netskope API call

### Tag Validation & Constraints
- **Maximum tag length:** 80 characters
- **Maximum tags per device:** 5 (Netskope platform limit) — for a Replace action, only the first 5 sorted tags are applied
- **Tag name format:** alphanumeric characters, hyphens, and spaces only

### Comma-Separated Value Handling
- **Tags** and **User Key** — split and processed as individual values
- **Device UID** — comma-separated values are rejected
- Empty values after splitting are detected and rejected with validation errors

---

## 1. View Existing Device Tags

Retrieve all device classification tags in your Netskope environment.

**Endpoint:** `GET /api/v2/deviceclassification/tags`

```bash
curl -X GET "https://<tenant>.goskope.com/api/v2/deviceclassification/tags" \
  -H "Authorization: Bearer <token>" \
  -H "User-Agent: Partner-RiskAssessment-1.0"
```

**Response:**
```json
[
  {
    "id": 8505,
    "priority": 7,
    "name": "Low Risk",
    "description": "the risk of this device is low",
    "modifiedBy": "test@test.com",
    "modifiedTime": "2025-07-28T08:48:14.000Z",
    "policyNames": ["Low-Risk device"]
  }
]
```

**Use case:** Check what classification tags already exist (and which policies reference them) before creating new ones.

---

## 2. Create a New Device Classification Tag

Create a new classification tag (label). The request body must be an **array of tag objects**.

**Endpoint:** `POST /api/v2/deviceclassification/tags`

```bash
curl -X POST "https://<tenant>.goskope.com/api/v2/deviceclassification/tags" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -H "User-Agent: Partner-RiskAssessment-1.0" \
  -d '[
    {
      "name": "low risk",
      "description": "Devices with low risk classification"
    }
  ]'
```

**Response:**
```json
{
  "status": true,
  "data": [18006]
}
```

**Important:** A newly created classification tag has no rule or OS association yet — it returns success but won't apply to any devices until you create a classification rule (workflow 3) that references it by name via `label`.

---

## 3. Create a Device Classification Rule

Rules define how devices are automatically classified based on criteria such as OS version or the presence of a device tag. The request body must be an **array of rule objects**, and typically **two POSTs** are needed — one for OS criteria, one for the device tag criteria.

**Endpoint:** `POST /api/v2/deviceclassification/rules`

**Request 1 — OS criteria:**
```bash
curl -X POST "https://<tenant>.goskope.com/api/v2/deviceclassification/rules" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -H "User-Agent: Partner-RiskAssessment-1.0" \
  -d '[
    {
      "name": "low risk rule - Windows All",
      "label": "low risk",
      "os": "windows",
      "conditions": {
        "$and": [
          {
            "$and": [
              {
                "$or": [
                  {
                    "min_os_version_check": {
                      "min_os_version": "0",
                      "edition": "Windows All"
                    }
                  }
                ]
              }
            ]
          }
        ]
      }
    }
  ]'
```

**Request 2 — Device tag criteria (required in addition to Request 1):**
```bash
curl -X POST "https://<tenant>.goskope.com/api/v2/deviceclassification/rules" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -H "User-Agent: Partner-RiskAssessment-1.0" \
  -d '[
    {
      "name": "low risk tag rule",
      "label": "low risk",
      "os": "windows",
      "conditions": {
        "$and": [
          {
            "$and": [
              {
                "$and": [
                  {"device_tag_check": {"tag_id": 1137}}
                ]
              }
            ]
          }
        ]
      }
    }
  ]'
```

**Response (both requests):**
```json
{
  "status": true,
  "data": [42]
}
```

**Note:** The `tag_id` in the second request refers to a **device tag** ID (not a classification tag ID) — use `/api/v2/devices/device/tags/gettags` to look it up (workflow 6).

**Important:**
- Request body must be an array of rule objects
- Use `label` with the tag *name* (not the tag ID)
- Use `os` to specify the operating system: `windows`, `mac`, `linux`, `android`, `ios`
- Conditions support multiple criteria types: `min_os_version_check`, `device_tag_check`, `av_check`, `domain_check`, `file_check`, etc.
- For OS edition criteria, use `min_os_version_check` with edition values like `"Windows All"`, `"Windows 10"`, `"Windows 11"`
- HTTP 201 indicates successful rule creation

---

## 4. Find Devices by Status Query

Before applying tags, look up the target device's `nsdeviceuid` using the client status search endpoint.

**Endpoint:** `GET /api/v2/events/datasearch/clientstatus`

**Parameters:**
- `starttime` / `endtime` — Unix timestamps for the query window
- `fields` — comma-separated list of fields to return

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
    },
    {
      "_id": "fc18ce47-e4f7-429c-9345-7853fe29ffbe",
      "client_version": "135.1.10.2611",
      "nsdeviceuid": "AB2E7066-747D-8728-71F9-6163532C2BD0",
      "hostname": "Jenga-Surface",
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

**Use case:** Resolve a hostname or user to the `nsdeviceuid` required by every device-tagging call below.

---

## 5. Create a Device Tag

Register a new device tag. If the tag might already exist, check first with workflow 6 to avoid creating a duplicate.

**Endpoint:** `POST /api/v2/devices/device/tags`

```bash
curl -X POST "https://<tenant>.goskope.com/api/v2/devices/device/tags" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -H "User-Agent: Partner-RiskAssessment-1.0" \
  -d '{
    "name": "low risk",
    "description": "Devices with low risk classification"
  }'
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": 1137,
    "name": "low risk",
    "description": "Devices with low risk classification"
  }
}
```

**Use case:** Pre-create the set of tags your integration will need (e.g., `low risk`, `medium risk`, `compromised`) before your response workflows run.

---

## 6. Find Existing Device Tags

Look up every device tag defined in the tenant, with IDs and names. There's no direct "list all tags" endpoint — query any device's tags and the response includes the full tag catalog.

**Endpoint:** `POST /api/v2/devices/device/tags/gettags`

```bash
curl -X POST "https://<tenant>.goskope.com/api/v2/devices/device/tags/gettags" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -H "User-Agent: Partner-RiskAssessment-1.0" \
  -d '{"device_id": "AB2E7066-747D-8728-71F9-6163532C2BD0"}'
```

**Response:**
```json
{
  "data": [
    {"id": 1137, "name": "low risk"},
    {"id": 912, "name": "Medium Risk"},
    {"id": 7841, "name": "Risky Device"}
  ]
}
```

**Use case:** Resolve a tag name to its ID before calling bulk replace (workflow 7), or before referencing it in a classification rule's `device_tag_check` (workflow 3).

---

## 7. Apply Tags to Devices

Apply one or more tags to one or more devices with the bulk replace endpoint. **This replaces all tags** on the specified devices with the tags provided — it is not additive.

**Endpoint:** `POST /api/v2/devices/device/tags/bulkreplace`

```bash
curl -X POST "https://<tenant>.goskope.com/api/v2/devices/device/tags/bulkreplace" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -H "User-Agent: Partner-RiskAssessment-1.0" \
  -d '{
    "tags": [1137],
    "devices": [
      {
        "nsdeviceuid": "AB2E7066-747D-8728-71F9-6163532C2BD0",
        "userkey": "AB2E7066-747D-8728-71F9-6163532C2BD0",
        "hostname": "Jenga-Surface"
      }
    ],
    "device_classifications": []
  }'
```

**Parameters:**
- `tags` — array of device tag IDs to apply
- `devices` — array of device objects with `nsdeviceuid` and `userkey`
- `device_classifications` — array of device classification IDs (use an empty array when applying device tags only)

**Response:**
```json
{
  "success": true,
  "data": {
    "affected_device_tags": 1,
    "affected_device_classification_tags": 0,
    "message": "Tags replaced successfully"
  }
}
```

**Use case:** In response to an XDR/EDR finding, write a risk tag to the affected device(s). A pre-configured classification rule (workflow 3) then promotes the tag into a classification that real-time policy can act on.

---

## Verifying Enforcement

The API calls above create and apply tags/classifications, but two pieces of setup happen outside the API, in the Netskope UI, and are required before any policy action actually fires:

1. **Classification rule must exist and reference the tag** (workflow 3) — without it, a device tag alone does not become a classification.
2. **A Real-Time Policy must be configured to match on the resulting classification** — go to **Policies > Real-time Protection**, create or edit a policy, and add a condition that matches your classification/tag, then choose an enforcement action (Alert, Block, etc.). Netskope does not expose an API to create or modify these policies; they must be pre-configured by the tenant admin.

To confirm a tag or classification took effect without relying on the admin console, poll the same endpoints you used to set it:
- Re-run workflow 6 (`gettags`) for the device to confirm the tag is present
- Re-run workflow 1 (`deviceclassification/tags`) to confirm the classification's `policyNames` field is populated, which indicates a policy references it

**Example end-to-end sequence** (create a "medium risk" tag, apply it, and confirm):
```
1. POST /api/v2/deviceclassification/tags   → create "medium risk" classification tag (workflow 2)
2. POST /api/v2/deviceclassification/rules  → create rule linking device_tag_check to the tag (workflow 3)
3. GET  /api/v2/events/datasearch/clientstatus → resolve target device's nsdeviceuid (workflow 4)
4. POST /api/v2/devices/device/tags          → create the underlying device tag if needed (workflow 5)
5. POST /api/v2/devices/device/tags/bulkreplace → apply the tag to the device (workflow 7)
6. POST /api/v2/devices/device/tags/gettags  → confirm the tag is now present on the device (workflow 6)
```
Enforcement then depends on a Real-Time Policy (configured in the UI) matching on that classification.

---

## Best Practices

- Use meaningful names and descriptions for tags to ensure clarity across your organization
- Establish a priority numbering scheme that aligns with your risk assessment framework (lower numbers = higher priority)
- Test tag creation and application in a non-production environment before deploying to production
- Always include a `User-Agent` header in API requests for proper tracking and support
- Implement error handling and retry logic for API calls to ensure robustness
- Use PATCH requests for partial updates when you only need to modify specific fields

---

## Integration Tips

### Error Handling
- Check for HTTP 429 (rate limit exceeded) and back off exponentially
- Validate API tokens and functional area permissions before bulk operations
- Confirm a classification rule exists before assuming a device tag will affect policy

### Idempotency
- Query existing tags (workflow 6) before creating new ones to avoid duplicates
- Remember `bulkreplace` overwrites all tags on a device — fetch current tags first if you need to preserve any

### Batching & Timing
- Split device lists into batches of 100 before calling `bulkreplace`
- Classification rule changes and tag assignments should be verified by polling, since there is no webhook/event for tag application

### Testing
- Start with a single test device and tag before running bulk operations
- Verify the classification rule was created correctly (workflow 1) before relying on it in production
- Use a non-production tenant if available

---

## Common Patterns

### Real-Time Risk Tagging
```
1. Third-party tool detects elevated device risk
2. Look up device tag ID (workflow 6) or create one if it doesn't exist (workflow 5)
3. Find the device's nsdeviceuid (workflow 4)
4. Apply the tag via bulkreplace (workflow 7)
5. Pre-configured classification rule promotes the tag into a classification
6. Real-time policy matches on the classification and enforces (e.g., alert, block)
```

### Batch Device Tagging
```
1. Query devices in scope (workflow 4)
2. Split device list into batches of 100 (platform limit)
3. For each batch, call bulkreplace (workflow 7)
4. Monitor affected_device_tags count in each response
```

---

## Troubleshooting

**Problem:** Tag applied successfully but no policy action occurs
- **Solution:** Confirm a classification rule exists that references the tag (workflow 3), and that a Real-Time Policy in the Netskope UI matches on that classification. Tags alone don't trigger enforcement.

**Problem:** `bulkreplace` removed tags I expected to keep
- **Solution:** This endpoint replaces the full tag set on a device. Fetch current tags first (workflow 6) and include them in the `tags` array alongside the new tag.

**Problem:** Classification rule creation succeeds but the classification never applies
- **Solution:** Verify both required rules were created — one for OS criteria and one for the `device_tag_check` criteria (workflow 3 requires both).

**Problem:** "Maximum tags per device exceeded" or unexpected tags dropped
- **Solution:** Netskope allows a maximum of 5 tags per device. For a replace action, only the first 5 sorted tags are applied — reduce the tag count or split by tag priority.

**Problem:** Comma-separated device UID request rejected
- **Solution:** Unlike tags and user keys, `nsdeviceuid` does not support comma-separated batching in a single object — submit one device object per entry in the `devices` array instead.

---

## API Reference

| Workflow | Endpoint | Method | Purpose |
|----------|----------|--------|---------|
| 1 | `/api/v2/deviceclassification/tags` | GET | View existing classification tags |
| 2 | `/api/v2/deviceclassification/tags` | POST | Create a classification tag |
| 3 | `/api/v2/deviceclassification/rules` | POST | Create a classification rule |
| 4 | `/api/v2/events/datasearch/clientstatus` | GET | Find devices / get nsdeviceuid |
| 5 | `/api/v2/devices/device/tags` | POST | Create a device tag |
| 6 | `/api/v2/devices/device/tags/gettags` | POST | Find existing device tags |
| 7 | `/api/v2/devices/device/tags/bulkreplace` | POST | Apply tags to devices |

**Note:** The Device Classification API is currently in **beta** — contact your Netskope account team for access.

---

## Support & Additional Resources

- **Netskope Help Center:** https://docs.netskope.com
- **Questions?** Contact your Netskope account team for API support and troubleshooting.
