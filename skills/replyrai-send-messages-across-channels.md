---
name: replyrai-send-messages-across-channels
description: >-
  Deliver outbound messages to a Replyr contact on WhatsApp, Messenger,
  Instagram, SMS, Telegram, Viber, RCS or web chat — plain text, files, products,
  designed flows, and multi-message/multi-action sends — safely, on an API with
  no idempotency and no delivery webhook.
api: Replyr Platform API
base_url: https://app.replyr.ai/api
generated: '2026-08-13'
method: generated
source: openapi/replyrai-platform-api-swagger.json
operations:
  - findByCustomField
  - getUserByName
  - sendTextMessage
  - sendFile
  - sendFlowToUser
  - sendContent
  - sendProduct
  - getAccountFlows
  - aiSaveMessages
  - getAccountInformation
---

# Send messages across channels

Everything outbound in Replyr is addressed by `contact_id` and delivered on a
channel the account has connected. This skill covers the five send operations,
the channel enums that actually differ between them, and the safety rules that
matter because **this API has no idempotency key and no delivery callback.**

## The safety rule that comes first

Every operation here delivers a message to a real person on a real messaging
platform. The API gives you:

- no idempotency key or request-deduplication parameter,
- no delivery receipt, webhook, or event stream,
- no `Retry-After` header and no documented 429,
- and, on `sendFlowToUser`, `sendContent` and `sendProduct`, only a `default`
  response with **no schema at all**, so you cannot even reliably read a success
  flag.

Consequence: **a timeout is not a failure.** If a send call does not return, the
message may well have been delivered. Do not blind-retry. Record your own send
key (contact id + template + a coarse timestamp bucket) before the call and check
it before any retry. `sendTextMessage` and `sendFile` at least return
`{"success": true}`; the other three do not.

## Step 0 — Resolve the contact and confirm the connected channels

Get `contact_id` from `findByCustomField` (`field_id=phone` or `email`) or, if
you already hold it, verify with `getUserByName` (GET `/contacts/{contact_id}`).

`getAccountInformation` (GET `/accounts/me`) tells you what the account can
actually reach: `fb_page_id`, `instagram_id`, `waba_id` (WhatsApp Business
Account), `wa_phone_id` and `viber_id`. A channel with no connected id will not
deliver, and the API does not warn you in advance.

The `Contact` object also carries an integer `channel` field indicating where it
originated — but Replyr publishes no enum mapping those integers to channel
names, so do not try to interpret it. Omit `channel` on send and let Replyr route
to the contact's own channel, or pass `omnichannel`.

## Step 1 — Plain text

`sendTextMessage` — POST `/contacts/{contact_id}/send/text`, `application/json`:

```json
{ "text": "Your appointment is confirmed for Tuesday 3pm.", "channel": "whatsapp" }
```

`text` is required. `channel` is optional, enum:
`messenger`, `whatsapp`, `sms`, `webchat`, `telegram`, `instagram`, `viber`,
`omnichannel`.

Returns `{"success": true}`. This is the only send operation with both a declared
schema and a usable success signal — prefer it when a single line of text will do.

## Step 2 — Files

`sendFile` — POST `/contacts/{contact_id}/send/file`. Use for treatment
information sheets, pre/post-care PDFs, before/after images. Returns
`{"success": true}`.

## Step 3 — Designed flows

`sendFlowToUser` — POST `/contacts/{contact_id}/send/{flow_id}`.

List available flows with `getAccountFlows` (GET `/accounts/flows`) and resolve
by name; flow ids are account-specific and must not be hard-coded across
environments. Replyr publishes no schema for the Flow object, so treat the
listing as opaque apart from its id and name.

A flow is the right tool whenever the interaction has branches — qualification
question trees, booking, re-engagement sequences. It runs inside Replyr, so the
AI agent handles the replies rather than your code polling for them.

`sendFlowToUser` declares only a `default` response. No success flag.

## Step 4 — Several messages and actions in one call

`sendContent` — POST `/contacts/{contact_id}/send_content`, `application/json`.
The most capable send, and the one that works across every channel:

```json
{
  "messages": [ { "message": { "text": "Hello world" } } ],
  "actions": [],
  "channel": "whatsapp"
}
```

Note its `channel` enum is **not the same** as `sendTextMessage`'s — it adds
`googleBM` and `rcs`, and drops `webchat`:
`messenger`, `sms`, `whatsapp`, `googleBM`, `telegram`, `instagram`, `rcs`,
`viber`, `omnichannel`. Validate against the right enum for the operation you are
calling; a value valid on one is not necessarily valid on the other.

`actions` accepts the same action vocabulary as `createNewContact` — `add_tag`,
`set_field_value`, `send_flow` — so you can message and update contact state in a
single round trip. Because there is no idempotency, doing it in one call is also
safer than doing it in three: one partial failure instead of three.

Declares only a `default` response. No success flag.

## Step 5 — Products

`sendProduct` — POST `/contacts/{contact_id}/send/products` sends a product
message the contact can act on. It leads into the cart and order surface:
`addProductCart`, `getUserCart`, `getUserOrder`, `payOrder`. Treat that as a
separate, higher-consequence workflow — `payOrder` changes payment state, is not
idempotent, and its documented 402 carries no decline code to branch on.

## Step 6 — Keep the AI agent's context straight

`aiSaveMessages` — POST `/contacts/{contact_id}/ai/save_messages` appends a
message to the contact's AI message history.

Use it when your system sent something **outside** Replyr that the AI agent needs
to know about — a call summary, an email, a message sent by another tool.
Skipping it is how an AI agent ends up repeating a question a human already
answered elsewhere. Declares only a `default` response.

## Handling failures

| Status | Meaning | What to do |
|---|---|---|
| 401 | `{"error":{"code":401,"message":"No valid API key provided."}}` | Missing/invalid/revoked key, or wrong account. Also returned for paths that do not exist. |
| 400 | Invalid parameters | Most often a `channel` value from the wrong enum, or a missing required `text`. |
| 404 | Contact not found | Re-resolve with `findByCustomField`. |
| timeout / no response | **Unknown, not failed** | Do not retry blind. Check your own send log. |

The error body is JSON served as `Content-Type: text/html` — parse on shape. No
rate-limit headers, no 429, no quota published: if you are sending in bulk,
throttle conservatively on your side and stagger sends, because the first signal
that you have exceeded an undocumented limit may be a platform-level block on the
underlying WhatsApp or Meta channel rather than an API error you can see.
