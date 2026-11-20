# Changelog

## 3.4.0-cf7.1 (2026-11-20)

Privacy and data-minimisation release. Every change is config-controlled: `payload: 'full'`, `failedPayload: 'full'` and `piiMode: 'plain'` reproduce the v3.3.0 payload, apart from the safety fixes marked *.

- `event` is the first key in every push
- New `piiMode`: `'plain'` (default), `'hashed'` (email and phone only as SHA-256) or `'none'` (no contact data)
- Hashed mode waits up to `hashWaitMs` (300 ms) for the hash, so an instant submit still carries it
- `rawFields` accepts an array as an allowlist, e.g. `['service', 'location']`
- New `formNameField` (default `'form_name'`): a hidden CF7 field whose value is sent as `form_name`
- New `failedPayload: 'minimal'` (default): failed events carry no contact data, hashes or field values, only field names, count and reason
- New `payload: 'lean'` (default): no `ec_attribution` object, compact attribution keys, no blank keys, no `form_url` when it matches the page, `ec_tracker_version` and `ec_dedupe_basis` only with `debug: true`
- Keys left by an earlier push are reset before the next push, so GTM variables never read another form's values
- In `hashed` and `none` mode the thank-you recovery copy holds no plain contact data and lives in sessionStorage only
- \* API key, secret, access/auth token, CSRF and nonce fields are always redacted
- \* Hidden inputs are never used as name or address values
- \* With PII consent denied, the name and address keys added in v3.3.0 are removed too

## 3.3.0-cf7.1 (2026-10-28)

First public standalone Contact Form 7 release, cut from the universal tracker v3.3.0. The shared core and the Contact Form 7 handler match the universal build.

- `cf7_form_success` and `cf7_form_failed` events, plus `ec_form_interaction` and `ec_tracker_initialized`
- `ec_event_id` and `ec_detection_method` on every event
- Consent gate (Consent Mode v2, OneTrust, Cookiebot, `EC_FORM_CONSENT_CHECK`) with queue and replay
- PII redaction, optional SHA-256 hashing, always-redacted sensitive field names
- 5-minute dedupe ledger
- First and last touch attribution with click IDs
- E.164 phone normalization with `defaultCountryCode` and `countryCodeField`
- `strictSuccess` option, failure diagnostics, SPA route handling
- GTM paste-ready file with production defaults (`debug: false`, `rawFields: false`) and a raw `.js` whose defaults never override a site's own config
