---
name: elemental-machines-equipment-utilization-report
description: >-
  Build an equipment utilization report for a lab or facility from the Elemental Machines API —
  aggregated and hourly utilization for a customer group, sliced by location and equipment category,
  against a working-hours calendar. Use when asked how heavily instruments are used, which assets are
  idle, whether to relocate or buy equipment, or to produce a capital-planning readout.
api: Elemental Machines API
base_url: https://api.elementalmachines.io
operations:
  - oauthToken
  - customerGroupsMy
  - customerGroupsIndex
  - machinesIndex
  - machinesUsageAggregated
  - machinesUsageHourly
  - machinesUsageStatus
generated: '2026-08-12'
method: generated
source: openapi/elemental-machines-api-openapi.yml
---

# Equipment utilization report

Grounded in `openapi/elemental-machines-api-openapi.yml`. Every parameter named below is declared in
the provider's own Swagger 1.2 contract.

## 1. Token and tenancy

`oauthToken` — `POST /oauth/token` (form: `username`, `password`, `client_id`, `client_secret`,
`grant_type`).

`customerGroupsMy` — `GET /api/customer_groups/my.json` — gives the caller's group.
`customerGroupsIndex` — `GET /api/customer_groups.json` — lists all groups the caller can reach,
for multi-site callers.

**`customer_group_uuid` is REQUIRED on all three usage operations.** There is no default. Resolve it
first.

## 2. Required pagination

All three usage operations declare `page` and `per_page` as **required**. Omitting either returns
`400 Invalid parameters.` — this is the single most common failure against this API. There is no
implicit first page.

## 3. The three shapes

| Operation | Path | What it answers |
|---|---|---|
| `machinesUsageAggregated` | `/api/machines/usage/aggregated.json` | Utilization totals per machine over a period — the headline report |
| `machinesUsageHourly` | `/api/machines/usage/hourly.json` | Utilization by hour — the shape of the day, peak load, shift patterns |
| `machinesUsageStatus` | `/api/machines/usage/status.json` | Current usage status per machine — a live board |

## 4. Scope the period and the calendar

`machinesUsageAggregated` and `machinesUsageHourly` accept:

- `start_date`, `end_date` — string dates (note: **not** the epoch `from`/`to` integers the
  telemetry endpoints use)
- `time_zone` — set this explicitly; the report is meaningless if the day boundary is wrong
- `start_work_hour`, `end_work_hour` — integers bounding the working day
- `work_days[]` — the working-week calendar

The working-hours parameters are what turn raw runtime into a utilization *rate*. A freezer running
at 3am is expected; a centrifuge idle from 9 to 5 is the finding. Always set the calendar, and state
in the report what calendar was used.

## 5. Slice the fleet

All three usage operations accept:

- `location_tags[]` — per the provider's own note, enter a **single** text value in this form
  (their example: `Lab 24`). The multi-value encoding is not published — the contract says to
  contact Customer Support. If you need several locations, issue one request per tag and merge.
- `equipment_category_tags[]` — same convention.
- `machine_uuids[]` — restrict to named machines. Resolve these from `machinesIndex`
  (`GET /api/machines.json`).
- `sort_by`, `sort_direction` — server-side ordering, so the top-N slice does not require pulling
  every page.

## 6. Assemble

1. `machinesUsageAggregated` with the calendar set → per-machine totals, sorted by utilization
   ascending → the idle-asset list.
2. `machinesUsageHourly` for the same window → the daily profile → whether low utilization is
   genuine idleness or a shift-pattern artifact.
3. `machinesUsageStatus` → a point-in-time board to sanity-check the aggregate against reality.
4. `machinesIndex` → machine names, so the report reads in asset names rather than UUIDs.

## Rules

- Send `customer_group_uuid`, `page` and `per_page` on **every** usage call. Required, all three.
- Page to exhaustion using `page`/`per_page`; there is no cursor and no documented total-count
  field, so keep requesting until a page comes back short.
- Use `sort_by`/`sort_direction` rather than sorting client-side across pages — cross-page
  client-side sorting on an unbounded set will be wrong.
- One `location_tags[]` value per request. Merge in your own code.
- **401 → refresh the token once** and retry. **400 → check the required trio first**, then date
  formats and integer typing on `start_work_hour`/`end_work_hour`.
- No rate limit is published and no `Retry-After` is returned. A fleet-wide report can be hundreds
  of paged calls — pace it deliberately.
- Log `x-request-id` from every response.
- This API is read-only. It supports a relocate/retire/purchase recommendation; it cannot enact one.
