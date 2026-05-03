# CLAUDE.md — UB-Mager Docs

> Context file for AI assistants working on this codebase.

## Project Overview

Bruno API collection for the **UB-Mager** ride-hailing platform. Contains all API endpoint definitions, request examples, and environment configurations for testing the backend API.

## Tool

- **Bruno** — Open-source API client (alternative to Postman)
- Collection format: `.bru` files (Bruno markup)
- Website: https://www.usebruno.com/

## Collection Structure

```
Auth/
├── Register.bru
├── Login.bru
├── Refresh Token.bru
└── Logout.bru

Passenger/
├── (Profile, booking, history, rating endpoints)

Driver/
├── (Registration, status, location, ride lifecycle, earnings)

Admin/
├── Dashboard Stats.bru
├── List Drivers.bru
├── Get Driver Detail.bru
├── Get Driver Rides.bru
├── Verify Driver.bru
├── Toggle Driver Status.bru
├── List Rides.bru
├── Get Ride Detail.bru
├── Cancel Ride.bru
├── Bulk Cancel Stuck Rides.bru
└── Recent Activity.bru

Analytics/
├── Revenue.bru
├── Daily Revenue.bru
├── Ride Stats.bru
├── Peak Hours.bru
├── Driver Leaderboard.bru
├── Demand Heatmap.bru
└── Demand Prediction.bru

WebSocket/
├── (WebSocket connection reference)

environments/
├── Local.bru          → localhost configuration
└── Development.bru    → dev server configuration
```

## How to Use

1. Install [Bruno](https://www.usebruno.com/)
2. Open this folder as a Bruno collection
3. Select environment: `Local` or `Development`
4. Start with `Auth > Login` to get an access token (auto-set as variable)
5. Use other endpoints — token is automatically included

## Environments

- **Local** — Points to `http://localhost:8081` (local API server)
- **Development** — Points to deployed dev server

## Conventions

- **File naming**: Endpoint name as filename (e.g., `Dashboard Stats.bru`)
- **Folder structure**: Mirrors API domain grouping (Auth, Admin, Driver, etc.)
- **Variables**: Use Bruno environment variables for base URL and auth tokens
- **Auth flow**: Login response token is stored as collection variable, auto-attached to subsequent requests

## Adding New Endpoints

1. Create a `.bru` file in the appropriate folder
2. Use Bruno's format:
   ```
   meta {
     name: Endpoint Name
     type: http
     seq: 1
   }

   post {
     url: {{baseUrl}}/api/v1/path
     body: json
     auth: bearer
   }

   auth:bearer {
     token: {{accessToken}}
   }

   body:json {
     {
       "field": "value"
     }
   }
   ```

## Related Repos

- `ub-mager-api` — Main backend API (Go/Gin, port 8081)
- `ub-mager-ml` — ML demand prediction service (Python/FastAPI, port 8000)
- `ub-mager-dashboard` — Admin dashboard (Next.js, port 3000)
