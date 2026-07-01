# Netskope API Integration Guide

Welcome! This guide provides everything you need to build integrations with Netskope using the RESTful API v2 and SCIM endpoints. Whether you're building XDR correlation, threat response automation, or policy enforcement workflows, you'll find the endpoints, examples, and best practices here.

---

## Quick Start

### Prerequisites

- Netskope tenant URL: `https://<customer-tenant>.goskope.com`
- API token: Generate from Admin Settings → API Tokens
- Rate limit: 4 requests/second per endpoint
- HTTPS required for all API calls
- User-agent string for all API calls

### Authentication

Include your API token in all requests:

```bash
-H 'Netskope-Api-Token: <your-token>'
```

---

## Key Concepts

### Adaptive Risk Response

Netskope enables dynamic policy enforcement by populating objects that influence user experience and security posture:

- **User identity groups** — Move users to restrictive groups
- **User Confidence Index (UCI)** — Adjust individual user risk scores (1–1000, where 1000 is perfect)
- **Application labels** — Tag apps for policy matching
- **Custom URL lists** — Add malicious or suspicious destinations
- **File hash lists** — Leverage for threat prevention

Policies are fixed, but by adding/removing data from these objects, you change which policies apply to users, destinations, files, and apps.

### User Confidence Index (UCI)

A machine learning-derived score that measures user risk. Updates when user behavior anomalies (UBA alerts) are detected. Penalties decay every 24 hours.

**Requirements:** Advanced UBA subscription. (All inline customers can use SCIM for group-based restrictions.)

### Cloud Confidence Index (CCI)

A machine learning-derived score (1–100) measuring SaaS application safety. Use CCI scores and Cloud Confidence Levels (CCL) in policies, or apply custom string-based labels.

**Available to:** All NG-SWG and CASB customers.

### Zero Trust Network Access (ZTNA/NPA)

A successor to VPN that grants access to specific IP/TCP or IP/UDP combinations tied to dedicated workloads. Applications can be labeled to move between policy rules, enabling dynamic enforcement.

### Rate Limits

Netskope enforces a **4 requests/second per endpoint** rate limit. Implement exponential backoff for retries and use pagination (`limit` and `offset`) for large result sets.

---

## Integration Guides

Choose the guide(s) matching your integration type:

### 🎯 **User Risk Management**
Manage UCI scores, detect anomalies, and restrict high-risk users.

**Use this if you're building:**
- XDR correlation with user behavior
- Response automation based on risk scores
- User quarantine/isolation workflows

[Read User Risk Management Guide →](User-Risk-Management.md)

**Key workflows:**
- Retrieve UBA alerts
- Get/update UCI scores
- Move users to restrictive groups
- Revert false positives

---

### 📱 **Application Management**
Tag and label cloud applications for policy enforcement.

**Use this if you're building:**
- App risk assessment integrations
- Incident response with app quarantine
- Sanctioned/unsanctioned app workflows

[Read Application Management Guide →](Application-Management.md)

**Key workflows:**
- Identify apps in use
- Extract CCI scores and labels
- Query and apply custom tags
- Manage application instances

---

### 🔗 **Custom URL Lists**
Populate destination-based blocklists and allowlists.

**Use this if you're building:**
- Malware/C2 infrastructure blocking
- Phishing detection automation
- Threat feed integration

[Read Custom URL Lists Guide →](Custom-URL-Lists.md)

**Key workflows:**
- Create and manage URL lists
- Add/replace URLs
- Deploy changes

---

### 🚨 **DLP Incident Management**
Monitor and respond to data loss prevention incidents.

**Use this if you're building:**
- Incident triage and automation
- DLP alert enrichment
- Investigation workflows

[Read DLP Incident Management Guide →](DLP-Incident-Management.md)

**Key workflows:**
- Retrieve incidents
- Update severity, assignee, status

---

### ☁️ **Application Instances**
Label specific cloud app instances (e.g., particular S3 buckets, Office 365 tenants).

**Use this if you're building:**
- Instance-level threat response
- CNAPP/XDR instance correlation
- Sanctioned/unsanctioned instance labeling

[Read Application Instances Guide →](Application-Instances.md)

**Key workflows:**
- Add/update/delete instances
- Apply sanctioned/unsanctioned labels
- List all instances

---

### 🔐 **Private App Management (ZTNA)**
Configure and manage Zero Trust Network Access applications.

**Use this if you're building:**
- Dynamic ZTNA orchestration
- Private app risk-based routing
- Workload isolation workflows

[Read Private App Management Guide →](Private-App-Management.md)

**Key workflows:**
- Gather app and publisher info
- Create private apps with tags
- Modify app labels

---

### 📡 **NAC Device Integration**
Push Network Access Control (NAC) device posture information into Netskope via Cloud Exchange.

**Use this if you're building:**
- NAC/device posture correlation with Netskope policy

There is no direct RESTful API workflow for this today — device information is pushed via a Cloud Exchange plugin. See the [Cloud Exchange NAC Plugin community post](https://community.netskope.com/netskope-cloud-exchange-ce-67/cloud-exchange-beta-nac-plugin-6925) for setup details.

---

## Best Practices

### Audit Trail
- Always populate `source` with your product name/version (e.g., `YourProduct/v1.0`)
- Include descriptive `reason` strings for all policy changes
- Store returned IDs (anomalyId, groupId) for potential reversions

### Error Handling
- Check for HTTP 429 (rate limit exceeded) and back off exponentially
- Validate API tokens and tenant URLs before bulk operations
- Use the `markAsAllowed` flag to prevent double-penalizing users

### Idempotency
- Store IDs locally to avoid duplicate operations
- Implement deduplication for batch updates
- Use GET before PATCH/POST to confirm current state

### Synchronization & Timing
- Group membership changes: up to 40 minutes to propagate globally
- UCI score impacts: up to 15 minutes for enforcement
- Poll periodically to confirm enforcement or watch for webhooks

### Testing
- Start with a small user/URL set to validate workflows
- Test group addition/removal and verify policy behavior before production
- Validate custom URL list capacity before large uploads
- Use a non-production tenant if available

---

## Common Integration Patterns

### Pattern 1: XDR User Response
```
1. XDR detects suspicious user behavior
2. Query user's current UCI score (User Risk Management)
3. Check if anomaly already marked as allowed
4. If not allowed, reduce UCI score by X points
5. Optionally add user to "review" group (SCIM)
```

### Pattern 2: App Threat Response
```
1. XDR identifies compromised SaaS app
2. Query apps in use (Application Management)
3. Apply "compromised" tag to app
4. (Pre-configured policy) blocks/coaches users on tagged app
```

### Pattern 3: Malware Infrastructure Blocking
```
1. Threat feed detects new C2 domain
2. Check available space in custom URL list
3. Add URL to "malware-c2" list
4. Deploy list changes
5. (Pre-configured policy) blocks access
```

### Pattern 4: DLP Incident Triage
```
1. DLP incident triggered
2. Pull incident via API
3. Enrich with your threat intelligence
4. Update severity/assignee/status in Netskope
5. Assign to SOC analyst
```

---

## API Reference Quick Links

- **Base URL:** `https://<customer-tenant>.goskope.com/api/v2`
- **API Docs (Swagger):** `https://<customer-tenant>.goskope.com/apidocs`
- **SCIM Endpoint:** `/api/v2/scim`
- **Events/Incidents:** `/api/v2/events/dataexport`, `/api/v2/incidents`
- **Policy/Services:** `/api/v2/policy`, `/api/v2/services`

---

## Support & Resources

- **Netskope Swagger Documentation:** `https://<customer-tenant>.goskope.com/apidocs`
- **Community:** [Netskope Community Portal](https://community.netskope.com)
- **Questions?** Contact your Netskope Tech Alliance team

---

## Integration Guide Structure

Each guide is self-contained and includes:
- Clear endpoint descriptions
- Practical curl examples
- Request/response JSON samples
- Key requirements and caveats
- Integration tips

**Recommended reading order:**
1. This README (you are here)
2. The guide(s) matching your use case
3. Integration Tips section in each guide

---

**Last updated:** July 2026  
**API Version:** v2 (RESTful), v2 (SCIM)

## License

This guide is licensed under the [MIT License](LICENSE).
