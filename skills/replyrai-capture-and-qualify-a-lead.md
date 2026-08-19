---
name: replyrai-capture-and-qualify-a-lead
description: >-
  Take an inbound clinic enquiry (a phone number, optionally an email and name),
  find or create the Replyr contact, tag and enrich it with qualification data,
  and send the first reply on the contact's chat channel.
api: Replyr Platform API
base_url: https://app.replyr.ai/api
generated: '2026-08-13'
method: generated
source: openapi/replyrai-platform-api-swagger.json
operations:
  - findByCustomField
  - createNewContact
  - getAccountTags
  - getTagByName
  - createTag
  - addTagToUser
  - getAccountCustomFields
  - getCustomFieldByName
  - addFieldToUser
  - sendTextMessage
  - getUserByName
---

# Capture and qualify a lead

Replyr's core loop: an enquiry arrives, it becomes a contact, the contact gets
labelled with what you know about them, and someone (or an AI agent) replies.
Every step below uses an operationId that exists in the Replyr Swagger 2.0
document at `https://app.replyr.ai/api`.

## Before you start

- Every call requires the header `X-ACCESS-TOKEN: <your account key>`. There are
  no scopes — this one key can do everything in this skill *and* everything else
  in the API, including sending messages to every contact on the account and
  changing order payment state. Treat it as a full-account credential.
- The base URL is `https://app.replyr.ai/api`. There is no version segment.
- **There is no idempotency key on this API.** Steps 4 and 6 have real-world side
  effects — a duplicate `createNewContact` and a duplicate `sendTextMessage` both
  produce a duplicate outcome a human will see. Deduplicate on your side before
  you retry: keep the phone number as your own idempotency key and check step 1
  again before re-issuing a write.

## Step 1 — Look for the contact before creating one

`findByCustomField` (GET `/contacts/find_by_custom_field`) is the only lookup by
a natural key. Pass `field_id=phone` (or `field_id=email`) and `value=<the
number>`.

```
GET /contacts/find_by_custom_field?field_id=phone&value=%2B60123456789
X-ACCESS-TOKEN: <key>
```

Returns `{"data": [ Contact, ... ]}`. Two things to know:

- The cap is **100 contacts** and there is no pagination on this operation. If a
  lookup can plausibly match more than 100, narrow the value — you cannot page
  past the cap.
- Results are sorted by the last custom-field update, not by relevance.

If `data` is non-empty, take `data[0].id` as your `contact_id` and skip to
step 3.

## Step 2 — Create the contact if it does not exist

`createNewContact` (POST `/contacts`, `application/json`). `phone` is the only
required field, and it must carry a country code.

```json
{
  "phone": "+60123456789",
  "email": "lead@example.com",
  "first_name": "Aisha",
  "last_name": "Rahman",
  "gender": "female"
}
```

Two contract defects to code around:

- `gender` is a **string enum** (`male` / `female` / `unknown`) on the request,
  but an **integer** on the `Contact` object you read back. Do not round-trip it.
- This operation declares only a `default` response with no schema, so the shape
  of what comes back is not specified. Do not depend on it — re-read with
  `findByCustomField` or `getUserByName` if you need the contact id confirmed.

`createNewContact` also accepts an `actions` array, which lets you fold steps 3
and 5 into this one call:

```json
"actions": [
  {"action": "add_tag", "tag_name": "inbound-whatsapp"},
  {"action": "set_field_value", "field_name": "source", "value": "meta-ads"},
  {"action": "send_flow", "flow_id": 11111}
]
```

Prefer this when you are creating the contact anyway — it is one round trip
instead of four, and `tag_name` / `field_name` mean you do not have to resolve
numeric ids first.

## Step 3 — Resolve the tag you want to apply

Tags are account-scoped and addressable **by name**, which is what you want —
never hard-code a numeric tag id across environments.

- `getTagByName` (GET `/accounts/tags/name/{tag_name}`) — resolve name → id.
- `getAccountTags` (GET `/accounts/tags`) — list everything if you need to pick.
- `createTag` (POST `/accounts/tags`, form field `name`) — create it if missing.

## Step 4 — Tag the contact

`addTagToUser` (POST `/contacts/{contact_id}/tags/{tag_id}`). No body. Returns
`{"success": true}`.

Use `removeTagFromUser` (DELETE, same path) to reverse it. Tagging is how
qualification state is represented in Replyr — there is no lead-status field.

## Step 5 — Write the qualification data onto the contact

Custom fields are the account's own schema. Resolve by name first, then set the
value.

- `getCustomFieldByName` (GET `/accounts/custom_fields/name/{custom_field_name}`)
- `createCustomField` (POST `/accounts/custom_fields`) if it does not exist
- `addFieldToUser` (POST `/contacts/{contact_id}/custom_fields/{custom_field_id}`)

`addFieldToUser` consumes **`application/x-www-form-urlencoded`**, not JSON —
send `value=<string>` as a form field. This is inconsistent with the JSON write
operations elsewhere in the API and is a common cause of a silent 400.

```
POST /contacts/8891/custom_fields/2334
Content-Type: application/x-www-form-urlencoded

value=acne-scarring
```

Read a single value back with `getFieldValue`, or the whole set with
`getUserFields`.

## Step 6 — Send the first reply

`sendTextMessage` (POST `/contacts/{contact_id}/send/text`, `application/json`):

```json
{ "text": "Thanks for reaching out! ...", "channel": "whatsapp" }
```

`channel` is optional and accepts `messenger`, `whatsapp`, `sms`, `webchat`,
`telegram`, `instagram`, `viber`, `omnichannel`. Omit it to use the contact's own
channel; use `omnichannel` to let Replyr route.

**This delivers a real message to a real person.** There is no idempotency key
and no dry-run. If the call times out, do not blind-retry — the message may
already have been delivered and the API gives you no way to find out.

For anything richer than a line of text, use `sendFlowToUser` (POST
`/contacts/{contact_id}/send/{flow_id}`) to trigger a designed flow, or
`sendContent` (POST `/contacts/{contact_id}/send_content`) to send several
messages and run several actions in one call.

## Handling failures

There is no error-code registry — branch on the HTTP status and nothing else.

| Status | Meaning | What to do |
|---|---|---|
| 401 | `{"error":{"code":401,"message":"No valid API key provided."}}` | Header missing, malformed, revoked, or for another account. The same body is returned for all four, and for paths that do not exist — a 401 is not proof the operation is real. |
| 400 | Invalid parameters | Check types against the spec, and check you used form encoding where the operation demands it. |
| 404 | Contact not found | Re-resolve with `findByCustomField`. |

Note the envelope is JSON but is served with `Content-Type: text/html` — parse on
shape, not on content type. No `Retry-After` or rate-limit header is ever
returned, and no 429 is documented, so back off on your own schedule.
