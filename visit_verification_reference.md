# Visit Verification API Reference

The Visit Verification API lets home health and personal care agencies record, retrieve, and validate caregiver visit data — including check-in/check-out events, GPS location capture, and service documentation — to support Electronic Visit Verification (EVV) compliance.

**Base URL:** `https://api.kantimehealth.example.com/v1`

---

## Authentication

All requests require a bearer token in the `Authorization` header, issued per agency account.

```
Authorization: Bearer <your_access_token>
```

Tokens expire after 60 minutes. Requests made without a valid token return `401 Unauthorized`.

```bash
curl -X POST https://api.kantimehealth.example.com/v1/oauth/token \
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

## Visits

### List visits

```
GET /visits
```

Returns a paginated list of scheduled and completed visits.

**Query parameters**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `patient_id` | string | No | Filter visits for a specific patient |
| `caregiver_id` | string | No | Filter visits assigned to a specific caregiver |
| `status` | string | No | `scheduled`, `in_progress`, `completed`, `missed` |
| `date_from` / `date_to` | date | No | Filter by visit date range (`YYYY-MM-DD`) |
| `limit` | integer | No | Results per page (default `25`, max `100`) |

**Example request**

```bash
curl -X GET "https://api.kantimehealth.example.com/v1/visits?status=completed&date_from=2026-07-01&date_to=2026-07-07" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

**Example response** — `200 OK`

```json
{
  "data": [
    {
      "visit_id": "vis_7f21ac",
      "patient_id": "pat_4021",
      "caregiver_id": "cg_1187",
      "service_type": "personal_care",
      "scheduled_start": "2026-07-03T09:00:00Z",
      "scheduled_end": "2026-07-03T10:00:00Z",
      "status": "completed",
      "evv_verified": true
    }
  ],
  "pagination": {
    "next_cursor": "eyJvZmZzZXQiOjF9",
    "has_more": false
  }
}
```

---

### Check in to a visit

```
POST /visits/{visit_id}/check-in
```

Records the start of a visit, including the caregiver's GPS location at the time of check-in. This is the primary event used for EVV compliance.

**Request body**

| Field | Type | Required | Description |
|---|---|---|---|
| `timestamp` | string (ISO 8601) | Yes | Time of check-in |
| `latitude` | number | Yes | GPS latitude at check-in |
| `longitude` | number | Yes | GPS longitude at check-in |
| `method` | string | Yes | `mobile_app`, `telephony`, or `manual` |

**Example request**

```bash
curl -X POST https://api.kantimehealth.example.com/v1/visits/vis_7f21ac/check-in \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "Content-Type: application/json" \
  -d '{
        "timestamp": "2026-07-03T09:02:14Z",
        "latitude": 39.7684,
        "longitude": -86.1581,
        "method": "mobile_app"
      }'
```

**Example response** — `200 OK`

```json
{
  "visit_id": "vis_7f21ac",
  "status": "in_progress",
  "check_in": {
    "timestamp": "2026-07-03T09:02:14Z",
    "latitude": 39.7684,
    "longitude": -86.1581,
    "method": "mobile_app",
    "within_geofence": true
  }
}
```

> **Note:** If the check-in location falls outside the patient's registered service address radius, `within_geofence` returns `false` and the visit is flagged for agency review rather than rejected outright.

---

### Check out of a visit

```
POST /visits/{visit_id}/check-out
```

Records the end of a visit and, optionally, service documentation notes.

**Request body**

| Field | Type | Required | Description |
|---|---|---|---|
| `timestamp` | string (ISO 8601) | Yes | Time of check-out |
| `latitude` | number | Yes | GPS latitude at check-out |
| `longitude` | number | Yes | GPS longitude at check-out |
| `tasks_completed` | array of strings | No | Service task codes completed during the visit |
| `notes` | string | No | Free-text visit notes |

**Example request**

```bash
curl -X POST https://api.kantimehealth.example.com/v1/visits/vis_7f21ac/check-out \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "Content-Type: application/json" \
  -d '{
        "timestamp": "2026-07-03T10:01:40Z",
        "latitude": 39.7684,
        "longitude": -86.1581,
        "tasks_completed": ["bathing_assist", "medication_reminder"],
        "notes": "Patient reported stable condition, no concerns noted."
      }'
```

**Example response** — `200 OK`

```json
{
  "visit_id": "vis_7f21ac",
  "status": "completed",
  "evv_verified": true,
  "duration_minutes": 59
}
```

---

## Patients

### Get patient details

```
GET /patients/{patient_id}
```

Returns demographic and care-plan summary information for a patient.

**Example response** — `200 OK`

```json
{
  "patient_id": "pat_4021",
  "name": "J. Alvarez",
  "service_address": {
    "line1": "482 Oakhurst Ln",
    "city": "Indianapolis",
    "state": "IN",
    "zip": "46208"
  },
  "care_plan_id": "cp_2290",
  "active": true
}
```

Patient names and identifying details are masked or tokenized in non-production environments in accordance with HIPAA data-handling requirements.

---

## Errors

| Status | Code | Meaning |
|---|---|---|
| `400` | `invalid_request` | A required field is missing or malformed |
| `401` | `unauthorized` | Missing or expired access token |
| `404` | `visit_not_found` | No visit exists with the given ID |
| `409` | `visit_already_checked_in` | Check-in already recorded for this visit |
| `422` | `geofence_violation` | Check-in location is outside the allowed service radius (visit is flagged, not blocked) |
| `429` | `rate_limited` | Too many requests — see [Rate limits](#rate-limits) |
| `500` | `internal_error` | Something went wrong on our end — safe to retry |

---

## Rate limits

Requests are limited to **120 requests per minute** per access token. Exceeding this returns `429 Too Many Requests` with a `Retry-After` header (in seconds).

---

## Changelog

| Version | Date | Change |
|---|---|---|
| v1.2 | 2026-05-10 | Added `within_geofence` field to check-in response |
| v1.1 | 2026-02-18 | Added `tasks_completed` to check-out request body |
| v1.0 | 2025-10-01 | Initial release |

---

*This is a sample reference document written to demonstrate API documentation style for a home health / EVV domain. It is an illustrative writing sample only — endpoints, data, and the "kantimehealth.example.com" domain are fictional and not sourced from or representative of any real vendor's actual API.*
