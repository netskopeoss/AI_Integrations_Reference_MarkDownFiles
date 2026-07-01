# User Risk Management Integration Guide

Manage User Confidence Index (UCI) scores, detect behavioral anomalies, and restrict high-risk users through identity group management. This guide covers all workflows for XDR correlation, risk response automation, and user isolation.

---

## Overview

Use these endpoints to:
- Retrieve User Behavior Analytics (UBA) alerts
- Get current UCI scores for single or multiple users
- Reduce UCI scores in response to security findings
- Move high-risk users to restricted groups
- Revert false positives

**Prerequisites:**
- Advanced UBA subscription (for UCI score modifications)
- Pre-configured policies that trigger on UCI score ranges
- SCIM client for group management (all inline customers)

---

## 1. Retrieve UBA Alerts (RESTful API v2)

Fetch all User Behavior Analytics alerts. Use this if you're not ingesting logs via a SIEM, or if you need to filter by timestamp or user.

**Note:** These logs do NOT include UCI scores; if no UBA alerts exist, UCI is not configured.

**Endpoint:** `GET /api/v2/events/data/alert`

```bash
curl -X GET \
  'https://<customer-tenant>.goskope.com/api/v2/events/data/alert?type=uba&limit=5000&offset=0&starttime=0&insertionstarttime=0' \
  -H 'accept: application/json' \
  -H 'Netskope-Api-Token: <token>'
```

**Response:**
```json
{
  "ok": 1,
  "result": [
    {
      "access_method": "string",
      "alert": "Unusual Multiple File Downloads",
      "alert_name": "string",
      "user": "john.doe@company.com",
      "user_id": "user-123",
      "activity": "Download",
      "timestamp": 1654823213000
    }
  ]
}
```

**Use case:** Monitor incoming UBA alerts in real-time, filter for specific users or behaviors, correlate with your XDR findings.

---

## 2. Retrieve UBA Alerts in Bulk

Use this if you want Netskope to send all UBA alerts without filtering. The system takes responsibility for delivery.

**Endpoint:** `GET /api/v2/events/dataexport/alerts/uba`

```bash
curl -X GET \
  'https://<customer-tenant>.goskope.com/api/v2/events/dataexport/alerts/uba?operation=1658502533' \
  -H 'accept: application/json'
```

**Response:** Full UBA alert objects with all fields (similar to endpoint 1, but via bulk export).

**Use case:** Batch ingestion of UBA alerts for your SIEM or enrichment pipeline.

---

## 3. Validate Anomalies (Check for Allowed Impact Score)

Before adjusting a user's UCI score, verify the anomaly hasn't already been "allowed" (waived by a Netskope admin). This prevents double-penalizing a user for the same behavior.

**Endpoint:** `GET /api/v2/incidents/anomalies/{anomalyId}`

```bash
curl -X GET \
  'https://<customer-tenant>.goskope.com/api/v2/incidents/anomalies/61b2e6a1364e11747c40fbd9' \
  -H 'accept: application/json'
```

**Response:**
```json
{
  "anomalyId": "61b2e6a1364e11747c40fbd9",
  "user": "john.doe@company.com",
  "time": 1654823213000,
  "score": 0,
  "activity": "Multiple File Downloads",
  "markAsAllowed": true,
  "reasonForAllowed": "User approved for bulk export",
  "source": "Netskope Admin",
  "anomalyCreatedTime": "2023-06-06T21:34:52.122Z"
}
```

**Important:** If `markAsAllowed` is `true`, skip UCI score reduction for this user to avoid double penalties.

---

## 4. Get All Users' UCI Scores

Retrieve current UCI scores for up to 500 users (comma-separated). Use `capPerUser: 1` to get only the most recent score per user.

**Endpoint:** `POST /api/v2/incidents/uba/getuci`

**Rate Recommendation:** Run no more than once every 5 minutes unless actively investigating.

```bash
curl -X POST \
  'https://<customer-tenant>.goskope.com/api/v2/incidents/uba/getuci' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Netskope-Api-Token: <token>' \
  -d '{
    "users": [
      "aaron.bowen@bobsbank.com",
      "berniece.ames@bobsbank.com"
    ],
    "fromTime": 1702661501,
    "capPerUser": 1
  }'
```

**Response:**
```json
{
  "usersUci": [
    {
      "userId": "aaron.bowen@bobsbank.com",
      "confidences": [
        {
          "start": 1702598400000,
          "confidenceScore": 999
        }
      ]
    },
    {
      "userId": "berniece.ames@bobsbank.com",
      "confidences": [
        {
          "start": 1702598400000,
          "confidenceScore": 1000
        }
      ]
    }
  ]
}
```

**Use case:** Batch polling for user risk scores. Query 500 users at a time, store locally, and determine who needs investigation.

---

## 5. Get Single User's UCI Score

Retrieve a specific user's UCI score for targeted enrichment actions.

**Endpoint:** `POST /api/v2/ubadatasvc/user/uci`

```bash
curl -X POST \
  'https://<customer-tenant>.goskope.com/api/v2/ubadatasvc/user/uci' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Netskope-Api-Token: <token>' \
  -d '{
    "user": "aaron.bowen@bobsbank.com",
    "fromTime": 1702598400000
  }'
```

**Response:**
```json
{
  "userId": "aaron.bowen@bobsbank.com",
  "confidences": [
    {
      "start": 1702598400000,
      "confidenceScore": 999
    }
  ]
}
```

**Use case:** Look up a specific user's score after a security incident, or enrich your alert with Netskope's risk assessment.

---

## 6. Get Users by Risk Rating

Retrieve users grouped by risk category: Poor (0–350), Moderate (351–650), or Good (651–1000). These users must be active in the last 48 hours.

**Endpoint:** `POST /api/v2/incidents/users/getactiveuci`

```bash
curl -X POST \
  'https://<customer-tenant>.goskope.com/api/v2/incidents/users/getactiveuci?offset=0&limit=1000&sortby=confidenceScore&sortorder=asc' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Netskope-Api-Token: <token>' \
  -d '{
    "watchlist": "uci-watchlist",
    "ratingRank": "poor"
  }'
```

**Response:**
```json
{
  "totalCount": 1,
  "results": [
    {
      "user": "taylor@eras.tour",
      "startTime": 1717632000000,
      "uciScore": 190
    }
  ]
}
```

**Use case:** Find all "poor" rated users for immediate investigation or automated response.

---

## 7. Get Users by Risk Rating (Extended Filtering)

Like workflow 6, but with extended filtering options (time range, user search patterns, etc.).

**Endpoint:** `POST /api/v2/incidents/users/getuci`

```bash
curl -X POST \
  'https://<customer-tenant>.goskope.com/api/v2/incidents/users/getuci?statistics=true&offset=0&limit=1000&sortby=confidenceScore&sortorder=asc' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Netskope-Api-Token: <token>' \
  -d '{
    "usersActiveInLastDays": 14,
    "ratingRank": "moderate",
    "user": {
      "contains": "demo"
    }
  }'
```

**Response:**
```json
{
  "totalCount": 74,
  "poorUserCount": 1,
  "moderateUserCount": 0,
  "goodUserCount": 73,
  "results": [
    {
      "user": "taylor@eras.tour",
      "start": 1717632000000,
      "confidenceScore": 190
    }
  ]
}
```

**Use case:** Advanced filtering for periodic risk assessments or department-specific reviews.

---

## 8. Reduce a User's UCI Score

Decrement a user's UCI score by X points in response to XDR findings. The impact takes effect within 15 minutes.

**Endpoint:** `POST /api/v2/incidents/user/uciimpact`

**Important Notes:**
- Customer must have Advanced UBA subscription
- A pre-configured policy must trigger on the new UCI score range for changes to take effect
- Timestamp must be epoch with milliseconds
- Store the returned `anomalyId` for potential reversion

```bash
curl -X POST \
  'https://<customer-tenant>.goskope.com/api/v2/incidents/user/uciimpact' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Netskope-Api-Token: <token>' \
  -d '{
    "user": "user@example.com",
    "score": 50,
    "timestamp": 1654823213000,
    "source": "YourProduct/v1.0",
    "reason": "Suspicious login pattern detected - multiple failed auth attempts"
  }'
```

**Response:**
```json
{
  "anomalyId": "61b2e6a1364e11747c40fbd9",
  "user": "user@example.com",
  "time": 1654823213000,
  "score": 50,
  "activity": "string",
  "source": "YourProduct/v1.0",
  "reason": "Suspicious login pattern detected - multiple failed auth attempts",
  "anomalyCreatedTime": "2023-06-06T21:59:21.782Z"
}
```

**Best Practice:** 
- Include your product name and version in `source` (e.g., `CiscoXDR/1.2.1`)
- Use descriptive `reason` strings for audit logging (this helps customers understand why scores changed)
- Store `anomalyId` for potential reversion if investigation proves false positive

---

## 9. Revert an Anomaly

Revert a previously applied UCI score reduction if it was a false positive or investigation cleared the user.

**Endpoint:** `POST /api/v2/incidents/anomalies/{anomalyId}/allow`

```bash
curl -X POST \
  'https://<customer-tenant>.goskope.com/api/v2/incidents/anomalies/61b2e6a1364e11747c40fbd9/allow' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Netskope-Api-Token: <token>' \
  -d '{
    "reason": "Investigation confirmed false positive - user authorized for bulk data export"
  }'
```

**Response:**
```json
{
  "anomalyId": "61b2e6a1364e11747c40fbd9",
  "user": "user@example.com",
  "markAsAllowed": true,
  "reasonForAllowed": "Investigation confirmed false positive - user authorized for bulk data export",
  "anomalyCreatedTime": "2023-06-06T23:27:53.478Z"
}
```

**Use case:** After SOC investigation, clear a user's penalty and explain the reversal in audit logs.

---

## 10. Manage Users via SCIM Groups

Move risky users to a restricted group for tighter policy enforcement (e.g., block uploads/downloads, require additional authentication).

**Important Notes:**
- Applicable to all real-time subscription customers (NG-SWG, NPA, RBI, Cloud Firewall)
- Group synchronization takes up to 40 minutes globally
- Policy must be pre-configured to use the group and trigger on it
- Place the "risky users" policy above the user's default policy in the rule stack (first match wins)

### Create a New Group

**Endpoint:** `POST /api/v2/scim/Groups`

```bash
curl -X POST \
  'https://<customer-tenant>.goskope.com/api/v2/scim/Groups' \
  -H 'accept: application/scim+json;charset=utf-8' \
  -H 'Content-Type: application/scim+json;charset=utf-8' \
  -H 'Netskope-Api-Token: <token>' \
  -d '{
    "schemas": ["urn:ietf:params:scim:schemas:core:2.0:Group"],
    "displayName": "high-risk-users",
    "externalId": "high-risk-users-ext",
    "meta": {
      "resourceType": "Group"
    }
  }'
```

**Response:**
```json
{
  "id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "schemas": ["urn:ietf:params:scim:schemas:core:2.0:Group"],
  "displayName": "high-risk-users",
  "externalId": "high-risk-users-ext",
  "meta": {
    "resourceType": "Group"
  },
  "status": 201
}
```

### Get User SCIM ID

**Endpoint:** `GET /api/v2/scim/Users/{userId}`

```bash
curl -X GET \
  'https://<customer-tenant>.goskope.com/api/v2/scim/Users/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx' \
  -H 'accept: application/scim+json;charset=utf-8' \
  -H 'Netskope-Api-Token: <token>'
```

**Response:**
```json
{
  "schemas": ["urn:ietf:params:scim:schemas:core:2.0:User"],
  "id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "userName": "john.doe@company.com",
  "name": {
    "familyName": "Doe",
    "givenName": "John"
  },
  "emails": [
    {
      "value": "john.doe@company.com",
      "primary": true
    }
  ]
}
```

### Add User to Group

**Endpoint:** `PATCH /api/v2/scim/Groups/{groupId}`

```bash
curl -X PATCH \
  'https://<customer-tenant>.goskope.com/api/v2/scim/Groups/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx' \
  -H 'accept: application/scim+json;charset=utf-8' \
  -H 'Content-Type: application/scim+json;charset=utf-8' \
  -H 'Netskope-Api-Token: <token>' \
  -d '{
    "schemas": ["urn:ietf:params:scim:api:messages:2.0:PatchOp"],
    "Operations": [{
      "path": "members",
      "op": "add",
      "value": {
        "value": {
          "value": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
        }
      }
    }]
  }'
```

**Response:**
```json
{
  "status": "204",
  "description": "PATCH Completed"
}
```

### Remove User from Group

**Endpoint:** `PATCH /api/v2/scim/Groups/{groupId}`

```bash
curl -X PATCH \
  'https://<customer-tenant>.goskope.com/api/v2/scim/Groups/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx' \
  -H 'accept: application/scim+json;charset=utf-8' \
  -H 'Content-Type: application/scim+json;charset=utf-8' \
  -H 'Netskope-Api-Token: <token>' \
  -d '{
    "schemas": ["urn:ietf:params:scim:api:messages:2.0:PatchOp"],
    "Operations": [{
      "path": "members",
      "op": "remove",
      "value": {
        "value": {
          "value": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
        }
      }
    }]
  }'
```

---

## Integration Tips

### Error Handling
- Always check for HTTP 429 (rate limit exceeded) and back off exponentially
- Validate API tokens before running batch operations
- Use the `markAsAllowed` flag to prevent double-penalizing users

### Idempotency
- Store `anomalyId` values locally to track reversions
- Check `markAsAllowed` before applying score reductions
- Deduplicate requests if retrying

### Timing & Synchronization
- **UCI score impact:** Up to 15 minutes for enforcement
- **Group membership:** Up to 40 minutes to propagate globally
- Poll periodically to confirm enforcement, or watch for webhook events

### Audit Trail
- Always populate `source` with your product name/version
- Use descriptive `reason` strings (helps customers understand changes)
- Log all reductions and reversions for compliance

### Testing
- Start with a small user set (5–10 users)
- Test group addition/removal and verify policy behavior before production
- Validate that pre-configured policies trigger on score changes
- Use a non-production tenant if available

---

## Common Patterns

### XDR User Response Workflow
```
1. XDR detects suspicious behavior (multiple failed logins, unusual file access)
2. Query user's current UCI score (workflow 5)
3. Check if anomaly already marked as allowed (workflow 3)
4. If not allowed, reduce UCI score by 25–100 points (workflow 8)
5. Store anomalyId for potential reversion
6. Monitor user's next 10 minutes of activity
7. If investigation clears user, revert anomaly (workflow 9)
8. If confirmed threat, optionally add user to "review" group (workflow 10)
```

### Bulk Risk Assessment
```
1. Query all active users by risk rating (workflow 6)
2. Filter for "poor" rated users (score < 350)
3. For each user:
   a. Get detailed score (workflow 5)
   b. Retrieve recent UBA alerts (workflow 1)
   c. Correlate with your threat intelligence
4. Create report for SOC team
5. If risk confirmed, reduce scores (workflow 8)
```

### False Positive Management
```
1. SOC analyst reviews UCI score reduction
2. Investigates incident and confirms false positive
3. Calls revert anomaly API (workflow 9)
4. Documents reason for audit trail
5. System restores user's score within 15 minutes
```

---

## Troubleshooting

**Problem:** UCI score reduction has no effect on policies
- **Solution:** Verify customer has pre-configured policy that triggers on UCI score ranges. Policies don't auto-create.

**Problem:** Group membership takes longer than expected to apply
- **Solution:** Normal behavior is up to 40 minutes. Restart Netskope client if needed. Verify policy is above default policy in rule stack.

**Problem:** User marked as allowed, but we still need to reduce their score
- **Solution:** Cannot reduce scores for allowed anomalies. Create a new impact request with different `score` value and new `timestamp`.

---

## API Reference

| Workflow | Endpoint | Method | Purpose |
|----------|----------|--------|---------|
| 1 | `/api/v2/events/data/alert` | GET | Retrieve UBA alerts |
| 2 | `/api/v2/events/dataexport/alerts/uba` | GET | Bulk UBA alerts |
| 3 | `/api/v2/incidents/anomalies/{anomalyId}` | GET | Check anomaly status |
| 4 | `/api/v2/incidents/uba/getuci` | POST | Get multiple users' UCI scores |
| 5 | `/api/v2/ubadatasvc/user/uci` | POST | Get single user's UCI score |
| 6 | `/api/v2/incidents/users/getactiveuci` | POST | Get users by risk rating |
| 7 | `/api/v2/incidents/users/getuci` | POST | Get users (filtered) |
| 8 | `/api/v2/incidents/user/uciimpact` | POST | Reduce user's UCI score |
| 9 | `/api/v2/incidents/anomalies/{anomalyId}/allow` | POST | Revert anomaly |
| 10a | `/api/v2/scim/Groups` | POST | Create group |
| 10b | `/api/v2/scim/Users/{userId}` | GET | Get user SCIM ID |
| 10c | `/api/v2/scim/Groups/{groupId}` | PATCH | Add/remove user from group |
