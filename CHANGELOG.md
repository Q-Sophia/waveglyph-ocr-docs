# WaveGlyph OCR changelog

## Documentation revision 2026-10-16-r2

- Completed V2 authentication specification: HMAC-SHA256 request binding, canonical
  JSON, UTC timestamp, nonce replay protection and independent conformance vector.
- Clarified that every status GET is authenticated and 401 does not reveal which
  authentication check failed. Public test vector credentials are illustrative only.

## 2.0.0 — 2026-10-16 (synthetic documentation release)

- Replaced synchronous recognition with asynchronous job submission and polling.
- Added nested typed image input; removed the flat `image_url` request.
- Added queued, processing, succeeded and failed states.
- Replaced top-level text with ordered document blocks.
- Retired V1 recognition with HTTP 410. No automatic compatibility fallback.

## 1.0.0

Synchronous `POST /v1/recognize` accepts `image_url` and returns `text`.
This is a synthetic version history, not a record of a real production incident.
