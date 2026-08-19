---
name: replyrai-run-a-sales-pipeline
description: >-
  Create and move an opportunity through a Replyr sales pipeline — list
  pipelines and stages, open an opportunity against a contact, update its stage
  and value, comment on it, transfer it to another pipeline, and close it out.
api: Replyr Platform API
base_url: https://app.replyr.ai/api
generated: '2026-08-13'
method: generated
source: openapi/replyrai-platform-api-swagger.json
operations:
  - getPipelines
  - getPipeline
  - getPipelineStages
  - getPipelineCustomFields
  - pipelinesGetCards
  - pipelinesAddCard
  - pipelinesGetCard
  - pipelinesUpdateCard
  - pipelinesTransferCard
  - pipelinesGetComments
  - pipelinesAddComment
  - pipelinesDeleteComment
  - pipelinesDeleteCard
---

# Run a sales pipeline

Replyr's pipeline surface is the most complete part of the API — 13 operations,
real pagination, and the only place the contract consistently declares response
schemas. Terminology is inconsistent in the published docs: an **opportunity**,
a **ticket** and a **card** are all the same object, and the operationIds use
`Card` while the paths use `opportunities`.

## Before you start

- Header on every call: `X-ACCESS-TOKEN: <account key>`.
- Base URL: `https://app.replyr.ai/api`.
- Writes are not idempotent. `pipelinesAddCard` called twice creates two
  opportunities. Check with `pipelinesGetCards?contact_id=<id>` before retrying.

## Step 1 — Find the pipeline

`getPipelines` (GET `/pipelines`) returns `{"data": [Pipeline]}` where a Pipeline
is just `{id, name}`.

This is one of only three operations in the whole API with real pagination:

```
GET /pipelines?offset=0&limit=100
```

`offset` defaults to 0, `limit` defaults to 100. There is no total count in the
response, so page until you get fewer rows than `limit`.

`getPipeline` (GET `/pipelines/{pipeline_id}`) fetches one.

## Step 2 — Read the stages

`getPipelineStages` (GET `/pipelines/{pipeline_id}/stages`) returns
`{"data": [PipelineStage]}`, each `{id, name}`. Resolve the stage you want by
name and hold its id — you need it for step 3 and step 5.

`getPipelineCustomFields` (GET `/pipelines/{pipeline_id}/custom_fields`) returns
the custom fields defined on this pipeline. These are pipeline-scoped and are not
the same set as the account-level contact custom fields.

## Step 3 — Open an opportunity

`pipelinesAddCard` (POST `/pipelines/{pipeline_id}/opportunities`,
`application/json`). `contact_id` and `title` are required:

```json
{
  "contact_id": 8891,
  "title": "Fractional CO2 — 3 session package",
  "description": "Inbound from Meta ad, asked about deeper scarring.",
  "stage_id": 4412
}
```

If `stage_id` is omitted the opportunity lands in the pipeline's default entry
stage. The operation declares only a `default` response with no schema, so do not
depend on the created object coming back — re-read with `pipelinesGetCards`
filtered by `contact_id` to get the new opportunity id.

## Step 4 — List what is in flight

`pipelinesGetCards` (GET `/pipelines/{pipeline_id}/opportunities`) is the
workhorse. It takes three optional query parameters:

- `contact_id` — filter to one contact's opportunities. This is how you avoid
  creating a duplicate.
- `offset` / `limit` — real pagination, same defaults as step 1.

Returns `{"data": [Opportunity]}`. An `Opportunity` carries `id`, `contact_id`,
`title`, `description`, `value`, `status`, `priority`, an embedded `stage`
object, `assigned_admins`, and `created_at` / `created_by` / `updated_at` /
`updated_by`.

Note `stage` is embedded as an object rather than exposed as a `stage_id`
reference on reads, while writes take `stage_id`. Read the id out of the nested
object.

`pipelinesGetCard` (GET `/pipelines/{pipeline_id}/opportunities/{opportunity_id}`)
fetches one.

## Step 5 — Move it

`pipelinesUpdateCard` (**POST**, not PATCH or PUT, to
`/pipelines/{pipeline_id}/opportunities/{opportunity_id}`). Send the fields you
want to change:

```json
{ "stage_id": 4415, "value": 3600, "priority": "high" }
```

`value` is a number. The API does not publish a currency on the Opportunity
object — currency lives on `Cart` and `Order`, not here — so agree the unit with
the account owner before writing values.

## Step 6 — Leave a trail

- `pipelinesAddComment` (POST `.../opportunities/{opportunity_id}/comments`)
- `pipelinesGetComments` (GET, same path) — paginated with `offset`/`limit`
- `pipelinesDeleteComment` (DELETE `.../comments/{comment_id}`)

A comment is `{id, data, created_at, created_by}` — the body text is in `data`,
and `created_by` is an admin id you can resolve with `getAccountAdmins`.

Comments are the only audit surface on an opportunity. There is no change
history and no webhook, so if you need a record of *why* a stage moved, write a
comment as part of the same workflow step.

## Step 7 — Transfer or close

`pipelinesTransferCard` (POST
`.../opportunities/{opportunity_id}/transfer-to-pipeline`) moves an opportunity
to a different pipeline — for example handing a qualified enquiry from a
marketing pipeline to a treatment-scheduling one.

`pipelinesDeleteCard` (DELETE `.../opportunities/{opportunity_id}`) removes it.
This is destructive, not idempotent-safe, and there is no undo or soft-delete in
the contract. Prefer moving to a "Closed lost" stage over deleting.

## Handling failures

The pipeline operations declare only `200` and `default` responses — no 400, no
404, no 401 — even though every one of them requires the API key. Expect and
handle:

| Status | Meaning | What to do |
|---|---|---|
| 401 | `{"error":{"code":401,"message":"No valid API key provided."}}` | Fix the header. Returned for unknown paths too. |
| 400 | Malformed body or wrong parameter type | `pipeline_id` and `opportunity_id` are typed `number`, `contact_id` on the body is `number` — send numerics, not strings. |
| 404 | Unknown pipeline or opportunity | Re-list with `getPipelines` / `pipelinesGetCards`. |

No rate-limit headers are returned and no 429 is documented. Bound your own
concurrency when walking every opportunity in a large pipeline.
