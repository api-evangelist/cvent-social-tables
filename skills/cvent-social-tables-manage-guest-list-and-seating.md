---
name: cvent-social-tables-manage-guest-list-and-seating
description: >-
  Build and run a Social Tables guest list end to end — create it, load guests in bulk, tag and
  assign meals, seat people on the diagram, check them in on the day, and restore anything
  deleted by mistake.
api: cvent-social-tables:cvent-social-tables-guest-lists-api
generated: '2026-09-07'
method: generated
source: openapi/_original/cvent-social-tables-openapi.json
operations:
  - 'POST /4.0/guestlists'
  - 'GET /4.0/guestlists/{guestlist_id}'
  - 'POST /4.0/guestlists/{guestlist_id}/guests/_bulk'
  - 'POST /4.0/guestlists/{guestlist_id}/guests/_dedupe'
  - 'POST /4.0/guestlists/{guestlist_id}/tags'
  - 'POST /4.0/guestlists/{guestlist_id}/meals'
  - 'POST /4.0/guestlists/{guestlist_id}/groups'
  - 'POST /4.0/guestlists/{guestlist_id}/guests/{guest_id}/seating'
  - 'PATCH /4.0/diagrams/{id}/auto-seat'
  - 'POST /4.0/guestlists/{guestlist_id}/guests/checkin'
  - 'POST /4.0/guestlists/{guestlist_id}/restore'
  - 'POST /4.0/guestlists/{guestlist_id}/guests/{guest_id}/restore'
note: Operations are named by method and path — the contract declares no operationId.
---

# Run a guest list from import to check-in

Authenticate first — see `cvent-social-tables-authenticate-and-identify`.

## 1. Create the list

`POST /4.0/guestlists`. The request body takes `diagram_id`; the sibling `event_id` property is
marked **"deprecated, use diagram_id"** in the contract — the only deprecation marker anywhere
in this API. Read it back with `GET /4.0/guestlists/{guestlist_id}`.

`POST /4.0/guestlists/{guestlist_id}/_clone` copies an existing list, which is usually faster
than re-importing for a repeating event.

## 2. Load guests

* `POST /4.0/guestlists/{guestlist_id}/guests` — one guest.
* `POST /4.0/guestlists/{guestlist_id}/guests/_bulk` — an array.
* `POST /4.0/guestlists/{guestlist_id}/guests/_replace` — replaces the whole list. Destructive.
* `POST /4.0/guestlists/{guestlist_id}/guests/_dedupe` — de-duplicates after a messy import.
* `GET /4.0/guestlists/{guestlist_id}/column_mapping` — how imported columns map to fields.

None of the bulk operations documents partial-failure semantics and none accepts an idempotency
key, so a retry after a timeout can duplicate rows. Run `_dedupe` after any retried bulk write.

## 3. Tag, group and feed

* `GET|POST /4.0/guestlists/{guestlist_id}/tags`, then
  `GET|PUT|DELETE /4.0/guestlists/{guestlist_id}/tags/{tag_title_or_id}` — the path parameter
  accepts a title *or* an id.
* `POST /4.0/guestlists/{guestlist_id}/meals` and
  `PUT|DELETE /4.0/guestlists/{guestlist_id}/meals/{meal_title}` — meals are keyed by title.
* `POST /4.0/guestlists/{guestlist_id}/groups`, then
  `POST /4.0/guestlists/{guestlist_id}/groups/{group_id}/guests/_bulk` (and `/create`,
  `/delete`) to move people in and out of a group;
  `PUT /4.0/guestlists/{guestlist_id}/groups/{group_id}/name/{name}` renames it.

## 4. Seat them

* `POST /4.0/guestlists/{guestlist_id}/guests/{guest_id}/seating` — seat one guest;
  `GET` the same path reads the assignment.
* `GET /4.0/guestlists/{guestlist_id}/guests/seating` — the whole seating map.
* `PATCH /4.0/diagrams/{id}/seat` and `PATCH /4.0/diagrams/{id}/auto-seat` — seat against the
  diagram. Both carry an `update_id`; treat it as the concurrency token and send back the one
  you last read.
* `GET /4.0/diagrams/guestlist/{guestlistId}` — the diagram behind a list.

## 5. Check in on the day

`GET /4.0/guestlists/{guestlist_id}/checkin` (status),
`POST /4.0/guestlists/{guestlist_id}/guests/checkin`,
`POST /4.0/guestlists/{guestlist_id}/guests/{guest_id}/checkin`,
`POST /4.0/guestlists/{guestlist_id}/guests/update-checkin-status`.

## 6. Undo

Deletes on this surface are soft and reversible:

* `POST /4.0/guestlists/{guestlist_id}/restore` → 204, "successfully restored a deleted guestlist"
* `POST /4.0/guestlists/{guestlist_id}/guests/{guest_id}/restore` → 200, restores the guest and
  their group where applicable

**No restore window is published.** Do not promise a caller that a deletion is recoverable after
some period — the provider has never stated one.

## Reading the status codes

`410 Gone` on any guest-list-scoped path means "guestlist is deleted" — 38 operations declare
it. That is resource state, not deprecation, and the restore path above is the fix. `404` means
wrong or missing id (check the 4.0 vs legacy generation). `409` on favorites and layouts means
a duplicate name.
