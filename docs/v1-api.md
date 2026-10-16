# WaveGlyph OCR V1 synchronous API

`POST /v1/recognize`, `Content-Type: application/json`:

```json
{"image_url":"https://images.example.test/receipt.png"}
```

HTTP 200:

```json
{"text":"Wave Cafe\nTotal: 18.50"}
```

The URL must use HTTP or HTTPS. Extra request keys are rejected. Invalid inputs or
unreadable fixture images return 422. Clients should enforce an HTTP timeout and
report non-success responses. Known demonstration fixtures are `receipt.png` and
`note.png` at `images.example.test`. V1 performs no outbound image downloads.

On a **V2** deployment this route returns HTTP 410, `{"error":{"code":"api_retired"}}`.
See the migration guide instead of retrying the retired route.
