---
name: jit-parked-access-requests
description: >-
  Build a 30-day milestone table of parked JIT access requests
  (CLASSIFY_JIT_PARKED) for an IdentityNow org. Use when the user gives an
  org name and asks for this JIT, parked-connection, or access-request
  milestone table, durations, or last-30-days timing for grants and revokes.
---

# Parked JIT access-request table

Given an org, produce the milestone table for the last 30 days. Read the canvas skill and render the table as a canvas. Do not dump it as a markdown table.

This flow does not provision the source. The JIT entitlement is filtered out of the plan and parked with `intent=CLASSIFY_JIT_PARKED`. Do not label any column provisioning.

## Query

Grafana MCP `query_elasticsearch`.

- Datasource: `aea6zw4ypcwe8e` (prod us-east-1 OpenSearch). If the org has no hits, try the other `grafana-opensearch-datasource-prod-*` datasources.
- Index: `logs-all-*`
- Window: last 30 days through now. Exclude `stack:oathkeeper`.
- Lucene only. Limit is 100 and results are newest first. If a query returns 100, page with `endTime` just before the oldest hit.
- On a TLS certificate error, retry the same query once.

Org filter: `org:<org>`.

## Find the requests

```
"CLASSIFY_JIT_PARKED" AND org:<org> AND NOT stack:oathkeeper
```

Each request id appears on `wps` and `gov-ent-connections`. Keep one row per `request_id`. Action is Grant when the access request is `GRANT_ACCESS`, Revoke when it is `REVOKE_ACCESS`. Entitlement name comes from the `ACCESS_REQUEST_COMPLETED` item, or from the connection `attributeValue` CN.

## Milestones

Times are the log `@timestamp`. A missing line stays blank. Do not invent a time.

| Column | System | Log |
|---|---|---|
| Submitted | rats | `AccessRequest of type GRANT_ACCESS requested` or `REVOKE_ACCESS requested` (`AccessRequestV3Resource.logAccessRequestRequested`). This is the start. |
| JIT policy | gov-jit-access | `evaluatePolicy completed`. Grants only. |
| SOD passed | wps | `Preventative SOD check completed`. Grants only. |
| Approved | wps | `Successfully submitted the approval decisions` if present, otherwise `Approval complete for identityRequestId`. |
| Connection published | gov-ent-connections | Earliest `connectionId=` on `BaseConnectionEventPublisher` or `ConnectionEventProcessor`. If that line is absent, use aer `AuditEventService` `connectionId=`. Say so on that row. |
| Request completed | wps | `Publishing ACCESS_REQUEST_COMPLETED`. If that line is absent, use the wps `AccessRequestProcessed` audit and say so. |
| Flink emitted | flink | Search ingested minus that same line's `latency`. Blank when Search ingested is blank. |
| Search ingested | sp-search-ciem-ingest | Defined below. |

Normalize connection ids by removing dashes. Search logs use the undashed id.

## Search end time

Search `stack:sp-search-ciem-ingest` for the undashed connection id. Grant matches `Submit index`. Revoke matches `Submit delete`.

The same connection id is reused by later grants and revokes. Do not take the first search log after the publish.

Record time = search `@timestamp` minus that line's `latency`. That record time is the Flink emitted column. Keep candidates whose record time is at or after the connection publish (allow 2 seconds of clock skew). Search ingested is the `@timestamp` of the candidate with the earliest record time. A record created before the publish must not be used. That is what makes ingest lag longer than the request.

If no candidate exists, leave Flink emitted and Search ingested blank.

## Durations

Under every milestone except Submitted, show elapsed time since Submitted in parentheses. Under Search ingested, show elapsed time since Flink emitted instead. That duration is the search log's latency. Keep the seconds. Do not drop them once the elapsed time reaches an hour or a day: 24 hours and 33 seconds is `1d 33s`, not `1d 0h 0m`. Under one second, show milliseconds. Color it red when elapsed time is over 5 minutes. Use `useHostTheme().category.red` for the red. Do not hardcode a hex.

## Canvas

One row per request, oldest submit first. Headers are two lines: milestone name, then the system name from the table above.

Columns: Request, Action, Entitlement, Connection id, Submitted, JIT policy, SOD passed, Approved, Connection published, Request completed, Flink emitted, Search ingested.

Connection id shows the dashed form and, under it, the plain form used in search logs.

Keep the table in its normal frame. Do not widen the canvas to fit the columns.

Blank milestone cells are an em dash. Row tone warning when Search ingested is blank.

Caption: org, pod if one pod dominates, UTC, last 30 days, and that the parenthetical is time since submit except under Search ingested, where it is time since Flink emitted. Call out rows with no search ingest and any connection time that came from aer.

In chat, give the canvas link, the request count, and the rows still missing a search time. Do not repeat the full table.
