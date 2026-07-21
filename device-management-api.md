# Device Management API Reference

The Device Management API lets you register, configure, and monitor access-control devices (readers, controllers, and gateways) programmatically. Use it to integrate device provisioning into your own applications or automation scripts.

**Base URL:** `https://api.accesscloud.example.com/v1`

---

## Authentication

All requests require a bearer token in the `Authorization` header.

```
Authorization: Bearer <your_access_token>
```

Tokens are issued via the [OAuth 2.0 Client Credentials flow](#) and expire after 60 minutes. Requests made without a valid token return `401 Unauthorized`.

```bash
curl -X POST https://api.accesscloud.example.com/v1/oauth/token \
  -d "grant_type=client_credentials" \
  -d "client_id=YOUR_CLIENT_ID" \
  -d "client_secret=YOUR_CLIENT_SECRET"
```

**Response**

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

---

## Devices

### List devices

```
GET /devices
```

Returns a paginated list of devices registered to your account.

**Query parameters**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `status` | string | No | Filter by device status: `online`, `offline`, `maintenance` |
| `site_id` | string | No | Filter devices belonging to a specific site |
| `limit` | integer | No | Number of results per page (default: `25`, max: `100`) |
| `cursor` | string | No | Pagination cursor from a previous response |

**Example request**

```bash
curl -X GET "https://api.accesscloud.example.com/v1/devices?status=online&limit=2" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

**Example response** — `200 OK`

```json
{
  "data": [
    {
      "device_id": "dev_8f2a1c",
      "name": "Main Entrance Reader",
      "type": "card_reader",
      "site_id": "site_014",
      "status": "online",
      "firmware_version": "3.2.1",
      "last_seen": "2026-07-20T09:14:22Z"
    },
    {
      "device_id": "dev_3b7e90",
      "name": "Loading Dock Controller",
      "type": "door_controller",
      "site_id": "site_014",
      "status": "online",
      "firmware_version": "3.2.1",
      "last_seen": "2026-07-20T09:15:01Z"
    }
  ],
  "pagination": {
    "next_cursor": "eyJvZmZzZXQiOjJ9",
    "has_more": true
  }
}
```

---

### Get a device

```
GET /devices/{device_id}
```

Returns full configuration and status for a single device.

**Path parameters**

| Parameter | Type | Description |
|---|---|---|
| `device_id` | string | Unique identifier of the device |

**Example response** — `200 OK`

```json
{
  "device_id": "dev_8f2a1c",
  "name": "Main Entrance Reader",
  "type": "card_reader",
  "site_id": "site_014",
  "status": "online",
  "firmware_version": "3.2.1",
  "ip_address": "10.4.2.18",
  "last_seen": "2026-07-20T09:14:22Z",
  "created_at": "2025-11-03T08:00:00Z"
}
```

If no device matches the given ID, the API returns `404 Not Found`.

---

### Register a new device

```
POST /devices
```

Registers a new device and returns its assigned `device_id`.

**Request body**

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Display name for the device |
| `type` | string | Yes | One of `card_reader`, `door_controller`, `gateway` |
| `site_id` | string | Yes | Site the device belongs to |
| `serial_number` | string | Yes | Manufacturer serial number |

**Example request**

```bash
curl -X POST https://api.accesscloud.example.com/v1/devices \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "Content-Type: application/json" \
  -d '{
        "name": "West Wing Reader",
        "type": "card_reader",
        "site_id": "site_014",
        "serial_number": "SN-88213-A"
      }'
```

**Example response** — `201 Created`

```json
{
  "device_id": "dev_a91cf2",
  "name": "West Wing Reader",
  "type": "card_reader",
  "site_id": "site_014",
  "status": "provisioning",
  "created_at": "2026-07-21T11:02:44Z"
}
```

> **Note:** Newly registered devices start in `provisioning` status and transition to `online` once they complete their first successful heartbeat, typically within 2 minutes.

---

### Update a device

```
PATCH /devices/{device_id}
```

Updates one or more fields on an existing device. Only include the fields you want to change.

**Example request**

```bash
curl -X PATCH https://api.accesscloud.example.com/v1/devices/dev_a91cf2 \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "Content-Type: application/json" \
  -d '{ "name": "West Wing Reader (Ground Floor)" }'
```

**Example response** — `200 OK`

```json
{
  "device_id": "dev_a91cf2",
  "name": "West Wing Reader (Ground Floor)",
  "status": "online",
  "updated_at": "2026-07-21T11:10:02Z"
}
```

---

### Delete a device

```
DELETE /devices/{device_id}
```

Permanently removes a device from your account. This action cannot be undone — the device must be re-registered to restore access.

**Example response** — `204 No Content`

---

## Errors

The API uses standard HTTP status codes. Error responses include a machine-readable `code` and a human-readable `message`.

```json
{
  "error": {
    "code": "device_not_found",
    "message": "No device exists with ID 'dev_xxxxxx'."
  }
}
```

| Status | Code | Meaning |
|---|---|---|
| `400` | `invalid_request` | A required field is missing or malformed |
| `401` | `unauthorized` | Missing or expired access token |
| `403` | `forbidden` | Token is valid but lacks permission for this action |
| `404` | `device_not_found` | No device exists with the given ID |
| `409` | `duplicate_serial_number` | A device with this serial number is already registered |
| `429` | `rate_limited` | Too many requests — see [Rate limits](#rate-limits) |
| `500` | `internal_error` | Something went wrong on our end — safe to retry |

---

## Rate limits

Requests are limited to **120 requests per minute** per access token. When you exceed this limit, the API returns `429 Too Many Requests` with a `Retry-After` header (in seconds).

```
HTTP/1.1 429 Too Many Requests
Retry-After: 8
```

---

## Changelog

| Version | Date | Change |
|---|---|---|
| v1.3 | 2026-06-15 | Added `firmware_version` to device list and detail responses |
| v1.2 | 2026-04-02 | Added `site_id` filter to `GET /devices` |
| v1.1 | 2026-01-20 | Added `PATCH /devices/{device_id}` endpoint |
| v1.0 | 2025-11-03 | Initial release |

---

*This is a sample reference document written to demonstrate API documentation style and structure — endpoints and data shown are illustrative, not a live API.*
