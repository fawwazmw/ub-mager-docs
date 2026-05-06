# UB Mager API Documentation

> Bruno API collection for the UB Mager ride-hailing platform.

## Quick Start

1. Install [Bruno](https://www.usebruno.com/)
2. Open this folder as a collection
3. Select environment: `Local` or `Development`
4. Start with `Auth > Login` to get access token
5. All other endpoints auto-include the token

## Base URLs

| Environment | API | WebSocket |
|-------------|-----|-----------|
| Local | `http://localhost:8081/api/v1` | `ws://localhost:8081/ws` |
| Health/Ready | `http://localhost:8081/health` | — |

## Collections

| Folder | Description | Auth Required |
|--------|-------------|---------------|
| **Auth** | Register, Login, Refresh, Logout | No (public) |
| **Passenger** | Profile, ride booking, history, rating, chat | Yes |
| **Driver** | Registration, status, location, ride lifecycle | Yes |
| **Admin** | Dashboard, driver/ride/user management, reports | Yes (ADMIN role) |
| **Analytics** | Revenue, ride stats, peak hours, leaderboard | Yes (ADMIN role) |
| **Chat** | Send/get messages within a ride | Yes |
| **Reports** | Submit user reports | Yes |
| **Health** | Health check, readiness, version | No (public) |
| **WebSocket** | Real-time connection reference | Token via query param |

---

## API Conventions

### Response Format

Every response follows this structure:

```json
{
  "success": true,
  "data": { ... },
  "meta": { "page": 1, "per_page": 20, "total": 100, "total_pages": 5 },
  "error": null
}
```

**Success:**
```json
{ "success": true, "data": { "id": "...", "name": "..." } }
```

**Error:**
```json
{ "success": false, "error": { "code": "RIDE_NOT_FOUND", "message": "Ride not found" } }
```

**Paginated:**
```json
{
  "success": true,
  "data": [ ... ],
  "meta": { "page": 1, "per_page": 20, "total": 57, "total_pages": 3 }
}
```

### Error Codes

| Code | HTTP | Meaning |
|------|------|---------|
| `VALIDATION_ERROR` | 400 | Request body failed validation |
| `INVALID_ID` | 400 | UUID parameter is malformed |
| `INVALID_COORDINATES` | 400 | Lat/lng out of range (±90/±180) |
| `INVALID_EMAIL_DOMAIN` | 400 | Email must be @student.ub.ac.id |
| `UNAUTHORIZED` | 401 | Missing or invalid token |
| `TOKEN_EXPIRED` | 401 | Access token expired (use refresh) |
| `INVALID_CREDENTIALS` | 401 | Wrong phone/password |
| `FORBIDDEN` | 403 | Not authorized for this action |
| `RIDE_NOT_FOUND` | 404 | Ride doesn't exist |
| `DRIVER_NOT_FOUND` | 404 | Driver profile doesn't exist |
| `USER_NOT_FOUND` | 404 | User doesn't exist |
| `PHONE_EXISTS` | 409 | Phone already registered |
| `EMAIL_EXISTS` | 409 | Email already registered |
| `ACTIVE_RIDE_EXISTS` | 409 | User already has an active ride |
| `RIDE_UNAVAILABLE` | 409 | Ride already taken by another driver |
| `INTERNAL_ERROR` | 500 | Server error (never exposes details) |

### Authentication Flow

```
1. POST /auth/register  →  { access_token, user }  +  refresh_token (httpOnly cookie)
2. POST /auth/login     →  { access_token, user }  +  refresh_token (httpOnly cookie)
3. Use access_token in header:  Authorization: Bearer <token>
4. When token expires (TOKEN_EXPIRED error):
   POST /auth/refresh   →  new { access_token }  (uses cookie automatically)
5. POST /auth/logout    →  clears refresh cookie
```

**Access token:** expires in 15 minutes
**Refresh token:** expires in 7 days (httpOnly cookie, auto-sent)

### Registration Rules

- Email must end with `@student.ub.ac.id` (campus email only)
- Password minimum 8 characters
- Phone must be unique
- Role: `passenger` or `driver`

### Pagination

All list endpoints support:

| Param | Default | Description |
|-------|---------|-------------|
| `page` | 1 | Page number |
| `per_page` | 20 | Items per page (max 100) |

### Rate Limiting

| Endpoint | Limit |
|----------|-------|
| `POST /auth/register` | 5/min |
| `POST /auth/login` | 10/min |
| `POST /auth/refresh` | 30/min |
| `POST /rides` | 3/min per user |

---

## Ride Lifecycle

```
SEARCHING → MATCHED → DRIVER_EN_ROUTE → ARRIVED_AT_PICKUP → IN_PROGRESS → COMPLETED
                                                                          ↘ CANCELLED
```

| Status | Who triggers | Endpoint |
|--------|-------------|----------|
| SEARCHING | Passenger requests ride | `POST /rides` |
| MATCHED | Driver accepts | `PUT /rides/:id/accept` |
| DRIVER_EN_ROUTE | Driver starts driving to pickup | `PUT /rides/:id/status` |
| ARRIVED_AT_PICKUP | Driver arrives | `PUT /rides/:id/status` |
| IN_PROGRESS | Trip starts | `PUT /rides/:id/status` |
| COMPLETED | Trip ends | `PUT /rides/:id/status` |
| CANCELLED | Either party | `PUT /rides/:id/cancel` |

### Status Update Body

```json
{ "status": "DRIVER_EN_ROUTE" }
```

Valid transitions only — server rejects invalid jumps.

---

## WebSocket Protocol

**Connect:** `ws://localhost:8081/ws?token=<access_token>`

### Message Format

```json
{
  "type": "MESSAGE_TYPE",
  "payload": { ... },
  "timestamp": 1700000000000
}
```

### Client → Server

| Type | Payload | Who |
|------|---------|-----|
| `PING` | — | Any |
| `DRIVER_LOCATION` | `{ lat, lng, speed, heading }` | Driver |
| `SUBSCRIBE_RIDE` | `{ ride_id }` | Any (subscribe to ride updates) |
| `RIDE_ACCEPT` | `{ ride_id }` | Driver |
| `RIDE_REJECT` | `{ ride_id }` | Driver |

### Server → Client

| Type | Payload | Sent to |
|------|---------|---------|
| `CONNECTED` | — | On connect |
| `PONG` | — | Response to PING |
| `RIDE_REQUEST` | `{ ride_id, passenger_name, pickup, dropoff, fare, vehicle_type }` | Nearby drivers |
| `RIDE_MATCHED` | `{ ride_id, driver_id }` | Passenger |
| `RIDE_STATUS` | `{ ride_id, status }` | Both parties |
| `LOCATION_UPDATE` | `{ lat, lng, speed, heading }` | Passenger (driver's location) |
| `CHAT_MESSAGE` | `{ message_id, ride_id, sender_id, content, created_at }` | Other party in ride |
| `ERROR` | `{ message }` | Sender |

### Subscribe to Ride

After accepting/requesting a ride, subscribe to get real-time updates:

```json
{ "type": "SUBSCRIBE_RIDE", "payload": { "ride_id": "uuid" } }
```

---

## Chat

In-ride messaging between passenger and driver.

**Send:** `POST /rides/:id/messages` with `{ "content": "message text" }`
**Get history:** `GET /rides/:id/messages?limit=50`

Messages are also delivered in real-time via WebSocket (`CHAT_MESSAGE` type) to the other party if they're subscribed to the ride.

---

## Vehicle Types

| Value | Description |
|-------|-------------|
| `motorcycle` | Motor (2-wheel) |
| `car` | Mobil (4-wheel) |
| `car_xl` | Mobil besar (MPV/SUV) |

## Payment Methods

| Value | Description |
|-------|-------------|
| `cash` | Bayar tunai |
| `ewallet` | E-wallet |

## Campus Zones

Optional tagging for pickup/dropoff locations:

`FILKOM`, `FTP`, `FEB`, `FH`, `FK`, `FKG`, `FIA`, `FISIP`, `FMIPA`, `FPIK`, `FT`, `FPET`, `REKTORAT`, `GOR`, `OTHER`

---

## User Roles & Permissions

| Role | Can do |
|------|--------|
| `PASSENGER` | Request rides, rate drivers, send messages, submit reports |
| `DRIVER` | Accept rides, update location/status, send messages |
| `ADMIN` | All of above + manage drivers/users/rides/reports, view analytics |

## Badges (returned in driver profile)

| Badge | Criteria |
|-------|----------|
| `VERIFIED_STUDENT` | Registered with @student.ub.ac.id email |
| `VERIFIED_DRIVER` | Admin-verified driver |
| `TRUSTED_HELPER` | 50+ completed trips |
| `EXPERIENCED` | 100+ completed trips |
| `TOP_RATED` | Rating ≥ 4.8 with 20+ trips |

---

## Testing with Bruno

1. **Login first** — `Auth > Login` stores token automatically
2. **Use path params** — Set variables like `{{ride_id}}`, `{{driver_id}}` in Bruno
3. **Check environments** — Switch between Local/Development as needed
4. **WebSocket** — Use `wscat` or Postman for WS testing (Bruno doesn't support WS natively)

```bash
# WebSocket test with wscat
wscat -c "ws://localhost:8081/ws?token=YOUR_ACCESS_TOKEN"

# Send ping
{"type":"PING"}

# Subscribe to ride
{"type":"SUBSCRIBE_RIDE","payload":{"ride_id":"uuid-here"}}
```
