## On-Prem to Cloud: Vendor-Managed Integrations

## Context

Convergint is evolving how we integrate with physical security platforms.

Historically, many integrations to on-prem systems have relied on **on-prem agents/connectors** that Convergint develops, distributes, and maintains. This works, but it pushes ongoing complexity onto Convergint (SDK/version drift, customer network constraints, credential handling, upgrades, and site-specific edge cases).

This document describes the next evolution: vendors own and operate the on-prem integration surface (if any) and Convergint integrates **cloud-to-cloud**. We "meet in the cloud."

This is a **high-level** document intended to align on the integration contract (inventory + events). We expect to collaborate on the design specifics with each vendor.

**What changes**

- **Responsibility shift**: vendor manages the connector lifecycle and on-prem complexity behind the scenes.
- **Integration surface**: Convergint consumes inventory via vendor cloud APIs and receives live events via vendor webhooks.
- **Outcome**: faster onboarding, fewer on-prem dependencies, and a clearer shared contract for inventory + events.

**Why vendors are best positioned**

- Vendors have first-party knowledge of their platform and control over internal interfaces, schemas, and release cycles.
- Vendors can often provide a more reliable cloud pipeline (APIs + webhooks) than third parties constrained to public/on-prem SDK surfaces.

## What We Need From Vendors

- **Inventory API**: provide a cloud API that returns device inventory (including latest known status) for a given org/site.
- **Events Webhook**: publish at least device health events (online/offline) to a Convergint webhook endpoint. More event families are strongly preferred.
- **Org/Site model**: expose stable org/account and site/location identifiers, plus basic site metadata.

## Today's World (On-Prem Agents Owned by Convergint)

Convergint-owned agent runs at the customer site and forwards inventory/events to Convergint Cloud.

![Today's World](diagrams/diagram-1.svg)

## Target World (Vendor-Managed Cloud Model)

Vendor-owned connector (if any) runs at the customer site; Convergint integrates directly with the vendor cloud via APIs + webhooks.

![Target World](diagrams/diagram-2.svg)

## Scope

We're looking for two core capabilities from a vendor's cloud platform:

1. **Asset Inventory**
   - Pull a full list of devices/assets for a given Convergint customer/site.
   - At minimum, inventory must reflect each device's **latest known status**.
   - Inventory API must support **pagination** (inventories can contain hundreds or thousands of devices).
   - Inventory API must support **filtering by site/location** (e.g., "for this site, return all assets").
2. **Live Events (Push Preferred)**
   - **Baseline**: device health events (online/offline/last_seen) delivered via webhooks. *(If webhooks are not yet available, Convergint can poll inventory as an interim measure, provided each asset includes its last known status.)*
   - **Preferred**: additional event families the vendor already exposes through on-prem SDKs (alarms/trouble, access events, video/VMS events, etc.).

Every payload (inventory or event) must include enough metadata to reliably map vendor entities to the correct Convergint customer and site.

## Integration Model

### Mapping / Binding (Tenant + Site)

We prefer **credential-bound integrations**:

- Convergint stores vendor-issued credentials (API keys, OAuth tokens, etc.) and binds them to a specific Convergint customer and one or more sites.
- Vendors send webhooks to a Convergint endpoint secured by a vendor-provided signing secret.
- The webhook payload must include the vendor's `org_id` and `site_id` (or equivalent), and Convergint resolves those to our internal customer/site mapping.

This is consistent with how many modern cloud integrations behave today, where vendor "Account/Org" and "Location/Site" identifiers exist and can be used for resolution.

![Mapping/Binding](diagrams/diagram-3.svg)

## Minimum Data Requirements

At a minimum, we need the following fields.

**Identifier Best Practice**: We strongly recommend using **UUIDs** (or equivalent globally unique identifiers) for all entity IDs—devices, sites, accounts, and events. Globally unique IDs simplify event processing and eliminate ambiguity when correlating devices across locations.

### Account Metadata (via API key)

- **Vendor org/account ID**
- **Account name**

### Site Metadata (per site)

- **Vendor site/location ID**
- **Site name**
- **Site address** (or coordinates) and **timezone** (preferred; at minimum, one stable site descriptor)

### Inventory (per device)

- Unique device ID
- Device name or label
- **Device type** (camera, door, panel, etc.)
- **Vendor site/location ID** (or a stable equivalent)
- Online/offline status (optional if fully covered by events, but recommended)
- MAC address

**Nice to Have**

- IP
- Hostname
- Site name
- Site address and/or coordinates
- Timezone
- Last seen timestamp (if not covered in events)
- Vendor model / firmware version
- Warranty expiry date
- End of support date
- System/server ID (to distinguish devices when multiple logical systems of the same vendor exist at one site)

### Health Events (baseline)

- Unique device ID
- **Status** (`online` / `offline` or equivalent)
- Timestamp
- **Vendor org/account ID**
- **Vendor site/location ID**
- Stable event identifier (preferred) to support idempotency/deduplication

### Additional Events (preferred)

The vendor should expose an event catalog. Convergint prefers webhooks for all event families the platform can publish, but will start with health.

Examples of event families we may adopt (depending on vendor support):

- Alarm/trouble conditions
- Access-control activity (granted/denied, door forced/held, etc.)
- Video/VMS health (recording failure, stream loss, storage issues)

## Sample API Request/Response

The following examples illustrate the expected shape of API responses. These are illustrative—vendors may adapt field names and structure as needed, as long as the required data is present.

### Account API

Returns the account associated with the current API key. This solves the bootstrap problem: Convergint can discover which vendor account the credentials belong to.

```
GET /api/v1/account
```

```json
{
  "account_id": "a1b2c3d4-5678-90ab-cdef-1234567890ab",
  "name": "Acme Security Corp"
}
```

### Sites API (paginated)

Since the API key is bound to a single account, the account is implicit.

```
GET /api/v1/account/sites?page=1&per_page=50
```

```json
{
  "sites": [
    {
      "site_id": "5e6693c0-091d-47a2-b90a-6c15531b3c50",
      "name": "US - 101 Chicago, IL",
      "address": "2000 Center Drive, Suite 315A, Hoffman Estates, IL 60192",
      "timezone": "America/Chicago"
    }
  ],
  "pagination": {
    "page": 1,
    "per_page": 50,
    "total_pages": 3,
    "total_count": 127
  }
}
```

### Inventory API (paginated)

```
GET /api/v1/sites/{site_id}/inventory?page=1&per_page=100
```

```json
{
  "devices": [
    {
      "device_id": "652efeac-8567-4253-9d61-bbc842863d33",
      "name": "Front Lobby Door",
      "type": "door",
      "status": "online",
      "mac_address": "00:0f:e5:14:b2:1e",
      "site_id": "5e6693c0-091d-47a2-b90a-6c15531b3c50"
    }
  ],
  "pagination": {
    "page": 1,
    "per_page": 100,
    "total_pages": 5,
    "total_count": 487
  }
}
```

### Health Event Payload (webhook)

```json
{
  "event_id": "698219ad-4de9-896d-3be0-66cb00000001",
  "event_type": "health",
  "device_id": "652efeac-8567-4253-9d61-bbc842863d33",
  "timestamp": "2026-02-04T14:32:00Z",
  "account_id": "a1b2c3d4-5678-90ab-cdef-1234567890ab",
  "site_id": "5e6693c0-091d-47a2-b90a-6c15531b3c50",
  "data": {
    "status": "offline"
  }
}
```

Initially, we are focused on health events. However, this structure should be flexible enough to accommodate other event types (access, alarm, etc.) in the future. The `event_type` field distinguishes between event families, and `data` contains the event-specific attributes.

## Target Workflows

### 1. Inventory Sync (Poll)

- Convergint calls vendor cloud APIs to fetch inventory.
- Convergint receives a complete inventory payload from the vendor cloud.

![Inventory Sync](diagrams/diagram-4.svg)

### 2. Real-Time Events (Push via Webhooks)

- Vendor cloud publishes events to a Convergint webhook endpoint.
- Convergint verifies the signature and processes the event.
- Device status (and related metadata) is updated; events can be recorded for downstream use.

![Webhooks](diagrams/diagram-5.svg)

### Fallback Behavior (Empty / Delayed / Offline)

We expect real-world gaps. A vendor integration should support:

- **Empty/bootstrapping state**: initial inventory sync can run before any webhooks have been received.
- **Webhook delays**: timestamps and idempotency keys (or stable event IDs) to dedupe/replay safely.
- **Temporary webhook outage**: Convergint can fall back to periodic polling to reconcile device status.
- **Partial coverage**: if certain device classes cannot publish events, inventory must include their latest status.

## Integration Lifecycle

1. **Onboarding**
   - Configure credentials (API key / OAuth app).
   - Bind vendor org + site(s) to Convergint customer/site records.
   - Configure webhook destination + signing secret.
2. **Collection**
   - Poll inventory on a schedule.
   - Receive events continuously via webhooks.
3. **Data Mapping**
   - Convergint maps vendor device taxonomy and statuses into a consistent Convergint schema.
4. **Operational Expectations**
   - Signed webhooks with replay protection (timestamp window recommended).
   - Retry semantics and idempotency to prevent duplicates.

## Next Steps (Collaborative)

We recognize this model may be net-new work for vendors that historically integrate primarily via on-prem SDKs.

If you're open to pursuing this, we'd like to collaborate on the design and implementation details together.

**What we can provide**

- A sample schema for inventory + baseline health events
- A test webhook endpoint and guidance for signing/replay protection
- Working sessions to align on your org/site model and event catalog

*For questions or next steps, contact:*

**Ryan Johnston** – [ryan.johnston@convergint.com](mailto:ryan.johnston@convergint.com) Director Engineering

**Juan Pemberthy** – [juan.pemberthy@convergint.com](mailto:juan.pemberthy@convergint.com) Principal Software Engineer
