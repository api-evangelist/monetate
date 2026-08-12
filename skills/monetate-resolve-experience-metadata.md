---
name: monetate-resolve-experience-metadata
description: Turn the opaque integer ids that come back in Monetate engine responses and session-stream exports — experience/variant ids, page event ids, custom target ids, account ids — into human-readable names using the Metadata API.
api: Monetate Metadata API
generated: '2026-08-12'
method: generated
source: openapi/monetate-metadata-api-openapi.yml, https://developer.monetate.com/metadata-api/get-interpretable-values-with-the-metadata-api
operations:
  - GET /metadata/account                                  # overlay operationId: listAccounts
  - GET /metadata/account/{account-id}                     # overlay operationId: getAccount
  - GET /metadata/variant                                  # overlay operationId: listExperienceDetails
  - GET /metadata/variant/{variant-id}                     # overlay operationId: getExperienceDetails
  - GET /metadata/experience-summary                       # overlay operationId: listExperienceSummaries
  - GET /metadata/experience-summary/{experience-summary-id} # overlay operationId: getExperienceSummaryDetails
  - GET /metadata/pageevent                                # overlay operationId: listPageEvents
  - GET /metadata/pageevent/{pageevent-id}                 # overlay operationId: getPageEventDetails
  - GET /metadata/customtarget                             # overlay operationId: listCustomTargets
  - GET /metadata/customtarget/{customtarget-id}           # overlay operationId: getCustomTargetDetails
base_url: https://api.monetate.net/api/metadata/v1/{retailerShortname}/production
---

# Resolve Monetate ids to names

Session-stream exports and Engine API responses are full of bare integers — `page_event_ids`, variant
ids, custom target ids. The Metadata API exists for exactly one job: turning those into something a
human or a report can read. It is entirely read-only; all ten operations are `GET`.

Authenticate first — see `monetate-mint-api-token`. Every call needs
`Authorization: Token <token_string>`.

## The pattern

Do **not** resolve ids one at a time. Every resource has a list endpoint and a detail endpoint; pull the
full list once with `page_size=1000`, build a dict keyed by `id`, and join locally. Per-id lookups will
burn requests against an API whose rate limits are not published.

```python
resp = requests.get(BASE + 'metadata/pageevent/',
                    headers={'Authorization': f'Token {auth_token}'},
                    params={'page_size': 1000})
page_events = {d['id']: d for d in resp.json()['data']}
```

`1000` is the documented maximum for a single request.

## What each resource gives you

| Endpoint | Entity | Useful fields |
|---|---|---|
| `/metadata/account` | Account | `id`, `type` (production/development), `kind` (retail/travel/…), `account_domain`, `archived` |
| `/metadata/variant` | Variant (an experience variant) | `id`, `experience_id`, `experience_name`, `account_domain` |
| `/metadata/experience-summary` | Experience-Summary | `id`, `experience_name`, `experience_type`, `account_domain` |
| `/metadata/pageevent` | PageEvent | `id`, `title` |
| `/metadata/customtarget` | CustomTarget | `id`, `title`, `description`, `is_identifier` |

Use `/metadata/variant` when you have a variant-level id from a decision response, and
`/metadata/experience-summary` when you want one row per experience rather than per variant. `Variant`
carries `experience_id`, which is the join back to the summary.

## Pagination

Responses use `meta` for pagination as well as status:

```json
{"meta": {"code": 200, "count": 1000, "next": "...", "previous": "...", "errors": [], "warnings": []},
 "data": [ ... ]}
```

Follow `meta.next` until it is absent. The `page` query parameter appears in Monetate's own Python
walkthrough but is **not declared** in the published specification — prefer `meta.next`, which is.

Ids from JSON object keys come back as strings; the API returns them as integers. Coerce before joining
or the dict lookup will silently miss.

## Error handling

- `401` — token missing/invalid. Re-mint and retry once.
- `403` — the API key was revoked. Stop; a human must re-issue it.
- `404` — the resource does not exist or was deleted. Expect this on ids from old session data:
  experiences and page events get deleted while historical exports still reference them. Collect the
  misses into a `missing_page_events` list and report them rather than failing the whole join.
- `500` — retry with backoff.

No `429` is declared on any Metadata API operation, and no rate-limit headers are documented — which
means there is no published signal to back off against. Batch with `page_size=1000` and cache.
