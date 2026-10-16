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
| Unsigned requests | Every V2 POST and GET requires canonical HMAC-SHA256, timestamp and fresh nonce |
| One request timeout | One overall deadline across all HTTP calls and sleeps |

The old route is removed, not redirected: it returns **410** with `api_retired`.
Changing only the URL does not constitute migration. V2 rejects flat input with
422 `invalid_request`; there is no V1 payload compatibility mode.

## V2 request authentication and canonicalization

V2 authenticates **every `/v2/` request**, including every status GET, with
HMAC-SHA256. This binds the method, route, payload, access identity and freshness
information so a captured submission cannot be changed or replayed as another
operation. Credentials are issued by the service operator and are available to
the integration developer. There is no hidden password or challenge value.
The algorithm specification, rather than the credential, is migration knowledge.

Provision the access identifier and UTF-8 secret through your application's
credential configuration. In the demonstration environment these are
`WAVEGLYPH_API_KEY` and `WAVEGLYPH_API_SECRET`. Never log either credentials or
canonical authenticated requests. Use HTTPS outside an isolated local sandbox.

### Required request headers

| Header | Value |
| --- | --- |
| `X-WaveGlyph-Key-Id` | The provisioned access identifier, verbatim |
| `X-WaveGlyph-Timestamp` | Current UTC Unix time in whole **seconds**, decimal ASCII, no leading zero |
| `X-WaveGlyph-Nonce` | Fresh per-request 16–64 ASCII characters from `A-Z a-z 0-9 _ -`; UUID hex is suitable |
| `Authorization` | `WG-HMAC-SHA256 ` followed by a lowercase 64-character hex signature |

Header names are case-insensitive. Each required header must occur exactly once;
values are case-sensitive and are not trimmed or otherwise normalized. Timestamp
syntax accepts 10–12 digits and the server accepts clock skew of at most 60 seconds.
A nonce is fresh for **each HTTP attempt**, not merely for each OCR job. Do not
reuse the POST nonce for GET, or reuse the nonce of an earlier poll.

### Canonical body

For a bodyless GET the canonical body is zero bytes, whose SHA256 is
`e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855`.
It is not `{}`, `null`, a space or a newline. GET bodies are rejected.

For POST, parse the UTF-8 JSON then serialize it as follows:

1. Include **every body member**, recursively. Do not sign only `uri` or omit
   metadata because it appears unimportant. Unknown input members still fail
   schema validation even when signed.
2. Sort every object's keys by ascending Unicode code point. Preserve array order.
3. Use `,` between entries and `:` between key and value, with no additional space.
4. Serialize strings as UTF-8 JSON: do not escape non-ASCII characters or `/`;
   use standard JSON escapes for quotes, backslashes and controls (lowercase hex
   for `\u00xx`, and short `\b`, `\t`, `\n`, `\f`, `\r` escapes). Do not perform NFC/NFD
   Unicode normalization. There is no BOM or final newline.
5. Duplicate object keys, invalid UTF-8, nonfinite values and floating-point JSON
   numbers are rejected at authentication. V2 image fields are strings. JSON
   integers, booleans and null use their ordinary compact JSON spelling, but may
   still be rejected by the request schema.

Equivalent input whitespace, object order and Unicode escape spelling therefore
produce the same digest. This is an explicitly restricted canonical JSON profile,
**not a claim of RFC 8785/JCS compliance**. In Python, for valid supported values,
`json.dumps(value, sort_keys=True, ensure_ascii=False, separators=(",", ":"), allow_nan=False).encode("utf-8")`
implements the required serialization. The maximum wire body is 65,536 bytes.

### Canonical request and HMAC

Form the following **seven lines in precisely this order**, separated by one LF
(`0x0a`), **with no trailing LF**:

```text
WG-HMAC-SHA256
UPPERCASE_HTTP_METHOD
ENCODED_ABSOLUTE_PATH
KEY_ID
TIMESTAMP
NONCE
LOWERCASE_SHA256_HEX_OF_CANONICAL_BODY
```

The path is the exact ASCII percent-encoded path sent on the wire, beginning with
`/`. It excludes scheme, hostname, port and fragment. Do not URL-decode or normalize
it. Current V2 routes accept **no query parameters**; any query is rejected. The
standard routes use ASCII paths (`/v2/jobs` and `/v2/jobs/{opaque-id}`).

Compute HMAC-SHA256 using the **UTF-8 bytes of the provisioned secret** as key and
UTF-8 bytes of the seven-line canonical request as message. Do not base64-decode
the secret, hash the secret first, use ordinary SHA256(key+message), or encode the
result as base64. Render the 32 output bytes as lowercase hex and add the exact
`WG-HMAC-SHA256 ` prefix to the Authorization header. Content-Type is not a signed
line, but POST should send `application/json` for normal request parsing.

### Authentication failure, freshness and replay

Missing, invalid, stale, wrong-key, wrong-signature and replayed requests all
return **401** with exactly `{"error":{"code":"authentication_failed"}}`.
The response deliberately does not reveal the failed check, expected signature,
canonical request, key or header names. All `/v2/` routes require authentication
before schema validation or route dispatch. `/health` remains unauthenticated and
retired `POST /v1/recognize` still returns 410. An unsigned request with malformed
input does not receive field-by-field validation assistance.

A successfully authenticated request consumes its `(key ID, nonce)` **even if**
subsequent schema validation returns 422, the job is unknown, or application
capacity returns 503. The server retains the nonce until its signed timestamp
plus 60 seconds. Failed authentication does not reserve a nonce. The replay cache
has 50,000 entries; exhausted authentication capacity returns 503 with
`capacity_exceeded` before reserving a new nonce. Capacity also applies to job storage.

Every poll must use its own current timestamp, fresh nonce, current route and
empty-body digest. Do not automatically retry 401: surface an authentication
failure. For a deliberate retry after another transport failure, recompute the
signature with a new nonce and stay within the same overall deadline. A fresh
nonce does not make POST idempotent; resubmission may create another job.

### Protocol conformance vector (public example credentials only)

The following fixed values are **public test data**, not service credentials.
The timestamp is intentionally historical: this vector validates local signing,
not a live request, and will fail the freshness check if sent today. Independent
implementations can compare bytes and hashes before testing their issued key.

<!-- signing-vector -->
```json
{
  "method": "POST",
  "path": "/v2/jobs",
  "key_id": "example-client",
  "timestamp": "1700000000",
  "nonce": "00112233445566778899aabbccddeeff",
  "example_secret": "public-example-key-not-a-real-credential",
  "wire_body": "{ \"input\": { \"image\": { \"uri\": \"https://images.example.test/sample/你好.png\", \"kind\": \"url\" } } }",
  "canonical_body": "{\"input\":{\"image\":{\"kind\":\"url\",\"uri\":\"https://images.example.test/sample/你好.png\"}}}",
  "body_sha256": "5b0eb0075d1a01360e43ddeacf76de2ca5751c056f426482ab26cf28b93cd4ca",
  "canonical_request": "WG-HMAC-SHA256\nPOST\n/v2/jobs\nexample-client\n1700000000\n00112233445566778899aabbccddeeff\n5b0eb0075d1a01360e43ddeacf76de2ca5751c056f426482ab26cf28b93cd4ca",
  "signature": "bd403fc5cdbb8a66f372efec6b4b351981ee2009a5ecdbf3843d001154caee50"
}
```

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
# Compute POST_AUTH, KEY_ID, TS and NONCE using the signing specification above.
curl -sS -X POST "$BASE/v2/jobs" \
  -H "Authorization: $POST_AUTH" -H "X-WaveGlyph-Key-Id: $KEY_ID" \
  -H "X-WaveGlyph-Timestamp: $TS" -H "X-WaveGlyph-Nonce: $NONCE" -H 'Content-Type: application/json' \
  -d '{"input":{"image":{"kind":"url","uri":"https://images.example.test/receipt.png"}}}'
# Read the returned job.id, wait poll_after_ms milliseconds, then:
# Recompute GET_AUTH with the returned path, empty body, current TS and a NEW_NONCE.
curl -sS "$BASE/v2/jobs/REPLACE_WITH_RETURNED_ID" \
  -H "Authorization: $GET_AUTH" -H "X-WaveGlyph-Key-Id: $KEY_ID" \
  -H "X-WaveGlyph-Timestamp: $TS" -H "X-WaveGlyph-Nonce: $NEW_NONCE"
```

Repeat GET only for a nonterminal state. This example illustrates the public API;
it is not a complete integration implementation. Business functions can retain
synchronous return semantics by waiting internally for a terminal result.
