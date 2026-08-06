---
name: Pull BlockID reports, metrics and audit logs
description: Query the 1Kosmos reporting service for events, metrics, login activity, application usage, last-seen and audit-log data within the 90-day window.
api: openapi/1kosmos-blockid-openapi.yml
operations: [reportsEvents, reportsMetrics, reportsAuditLogReport, reportsLoginActivityReport, reportsPasswordlessLoginActivityReport, reportsApplicationUsageReport, reportsLastSeenReport]
generated: '2026-08-05'
method: generated
source: openapi/1kosmos-blockid-openapi.yml + conventions/1kosmos-conventions.yml
---

# Pull BlockID reports, metrics and audit logs

Seven POST endpoints on the `reports` service, all scoped
`/reports/tenant/{tenantID}/community/{communityID}/…`, all paged, all bounded to a rolling 90 days.

## Prerequisite

Complete `1kosmos-bootstrap-tenant.md`. These endpoints additionally require the `X-TenantTag`
header.

## The endpoints

| Operation | Path suffix | What it returns |
|---|---|---|
| `reportsEvents` | `/events` | Raw event stream (`E_IDV_*`, `E_LOGIN_SUCCEEDED`, …) |
| `reportsMetrics` | `/metrics` | Aggregate counters by `metricsName` |
| `reportsAuditLogReport` | `/audit_log` | Administrative audit trail |
| `reportsLoginActivityReport` | `/login_activity_report` | All login activity |
| `reportsPasswordlessLoginActivityReport` | `/passwordless_login_activity_report` | Passwordless logins only |
| `reportsApplicationUsageReport` | `/application_usage_report` | Usage per integrated application |
| `reportsLastSeenReport` | `/last_seen_report` | Last-seen per `user_id` / `device_id` / `eventName` |

## Request shape

```json
{ "pIndex": 0, "pSize": 10,
  "from": "2026-05-01 00:00:00.000",
  "to":   "2026-06-01 00:00:00.000" }
```

`reportsMetrics` replaces the paging fields with `metricsName` — published example values include
`M_C_LOGINS` and `M_GT_SUCCESSFUL_AUTHENTICATIONS`.

## Rules

- **90 days, hard.** `from` older than now-90d fails with
  `400 {"errors":[{"param":"from","message":"Field should be from current time to minus 90 days"}]}`.
  Archive anything you need beyond that yourself.
- **`to` must be after `from`.** Some endpoints allow equality, some do not — the error message tells
  you which.
- **Timestamps are `YYYY-MM-DD HH:mm:ss.SSS` strings**, not ISO 8601 with a zone. Format them exactly.
- **Paging is `pIndex` (zero-based) + `pSize` in the body**, not query parameters — these are POST
  endpoints.
- This service uses the third error envelope: `{"errors": [{"message": …, "param": …}]}`. It is the
  only one that names the offending field, so read `param` before retrying.
- `400 "This header can't be decrypted"` on `param: licensekey` means a stale shared secret, not a
  bad report request.
- Audit-log and login-activity output is PII. Handle under the tenant's retention policy.
