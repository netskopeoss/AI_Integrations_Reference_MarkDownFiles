# Custom URL Lists Integration Guide

Populate destination-based blocklists and allowlists for policy enforcement. Use these workflows to create URL lists, manage malware/C2 infrastructure, integrate threat feeds, and block phishing URLs in real-time.

---

## Overview

Use these endpoints to:
- Create and manage custom URL lists
- Add malicious URLs from threat feeds
- Block phishing, C2, or malware infrastructure
- Replace or append URLs to existing lists
- Deploy changes to activate policies

**Prerequisites:**
- NG-SWG or CASB subscription
- Pre-configured policies that reference custom URL lists
- Lists must be referenced in policies to take effect

**Constraints:**
- Max 300,000 items per tenant across all lists (items must be removed to free space)
- Individual uploads must be < 16 MB
- Wildcard and regex not supported (except leading wildcards: `*.url.domain`)
- List names must exactly match the names in your real-time policies

---

## 1. Retrieve Existing URL Lists

Get names of all custom URL lists to identify write-enabled lists and avoid duplicates.

**Endpoint:** `GET /api/v2/policy/urllist`

```bash
# Get list names only
curl -X GET \
  'https://<customer-tenant>.goskope.com/api/v2/policy/urllist?pending=0&field=name' \
  -H 'accept: application/json'

# Or get all list data
curl -X GET \
  'https://<customer-tenant>.goskope.com/api/v2/policy/urllist?pending=0' \
  -H 'accept: application/json'
```

**Response:**
```json
[
  {
    "name": "Form Post Sites"
  },
  {
    "name": "xdr-threats"
  },
  {
    "name": "phishing-urls"
  },
  {
    "name": "Demo1"
  }
]
```

**Parameters:**
- `pending=0` — Get active lists
- `pending=1` — Get pending/draft lists
- `field=name` — Return only name field

**Use case:** Check if your list already exists before creating a new one.

---

## 2. Count Items and Check Capacity

Retrieve all URLs in each custom URL list to confirm available space before adding new entries. Tenant can have max 300,000 items total.

**Endpoint:** `GET /api/v2/policy/urllist`

```bash
curl -X GET \
  'https://<customer-tenant>.goskope.com/api/v2/policy/urllist' \
  -H 'accept: application/json'
```

**Response (partial):**
```json
[
  {
    "id": 1,
    "name": "Form Post Sites",
    "data": {
      "urls": [
        "*.dlptest.com",
        "*.dataleaktest.com",
        "dlptest.com",
        "dataleaktest.com"
      ],
      "type": "exact"
    },
    "modify_by": "admin@netskope.com",
    "modify_time": "2020-07-31T13:46:42.000Z",
    "modify_type": "Created",
    "pending": 0
  }
]
```

**Use case:** Before large bulk uploads, verify there's enough capacity (300,000 - current_count).

---

## 3. Create New Custom URL List

Create a new list and populate with initial URLs. List will be in "pending" status until deployed.

**Endpoint:** `POST /api/v2/policy/urllist`

```bash
curl -X POST \
  'https://<customer-tenant>.goskope.com/api/v2/policy/urllist' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Netskope-Api-Token: <token>' \
  -d '{
    "name": "xdr-threats",
    "data": {
      "urls": [
        "malware.example.com",
        "c2-server.io",
        "phishing.malicious.com"
      ],
      "type": "exact"
    }
  }'
```

**Response:**
```json
[
  {
    "id": 0,
    "name": "xdr-threats",
    "data": {
      "urls": [
        "malware.example.com",
        "c2-server.io",
        "phishing.malicious.com"
      ],
      "type": "exact"
    },
    "modify_type": "Created",
    "modify_by": "Netskope API",
    "modify_time": "2024-01-01 00:00:00",
    "pending": 0
  }
]
```

**Parameters:**
- `name` — List name (must match names in policies)
- `data.urls` — Array of URLs
- `data.type` — `exact` (literal match) or other types

**Use case:** Create a new list for your threat feed or incident response workflow.

---

## 4. Add or Replace URLs in Existing List

Append new URLs to an existing list or replace the entire contents.

**Endpoint:** `PATCH /api/v2/policy/urllist/{listId}/append` or `/replace`

```bash
# Append URLs (keep existing)
curl -X PATCH \
  'https://<customer-tenant>.goskope.com/api/v2/policy/urllist/1/append' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Netskope-Api-Token: <token>' \
  -d '{
    "name": "xdr-threats",
    "data": {
      "urls": [
        "new-malware.com",
        "new-c2.io"
      ],
      "type": "exact"
    }
  }'

# Replace URLs (overwrite entire list)
curl -X PATCH \
  'https://<customer-tenant>.goskope.com/api/v2/policy/urllist/1/replace' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Netskope-Api-Token: <token>' \
  -d '{
    "name": "xdr-threats",
    "data": {
      "urls": [
        "fresh-list.com"
      ],
      "type": "exact"
    }
  }'
```

**Response:**
```json
{
  "id": 1,
  "name": "xdr-threats",
  "data": {
    "urls": [
      "malware.example.com",
      "c2-server.io",
      "phishing.malicious.com",
      "new-malware.com",
      "new-c2.io"
    ],
    "type": "exact",
    "json_version": 2
  },
  "modify_by": "Netskope API",
  "modify_time": "2024-01-01",
  "modify_type": "Edited",
  "pending": 1
}
```

**Use case:** 
- **Append:** Add new threats as they're discovered
- **Replace:** Daily refresh from your threat feed

**Note:** Changes are marked `pending: 1` until deployed (workflow 5).

---

## 5. Deploy URL List Changes

Apply pending changes to all custom URL lists. This activates them for policy matching.

**Endpoint:** `POST /api/v2/policy/urllist/deploy`

```bash
curl -X POST \
  'https://<customer-tenant>.goskope.com/api/v2/policy/urllist/deploy' \
  -H 'accept: application/json' \
  -H 'Netskope-Api-Token: <token>'
```

**Response:**
```json
[
  {
    "id": 1,
    "name": "xdr-threats",
    "data": {
      "urls": [
        "malware.example.com",
        "c2-server.io",
        "phishing.malicious.com",
        "new-malware.com"
      ],
      "type": "exact",
      "json_version": 2
    },
    "modify_by": "Netskope API",
    "modify_time": "2024-01-01T00:00:00.000Z",
    "modify_type": "Edited",
    "pending": 0
  }
]
```

**Use case:** After adding/replacing URLs, deploy to activate them in policies.

---

## Integration Tips

### Threat Feed Integration

Typical workflow for integrating external threat feeds:

```bash
# Step 1: Check if list exists
GET /api/v2/policy/urllist?pending=0&field=name

# Step 2: If not, create list (workflow 3)
POST /api/v2/policy/urllist

# Step 3: Daily: Get latest threats from feed
# (your internal feed ingestion)

# Step 4: Replace list with fresh URLs (workflow 4)
PATCH /api/v2/policy/urllist/{listId}/replace

# Step 5: Deploy changes (workflow 5)
POST /api/v2/policy/urllist/deploy
```

### Capacity Management

Monitor capacity to avoid hitting the 300,000 item limit:

```bash
# Step 1: Get all lists and count URLs
GET /api/v2/policy/urllist

# Step 2: Calculate total_count = sum of all urls
total_urls = sum(len(list['data']['urls']) for list in response)
available = 300000 - total_urls

# Step 3: If available < 10000, remove old lists or archive
```

### Rate Limiting

Each endpoint has a 4 request/second limit. For large uploads:
- Split into multiple append operations
- Space out requests (250ms between each)
- Monitor for HTTP 429 responses

---

## Error Handling

**Problem:** "List not found" error
- **Solution:** Verify list ID from workflow 1. List may be pending or deleted.

**Problem:** "Capacity exceeded" error
- **Solution:** 300,000 item limit reached. Remove items from other lists or delete old lists.

**Problem:** Uploaded file is too large (> 16 MB)
- **Solution:** Split URLs into multiple requests using append (workflow 4).

**Problem:** URLs not matching in policies
- **Solution:** Verify list name exactly matches the name referenced in your policy. Case-sensitive.

---

## Common Patterns

### Real-Time Malware Blocking
```
1. Malware detection system identifies new C2 domain
2. Check capacity (workflow 2)
3. Append domain to "malware-c2" list (workflow 4)
4. Deploy changes (workflow 5)
5. Users accessing domain are blocked (policy triggers)
```

### Daily Threat Feed Sync
```
1. Fetch latest URLs from external threat feed
2. List contains 50,000 URLs
3. Replace "daily-threats" list (workflow 4)
4. Deploy changes (workflow 5)
5. Previous day's URLs are removed, new ones added
```

### Phishing Detection Integration
```
1. Phishing detection tool identifies suspicious URLs
2. Check if list "phishing-urls" exists (workflow 1)
3. If not, create (workflow 3)
4. Append new phishing URLs (workflow 4)
5. Deploy (workflow 5)
6. Policy blocks users from accessing phishing sites
```

---

## API Reference

| Workflow | Endpoint | Method | Purpose |
|----------|----------|--------|---------|
| 1 | `/api/v2/policy/urllist` | GET | List all URL lists |
| 2 | `/api/v2/policy/urllist` | GET | Get URL list contents |
| 3 | `/api/v2/policy/urllist` | POST | Create new URL list |
| 4 | `/api/v2/policy/urllist/{id}/append` | PATCH | Add URLs to list |
| 4b | `/api/v2/policy/urllist/{id}/replace` | PATCH | Replace list contents |
| 5 | `/api/v2/policy/urllist/deploy` | POST | Deploy changes |
