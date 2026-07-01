# Application Instances Integration Guide

Label specific cloud application instances (e.g., particular S3 buckets, Office 365 tenants, Salesforce orgs) as "sanctioned" or "unsanctioned" for policy enforcement. Use these workflows to manage instance-level risk assessment and dynamic policy application.

---

## Overview

Use these endpoints to:
- Add new cloud app instances with labels
- Update instance labels based on security findings
- List all configured instances and their labels
- Delete instances from Netskope
- Coordinate with CNAPP and XDR for instance correlation

**Prerequisites:**
- NG-SWG or CASB subscription
- Pre-configured policies matching on instance labels
- Instance ID and instance name (from logs or your CMDB)

**Note:** Only two labels are supported: `sanctioned` and `unsanctioned`

---

## 1. Add Application Instance

Create a new app instance with a label. All three fields (app name, instance ID, instance name) must be specified.

**Endpoint:** `GET /api/v1/app_instances?token=<token>&op=add`

```bash
curl --location \
  'https://<tenant-name>.goskope.com/api/v1/app_instances?token=<token>&op=add' \
  --header 'Content-Type: application/json' \
  --data '{
    "instances": [
      {
        "app": "Amazon S3",
        "instance_id": "427028034291",
        "instance_name": "S3-Unsanctioned",
        "tags": ["Unsanctioned"]
      }
    ]
  }'
```

**Response:**
```json
{
  "status": "success",
  "msg": "Successfully added instances"
}
```

**Parameters:**
- `app` — Application name (e.g., "Amazon S3", "Microsoft Office 365 OneDrive")
- `instance_id` — Unique instance identifier (e.g., AWS account ID, Office 365 tenant ID)
- `instance_name` — Friendly name for the instance
- `tags` — Array containing either `["Sanctioned"]` or `["Unsanctioned"]`

**Use case:** When your CNAPP discovers a new S3 bucket, add it with the appropriate label.

---

## 2. Update Application Instance

Modify labels on an existing instance. This is the key workflow for dynamic risk-based routing.

**Endpoint:** `GET /api/v1/app_instances?token=<token>&op=update`

```bash
curl --location \
  'https://<tenant-name>.goskope.com/api/v1/app_instances?token=<token>&op=update' \
  --header 'Content-Type: application/json' \
  --data '{
    "instances": [
      {
        "app": "Amazon S3",
        "instance_id": "427028034291",
        "instance_name": "S3-Unsanctioned",
        "tags": ["Sanctioned"]
      }
    ]
  }'
```

**Response:**
```json
{
  "status": "success",
  "msg": "Successfully updated instances"
}
```

**Use case:** After security review, change an instance from "Unsanctioned" to "Sanctioned" or vice versa.

---

## 3. List All Application Instances

Retrieve all configured instances and their current labels. Use this to understand what's managed and identify gaps.

**Endpoint:** `GET /api/v1/app_instances?token=<token>&op=list`

```bash
curl --location \
  'https://<tenant-name>.goskope.com/api/v1/app_instances?token=<token>&op=list' \
  --header 'Content-Type: application/json'
```

**Response:**
```json
{
  "status": "success",
  "data": [
    {
      "app": "Amazon Web Services Console",
      "instance_name": "karthik@portal26.ai",
      "instance_id": "427028034291",
      "type": "Sync",
      "tags": ["Sanctioned"],
      "last_modified": "2024-06-14 09:30:12",
      "custom": "0",
      "app_id": "152"
    },
    {
      "app": "Microsoft Office 365 OneDrive for Business",
      "instance_name": "MyNetskopeDemo",
      "instance_id": "mynetskopedemo",
      "type": "Custom",
      "tags": [null],
      "last_modified": "2022-04-29 07:46:03",
      "custom": "0",
      "app_id": "5682"
    },
    {
      "app": "Amazon S3",
      "instance_name": "S3-Unsanctioned",
      "instance_id": "123451234512",
      "type": "Custom",
      "tags": ["Unsanctioned"],
      "last_modified": "2024-07-03 09:11:43",
      "custom": "0",
      "app_id": "9528"
    }
  ]
}
```

**Use case:** Dashboard to see all labeled instances and their risk status.

---

## 4. Delete Application Instance

Remove an instance from Netskope management.

**Endpoint:** `GET /api/v1/app_instances?token=<token>&op=delete`

```bash
curl --location \
  'https://<tenant-name>.goskope.com/api/v1/app_instances?token=<token>&op=delete' \
  --header 'Content-Type: application/json' \
  --data '{
    "instances": [
      {
        "app": "Amazon S3",
        "instance_id": "427028034291",
        "instance_name": "S3-Unsanctioned"
      }
    ]
  }'
```

**Response:**
```json
{
  "status": "success",
  "msg": "Successfully deleted instances"
}
```

**Use case:** When an instance is decommissioned or no longer needs policy enforcement.

---

## Integration Tips

### CNAPP Correlation

Coordinate with your Cloud Native Application Protection Platform (CNAPP):

```
1. CNAPP discovers new S3 bucket
2. Extract: account_id, bucket_name, risk_level
3. If risk_level = high:
   - Call Add Instance with "Unsanctioned" tag
4. If risk_level = low:
   - Call Add Instance with "Sanctioned" tag
5. Pre-configured policy enforces access restrictions
```

### Risk-Based Routing

Update instances dynamically based on security findings:

```
1. Instance initially marked "Sanctioned"
2. Unusual activity detected (multiple downloads, unusual users)
3. Call Update Instance to "Unsanctioned"
4. Policy restricts access (requires justification, etc.)
5. After SOC review:
   - If benign activity: Update back to "Sanctioned"
   - If suspicious: Keep as "Unsanctioned"
```

### Inventory Management

Regularly sync your instance inventory:

```
1. List all instances (workflow 3)
2. Compare with your CMDB/CNAPP inventory
3. For new instances: Add them (workflow 1)
4. For deleted instances: Remove them (workflow 4)
5. For changed risk: Update labels (workflow 2)
```

---

## Error Handling

**Problem:** "Instance not found" error
- **Solution:** Verify instance_id and instance_name match exactly. Check workflow 3 (list) to see actual values.

**Problem:** "Invalid tag" error
- **Solution:** Only two tags allowed: `Sanctioned` or `Unsanctioned` (case-sensitive).

**Problem:** All three fields required
- **Solution:** You must provide app name, instance_id, and instance_name. All are required.

---

## Common Patterns

### Kubernetes Workload Protection
```
1. Kubernetes cluster running in AWS
2. Workload accesses S3 bucket (instance_id: aws-account-123)
3. CNAPP detects workload identity
4. If workload is approved:
   - Add instance with "Sanctioned" tag
   - Policy allows access
5. If workload is unapproved:
   - Add instance with "Unsanctioned" tag
   - Policy blocks or requires approval
```

### Database Instance Tagging
```
1. RDS instance detected in multi-tenant account
2. Instance_id: rds-prod-db-123
3. Instance_name: "Production-Database"
4. If data classification = confidential:
   - Add with "Unsanctioned"
   - Restrict access to authorized users only
5. Regular access review:
   - List instances (workflow 3)
   - Verify access patterns match expectations
   - Update tags if needed
```

### Incident Response
```
1. Security incident detected in S3 bucket
2. Bucket instance_id: prod-bucket-456
3. Immediately update to "Unsanctioned"
4. Policy blocks all access except forensics team
5. Investigation completes:
   - If contained: Update back to "Sanctioned"
   - If ongoing: Keep as "Unsanctioned"
```

---

## API Reference

| Workflow | Endpoint | Method | Purpose |
|----------|----------|--------|---------|
| 1 | `/api/v1/app_instances` | GET (op=add) | Add instance |
| 2 | `/api/v1/app_instances` | GET (op=update) | Update instance label |
| 3 | `/api/v1/app_instances` | GET (op=list) | List all instances |
| 4 | `/api/v1/app_instances` | GET (op=delete) | Delete instance |

**Note:** These use the v1 endpoint with query parameters instead of the v2 RESTful style.
