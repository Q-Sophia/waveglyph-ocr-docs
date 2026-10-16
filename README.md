# WaveGlyph OCR — API documentation

WaveGlyph OCR is a **synthetic, deterministic OCR API** for engineering demonstrations.
It is not a commercial OCR vendor and does not claim a real production migration.
The current API contract is **V2 / 2.0.0**. Documentation prepared: **2026-10-16**.
Documentation edition: **2026-10-16**.

If an integration stopped working after upgrading from the synchronous V1 API, see
[WaveGlyph OCR V1 to V2 Migration Guide](docs/migration-v1-to-v2.md).

- [V1 synchronous API](docs/v1-api.md)
- [V2 asynchronous recognition API](docs/v2-api.md)
- [V1 to V2 breaking changes and migration](docs/migration-v1-to-v2.md)
- [Changelog](CHANGELOG.md)

The HTTP service is installed privately by the application operator; only these
API documents are public. Replace `http://localhost:8000` in examples with the
configured service origin. There is no public OCR endpoint. V2 access credentials are provisioned by the local
operator; HMAC request signing is specified in the migration guide.
Documentation revision: **2026-10-16-r2**, adding the complete V2 authentication contract.
