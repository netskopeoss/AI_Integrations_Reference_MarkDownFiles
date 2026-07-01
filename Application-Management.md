# Application Management Integration Guide

Tag and label cloud applications for policy enforcement. Use these workflows to identify apps in use, extract risk scores, apply custom labels, and manage application instances based on XDR findings.

---

## Overview

Use these endpoints to:
- Identify all SaaS applications in use at customer's site
- Extract Cloud Confidence Index (CCI) scores and existing labels
- Query existing tags for applications
- Apply custom labels (e.g., "compromised", "under-investigation")
- Create new tags and manage application instances

**Prerequisites:**
- NG-SWG or CASB subscription (for CCI database access)
- Pre-configured policies that match on application labels
- Application names or IDs from event logs

---

## 1. Determine Applications in Use

Extract application names and IDs from event logs to identify which apps are active in the customer's environment. Use this as a foundation for app risk assessment.

**Endpoint:** `GET /api/v2/events/dataexport/events/application`

```bash
curl -X GET \
  'https://<customer-tenant>.goskope.com/api/v2/events/dataexport/events/application?operation=1658502533' \
  -H 'accept: application/json'
```

**Response:**
```json
{
  "result": [
    {
      "_id": "event-123",
      "app": "Salesforce",
      "app_id": "42",
      "activity": "Login",
      "user": "john.doe@company.com",
      "timestamp": 1654823213000
    },
    {
      "_id": "event-124",
      "app": "Slack",
      "app_id": "156",
      "activity": "File Upload",
      "user": "jane.smith@company.com",
      "timestamp": 1654823220000
    }
  ]
}
```

**Use case:** Inventory all SaaS applications and their usage patterns before applying labels or restrictions.

---

## 2. Extract CCI Score and Labels

Retrieve Cloud Confidence Index (CCI) scores and existing labels for applications in use. CCI ranges from 1 (poor) to 100 (excellent).

**Endpoint:** `GET /api/v2/events/dataexport/events/application`

**Key fields in response:**
- `cci`: Cloud Confidence Index (1–100)
- `ccl`: Cloud Confidence Level (category string)
- Custom labels already applied

```bash
curl -X GET \
  'https://<customer-tenant>.goskope.com/api/v2/events/dataexport/events/application?operation=1658502533' \
  -H 'accept: application/json'
```

**Sample response (excerpt):**
```json
{
  "result": [
    {
      "app": "Salesforce",
      "app_id": "42",
      "cci": 92,
      "ccl": "High",
      "activity": "Login",
      "user": "john.doe@company.com"
    }
  ]
}
```

**Use case:** Understand Netskope's risk assessment for each app before deciding on labels or policies.

---

## 3. Query Existing Tags for Applications

Get all tags currently applied to specified applications by name or ID. This helps avoid duplicate tags and understand what's already labeled.

**Endpoint:** `GET /api/v2/services/cci/tags`

```bash
# By application names (semicolon-separated, no spaces, max 100)
curl -X GET \
  'https://<customer-tenant>.goskope.com/api/v2/services/cci/tags?apps=Box%3BAmazon%20Database' \
  -H 'accept: application/json'

# By application IDs (semicolon-separated, max 100)
curl -X GET \
  'https://<customer-tenant>.goskope.com/api/v2/services/cci/tags?ids=4%3B7%3B11' \
  -H 'accept: application/json'
```

**Response:**
```json
{
  "data": {
    "Box": {
      "app_type": "Enterprise",
      "id": 4,
      "sanctioned": "No",
      "tags": ["preferred"]
    },
    "24SevenOffice": {
      "app_type": "Departmental",
      "id": 7,
      "sanctioned": "No",
      "tags": ["contract"]
    },
    "Slack": {
      "app_type": "Enterprise",
      "id": 156,
      "sanctioned": "Yes",
      "tags": ["approved", "collaboration"]
    }
  },
  "status": "Success",
  "status_code": 200
}
```

**Use case:** Before applying new labels, check what's already in place to avoid duplicates and understand the app's approval status.

---

## 4. Add or Modify Tags on Applications

Append or remove tags from applications (max 100 at a time). Use this to add existing tag names to apps you're investigating.

**Endpoint:** `PATCH /api/v2/services/cci/tags/{tag}`

```bash
curl -X PATCH \
  'https://<customer-tenant>.goskope.com/api/v2/services/cci/tags/approved' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Netskope-Api-Token: <token>' \
  -d '{
    "apps": ["Box", "Amazon Database"],
    "action": "append"
  }'
```

**Response:**
```json
{
  "tag": "approved",
  "action": "append",
  "apps": ["Box", "Amazon Database"],
  "message": "Tag updated successfully to the list of apps/ids.",
  "status": "Success",
  "status_code": 200
}
```

**Parameters:**
- `tag`: Tag name to add/remove
- `action`: `append` (add tag) or `remove` (remove tag)
- `apps`: Array of app names OR `ids` array of app IDs

**Use case:** Quickly apply pre-existing tags like "approved", "contract", "preferred" to multiple apps.

---

## 5. Create New Custom Tags

Create and apply new labels to applications based on your XDR findings. Netskope recommends pre-creating a set of tags ready for use in workflows (e.g., "compromised", "under-investigation", "suspicious").

**Endpoint:** `POST /api/v2/services/cci/tags`

```bash
curl -X POST \
  'https://<customer-tenant>.goskope.com/api/v2/services/cci/tags' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Netskope-Api-Token: <token>' \
  -d '{
    "tag": "under-investigation",
    "apps": ["24SevenOffice", "Campfire"],
    "ids": [7, 11]
  }'
```

**Response:**
```json
{
  "tag": "under-investigation",
  "apps": ["24SevenOffice", "Campfire"],
  "message": "Tag created successfully and associated to the list of apps/ids.",
  "status": "Success",
  "status_code": 200
}
```

**Parameters:**
- `tag`: New tag name (string)
- `apps`: Array of app names (optional)
- `ids`: Array of app IDs (optional)

**Recommended Tag Names:**
- `compromised` — App known to be breached or compromised
- `under-investigation` — App suspected of malicious activity
- `suspicious` — App showing abnormal behavior
- `recommend-block` — Recommended for blocking
- `monitor-only` — Monitor for now, don't block

**Use case:** In response to XDR findings, create a new tag for the app and associate it. Pre-configured policies then match on this tag.

---

## Integration Tips

### Creating Tags in Advance

Best practice: Create a set of standard tags before running integrations:

```bash
# Create "compromised" tag (empty initially)
curl -X POST \
  'https://<customer-tenant>.goskope.com/api/v2/services/cci/tags' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Netskope-Api-Token: <token>' \
  -d '{
    "tag": "compromised",
    "apps": []
  }'

# Create "under-investigation" tag
curl -X POST \
  'https://<customer-tenant>.goskope.com/api/v2/services/cci/tags' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Netskope-Api-Token: <token>' \
  -d '{
    "tag": "under-investigation",
    "apps": []
  }'
```

### Batch Tagging

Query all apps, then tag relevant ones in bulk:

```bash
# Step 1: Get apps in use from event logs
# Step 2: For each app, query existing tags (workflow 3)
# Step 3: If tag doesn't exist, create it (workflow 5)
# Step 4: Add app to tag (workflow 4)
```

### Policy Matching

Ensure customer has policies configured that match on your custom tags. Example policy:
- **Trigger:** Application is labeled "compromised"
- **Action:** Block access or require justification

---

## Error Handling

**Problem:** Tag creation fails with "Tag already exists"
- **Solution:** Use workflow 4 (append) instead to add apps to the existing tag.

**Problem:** App not found in response
- **Solution:** Verify app ID or name is correct. Check event logs for exact app spelling.

**Problem:** Tag not showing in policies
- **Solution:** Confirm customer created policies that match on this tag. Tags only take effect when referenced in a policy.

---

## Common Patterns

### Incident Response Tagging
```
1. XDR detects suspicious SaaS app (e.g., unusual file uploads to Dropbox)
2. Query apps in use (workflow 1)
3. Check existing tags for Dropbox (workflow 3)
4. If not already tagged, create "suspicious" tag (workflow 5)
5. Add Dropbox to "suspicious" tag (workflow 4)
6. Pre-configured policy blocks or alerts on "suspicious" apps
7. User tries to access Dropbox, policy triggers
```

### Breach Response
```
1. Third-party announces Slack breach
2. Create "compromised" tag (workflow 5)
3. Add Slack to "compromised" tag (workflow 4)
4. Pre-configured policy blocks all Slack access
5. Users receive message: "IT does not approve this application"
```

### Inventory & Assessment
```
1. Query all apps in use (workflow 1)
2. For each app:
   a. Get CCI score and existing labels (workflow 2)
   b. Check for custom tags (workflow 3)
3. Build dashboard of app risk
4. Tag high-risk apps with "under-investigation"
```

---

## API Reference

| Workflow | Endpoint | Method | Purpose |
|----------|----------|--------|---------|
| 1 | `/api/v2/events/dataexport/events/application` | GET | Get apps in use |
| 2 | `/api/v2/events/dataexport/events/application` | GET | Extract CCI scores |
| 3 | `/api/v2/services/cci/tags` | GET | Query existing tags |
| 4 | `/api/v2/services/cci/tags/{tag}` | PATCH | Add/remove tag from apps |
| 5 | `/api/v2/services/cci/tags` | POST | Create new tag |
