---
name: here-calculate-a-route
description: Calculate a driving, truck, pedestrian, bicycle, scooter or EV route between waypoints with the HERE Routing API v8, and read the resulting sections, maneuvers, tolls and EV charging stops.
api: HERE Routing API v8
spec: openapi/here-routing-v8-openapi.yml
base_url: https://router.hereapi.com/v8
operations:
  - Routing API v8 calculateRoutes
  - Routing API v8 calculateRoutesPost
  - Routing API v8 getRoutesByHandle
  - Routing API v8 importRoute
  - Routing API v8 getVersion
---

# Calculate a route with HERE Routing v8

## Authenticate

Pick one of two schemes — both are declared on every operation in the spec:

- `ApiKey`: append `apiKey=<key>` as a query parameter. Simplest; fine for server-side calls.
- `Bearer`: `Authorization: Bearer <token>` where the token comes from `POST https://account.api.here.com/oauth2/token`
  (`OAuth 2.0 Access Token getOAuth2AccessToken`, grant type `client_credentials`, `private_key_jwt` client auth).

Use `Bearer` when the caller is a backend service with rotating credentials; use `ApiKey` only where the key is not exposed to a browser.

## Calculate the route

`GET /routes` (`Routing API v8 calculateRoutes`) on `https://router.hereapi.com/v8`.

Required-in-practice parameters:

- `transportMode` — `car`, `truck`, `pedestrian`, `bicycle`, `scooter`, `bus`, `privateBus`, `taxi`
- `origin` and `destination` — `lat,lng`
- `return` — ask for what you need: `polyline,summary,actions,instructions,tolls,elevation`
- `via` — repeat the parameter for intermediate waypoints, in order

Use `POST /routes` (`Routing API v8 calculateRoutesPost`) when the request would exceed a safe URL length — long `via` lists, large `avoid[areas]` polygons, or detailed `ev[...]` consumption curves.

## Read the response

A `Route` has many `Section`. Each section carries `departure`, `arrival`, `transport`, `summary`
(`duration`, `length`, `baseDuration`), and — when requested — `polyline`, `actions` and `notices`.

The `polyline` is **Flexible Polyline** encoded, not Google-encoded. Decode it with HERE's own
`@here/flexpolyline` (npm, 0.1.0) or the reference implementations at
https://github.com/heremaps/flexible-polyline. Decoding it as a Google polyline produces coordinates that are silently wrong.

Always read `notices[]`. A route can come back `200` and still tell you it violated a constraint.

## EV routing

Set `ev[freeFlowSpeedTable]`, `ev[ascent]`, `ev[descent]`, `ev[auxiliaryConsumption]` and the charging
parameters to have HERE insert charging stops into the route. Two parameters shipped 2026-08-11 in 8.161.0:
`ev[maxDrivingTimeBeforeCharge]` and `ev[minTimeAtFirstChargingStation]`. Combining `arrivalTime` with any
`ev` parameter is an error as of 8.158.0 — the API returns 400 rather than silently ignoring one of them.

## Reuse a route instead of recalculating

`GET /routes/{routeHandle}` (`Routing API v8 getRoutesByHandle`) re-reads a previously calculated route by
its handle. Prefer this to recalculating when you only need a different `return` projection of the same route.

## Errors and retries

- No `Idempotency-Key` exists on this API. Routing is a read operation so a retry is safe, but do not assume
  idempotency semantics anywhere on the HERE platform — see `conventions/here-conventions.yml`.
- On `429`, honour `Retry-After` (seconds). Back off exponentially with jitter. HERE may return `429`/`503`
  for up to ~10 minutes during a demand surge even when your quota is not exhausted.
- Send your own `X-Correlation-ID` and keep it. HERE echoes it, and it is what support will ask for.
- `E605502` (400) means a `POST /import` trace was too complex to match within the request time limit — simplify the trace.

## Version drift

The `/v8` path is stable; the service version moves underneath it (8.162.0 as of 2026-09-07). Pin nothing to
the minor version, but read https://docs.here.com/routing/docs/routing-v8-changelog before assuming behaviour.
`mlDuration` was deprecated 2026-07-10: `return=mlDuration` is still accepted and returns nothing.
