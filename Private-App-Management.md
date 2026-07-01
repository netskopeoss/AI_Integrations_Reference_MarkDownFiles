# Private App Management (ZTNA) Integration Guide

Configure and manage Zero Trust Network Access (ZTNA / NPA) private applications. Use these workflows to define application access policies, manage publishers, and enable dynamic workload isolation based on security posture.

---

## Overview

Zero Trust Network Access is a successor to VPN that grants access to specific IP/TCP or IP/UDP combinations tied to dedicated workloads. Unlike VPN, users can only access the specific application they're authorized for, not the entire network.

Use these workflows to:
- Query existing private apps and publishers
- Create new private applications with tags
- Update application configurations
- Enable dynamic ZTNA orchestration based on risk

**Prerequisites:**
- Netskope Private Access (NPA) subscription
- NPA publishers configured and running
- Real-time policies that reference app tags

---

## 1. Gather Private Apps Information

Query all configured private applications and their details. Use this to understand what's already deployed.

**Endpoint:** `GET /api/v2/steering/apps/private`

```bash
curl -X GET \
  'https://<tenant>.goskope.com/api/v2/steering/apps/private' \
  -H 'accept: application/json' \
  -H 'Netskope-Api-Token: <token>'
```

**Response:**
```json
{
  "data": {
    "private_apps": [
      {
        "app_id": 2,
        "app_name": "[test server]",
        "clientless_access": false,
        "host": "172.31.34.5",
        "modified_by": "user@netskope.com",
        "modify_time": "2023-11-30 23:08:48",
        "policies": ["PrivateApps"],
        "private_app_protocol": "http",
        "protocols": [
          {
            "created_at": "2023-11-30T23:08:48.269Z",
            "id": 7,
            "port": "443",
            "service_id": 1,
            "transport": "tcp",
            "updated_at": "2023-11-30T23:08:48.269Z"
          },
          {
            "created_at": "2023-11-30T23:08:48.272Z",
            "id": 8,
            "port": "22",
            "service_id": 1,
            "transport": "tcp",
            "updated_at": "2023-11-30T23:08:48.272Z"
          }
        ],
        "public_host": "",
        "reachability": {
          "error_code": 0,
          "error_string": "",
          "reachable": true
        },
        "service_publisher_assignments": [
          {
            "publisher_id": 15,
            "publisher_name": "AWS-NPA",
            "reachability": {
              "error_code": 0,
              "error_string": "",
              "reachable": true
            },
            "service_id": 2
          }
        ],
        "tags": [],
        "trust_self_signed_certs": true,
        "use_publisher_dns": false
      }
    ]
  },
  "status": "success",
  "total": 1
}
```

**Key Fields:**
- `app_id` — Unique app identifier
- `app_name` — Application name (wrapped in brackets)
- `host` — Private IP address or hostname
- `protocols` — TCP/UDP ports and transport
- `service_publisher_assignments` — Publishers providing access
- `tags` — Labels for policy matching

**Use case:** Inventory existing private apps before creating or modifying.

---

## 2. Get Publishers Information

List all NPA publishers. Useful for identifying which publishers are available and their status.

**Endpoint:** `GET /api/v2/infrastructure/publishers`

```bash
curl -X GET \
  'https://<tenant>.goskope.com/api/v2/infrastructure/publishers' \
  -H 'accept: application/json' \
  -H 'Netskope-Api-Token: <token>'
```

**Response:**
```json
{
  "data": {
    "publishers": [
      {
        "apps_count": 1,
        "assessment": {
          "eee_support": true,
          "hdd_free": "3978469376Kb",
          "hdd_total": "8132173824Kb",
          "ip_address": "172.31.46.31",
          "latency": 1,
          "version": "110.0.0.8301"
        },
        "common_name": "f722b4b29620257e",
        "connected_apps": ["[test server]"],
        "lbrokerconnect": false,
        "publisher_id": 15,
        "publisher_name": "AWS-NPA",
        "publisher_upgrade_profiles_external_id": 1,
        "registered": true,
        "status": "connected",
        "stitcher_id": 249,
        "tags": [],
        "upgrade_failed_reason": null,
        "upgrade_request": false,
        "upgrade_status": {
          "upstat": "disabled"
        }
      }
    ]
  },
  "status": "success",
  "total": 1
}
```

**Key Fields:**
- `publisher_id` — Required for creating apps
- `publisher_name` — Publisher identifier
- `status` — `connected` (active) or disconnected
- `version` — Publisher agent version
- `connected_apps` — Apps already assigned to this publisher

**Use case:** Verify publisher is connected and operational before assigning apps to it.

---

## 3. Create Private App

Create a new private application with protocols, host, publishers, and tags. Tags are used for policy matching.

**Endpoint:** `POST /api/v2/steering/apps/private`

```bash
curl -X POST \
  'https://<tenant>.goskope.com/api/v2/steering/apps/private' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Netskope-Api-Token: <token>' \
  -d '{
    "app_name": "quarantine",
    "host": "172.31.34.6",
    "protocols": [
      {
        "type": "tcp",
        "port": "443"
      },
      {
        "type": "tcp",
        "port": "22"
      }
    ],
    "publishers": [
      {
        "publisher_id": "15",
        "publisher_name": "AWS-NPA"
      }
    ],
    "tags": [
      {
        "tag_name": "quarantine"
      }
    ],
    "use_publisher_dns": false,
    "clientless_access": true,
    "allow_unauthenticated_cors": true,
    "trust_self_signed_certs": true
  }'
```

**Response:**
```json
{
  "data": {
    "app_id": 3,
    "app_name": "[quarantine]",
    "clientless_access": true,
    "host": "172.31.34.6",
    "id": 3,
    "modified_by": "apigw",
    "modify_time": "2023-12-01 00:09:59",
    "name": "[quarantine]",
    "policies": [],
    "protocols": [
      {
        "created_at": "2023-12-01T00:09:59.548Z",
        "id": 9,
        "port": "443",
        "service_id": 2,
        "transport": "tcp",
        "updated_at": "2023-12-01T00:09:59.548Z"
      }
    ],
    "service_publisher_assignments": [
      {
        "primary": null,
        "publisher_id": 15,
        "publisher_name": "AWS-NPA",
        "reachability": null,
        "service_id": 3
      }
    ],
    "tags": [
      {
        "tag_id": 3,
        "tag_name": "quarantine"
      }
    ],
    "trust_self_signed_certs": true,
    "use_publisher_dns": false
  },
  "status": "success"
}
```

**Parameters:**
- `app_name` — Application name (becomes "[app_name]" in Netskope)
- `host` — IP address or hostname of the private app
- `protocols` — Array of {type, port} (tcp or udp)
- `publishers` — Array of {publisher_id, publisher_name}
- `tags` — Array of {tag_name} for policy matching
- `clientless_access` — Enable browser-based access
- `trust_self_signed_certs` — Accept self-signed certificates

**Use case:** Create a new private app for a risk-based workflow (e.g., "quarantine" app for restricted access).

---

## Integration Tips

### Risk-Based ZTNA Routing

Use tags to dynamically route based on user risk:

```
1. User with high UCI score attempts to access internal app
2. Pre-configured policy: if (user.risk == "high") route to "quarantine" app
3. Quarantine app has:
   - Limited ports (only HTTPS, no RDP/SSH)
   - Logging enabled
   - IP restrictions (office only)
4. User gets restricted access until risk clears
5. Normal users route to "production" app with full access
```

### Publisher Redundancy

Create multiple apps for the same backend, each behind different publishers:

```
1. Database server at 10.0.1.5:5432
2. Create app "db-primary" behind "Publisher-East"
3. Create app "db-backup" behind "Publisher-West"
4. Policy routes based on region or failover
```

### Application Isolation

Use tags to control access to different environments:

```
1. Create "production" app with tag "prod"
   - Limited to production team
2. Create "staging" app with tag "staging"
   - Accessible to dev team
3. Policy:
   - if (department == "prod") allow "production" app
   - if (department == "dev") allow "staging" app
4. User can only see and access their allowed apps
```

---

## Common Patterns

### Incident Response App Isolation
```
1. Incident detected in database
2. Create new private app "quarantine-db" with tag "quarantine"
3. Configure for logging and restricted access
4. Point policy: if (data_classification == "sensitive") route to quarantine-db
5. Sensitive app access only to forensics team
6. After incident: delete app or repurpose
```

### Gradual Rollout
```
1. Create private app for new internal tool
2. Initial tag: "pilot" (100 pilot users)
3. Monitor usage and performance
4. After validation:
   - Create "production" tag
   - Migrate users from "pilot" to "production"
5. Retire pilot app
```

### Multi-Environment Management
```
1. Database environments: prod, staging, dev
2. Create three private apps:
   - "prod-db" tag: "production" (locked down)
   - "staging-db" tag: "staging" (moderate restrictions)
   - "dev-db" tag: "development" (open access)
3. Policy routes users based on team and approval
4. Audit logs track all access
```

---

## Troubleshooting

**Problem:** Publisher shows "disconnected"
- **Solution:** Check publisher logs. NPA agent may need restart.

**Problem:** App reachability failing
- **Solution:** Verify host IP/hostname is correct. Check firewall rules from publisher to app.

**Problem:** Tags not showing in policy
- **Solution:** Tags only work in real-time policies (not in rules engine). Ensure you're creating tags in the right context.

---

## API Reference

| Workflow | Endpoint | Method | Purpose |
|----------|----------|--------|---------|
| 1 | `/api/v2/steering/apps/private` | GET | Query private apps |
| 2 | `/api/v2/infrastructure/publishers` | GET | Get publishers |
| 3 | `/api/v2/steering/apps/private` | POST | Create private app |

---

## Additional Resources

For comprehensive ZTNA/NPA documentation, see:
[Netskope Private App Management Documentation](https://docs.netskope.com/en/netskope-help/data-security/netskope-private-access/private-app-management/)
