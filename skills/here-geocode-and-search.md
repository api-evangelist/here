---
name: here-geocode-and-search
description: Turn text into coordinates and coordinates into addresses with the HERE Geocoding and Search API v7 — geocode, reverse geocode, autosuggest, autocomplete, discover, browse and lookup.
api: HERE Geocoding and Search API v7
spec: openapi/here-geocoding-and-search-v7-openapi.json
base_urls:
  - https://geocode.search.hereapi.com/v1
  - https://revgeocode.search.hereapi.com/v1
  - https://discover.search.hereapi.com/v1
  - https://autosuggest.search.hereapi.com/v1
  - https://autocomplete.search.hereapi.com/v1
  - https://browse.search.hereapi.com/v1
  - https://lookup.search.hereapi.com/v1
operations:
  - GET /geocode
  - GET /revgeocode
  - POST /multi-revgeocode
  - GET /discover
  - GET /autosuggest
  - GET /autocomplete
  - GET /browse
  - GET /lookup
  - POST /signals
---

# Geocode and search with HERE v7

> **Note on operationIds.** HERE's published Geocoding and Search v7 document declares no `operationId` on
> any of its 11 operations. Address them by method + path, as above. Do not invent operationIds; a generated
> client will name them from the path.

## One API, seven hosts

Each endpoint lives on its own subdomain. The path is the endpoint name and the host must match it —
`/geocode` on `geocode.search.hereapi.com`, `/revgeocode` on `revgeocode.search.hereapi.com`, and so on.
Calling `/discover` on the geocode host is the most common first mistake.

## Pick the right endpoint

| Intent | Endpoint |
|---|---|
| A complete address string to coordinates | `GET /geocode?q=` |
| Structured address fields to coordinates | `GET /geocode?qq=city=Berlin;country=Germany` |
| Coordinates to an address | `GET /revgeocode?at=lat,lng` |
| Many coordinates to addresses in one call | `POST /multi-revgeocode` |
| Keystroke-by-keystroke as a user types | `GET /autosuggest?q=&at=` |
| Address completion only, no places | `GET /autocomplete?q=` |
| Free-text place search near a point | `GET /discover?q=&at=` |
| Category browse near a point (e.g. all EV chargers) | `GET /browse?at=&categories=` |
| Re-fetch a known result by its HERE id | `GET /lookup?id=` |

`/autosuggest` is for interactive typing; it is tuned for partial input and returns query-refinement
suggestions as well as results. `/discover` is for a completed search intent. Using `/discover` per keystroke
wastes transactions and gives worse results.

## Shape the request

- `at=lat,lng` sets the search centre and is what makes results locally relevant.
- `in=countryCode:DEU,FRA` or `in=circle:lat,lng;r=5000` or `in=bbox:w,s,e,n` constrains the search.
- `limit` defaults to 20 and caps at 100.
- `lang` takes BCP 47 tags.
- `show=` opts into extra result sections (contacts, opening hours, time zone, street info).

## Read the response

`items[]` of `Item`. Each `Item` carries `title`, `id`, `resultType`, `address` (with `label` as the
formatted one-line form), `position` (`lat`/`lng`), and `scoring.queryScore`.

Never treat `queryScore` as a probability. Use `resultType` (`houseNumber`, `street`, `locality`,
`administrativeArea`, `place`) to decide whether the match is precise enough for what you are about to do
with it. A `locality` result for a request that needed a rooftop is a wrong answer that looks like a right one.

Keep `id` if you will need the same place again — `GET /lookup?id=` is cheaper and stable.

## Test without polluting the relevance model

Add the `X-OLP-Testing` header to any request not driven by a real user. HERE uses it to exclude synthetic
traffic from the search relevance model. There is no sandbox host; testing runs against production with a
free-tier key.

## Large volumes

Above a few thousand records, stop looping `/geocode` and use the HERE Batch API v7 instead —
see `skills/here-run-a-batch-geocoding-job.md`. It accepts up to 1,000,000 records or 500 MB per job.

## Deprecation in flight

Alternative property keys are being removed in two stages — stage 1 landed January 2026 (release 7.34.0),
stage 2 lands October 2026. Select only the leaf property keys the API actually returns.
https://docs.here.com/geocoding-and-search/docs/deprecation-and-change-notice
