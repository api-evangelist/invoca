# Invoca OpenAPI

## invoca-sms-messaging-openapi.json

- **method:** searched (harvested verbatim)
- **source:** https://developers.invoca.net/en/latest/_static/js/swagger-ui/swagger-initializer.js
- **fetched:** 2026-08-13 (HTTP 200)
- **spec:** OpenAPI 3.0.3 — `info.title` "SMS Messaging API", `info.version` "2022-08-01",
  8 paths / 12 operations, 2 component schemas, 4 component parameters.

### Where it was found

Invoca's developer portal (`developers.invoca.net`, a Read the Docs / Sphinx site) bundles
Swagger UI. Its initializer, `_static/js/swagger-ui/swagger-initializer.js`, does **not** point
Swagger UI at a spec URL — it embeds the whole document inline in the `spec:` option. The same
file (byte-identical, 27,853 bytes) is served under every published documentation version
(`/en/latest/`, `/en/2026-02-01/`, `/en/2022-08-01/`, `/en/2019-05-01/`, `/en/2019-02-01/`,
`/en/2017-04-01/`, `/en/2016-08-22/`); `/en/stable/` and `/en/2013-03-22/` 404.

No page in the portal carries a `<div id="swagger-ui">`, so the document is loaded on every
docs page but never rendered. It is nevertheless a real, publicly served, machine-readable
Invoca contract at a stable URL — this is the only OpenAPI Invoca publishes.

### Ownership (STEP 0c)

`servers[]` is empty, so the fetch host is not the only evidence. The document identifies
itself as Invoca's in its own content:

- `GET /phone_numbers` — summary: *"Gets summarized metadata for phone numbers assigned to an
  **Invoca Network**"*, and the response schema carries `network_id` / `virtual_line_id`,
  Invoca's own network and line identifiers.
- Every pagination example uses the path prefix `/sms/2022-08-01/...`, which matches the
  date-in-path versioning Invoca documents in
  https://developers.invoca.net/en/latest/basics/design_principles.html
- The error envelope in the spec (`{"message": "..."}`) matches Invoca's documented
  error-handling contract.
- It is served from `developers.invoca.net`, Invoca's own developer portal on its own
  `invoca.net` domain.

No sibling-brand or third-party markers appear anywhere in the document.

### Caveats

- `servers[]` is empty and the portal does not document an SMS Messaging base URL, so the
  callable host is **not** recorded here and has not been guessed. The `201` example uses the
  placeholder `https://example.org/sms/2022-08-01/phone_numbers/...`.
- Operations have no `operationId`s; artifacts derived from this spec reference operations as
  `METHOD /path`.
