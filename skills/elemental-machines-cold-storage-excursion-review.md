---
name: elemental-machines-cold-storage-excursion-review
description: >-
  Investigate a temperature excursion on an Elemental Machines monitored asset — pull the alert
  history for the machine, window the raw sensor samples around each alert, and summarize against
  the server-computed statistics for the same period. Use when asked to review a freezer, incubator
  or cold-storage alarm, build an excursion report, or explain what happened to a specific asset.
api: Elemental Machines API
base_url: https://api.elementalmachines.io
operations:
  - oauthToken
  - machinesIndex
  - machinesShow
  - alertLogsIndex
  - machineSamplesIndex
  - machineSampleStatsIndex
generated: '2026-08-12'
method: generated
source: openapi/elemental-machines-api-openapi.yml
---

# Cold storage excursion review

Every operationId below exists in `openapi/elemental-machines-api-openapi.yml`, converted from the
provider's own Swagger 1.2 contract. No response field names are given because the provider
publishes no response schemas — read the shapes at runtime.

## 1. Get a token

`oauthToken` — `POST /oauth/token`, form-encoded, with `username`, `password`, `client_id`,
`client_secret` and `grant_type`. Hold the returned token in memory only. Every subsequent call
carries it as an `access_token` **query parameter**; there is no Authorization-header option, so
never write a request URL to a log or a file.

## 2. Resolve the machine

`machinesIndex` — `GET /api/machines.json?access_token=…` — returns the machines the caller can
see. Match the asset by name from this list. Do not guess a UUID: `machinesShow`
(`GET /api/machines/{uuid}.json`) returns **403 Forbidden** when the UUID belongs to another
customer group and **401** when the token is stale. 403 means "not yours", not "not found".

Confirm tenancy with `customerGroupsMy` (`GET /api/customer_groups/my.json`) if the 403 is
unexpected.

## 3. Pull the alert history

`alertLogsIndex` — `GET /api/machines/{machine_uuid}/alert_logs.json`

Parameters: `access_token` (required), `from`, `to` (epoch integers), `order`, `limit`.

Window this to the period under review. Start wide, then narrow. Note that `from`/`to` are integers
— epoch seconds — not ISO dates; the date-string parameters (`start_date`, `end_date`) belong to
the usage operations, not to this one.

## 4. Window the raw samples around each alert

`machineSamplesIndex` — `GET /api/machines/{machine_uuid}/samples.json`

Parameters: `access_token` (required), `from`, `to`, `order`, `limit`.

For each alert timestamp `t`, request a window that brackets it — for example `t - 3600` to
`t + 3600` — so the report shows the approach to the threshold, the breach, and the recovery. The
platform samples at 15-second resolution, so a one-hour bracket is roughly 240 points per sensor.
**Always set `limit`.** There is no next-page token on this endpoint; you advance the window
yourself.

## 5. Establish the baseline

`machineSampleStatsIndex` — `GET /api/machines/{machine_uuid}/sample_stats.json?from=…&to=…`

Returns server-computed minimum, maximum, mean and median. Call this twice: once over the excursion
window, once over a comparable clean period. Do **not** compute these yourself from
`machineSamplesIndex` — the server already does it, and pulling raw series to average them is the
single most wasteful pattern against this API.

## 6. Check the rule that fired

`alertRulesIndex` — `GET /api/alert_rules.json?managed_machine_uuid={machine_uuid}` — gives the
configured condition, so the report can state the threshold the excursion crossed rather than
inferring it.

## Rules

- **401 → refresh once.** Token lifetime is not published. Re-run `oauthToken` and retry the call
  a single time; do not loop.
- **403 → stop.** It is a tenancy boundary, not a transient error. Retrying will not help.
- **404 on a nested read** means the `machine_uuid` is unknown. Go back to `machinesIndex`.
- **No rate limit is published and no `Retry-After` is returned.** Pace yourself; the server will
  not warn you.
- **Use `If-None-Match`.** Responses carry a weak ETag and `must-revalidate`, so repeat polls of an
  unchanged window are cheap.
- **Log `x-request-id`** from every response. It is the only correlation handle support can act on.
- This API is read-only. You cannot acknowledge, silence or resolve an alert through it — say so
  rather than implying the excursion has been actioned.
- If the platform itself looks wrong, `statusCheck` (`GET /api/status/check.json`) answers without
  a token and reports subsystem health; https://status.elementalmachines.io is the human view.

## Compliance note

`userActivitiesIndex` (`GET /api/user_activities.json`, filterable by `customer_group_uuid`,
`usage_type`, `action_type`, `from`, `to`) is the audit trail behind the company's 21 CFR Part 11 /
ALCOA+ claim. Include it when the excursion review has to be audit-ready — it shows who looked at
what and when, alongside the sensor record.
