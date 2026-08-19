---
name: Ingest an externally originated call into Invoca
description: >-
  Submit a call that happened outside Invoca's telephony — on your own switch, a partner
  network, or another provider — so Invoca can transcribe it, run Signal AI against it, and
  fold it into attribution reporting. Covers the endpoint, timestamp formats, recording
  handoff, and the error contract.
api: Invoca Call Ingestion API
docs: https://developers.invoca.net/en/latest/api_documentation/call_ingestion_api/index.html
operations:
  - POST https://invoca.net/api/2022-03-01/calls.json
method: generated
generated: '2026-08-13'
source: https://developers.invoca.net/en/latest/api_documentation/call_ingestion_api/index.html
---

# Ingest an externally originated call into Invoca

Use this when the call did **not** ride an Invoca tracking number but you still want Invoca's
conversation intelligence on it.

## Before you start

- **Credential.** An Invoca API token from `Integrations > Manage Integrations > Invoca APIs`.
  Header form: `Authorization: <token>`. See `authentication/invoca-authentication.yml`.
- **Requirements.** The developer portal documents a Requirements section for this API —
  check it before your first call, because ingestion depends on network configuration that is
  set up with Invoca, not through the API.

## Step 1 — POST the call

```
POST https://invoca.net/api/2022-03-01/calls.json
Content-Type: application/json
Accept: application/json
Authorization: <your-invoca-api-token>
```

The date segment `2022-03-01` is the API version. Substituting a version the route does not
support returns `400 Bad Request` with a message naming the correction.

## Step 2 — get the request parameters right

Three things trip integrations up, and Invoca documents all three explicitly:

1. **Timestamps.** The reference has a dedicated *Timestamp Formats* section. Follow it
   exactly — Invoca's own responses carry both an ISO 8601 string (`occurred_at_time`) and a
   Unix epoch integer (`occurred_at_time_t`), so be deliberate about which the field expects.
2. **Phone number fields.** There is a *Recommendations on Phone Number Fields* section.
   Elsewhere on the platform numbers are E.164 (`+18885550100`); do not assume a bare
   national format will be accepted.
3. **Recordings.** *Supported Recording Formats* and *Supported Recording Access Options*
   define how you hand Invoca the audio. Transcription and Signal AI depend on it, so a call
   ingested without a usable recording gives you attribution but not conversation
   intelligence.

## Step 3 — check the response code

Invoca returns HTTP status codes with meaning; check them rather than parsing the body first.
`403 Forbidden` on this platform means invalid input **or** validation failure as often as it
means a permission problem — read `errors.class` and `errors.invalid_data`. Full table in
`errors/invoca-problem-types.yml`.

## Step 4 — retries

Invoca's design principle is that "most interfaces are designed to be idempotent … it is
harmless to call them more than once with the same parameters." There is no
`Idempotency-Key` header. Send the same identifying parameters on a retry rather than
generating new ones.

**Known documentation gap:** the reference lists *Call Processing Error Notifications* and
*Retrying Failed Calls* as sections, but both read "Details on this process coming soon."
Invoca has not published how failed ingestion is signalled back to you or what the retry
procedure is. Build your own dead-letter handling and reconcile through the Transactions API
rather than waiting for a notification.

## Step 5 — verify, then enrich

Once the call is in, confirm it via the Transactions API
(`https://invoca.net/api/2018-02-01/transactions.json`), then attach outcomes with
`skills/invoca-report-signal.md`.

## Related

- `data-model/invoca-data-model.yml` — Call, Transaction, Transcript.
- `rate-limits/invoca-rate-limits.yml` — Invoca publishes no limits and no rate-limit headers;
  pace bulk backfills conservatively.
