---
name: canopy-monitor-policy-changes
description: >-
  Keep a Canopy Connect insurance connection live - create a Monitoring on an
  existing Pull, refresh it on a schedule or on demand, read what changed
  between refreshes, and reconnect it when the carrier demands re-authentication.
generated: '2026-09-05'
method: generated
source: >-
  openapi/canopy-openapi.json (operationIds verified against the spec) +
  https://docs.usecanopy.com/reference/how-to-use-monitoring-1
api: Canopy Connect API
base_url: https://app.usecanopy.com/api/v1.0.0
operations:
  - post-monitoring
  - get-monitorings
  - get-monitoring
  - patch-monitoring
  - delete-monitoring
  - post-monitoring-refresh
  - get-monitoring-events
  - get-monitoring-reconnect-token
  - post-reconnect-token
  - get-pull-by-id
---

# Monitor an insurance account with Canopy Connect

A Monitoring is an ongoing connection to a carrier account. Each refresh
produces a new Pull; the Monitoring is the schedule that produces them.

## Steps

1. **Create the Monitoring.** Call `post-monitoring` referencing a successful
   Pull. Set `interval` as an ISO-8601 duration. **The interval must be at least
   30 days** - anything shorter returns `400 INTERVAL_TOO_SHORT` /
   `400 INVALID_INTERVAL`.
2. **Adjust it.** `patch-monitoring` changes `interval`, `next_pull_date`,
   `stop_after_date` or `status`. `next_pull_date` must be at least 30 days
   after the latest Pull was created.
3. **Refresh on demand.** `post-monitoring-refresh` runs a refresh immediately.
   A refresh already in flight returns `400 CURRENTLY_REFRESHING`; an inactive
   Monitoring returns `400 MONITORING_INACTIVE`.
4. **Read the diff.** `get-monitoring-events` returns the changes between the
   two most recent Pulls. This requires monitoring events to be enabled on your
   plan - otherwise contact your Canopy Connect representative. Subscribe to the
   `MONITORING_EVENTS` webhook to be told when a refresh produced changes.
5. **Reconnect when authentication lapses.** When you receive
   `MONITORING_RECONNECT`, or the Pull status is `NOT_AUTHENTICATED`, call
   `get-monitoring-reconnect-token` (or `post-reconnect-token`) and send the
   consumer through the reconnect flow. A token already used returns
   `400 ALREADY_RECONNECTED`; an expired or wrong token returns
   `400 INVALID_RECONNECT_TOKEN`.
6. **Stop it.** `delete-monitoring` stops/pauses the Monitoring. Canopy
   describes this as stop/pause rather than destroy, and publishes no statement
   about what happens to the Pulls it already produced.
7. **Inspect at any time.** `get-monitorings` lists Monitorings and can filter
   by Pull ID or status; `get-monitoring` reads one.

## Rules that will bite you

- **30 days is a floor, not a default you can lower.** Both `interval` and
  `next_pull_date` are validated against it.
- **`delete-monitoring` has no published restore.** There is no undelete and no
  window. If you need it back, `patch-monitoring` the status instead of
  deleting, where your plan permits.
- **`PULL_TOO_OLD` and `NEWER_PULL_AVAILABLE`** are real 400s on this surface -
  do not refresh against a stale Pull reference; re-read with `get-pull-by-id`
  first.
- **No idempotency key.** A retried `post-monitoring-refresh` is guarded only by
  `CURRENTLY_REFRESHING`, which is a race guard, not replay protection.
