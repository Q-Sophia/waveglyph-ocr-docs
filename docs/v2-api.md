# WaveGlyph OCR V2 API reference (2.0.0)

## Authentication prerequisite

Every V2 request, including every poll, requires the HMAC-SHA256 protocol in the
[Migration Guide](migration-v1-to-v2.md#v2-request-authentication-and-canonicalization).
That guide is the normative source for canonicalization, signed field order,
header formatting, timestamp/nonce rules and conformance test vectors. The JSON
examples below are payloads only; they are not unsigned complete requests.
Authentication precedes schema validation; unauthenticated probes receive only 401.

## Submission

`POST /v2/jobs`, `Content-Type: application/json`:

```json
{"input":{"image":{"kind":"url","uri":"https://images.example.test/receipt.png"}}}
```

All three objects are closed: unknown keys are rejected. `kind` must be `url`;
`uri` is a valid HTTP(S) URL. The old flat `image_url` structure is invalid.

HTTP **202** (not a finished result):

```json
{"job":{"id":"example-job-id","state":"queued"},"poll_after_ms":25}
```

IDs are opaque URL-safe strings. The implementation generates UUIDs, but consumers
must not derive them from input, assume reuse, or parse them for business meaning.
Each submission creates an independent job. POST is not idempotent; do not blindly
retry a submission after an ambiguous network failure.

## Polling

`GET /v2/jobs/{id}` returns HTTP **200** for all known job states, including failure.

```json
{"job":{"id":"example-job-id","state":"processing"},"poll_after_ms":25}
```

Valid states: `queued` → `processing` → `succeeded` **or** `failed`.
Reads do not advance state; elapsed monotonic time does. A slow caller may skip
observing intermediate states. `queued` and `processing` are nonterminal.
`poll_after_ms` recommends the delay before the next request, in **milliseconds**.
It is nonnegative. If omitted, a 25 ms default is suitable for this demonstration.

Successful terminal response:

```json
{"job":{"id":"example-job-id","state":"succeeded"},"result":{"document":{"blocks":[{"order":1,"text":"Total: 18.50"},{"order":0,"text":"Wave Cafe"}]}}}
```

`result.document.blocks` is a list of text blocks. Every block has a unique,
nonnegative integer `order` and a string `text`. Sort by ascending `order`, then
join text with a single newline. Wire array order is not reading order. Indices
need not be contiguous. Preserve Unicode, blank text and whitespace. An empty
list represents successful empty text, not a failure. Unknown additional response
fields may be ignored.

Failed terminal response (still HTTP 200):

```json
{"job":{"id":"example-job-id","state":"failed"},"error":{"code":"image_unreadable","message":"Image could not be recognized"}}
```

A failed job has no recognition result. Stop polling and report the failure.
Unknown states, mismatched job IDs, missing required data, invalid JSON, duplicate
block indices or wrong types are protocol errors. Do not return an empty successful
result for these conditions, and do not poll an unknown state indefinitely.

## HTTP errors and lifecycle

| HTTP | Meaning | Body |
| --- | --- | --- |
| 202 | Job accepted | queued submission envelope |
| 200 | Known job or health | state envelope or health object |
| 401 | Authentication, clock or replay failure | `{"error":{"code":"authentication_failed"}}` |
| 410 | Retired V1 POST | `{"error":{"code":"api_retired"}}` |
| 422 | Invalid V2 request | `{"error":{"code":"invalid_request"}}` |
| 404 | Unknown/expired job | `{"error":{"code":"job_not_found"}}` |
| 503 | Capacity exhausted | `{"error":{"code":"capacity_exceeded"}}` |

Authenticated missing V2 routes and missing non-V2 routes return the framework's generic 404 `{"detail":"Not Found"}`;
unsupported methods return 405. Non-2xx HTTP responses are service errors.
Validation errors intentionally do not expose schema details.

Jobs are retained for 600 seconds and held in memory by a **single worker**.
Restarting the service loses outstanding jobs. Expired jobs return 404 and are
removed on later submission. Capacity is 10,000 retained jobs. Do not run multiple
workers against this in-memory demonstration implementation.

`GET /health` returns `{"status":"ok","product":"WaveGlyph OCR","version":"2.0"}`.
OpenAPI, Swagger, ReDoc, filesystem and debugging routes are not exposed.

## Time budget and demonstration images

Use an overall deadline covering submission, sleeps and all polls, not a new full
budget for each request. Cap each request and each delay by the remaining budget.
A caller may choose to treat transport errors as failure; automatic retries are
not required. Never retry terminal failure as if it were still processing.

The simulator does not fetch image bytes. URLs identify deterministic fixtures at
`https://images.example.test/`: `note.png`, `receipt.png`, `multiblock.png`,
`empty.png`, `failed.png`, and `slow.png`. Normal jobs queue for 80 ms and finish
at 200 ms; `slow.png` remains nonterminal for 30 seconds, then fails as unreadable.
Unknown images eventually fail. `sample/<token>.png` produces a generated document
containing the URL-decoded token and a verification line. This fixture feature is
for demonstrations and is not a real OCR claim.
