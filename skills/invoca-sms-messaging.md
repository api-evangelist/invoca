---
name: Send and track SMS through the Invoca SMS Messaging API
description: >-
  Send a text message from an Invoca-provisioned number, then follow its delivery through
  delivery reports and usage counters. This is the one Invoca surface with a machine-readable
  contract, so every operation below is grounded in the harvested OpenAPI document rather
  than in prose.
api: openapi/invoca-sms-messaging-openapi.json
docs: https://developers.invoca.net/en/latest/api_documentation/index.html
operations:
  - POST /messages
  - GET /messages
  - GET /messages/phone_number/{phone_number}
  - GET /messages/phone_number/{phone_number}/usage
  - GET /messages/uuid/{message_uuid}
  - DELETE /messages/uuid/{message_uuid}
  - GET /messages/uuid/{message_uuid}/delivery_reports
  - GET /messages/uuid/{message_uuid}/delivery_reports/uuid/{delivery_report_uuid}
  - GET /phone_numbers
  - POST /phone_numbers/{phone_number}
  - GET /phone_numbers/{phone_number}
  - DELETE /phone_numbers/{phone_number}
method: generated
generated: '2026-08-13'
source: openapi/invoca-sms-messaging-openapi.json
---

# Send and track SMS through the Invoca SMS Messaging API

The contract is OpenAPI 3.0.3, `info.version` `2022-08-01`, 8 paths / 12 operations. It is
harvested verbatim at `openapi/invoca-sms-messaging-openapi.json` — see `openapi/README.md`
for where it came from and why it is trustworthy.

## Read this first — two hard limits on this skill

1. **No base URL is published.** `servers[]` is empty in Invoca's own document, and no page of
   the developer portal names an SMS Messaging host. Paths are prefixed `/sms/2022-08-01/`
   (visible in every pagination URL in the spec's examples), and the `201` example uses the
   placeholder `https://example.org/...`. **Ask Invoca for your host.** Do not guess it from
   the other APIs — this skill deliberately does not.
2. **No securitySchemes are declared.** The document defines no auth at all, but every Invoca
   API requires a credential. Use the platform contract in
   `authentication/invoca-authentication.yml`: `Authorization: <invoca-api-token>`.

## Provision a number

```
POST {host}/sms/2022-08-01/phone_numbers/{phone_number}
```

`{phone_number}` is E.164 (`+18885550100`). On success you get `201` with the
`phone_number` object — `virtual_line_id`, `network_id`, `carrier_interface` (e.g.
`bandwidth`), `created_at`, `enabled_at`.

- `422` — the number is not valid.
- `409` — it is already on SMS Messaging.
- `403` — forbidden.

List with `GET /phone_numbers`, inspect one with `GET /phone_numbers/{phone_number}`, remove
with `DELETE /phone_numbers/{phone_number}` (which returns the record with `deleted_at` set —
a soft delete).

## Send a message

```
POST {host}/sms/2022-08-01/messages
Content-Type: application/json
```

The request body is an **array** of `MessageRequest`:

```json
[ { "to": ["+18885550100"], "from": "+18885550101", "text": "Your appointment is confirmed for Friday. Thank you." } ]
```

Note `to` is an array of strings while `from` is a single string.

A successful send returns **`202 Accepted`**, not `201` — the spec describes it as "creates a
message and forwards to kafka". Treat it as queued, not delivered.

- `400` — `{"message": "Problems parsing JSON"}`.
- `404` / `409` — see the `{"message": "..."}` envelope.

## Follow delivery

Delivery is asynchronous and you poll for it. There is **no webhook** in this contract.

```
GET {host}/sms/2022-08-01/messages/uuid/{message_uuid}
GET {host}/sms/2022-08-01/messages/uuid/{message_uuid}/delivery_reports
GET {host}/sms/2022-08-01/messages/uuid/{message_uuid}/delivery_reports/uuid/{delivery_report_uuid}
```

A `message` carries `direction`, `segments`, `delivered_at`, `failed_at`, `carrier_message_id`
and an embedded `delivery_reports` array. An individual delivery report adds `destination`,
`progress_bit`, `carrier_description` and `carrier_errorcode` — the carrier fields are where a
failure reason actually appears, so fetch the report rather than relying on `failed_at` alone.

`422` on the delivery-reports list tells you the sort column is unsupported and names the
supported ones (`created_at`, `progress_bit`).

## Paginate

Every list response wraps results next to a `pagination` object: `page`, `pages`, `count`,
`items`, `from`, `to`, `in`, `next`, `prev`, plus `next_url`, `prev_url`, `first_url`,
`last_url`, `page_url` and a `scaffold_url` template. Navigate by following `next_url` until
`next` is `null`; do not build page URLs yourself.

## Usage and clean-up

```
GET {host}/sms/2022-08-01/messages/phone_number/{phone_number}          # messages for a number
GET {host}/sms/2022-08-01/messages/phone_number/{phone_number}/usage    # incoming/outgoing message + segment counts
DELETE {host}/sms/2022-08-01/messages/uuid/{message_uuid}               # soft delete, returns deleted_at
```

Usage reports **segments** as well as messages — a long text bills as several segments, so
reconcile on segments.

## Conventions that apply

- Error envelope on this surface is `{"message": "<text>"}` — different from the rest of the
  Invoca platform, which returns an `errors` object. See `errors/invoca-problem-types.yml`.
- No rate limits and no rate-limit headers are published anywhere on Invoca
  (`rate-limits/invoca-rate-limits.yml`).
- The document declares no `operationId`s, which is why every operation above is referenced as
  `METHOD /path`.
