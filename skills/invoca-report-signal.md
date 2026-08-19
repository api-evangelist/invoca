---
name: Report a conversion Signal against an Invoca call
description: >-
  Attach a Signal (a boolean outcome such as a sale or quote) and/or Custom Data Fields to an
  Invoca call transaction after the call has ended, so the conversion is attributed back to
  the marketing source that drove it. Covers finding the right call, the natural-key
  idempotency rules that make retries safe, and how to read Invoca's error envelope.
api: Invoca Signal API
docs: https://developers.invoca.net/en/latest/api_documentation/signal_api/index.html
operations:
  - POST https://invoca.net/api/<version>/transactions.json
method: generated
generated: '2026-08-13'
source: https://developers.invoca.net/en/latest/api_documentation/signal_api/index.html
---

# Report a conversion Signal against an Invoca call

Use this when an outcome happened **after** the phone call — a sale closed in the CRM, a
quote was issued, an application was approved — and Invoca needs to know so the revenue lands
against the campaign that produced the call.

## Before you start

- **Credential.** You need an Invoca API token. It is created in the platform under
  `Integrations > Manage Integrations > Invoca APIs > + New API Credential`. There is no
  test-vs-live key: the token inherits the privileges of the user who created it and acts on
  production data. Generate your own; never use a token issued on behalf of an Advertiser or
  Publisher. See `authentication/invoca-authentication.yml`.
- **Version.** The API version is a **date in the URL path**. Using a date segment the route
  does not support returns `400 Bad Request`. See `lifecycle/invoca-lifecycle.yml`.
- **Format.** Request and response are `application/json`.

## Step 1 — authenticate

Send the token one of three documented ways. Prefer the header.

```
Authorization: <your-invoca-api-token>
```

For a JSON POST you may instead pass it in the body as `"oauth_token": "<token>"`; for a GET
it may be a `?oauth_token=` query parameter. Invoca does not use OAuth refresh tokens and
exposes no scopes, so the token is long-lived — treat it like a password and keep it out of
source control.

## Step 2 — identify the call

Every Signal is applied to a **transaction** (a call). Identify it inside the `search` object.
The strongest identifier is `transaction_id`. If you do not have one, Invoca also accepts
`call_record_id`, `external_call_unique_id`, or `call_start_time` combined with other search
filters.

If none of those is supplied you get `403 Forbidden`:

```json
{ "errors": { "class": "InvalidInput",
              "invalid_data": "transaction_id, call_record_id, external_call_unique_id, or call_start_time must not be empty" } }
```

## Step 3 — POST the signals

```
POST https://invoca.net/api/<version>/transactions.json
Content-Type: application/json
Accept: application/json
Authorization: <your-invoca-api-token>
```

```json
{
  "search":  { "transaction_id": "00000000-00000001" },
  "signals": [ { "name": "Quote", "partner_unique_id": "1" },
               { "name": "Quote", "partner_unique_id": "2" } ]
}
```

- `name` is the Signal — a boolean outcome marker (`sale`, `quote`, …). Your plan caps how
  many custom Signals you may define (5 / 50 / 100 on Pro / Enterprise / Elite).
- `partner_unique_id` is **your** identifier for the thing that converted.
- Use `custom_data` for alphanumeric values (account type, quality score) rather than
  booleans.
- Set `call_in_progress: true` when the call is still live; the response becomes `201 Created`
  instead of `200 OK`.

A successful response returns each signal with a `transaction_id`, `corrects_transaction_id`,
`occurred_at_time` / `occurred_at_time_t`, and `value`, plus the `call` it was applied to.

## Step 4 — retry safely

**A Signal is unique on `(name, partner_unique_id)`.** This is the whole idempotency contract
— there is no `Idempotency-Key` header on any Invoca API.

- Re-post the *identical* request → nothing happens. No duplicate, no update.
- Re-post the same `(name, partner_unique_id)` with *different* field values → the original
  signal is updated. Invoca writes a **new** record whose `corrects_transaction_id` points at
  the record it supersedes, leaving an audit chain (Invoca's "Self-Correction" principle).
- Change `partner_unique_id` → a **second** signal of the same name is added to the call.

So: retry freely, but never mint a fresh `partner_unique_id` on a retry or you will create a
duplicate conversion.

## Step 5 — handle errors

Read `conventions/invoca-conventions.yml` and `errors/invoca-problem-types.yml` first; the
short version:

| Status | What it actually means |
|---|---|
| 200 | Applied. |
| 201 | Applied, with `call_in_progress: true`. |
| 400 | Wrong API version date for this route, or unparseable JSON. |
| 401 | Missing or invalid token. |
| 403 | **Invalid input or validation failure** — not necessarily a permission problem. |
| 404 | No call matched the search. |
| 500 | Invoca-side. Body carries `"refer to error handle <n>"` — quote that handle to support. |

Do not branch on `403` as an authorization failure. Invoca overloads it for bad data. Inspect
`errors.class`: `InvalidInput` is your request's fault, and a body keyed by *field name* with
arrays of messages is a validation failure listing every problem at once.

There are **no published rate limits and no rate-limit response headers** on any Invoca API
(`rate-limits/invoca-rate-limits.yml`). Back off on 5xx and be conservative with concurrency.

## Related

- `data-model/invoca-data-model.yml` — Transaction, Signal, CustomDataField relationships.
- `skills/invoca-ingest-call.md` — get an externally originated call into Invoca first.
