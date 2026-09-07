---
name: cvent-social-tables-build-event-diagram
description: >-
  Lay out a room automatically with the Social Tables Layout Automation API — pick the property
  and bookable room, post a wizard specification, and clear the floor again if it is wrong.
api: cvent-social-tables:cvent-social-tables-layout-automation-api
generated: '2026-09-07'
method: generated
source: >-
  https://developer.socialtables.com/docs/api-usage/layout-automation,
  openapi/_original/cvent-social-tables-openapi.json
operations:
  - 'GET /4.0/properties'
  - 'GET /4.0/properties/{property_id}/rooms'
  - 'GET /4.0/rooms'
  - 'GET /4.0/rooms/{room_id}'
  - 'POST /4.0/layout-automation'
  - 'POST /4.0/layout-automation/custom-setup'
  - 'DELETE /4.0/layout-automation'
  - 'GET /4.0/diagrams'
  - 'GET /4.0/diagrams/{id}'
note: >-
  Operations are named by method and path — the contract declares no operationId. All values
  below come from the provider's own Layout Automation reference; none is invented.
---

# Lay out an event space automatically

Authenticate first — see `cvent-social-tables-authenticate-and-identify`.

## 1. Find the room

* `GET /4.0/properties` → `GET /4.0/properties/{property_id}/rooms`, or
* `GET /4.0/rooms` (cursor paging: `page_size` default 50, `after`, `before`) and
  `GET /4.0/rooms/{room_id}` for one room.

The Layout Automation reference states the bookable room id **must be prepended with an `S`**
when supplied as `venue_id`. This is a prose-only rule; the contract does not encode it.

## 2. Post the layout specification

`POST /4.0/layout-automation` — "actively calculates layout and creates an event/space/diagram".

Top-level event fields: `name` and `category` (gala, reception, conference, wedding, party,
other) for a new event, **or** `event_id` to update an existing one — one or the other, not
both. Optional `start_time` / `end_time` (ISO-8601), `user_email` (must be on the requestor's
team), `uses_metric`.

`spaces[]` — for a new space: `name`, `venue_id`, `wizard`. For an existing space: `space_id`
and `wizard`.

The `wizard` object: `attendees` (integer, required), `setup` (required — aligned, staggered,
classroom, conference, keynote, u-shape, reception, theatre, registration; the notes section
also lists conference-table and hollow-square), `table` (required), plus optional
`chevronAngle` and `rotation` (degrees).

`table`: `chairs`, `size` (`width` + `length`, **or** `radius` — never both), `spacing`
(`x`, `y`), `type` (circle, square, rectangle, theatre, crescent, oval, serpentine, high-boy,
chair), `position` (n, s, e, w, ne, nw, se, sw), `cullAdditionalTables`, `removedChairs`,
`rotateCrescent`. Optional `aisles.horizontal` / `aisles.vertical` with `floorElementsBetween`
and `width`.

**Units are inches** unless the event sets `uses_metric`, in which case lengths are
centimetres. Getting this wrong scales a whole ballroom by 2.54.

`POST /4.0/layout-automation/custom-setup` is the sibling for a caller-supplied setup rather
than a wizard preset.

## 3. Read the result

The 200 body is the full event created by the layout-automation service. Inspect the produced
diagrams with `GET /4.0/diagrams` and `GET /4.0/diagrams/{id}`.

## 4. Undo — read this before you write

`DELETE /4.0/layout-automation` takes a required `spaces` query parameter (a comma-separated
list of space IDs) and "removes floor elements from previous automated **and manual** layouts".
It does not distinguish work your integration created from work a human did in the UI, and
there is no restore path for a cleared floor. Treat it as destructive and confirm the space
list before calling it.

There is no dry-run or preview mode, and no `Idempotency-Key` header on any operation: a
retried `POST /4.0/layout-automation` creates a second event unless you passed `event_id`.
