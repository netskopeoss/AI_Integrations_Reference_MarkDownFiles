# DLP Incident Management Integration Guide

Monitor and respond to Data Loss Prevention (DLP) incidents in real-time. Use these workflows to pull incidents, enrich them with your threat intelligence, and update status, severity, and assignee.

---

## Overview

Netskope's Data Loss Prevention (DLP) logs policy violations when users attempt to upload sensitive data to cloud services or breach Content Control policies. Use these workflows to:
- Retrieve DLP incident events
- Extract incident details (user, app, matched rules, file info)
- Update incident severity, assignee, and status
- Integrate with your incident management system
- Automate incident triage and response

**Prerequisites:**
- DLP subscription
- DLP policies configured to log incidents
- Endpoint DLP (for Content Control incident capture)

---

## 1. Retrieve DLP Incident Events

Get all DLP incidents, including file details, matched rules, and severity. Events include both SaaS DLP and Endpoint DLP (Content Control) violations.

**Endpoint:** `GET /api/v2/events/dataexport/events/incident`

```bash
curl -X GET \
  'https://<customer-tenant>.goskope.com/api/v2/events/dataexport/events/incident?operation=1708955000&index=abc' \
  -H 'accept: application/json' \
  -H 'Netskope-Api-Token: <token>'
```

**Response:**
```json
{
  "ok": 1,
  "result": [
    {
      "_id": "5bd54d8290c6ffec9d64acc0a6519578f8690809d1cc7d76dce7e84b20532585",
      "access_method": "Client",
      "acting_user": "kmaheshwari@netskope.com",
      "activity": "Upload",
      "app": "Amazon S3",
      "app_session_id": 5870330112534445000,
      "assignee": "None",
      "dlp_incident_id": 6441942879614541000,
      "dlp_match_info": [
        {
          "dlp_action": "alert",
          "dlp_forensic_id": 6441942879614541000,
          "dlp_policy": "Block Action",
          "dlp_profile_name": "EU General Data Protection Regulation (GDPR) (narrow)",
          "dlp_rules": [
            {
              "dlp_data_identifiers": {
                "numbers/telephone_numbers/eu": 2,
                "persons/proper_names/eu/full": 2
              },
              "dlp_incident_rule_count": 2,
              "dlp_rule_name": "EU-Name-Phone (narrow)",
              "dlp_rule_score": 4,
              "dlp_rule_severity": "Low",
              "is_unique_count": false,
              "weighted": false
            }
          ]
        }
      ],
      "file_lang": "ENGLISH",
      "file_size": 59735,
      "file_type": "application/pdf",
      "md5": "5ce2a670ed85834f9b2d5051c67a4263",
      "object": "Resume (3).pdf",
      "severity": "Low",
      "site": "Amazon S3",
      "src_location": "Bengaluru",
      "status": "new",
      "timestamp": 1708955654,
      "title": "Resume (3).pdf",
      "user": "kmaheshwari@netskope.com"
    }
  ],
  "wait_time": 1,
  "timestamp_hwm": 1708955663
}
```

**Key Fields:**
- `dlp_incident_id` — Unique incident ID
- `object_id` — Required for updates (in update requests)
- `user` / `acting_user` — User who triggered incident
- `app` — SaaS application where violation occurred
- `dlp_match_info` — Rules, data identifiers, severity
- `status` — Current status (new, in_progress, resolved)
- `severity` — Low, Medium, High
- `assignee` — Current assignee (if any)

**Use case:** Pull incidents for real-time triage, automation, or integration with your SIEM.

---

## 2. Update DLP Incident Status

Update an incident's severity, assignee, or status (new, in_progress, resolved). You must provide the current value and the new value.

**Endpoint:** `PATCH /api/v2/incidents/update`

```bash
curl -X PATCH \
  'https://<customer-tenant>.goskope.com/api/v2/incidents/update' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Netskope-Api-Token: <token>' \
  -d '{
    "payload": [
      {
        "object_id": "hash_kmaheshwari@netskope.com_5ce2a670ed85834f9b2d5051c67a4263_b68d2e26d7deaf14b000a07182e2b228f2fa5c64",
        "field": "status",
        "old_value": "new",
        "new_value": "in_progress"
      },
      {
        "object_id": "hash_kmaheshwari@netskope.com_5ce2a670ed85834f9b2d5051c67a4263_b68d2e26d7deaf14b000a07182e2b228f2fa5c64",
        "field": "severity",
        "old_value": "Low",
        "new_value": "High"
      },
      {
        "object_id": "hash_kmaheshwari@netskope.com_5ce2a670ed85834f9b2d5051c67a4263_b68d2e26d7deaf14b000a07182e2b228f2fa5c64",
        "field": "assignee",
        "old_value": "None",
        "new_value": "jane.smith@company.com"
      }
    ]
  }'
```

**Response:**
```json
{
  "ok": 1,
  "result": "0"
}
```

**Parameters:**
- `object_id` — From incident event (required)
- `field` — `status`, `severity`, or `assignee`
- `old_value` — Current value (must match exactly)
- `new_value` — New value

**Valid Values:**
- **status:** `new`, `in_progress`, `resolved`
- **severity:** `Low`, `Medium`, `High`
- **assignee:** Email address or `None`

**Important:** You must retrieve the current `old_value` from the incident event. There's no API to query current state—it's assumed you have the incident already fetched.

---

## Integration Tips

### Incident Enrichment Workflow

```
1. Pull incidents (workflow 1)
2. For each incident:
   a. Query your threat intelligence for user/app/file hash
   b. Check if file hash matches known malware
   c. Check if user has prior security incidents
   d. Determine severity (Low/Medium/High)
   e. Route to appropriate analyst (assignee)
3. Update status/severity/assignee (workflow 2)
4. Send notification to analyst
```

### Automated Response

```
1. Pull incidents (workflow 1)
2. Filter for incidents matching your criteria:
   - File size > 5 MB AND severity > Low
   - Matched rule contains "PII" OR "GDPR"
   - App is unapproved (from your app inventory)
3. Auto-update to "in_progress" and assign to team
4. Optional: Reduce user's UCI score (User Risk Management guide)
5. Optional: Block user access (SCIM groups, User Risk Management)
```

### Daily Incident Report

```
1. Pull incidents from last 24 hours
2. Group by severity, user, app
3. Calculate statistics:
   - Total incidents
   - By severity breakdown
   - Top users, apps, rules
4. Identify trends
5. Send daily report to SOC
```

---

## Error Handling

**Problem:** "object_id not found" error
- **Solution:** Verify object_id matches the event. Copy it exactly from the incident response.

**Problem:** "old_value mismatch" error
- **Solution:** The current value doesn't match what you provided. Fetch the incident again to get the actual current state.

**Problem:** Status update didn't work
- **Solution:** Ensure `old_value` and `new_value` are valid. Valid status values are: `new`, `in_progress`, `resolved`.

---

## Common Patterns

### Real-Time Alert Triage
```
1. Customer receives DLP alert
2. Pull incident details (workflow 1)
3. Enrich with your threat intel:
   - Is file hash known malware?
   - Is user a known insider threat?
   - Is destination app approved?
4. If high-risk:
   - Update severity to High
   - Assign to senior analyst
5. If low-risk:
   - Update severity to Low
   - Archive (update status to resolved)
```

### Insider Threat Response
```
1. UEBA detects user downloading unusual amounts of data
2. Pull DLP incidents for that user
3. Check matching rules (are they PII/GDPR related?)
4. If sensitive data, escalate:
   - Update severity to High
   - Assign to insider threat team
   - Optionally reduce UCI score (User Risk Management)
   - Optionally add user to restricted group (SCIM)
```

### Compliance Reporting
```
1. Daily: Pull all DLP incidents
2. Filter for "GDPR" or "PII" rules
3. Group by user and count
4. Update incidents to "in_progress"
5. Generate compliance report
6. Archive when resolved
```

---

## API Reference

| Workflow | Endpoint | Method | Purpose |
|----------|----------|--------|---------|
| 1 | `/api/v2/events/dataexport/events/incident` | GET | Retrieve incidents |
| 2 | `/api/v2/incidents/update` | PATCH | Update incident (status, severity, assignee) |
