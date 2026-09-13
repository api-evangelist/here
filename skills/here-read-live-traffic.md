---
name: here-read-live-traffic
description: Read real-time traffic flow and incidents for an area or corridor with the HERE Traffic API v7, and choose between the JSON data API and the raster/vector tile APIs.
api: HERE Traffic API v7
spec: openapi/here-traffic-v7-openapi.yml
base_url: https://data.traffic.hereapi.com/v7
operations:
  - Traffic API v7 getFlow
  - Traffic API v7 getFlowWithBodyParams
  - Traffic API v7 getIncidents
  - Traffic API v7 getIncidentsById
  - Traffic API v7 getIncidentsWithBodyParams
  - Traffic API v7 getVersion
---

# Read live traffic with HERE v7

## Pick the right product first

| You want | Use |
|---|---|
| Traffic values you will compute with | **Traffic API v7** (`data.traffic.hereapi.com/v7`) — JSON flow and incidents |
| Traffic drawn on a map for a human | **Traffic Raster Tile API v3** (`traffic.maps.hereapi.com/v3`) — PNG tiles |
| Traffic styled by your own renderer | **Traffic Vector Tile API v2** (`traffic.vector.hereapi.com/v2`) — Mapbox Vector Tile |

Fetching raster tiles to extract numbers from pixels is the wrong shape and will cost you far more transactions.

## Flow

`GET /flow` (`Traffic API v7 getFlow`), constrained by exactly one spatial filter:

- `in=circle:lat,lng;r=meters`
- `in=bbox:west,south,east,north`
- `in=corridor:lat,lng;lat,lng;...;r=meters` — the one to use along a route

`locationReferencing=shape` returns geometry with each flow record; omit it if you already hold the geometry
and only need the values. Flow records carry `jamFactor` (0-10), `speed`, `freeFlow` and `confidence`.

Read `confidence` before you act on `speed`. Low-confidence flow is modelled, not observed.

## Incidents

`GET /incidents` (`Traffic API v7 getIncidents`) with the same `in=` filters. `GET /incidents/{originalId}`
(`getIncidentsById`) re-reads one incident by the id HERE issued.

Use the `POST` variants (`getFlowWithBodyParams`, `getIncidentsWithBodyParams`) when a corridor definition
is too long for a URL — a corridor along a full route usually is.

## Joining to the map

Both flow and incidents reference HERE topology segments (`segmentId` / `extSegmentId`) and, for TMC-based
references, location codes. If you are matching traffic onto your own geometry, match against the HERE
segment references rather than by coordinate proximity — see also the Route Matching v8 API
(`openapi/here-route-matching-v8-openapi.json`).

## Operating notes

- `GET /version` returns the running service version; useful in a support ticket.
- Traffic is high-cardinality and high-volume. Poll on a cadence your quota can sustain and honour
  `Retry-After` on `429`.
- Send `X-Correlation-ID`; HERE echoes it on the response.
- Changelog: https://docs.here.com/traffic-api/docs/traffic-api-changelog
