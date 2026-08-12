---
name: monetate-load-customer-data
description: Define a Monetate dataset with the Data API, push rows into it, watch the upload land, and designate a product schema as an account's default catalog — including the identifier-field rule that decides whether your data can be targeted at all.
api: Monetate Data API
generated: '2026-08-12'
method: generated
source: openapi/monetate-data-api-openapi.yml, https://developer.monetate.com/data-api
operations:
  - GET /schematype/                  # overlay operationId: listSchemaTypes
  - GET /schematype/{schema-type}/    # overlay operationId: getSchemaTypeDetails
  - GET /schema/                      # overlay operationId: getActiveSchema
  - POST /schema/                     # overlay operationId: createaNewSchema
  - GET /schema/{schema-name}/        # overlay operationId: getSchemaDetails
  - PATCH /schema/{schema-name}/      # overlay operationId: updateanExistingSchema
  - POST /data/{schema-name}/         # overlay operationId: postDataForexampleschema
  - GET /data/{schema-name}/          # overlay operationId: getDataForexampleschema
  - GET /upload/{schema-name}/        # overlay operationId: getUploadHistory
  - GET /defaultcatalog/              # overlay operationId: listAccountDefaults
  - POST /defaultcatalog/             # overlay operationId: setAccountDefault
  - POST /bulk-defaultcatalog/        # overlay operationId: bulkSetAccountDefaults
base_url: https://api.monetate.net/api/data/v1/{retailerShortname}/production
---

# Load customer and product data into Monetate

Monetate's Data API is meta: you first declare a **Schema** (your own dataset shape), and only then can
you send rows against it. The published specification uses `example_schema` in its paths as a
placeholder — your real paths are `/data/{your_schema_name}/`, which is why no generated client will
have the right operations until you substitute your own schema name.

Authenticate first — see `monetate-mint-api-token`. Every call needs
`Authorization: Token <token_string>`.

## Step 1 — Find the schema type you need

```
GET /schematype/
GET /schematype/{schema-type}/
```

A SchemaType is Monetate's template. The published enum is: `agil_one`, `attribute`,
`behavioral_trigger`, `custom_list`, `customer_data_privacy`, `email_metadata`, `event`, `inventory`,
`product`, `product_recommendation`, `purchase`.

`GET /schematype/{schema-type}/` returns the fields that type expects — start from that rather than
inventing a shape.

## Step 2 — Create the schema

```
POST /schema/
```

Body is a `Schema`: `name` plus `fields`.

- `name` must match `^[A-Za-z][A-Za-z0-9_]*$` and be **at most 64 characters**.
- `fields` is a map of field name → `Field`, and must have **at least one** entry.

Each `Field` requires a `data_type` from `STRING`, `MULTI_STRING`, `BOOLEAN`, `DATETIME`, `NUMBER`,
`GOOGLE_PRODUCT_CATEGORY`. (The spec's own prose says "Monetate supports five data types" while the enum
lists six — trust the enum.)

**The rule that catches everyone:** exactly one field must be `identifier: true`, and its `data_type`
must be `STRING`. That is the field Monetate links to a Person ID; without it your rows load but cannot
be targeted on. Other flags are `unique_key`, `event_time` and `required`.

`400 Validation error` means a value was the wrong format or a required value was missing. The error
body will not name the field — diff your payload against the `Schema`/`Field` schemas.

## Step 3 — Send rows

```
POST /data/{schema-name}/
```

Body is `{"schema_rows": [ ... ]}`, where each row's properties are exactly the fields you declared.
`DATETIME` values are sent as strings in the form `2016-09-30 12:10:45.000145+00:00`; `MULTI_STRING`
values are comma-separated in a single string.

**There is no idempotency key.** A retried POST is a second load. Before retrying a request whose
outcome you are unsure of, check the upload history rather than resending.

## Step 4 — Confirm the upload landed

```
GET /upload/{schema-name}/
```

Returns `FileUpload` records with `status`, `upload_time`, `import_start_time`, `import_end_time`.
Status is one of `PENDING`, `PROCESSING`, `COMPLETE`, `VALIDATION_ERROR`, `SYSTEM_ERROR`, `SKIPPED`,
`TIMEOUT_ERROR`, `MAX_LOAD_ATTEMPTS_REACHED`.

A `200` on the POST means accepted, not imported. Poll here until `COMPLETE`. `SKIPPED` and
`MAX_LOAD_ATTEMPTS_REACHED` are silent failures that will otherwise look like success.

## Step 5 — Read it back

```
GET /data/{schema-name}/
```

This is the **only operation in any published Monetate spec that declares a `429`** ("the user has sent
too many requests in a given amount of time to a rate-limited endpoint"). No numeric limit and no
`Retry-After` are published, so back off exponentially on 429 and do not hammer it in a loop. See
`rate-limits/monetate-rate-limits.yml`.

## Step 6 — Designate a default product catalog

```
GET  /defaultcatalog/
POST /defaultcatalog/          {"account": "123", "schema": "456"}
POST /bulk-defaultcatalog/     {"defaultaccountcatalogs": [{"account": "123", "schema": "456"}, ...]}
```

Both `account` and `schema` are required and are string ids. Use the bulk form when wiring more than one
account — the single form has no batching and there is no rate-limit guidance to size a loop against.

## Inspecting and evolving a schema

```
GET   /schema/                        # active schema; optional flags row_count, latest_upload, usable_in_accounts
GET   /schema/{schema-name}/          # one schema; optional flags row_count, latest_upload
PATCH /schema/{schema-name}/          # partial update
```

Those boolean query flags are Monetate's substitute for field expansion — there is no generic
`expand`/`fields` parameter.

## Pagination

List responses page with `page_size` (**maximum 1000**) and return `meta.count`, `meta.next` and
`meta.previous`. Follow `meta.next` rather than incrementing a page counter; the `page` parameter appears
in Monetate's own code samples but is not declared in the specification.

## Right to erasure

The `customer_data_privacy` schema type backs deletion requests. A `CustomerDataPrivacy` record reports
`status` of `PENDING`, `FOUND` or `NOT_FOUND` with a human-readable `description`. Note the spec declares
`status` as `type: integer` while its own example returns strings — parse defensively.
