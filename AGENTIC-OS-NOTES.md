# agentic-os fork — Wanderlog integration notes

This fork of `wanderlog-mcp` is used by Jon's agentic-os fleet to drive a
real Wanderlog trip from Obsidian-sourced research. This file documents
what was reverse-engineered beyond the upstream README, and the roadmap for
this fork. Upstream: `shaikhspeare/wanderlog-mcp`.

## Auth

Wanderlog has no public API. Auth is the browser session cookie
`connect.sid`. For agentic-os it is **stored in 1Password** (vault `claw`,
item `Wanderlog cookie`, field `password`), read at runtime via
`backend/op_client.py` — never written to disk, never echoed. The cookie is
machine-independent (a signed session id, not IP-bound) and valid ~1 year;
renew by updating the 1Password item from anywhere.

## ShareDB / WebSocket protocol

A trip is a ShareDB (JSON0 OT) document. The MCP wraps this; we also drive
it directly from Python (`websockets`) for capabilities the MCP lacks.

- **Connect:** `wss://wanderlog.com/api/tripPlans/wsOverall/{tripKey}?clientSchemaVersion=2`
  with headers `Cookie`, `Origin: https://wanderlog.com`, `User-Agent`.
- **Handshake:** send `{"a":"hs","id":null,"protocol":1,"protocolMinor":2}`;
  server replies `init` then `hs`.
- **Subscribe:** send `{"a":"s","c":"TripPlans","d":tripKey}`; server replies
  `{"a":"s","data":{"v":<version>,"data":<TripPlan>}}`.
- **Submit op:** `{"a":"op","c":"TripPlans","d":tripKey,"v":<version>,"seq":<n>,"x":{},"op":<Json0Op[]>}`.
  Server acks with `{"a":"op",...,"seq":<n>,"v":<applied version>}`; new
  version is `v+1`. Errors arrive as a frame with an `error` field.

### JSON0 op patterns

- Insert a block / section: `{p:[...path...,<index>], li:<value>}`
- Delete a block / section: `{p:[...path...,<index>], ld:<exact current value>}`
- Set / replace a scalar or object: `{p:[...path...], od:<old>, oi:<new>}`
  (omit `od` if the key does not exist yet)
- Edit rich text (note / place description): `{p:[...,"text"], t:"rich-text", o:[<Quill delta ops>]}`

Paths are rooted at `["itinerary","sections",<sectionIdx>,"blocks",<blockIdx>,...]`.

## Document structure

`itinerary.sections[]` — each section: `{id, type, mode, heading, date,
blocks[], placeMarkerColor?, placeMarkerIcon?, text}`.

- Section `type`: `textOnly` (Notes), `flights`, `hotels`, `rentalCars`,
  `normal`. `mode`: `placeList` (un-dated list) or `dayPlan` (a dated day).
- **Per-city to-do sections** = `type:"normal", mode:"placeList",
  heading:"<City> things to do", date:null`. Inserted above the dated day
  sections. Verified: create, rename (heading edit), title the default list.
- Dated days = `type:"normal"`, `date:"YYYY-MM-DD"`.

### Block types

`place`, `note`, `checklist`, `flight`, `train`, **`rentalCar`**. The
upstream `types.ts` maps all but `rentalCar` — see below.

- **place:** `{id, type:"place", place:<PlaceData>, text:<QuillDelta>,
  addedBy:{type:"user",userId}, imageSize:"small", upvotedBy:[],
  travelMode:null, attachments:[], imageKeys?:string[]}`. A **hotel** is a
  place block in a `hotels` section with a `hotel:{checkIn,checkOut,
  travelerNames,confirmationNumber}` sub-object.
- **rentalCar** (not in upstream `types.ts`): `{id, type:"rentalCar",
  addedBy, pickUp:{date,time,place:<PlaceData>}, dropOff:{date,time,place},
  ...}` where each leg's place also carries an `airportOrGeo` object. Lives
  in a `rentalCars` section. Captured from a real Wanderlog email import.

## REST endpoints (cookie-authed, `https://wanderlog.com`)

- `GET /api/placesAPI/autocomplete/v2?request=<urlencoded JSON>` —
  `{input, sessiontoken, location:{latitude,longitude}, radius, language}`
  → `{data:[PlaceSuggestion]}`.
- `GET /api/placesAPI/getPlaceDetails/v2?placeId=<id>&language=en` →
  `{data:<PlaceData>}`.
- Others (see `src/transport/rest.ts`): `/api/user`, `/api/tripPlans/home`,
  `/api/tripPlans/{key}`, `/api/geo/autocomplete/{q}`, create/delete trip.

## Place photo thumbnails (`imageKeys`) — SOLVED

Wanderlog renders the small place thumbnail from a block field
**`imageKeys`** — an array of Wanderlog-hosted image IDs (opaque 32-char
strings). UI-added places have them; the raw Google `place.photo_urls` are
present too but the UI does NOT render from those. Blocks inserted via raw
ShareDB ops (and via the upstream MCP — `buildPlaceBlock` never sets
`imageKeys`) have no `imageKeys`, hence no thumbnail.

**The image-ingestion endpoint** (reverse-engineered from the UI's
add-place network trace, 2026-05-18):

```
POST https://wanderlog.com/api/placePhotos/{placeId}
body: {"place": <PlaceData object>}   # the place from getPlaceDetails
→ {"success": true, "data": ["<imageKey>", ...]}
```

It ingests the Google photos server-side and returns the `imageKeys`.
Cookie-authed; works headlessly (verified — no browser needed).

**Fix for place-add:** after `getPlaceDetails`, `POST /api/placePhotos/
{placeId}` with `{"place": placeData}`, take the returned `data` array, and
set `block["imageKeys"] = data` on the place block before the ShareDB `li`
op. The thumbnail then renders like a native UI add.

## Verified working (driven directly via ShareDB ops, May 2026)

create trip-typed `rentalCar` entry · delete a block (`ld`) · delete a note
· add/edit a note (rich-text op) · add a points note to a hotel · place
search (REST autocomplete + details) · add a place to a dated day · add a
place to an un-dated list · create a section · rename / title a section.

## Fork roadmap

1. Map the `rentalCar` block in `types.ts`; add `add-car` tool.
2. Add `add-flight` tool (`FlightBlock` is already typed; no creator tool).
3. Surface `edit-note` / `remove-note` in the published build.
4. Wire `POST /api/placePhotos/{placeId}` into place-add so blocks get
   `imageKeys` and render thumbnails (endpoint solved — see above).
5. Per-city section helpers (`add-activity` with target section/day).
