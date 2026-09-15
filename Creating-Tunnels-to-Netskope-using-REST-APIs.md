# Creating Tunnels to Netskope using REST APIs

Automate IPsec (or GRE) tunnel creation to Netskope's NewEdge network for NG-SWG, NPA, and Cloud Firewall inline inspection. Use these workflows if you're an SD-WAN/networking partner building tunnels programmatically instead of configuring them by hand, so joint customers can stand up multi-vendor SASE connectivity without manual tunnel setup on either side.

---

## Overview

Use these endpoints to:
- Discover the closest Netskope NewEdge Points of Presence (POPs) to an end device
- Create an IPsec tunnel (with primary and backup POPs) to Netskope
- List existing tunnels and their status/throughput
- Delete tunnels that are no longer needed

**Prerequisites:**
- **User-Agent header:** Include a user-agent string formatted as `<Vendor-Product-Version>` on every API call — this is how Netskope attributes traffic to your integration.
- **API Scopes:** Enable the following in the **Network** functional area, under **Traffic Steering**:
  - `/api/v2/steering/ipsec/pops` and `/api/v2/steering/ipsec/tunnels` (for IPsec)
  - and/or `/api/v2/steering/gre/pops` and `/api/v2/steering/gre/tunnels` (for GRE — same pattern as IPsec, substituting `gre` for `ipsec` in the endpoint path)
- **Reference docs:** [REST API v2 overview](https://docs.netskope.com/en/netskope-help/admin-console/rest-api/rest-api-v2-overview-312207/), [Create a tunnel on Netskope](https://docs.netskope.com/en/create-a-tunnel-on-netskope.html)

---

## Key Concepts

### Establishing Tunnels Workflow
The end device calls your platform, which in turn queries the customer's Netskope tenant for the closest POPs by providing the device's latitude and longitude. Netskope returns a list of NewEdge POPs with distance and capability attributes. Your platform then calls Netskope again to create the tunnel, specifying a primary POP and one or more backup POPs.

### Tunnels Are Decoupled from Policy
Netskope policy can match on many source attributes — user, user group, source IP, source country, User Confidence Index, OS, browser, access method (IPsec, GRE, Client, etc.), device classification, and HTTP header — instead of being tied to a specific tunnel. This means you can add, remove, or rename tunnels without needing to update policy to match.

### Release Deployment Days
Netskope updates POPs on a monthly basis. Each POP's JSON includes a `releasedeploymentday` value. **Avoid pairing two POPs in the same datacenter/metro (e.g., two `NYC<x>` POPs) as primary and backup if they share the same deployment day** (e.g., both `Day-2`) — a shared maintenance window would put both tunnels at risk at the same time.

---

## 1. Get Netskope POP Locations

Query the closest Netskope NewEdge POPs to an end device by latitude/longitude.

**Endpoint:** `GET /api/v2/steering/ipsec/pops`

```bash
curl -X GET \
  'https://<customer-tenant>.goskope.com/api/v2/steering/ipsec/pops?lat=37.354107&long=-121.955238&offset=0&limit=100' \
  -H 'accept: application/json' \
  -H 'Netskope-Api-Token: <token>' \
  -H 'User-Agent: <Vendor-Product-Version>'
```

**Response (excerpt — closest POPs first):**
```json
{
  "total": 109,
  "result": [
    {
      "id": "0x00CA",
      "name": "sfo1",
      "region": "US-CA",
      "location": "San Francisco, CA, US",
      "gateway": "163.116.140.38",
      "probeip": "10.140.6.216",
      "distance": "30 miles",
      "acceptingtunnels": true,
      "options": {
        "phase1": {
          "ikeversion": "2",
          "encryptionalgo": "AES128-CBC, AES192-CBC, AES256-CBC",
          "integrityalgo": "SHA1, SHA256, SHA384, SHA512",
          "dhgroup": "14, 15, 16, 18",
          "salifetime": "8h",
          "dpd": true
        },
        "phase2": {
          "encryptionalgo": "AES128-CBC, AES128-GCM, AES192-GCM, AES256-CBC, AES256-GCM, Null",
          "integrityalgo": "SHA1, SHA256, SHA384, SHA512",
          "dhgroup": "14, 15, 16, 18",
          "pfs": true,
          "salifetime": "2h"
        }
      },
      "bandwidth": "50 mbps, 100 mbps, 150 mbps, 200 mbps, 250 mbps",
      "releasedeploymentday": "Day-3"
    },
    {
      "id": "0x00AA",
      "name": "lax1",
      "region": "US-CA",
      "location": "Los Angeles, CA, US",
      "gateway": "163.116.132.38",
      "probeip": "10.132.6.216",
      "distance": "309 miles",
      "acceptingtunnels": true,
      "options": {
        "phase1": {
          "ikeversion": "2",
          "encryptionalgo": "AES128-CBC, AES192-CBC, AES256-CBC",
          "integrityalgo": "SHA1, SHA256, SHA384, SHA512",
          "dhgroup": "14, 15, 16, 18",
          "salifetime": "8h",
          "dpd": true
        },
        "phase2": {
          "encryptionalgo": "AES128-CBC, AES128-GCM, AES192-GCM, AES256-CBC, AES256-GCM, Null",
          "integrityalgo": "SHA1, SHA256, SHA384, SHA512",
          "dhgroup": "14, 15, 16, 18",
          "pfs": true,
          "salifetime": "2h"
        }
      },
      "bandwidth": "50 mbps, 100 mbps, 150 mbps, 200 mbps, 250 mbps",
      "releasedeploymentday": "Day-2"
    }
  ]
}
```

**Key fields:**
- `name` — the POP identifier used later as an entry in the tunnel's `pops` array
- `acceptingtunnels` — only select POPs where this is `true`
- `distance` — results are sorted closest-first
- `options.phase1` / `options.phase2` — supported IKE/IPsec cipher suites, DH groups, and SA lifetimes for that POP
- `releasedeploymentday` — used to avoid pairing primary/backup POPs with overlapping maintenance windows

**Use case:** Resolve an end device's location to a primary and backup POP candidate before creating a tunnel.

---

## 2. Create an IPsec Tunnel

Create a tunnel to one or more POPs (typically a primary and a backup) in a single call.

**Endpoint:** `POST /api/v2/steering/ipsec/tunnels`

```bash
curl -X POST \
  'https://<customer-tenant>.goskope.com/api/v2/steering/ipsec/tunnels' \
  -H 'accept: application/json' \
  -H 'Netskope-Api-Token: <token>' \
  -H 'Content-Type: application/json' \
  -H 'User-Agent: <Vendor-Product-Version>' \
  -d '{
    "encryption": "AES128-CBC",
    "site": "remotesite1",
    "srcidentity": "192.168.1.1",
    "psk": "netskoperocks",
    "srcipidentity": "192.168.1.1",
    "pops": [
      "sfo1", "lax1"
    ],
    "bandwidth": 50,
    "notes": "string",
    "vendor": "jenga",
    "sourcetype": "Mixed",
    "template": "string",
    "enable": true,
    "options": {
      "rekey": false,
      "reauth": false,
      "xff": {
        "enable": true,
        "iplist": [
          "192.168.1.1"
        ]
      }
    }
  }'
```

**Response:**
```json
{
  "status": 201,
  "result": "tunnel created successfully",
  "data": [
    {
      "id": 199273,
      "site": "remotesite1",
      "enabled": true,
      "pops": [
        {
          "name": "sfo1",
          "gateway": "163.116.140.38",
          "probeip": "10.140.6.216",
          "primary": true,
          "status": "down",
          "since": "unknown",
          "throughput": "unknown"
        },
        {
          "name": "lax1",
          "gateway": "163.116.132.38",
          "probeip": "10.132.6.216",
          "primary": false,
          "status": "down",
          "since": "unknown",
          "throughput": "unknown"
        }
      ],
      "vendor": "jenga",
      "template": "string",
      "sourcetype": "Mixed",
      "notes": "string",
      "bandwidth": 50,
      "encryption": "AES128-CBC",
      "srcidentity": "192.168.1.1",
      "srcipidentity": "192.168.1.1",
      "options": {
        "rekey": false,
        "reauth": false,
        "xff": {
          "enabled": true,
          "iplist": ["192.168.1.1"]
        }
      },
      "version": 2
    }
  ]
}
```

**Field notes:**
- `pops` — array of POP `name` values from workflow 1 (first entry becomes `primary: true` in the response)
- `bandwidth` — a plain number (no `mbps` suffix), matching one of the values offered in the POP's `bandwidth` field
- `sourcetype` — one of `User`, `Machine`, `IoT`, `Guest Wifi`, `Mixed`
- `notes`, `vendor`, `template` — optional, freeform
- `options.rekey` / `options.reauth` — some SD-WAN vendors require these enabled; for example, Aruba EdgeConnect uses a rekey interval of `480` on its side
- `options.xff.enable` — must be `true` if you provide an `iplist`; leaving it `false` while supplying `iplist` returns a validation error
- Store the returned `id` — it's required to look up, update, or delete this tunnel later

**Use case:** Provision a new site's tunnel to Netskope as part of an automated onboarding flow.

---

## 3. List Tunnels

Retrieve all tunnels configured in the tenant, including status, throughput, and the tunnel `id` needed for update/delete calls.

**Endpoint:** `GET /api/v2/steering/ipsec/tunnels`

```bash
curl -X GET \
  'https://<customer-tenant>.goskope.com/api/v2/steering/ipsec/tunnels?offset=0&limit=100' \
  -H 'accept: application/json' \
  -H 'Netskope-Api-Token: <token>' \
  -H 'User-Agent: <Vendor-Product-Version>'
```

**Response:**
```json
{
  "status": 200,
  "total": 1,
  "result": [
    {
      "id": 38,
      "site": "remotesite99",
      "enabled": true,
      "pops": [
        {
          "name": "sfo1",
          "gateway": "163.116.140.38",
          "probeip": "10.140.6.216",
          "primary": true,
          "status": "down",
          "since": "unknown",
          "throughput": "unknown"
        },
        {
          "name": "lax1",
          "gateway": "163.116.132.38",
          "probeip": "10.132.6.216",
          "primary": false,
          "status": "down",
          "since": "unknown",
          "throughput": "unknown"
        }
      ],
      "vendor": "weinhardsvendor",
      "template": "string",
      "sourcetype": "Mixed",
      "notes": "weinhardsnote",
      "bandwidth": 50,
      "encryption": "AES128-CBC",
      "srcidentity": "192.168.99.1",
      "srcipidentity": "192.168.99.1",
      "options": {
        "rekey": false,
        "reauth": false,
        "xff": {
          "enabled": true,
          "iplist": ["192.168.99.1"]
        },
        "qos": {
          "enabled": false,
          "linkid": 0
        }
      },
      "version": 2
    }
  ]
}
```

**Use case:** Look up a tunnel's `id` before updating or deleting it, or poll `pops[].status` to confirm both the primary and backup tunnel have come `up` after creation.

---

## 4. Delete a Tunnel

Remove a tunnel using its `id` (from workflow 2 or 3).

**Endpoint:** `DELETE /api/v2/steering/ipsec/tunnels/{id}`

```bash
curl -X DELETE \
  'https://<customer-tenant>.goskope.com/api/v2/steering/ipsec/tunnels/38' \
  -H 'accept: application/json' \
  -H 'Netskope-Api-Token: <token>' \
  -H 'User-Agent: <Vendor-Product-Version>'
```

**Response:**
```json
{
  "status": 200,
  "result": "tunnel deleted successfully"
}
```

**Use case:** Decommission a tunnel when a site is offboarded or being re-provisioned to different POPs.

**Note:** Netskope also supports updating an existing tunnel's configuration by `id`. Consult the [Netskope REST API v2 documentation](https://docs.netskope.com/en/netskope-help/admin-console/rest-api/rest-api-v2-overview-312207/) for the current update endpoint and payload, as it was not included in the source material for this guide.

---

## Best Practices

- Always send a `User-Agent` string formatted as `<Vendor-Product-Version>` on every call — Netskope uses this to attribute integration traffic
- Only select POPs where `acceptingtunnels` is `true`
- Pick a primary and backup POP with **different** `releasedeploymentday` values, especially if they're in the same metro/datacenter
- Store the tunnel `id` returned at creation time — there's no way to search tunnels by `site` name alone in this API surface, so tracking IDs locally avoids extra list calls
- Set `enable: true`/`false` deliberately — a created tunnel can be provisioned but left disabled
- If your SD-WAN platform requires periodic rekeying or reauthentication (e.g., Aruba EdgeConnect), set `options.rekey` / `options.reauth` accordingly rather than leaving Netskope's defaults

---

## Integration Tips

### Error Handling
- If `options.xff.iplist` is populated, `options.xff.enable` must be `true` or the request is rejected
- Validate that the POP `name` values in your `pops` array come from a current call to workflow 1 — POP availability and capabilities can change with monthly deployments

### Idempotency
- Persist the tunnel `id` from the creation response (workflow 2) for later reference
- If the ID is lost, use workflow 3 to look up the tunnel by `site` or `vendor` field

### Timing
- Immediately after creation, POP tunnel `status` will show `down` — this is expected while the IKE/IPsec negotiation completes
- Poll workflow 3 until both `pops[].status` entries show `up` before considering the site fully provisioned

### Testing
- Provision a single test tunnel to a nearby POP before automating bulk rollout
- Confirm both the primary and backup POP negotiate successfully, not just the primary

---

## Common Patterns

### New Site Tunnel Provisioning
```
1. End device reports its latitude/longitude to your platform
2. Query Netskope for the closest POPs (workflow 1)
3. Select a primary and backup POP, avoiding matching releasedeploymentday values in the same metro
4. Create the tunnel referencing both POPs (workflow 2)
5. Poll tunnel status (workflow 3) until both POPs report status "up"
```

### Tunnel Decommissioning
```
1. List tunnels to find the target tunnel's id (workflow 3)
2. Delete the tunnel (workflow 4)
```

---

## Troubleshooting

**Problem:** Tunnel creation fails validation related to `xff`
- **Solution:** Set `options.xff.enable` to `true` whenever `options.xff.iplist` is populated — leaving `enable` as `false` with a populated list returns an error.

**Problem:** Both primary and backup tunnels go down at the same time
- **Solution:** Check whether the two POPs share the same `releasedeploymentday` in the same metro/datacenter (e.g., two `NYC<x>` POPs both on `Day-2`). Re-provision the backup to a POP with a different deployment day.

**Problem:** SD-WAN device repeatedly renegotiates or drops the tunnel
- **Solution:** Confirm `options.rekey` / `options.reauth` match what your SD-WAN vendor expects — some vendors (e.g., Aruba EdgeConnect) require rekey enabled with a specific interval on their side.

**Problem:** Tunnel `status` stays `down` long after creation
- **Solution:** Verify the selected POP had `acceptingtunnels: true` at the time of creation, and confirm the PSK/identity values match what's configured on your CPE/SD-WAN device.

---

## API Reference

| Workflow | Endpoint | Method | Purpose |
|----------|----------|--------|---------|
| 1 | `/api/v2/steering/ipsec/pops` | GET | Get nearby Netskope NewEdge POP locations |
| 2 | `/api/v2/steering/ipsec/tunnels` | POST | Create an IPsec tunnel |
| 3 | `/api/v2/steering/ipsec/tunnels` | GET | List existing tunnels |
| 4 | `/api/v2/steering/ipsec/tunnels/{id}` | DELETE | Delete a tunnel |

**GRE variant:** The same workflow pattern applies to GRE tunnels using `/api/v2/steering/gre/pops` and `/api/v2/steering/gre/tunnels` in place of the `ipsec` paths above.

---

## Support & Additional Resources

- **REST API v2 overview:** https://docs.netskope.com/en/netskope-help/admin-console/rest-api/rest-api-v2-overview-312207/
- **Create a tunnel on Netskope:** https://docs.netskope.com/en/create-a-tunnel-on-netskope.html
- **Questions?** Contact your Netskope account team for API support.
