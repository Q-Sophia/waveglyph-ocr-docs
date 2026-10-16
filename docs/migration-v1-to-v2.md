# WaveGlyph OCR V1 to V2 Migration Guide

**Target:** WaveGlyph OCR 2.0.0. **Prepared:** 2026-10-16.
Documentation edition: 2026-10-16. This is a synthetic service migration for engineering
exercises, not a claimed production incident.

## Breaking changes

| V1 | V2 |
| --- | --- |
| `POST /v1/recognize` synchronously returns text | `POST /v2/jobs` accepts work; poll `GET /v2/jobs/{id}` |
| `{"image_url":"..."}` | `{"input":{"image":{"kind":"url","uri":"..."}}}` |
| HTTP 200 immediately indicates recognition success | HTTP 202 means queued; a GET with HTTP 200 can still mean processing or failed |
| `text` at response root | `result.document.blocks`, sorted by ascending `order`, joined by newline |
| One request timeout | One overall deadline across all HTTP calls and sleeps |

The old route is removed, not redirected: it returns **410** with `api_retired`.
Changing only the URL does not constitute migration. V2 rejects flat input with
422 `invalid_request`; there is no V1 payload compatibility mode.

## New request and response

Send JSON to `/v2/jobs`:

```json
{"input":{"image":{"kind":"url","uri":"https://images.example.test/note.png"}}}
```

The only supported image kind is `url` and the URI must use HTTP or HTTPS.
Unknown keys in request objects are rejected. An accepted response is:

```json
{"job":{"id":"example-job-id","state":"queued"},"poll_after_ms":25}
```

Save the opaque ID and query `/v2/jobs/example-job-id`. Pending responses use
`job.state` (`queued` or `processing`) and `poll_after_ms`. It is a delay in
**milliseconds**, not seconds. Honor it subject to the remaining time budget.
Terminal responses are described below. The complete reference is [V2 API](v2-api.md).

## Asynchronous lifecycle and result extraction

Jobs progress `queued → processing → succeeded|failed`. Polling does not drive
execution and may miss intermediate states. Both terminal states return HTTP 200.

Success:

```json
{"job":{"id":"example-job-id","state":"succeeded"},"result":{"document":{"blocks":[{"order":8,"text":"Second line"},{"order":2,"text":"First line"}]}}}
```

Sort blocks by the integer `order`, **not** by wire order or text; join their `text`
values with `\n`. The resulting text here is `First line\nSecond line`.
Indices are unique and nonnegative, but may contain gaps. Empty blocks mean `""`.
Preserve Unicode, whitespace and line breaks. Response additions may be ignored.

Failure:

```json
{"job":{"id":"example-job-id","state":"failed"},"error":{"code":"image_unreadable","message":"Image could not be recognized"}}
```

Stop immediately and surface the service failure; do not return text or keep
polling. Treat unknown states, mismatched IDs and malformed success envelopes as
protocol errors. A terminal response stays terminal until retention expires.

## Status codes, timeout and retry policy

- POST 202: accepted, no recognition result yet.
- GET 200: inspect `job.state`; it does not alone indicate recognition success.
- 410: old API retired. 422: invalid input. 404: unknown/expired job.
- 503: job store full. Other non-success responses and transport failures are
  service errors. These can be surfaced; automatic retries are optional.
- Allocate one overall deadline before submitting. Deduct HTTP and sleep time,
  and cap every subsequent timeout and sleep by the remaining budget.
- Use `poll_after_ms` (25 ms default when absent). Avoid a busy loop.
- Do not resubmit automatically after a read timeout: POST has no idempotency key.
- Jobs expire after 600 seconds, are held in memory, and are lost on restart.

## Neutral command-line example

```sh
BASE=http://localhost:8000
curl -sS -X POST "$BASE/v2/jobs" -H 'Content-Type: application/json' \
  -d '{"input":{"image":{"kind":"url","uri":"https://images.example.test/receipt.png"}}}'
# Read the returned job.id, wait poll_after_ms milliseconds, then:
curl -sS "$BASE/v2/jobs/REPLACE_WITH_RETURNED_ID"
```

Repeat GET only for a nonterminal state. This example illustrates the public API;
it is not a complete integration implementation. Business functions can retain
synchronous return semantics by waiting internally for a terminal result.
