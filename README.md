# Contact Form 7 GTM Form Tracker

Confirmation-based form tracking for Contact Form 7 in Google Tag Manager.

This standalone tracker pushes dataLayer events when Contact Form 7 submissions actually succeed or fail. It is built for GTM, GA4, server-side GTM, Meta CAPI and Google Enhanced Conversions workflows where button-click tracking is too noisy.

Docs and payload examples: **[iyrexmedia.com/gtm-form-listeners](https://iyrexmedia.com/gtm-form-listeners/)**

## Why this exists

CF7 recipes usually listen for `wpcf7mailsent` and stop there. That event fires, but most recipes then send nothing useful with it, miss the spam and mail-failure outcomes, and count every redirect and refresh. This listener uses CF7's own events, reads field values from the event detail, and separates real leads from validation errors, spam blocks and mail failures.

## Files

| File | Use case |
| --- | --- |
| `cf7-form-tracker.gtm.html` | GTM Custom HTML paste-ready version. Includes the `<script>` wrapper and a production config block at the top. |
| `cf7-form-tracker.js` | Raw JavaScript for self-hosting. Its config block only applies when `window.EC_FORM_CONFIG` is not already set. Do not wrap it in another `<script>` tag when loading by URL. |

## Events

```text
cf7_form_success
cf7_form_failed
ec_form_interaction
ec_tracker_initialized
```

### Success event

`cf7_form_success` fires on the first of these:

- The `wpcf7mailsent` DOM event (CF7's own "mail sent" signal)
- The form's `data-status` changing to `sent`
- A same-domain thank-you page, matched to data stored at submit

### Failure event

`cf7_form_failed` fires on `wpcf7invalid` (validation), `wpcf7spam` (spam blocked) and `wpcf7mailfailed` (mail server error), each with its own `ec_detection_method`, after the visitor has interacted with the form.

By default (`failedPayload: 'minimal'`) a failed event carries no contact data, hashes or field values. It keeps the form metadata, `ec_detection_method`, `ec_failed_fields` (field names only) and `ec_failed_field_count`.

### Diagnostic events

`ec_form_interaction` fires once per form after the first real input. It carries metadata only, no PII.

`ec_tracker_initialized` fires once per page load so you can confirm the script loaded before a test submission.

## Payload example

Default settings (`piiMode: 'plain'`, `payload: 'lean'`):

```js
{
  event: "cf7_form_success",
  ec_name: "Jane Doe",
  ec_first_name: "Jane",
  ec_last_name: "Doe",
  ec_email: "jane@example.com",
  ec_phone: "+15551234567",
  form_cta: "Send",
  form_platform: "cf7",
  form_position: "top",
  form_index: 1,
  form_id: "5",
  ec_event_id: "uuid-v4",
  ec_detection_method: "confirmation_event",
  ec_submit_attempt: 1,
  ec_time_to_submit_s: 42,
  ec_fast_submit: false,
  ec_first_source: "google",
  ec_first_medium: "cpc",
  ec_last_source: "google",
  ec_last_medium: "cpc",
  ec_click_id: "Cj0K...",
  ec_click_id_type: "gclid"
}
```

- `event` is always the first key.
- Empty values are left out. `form_url` is only sent when it differs from the current page (thank-you page recall). `ec_tracker_version` and `ec_dedupe_basis` are only sent with `debug: true`.
- `payload: 'full'` restores the v3.3.0 key set, including the `ec_attribution` object, `ec_first_landing`, `ec_first_referrer`, `ec_first_ts` and `ec_time_to_submit_ms`.
- With `rawFields: true`, `ec_raw_fields` is added with every captured field and its `_bucket` classification. Pass an array instead (`rawFields: ['service', 'location']`) to send only those fields.
- With `hashPii: true`, `ec_email_sha256` and `ec_phone_sha256` are added.
- If a CF7 form has a hidden field named `form_name` (`[hidden form_name "contact-us"]`), its value is sent as `form_name`. A stable label per form purpose keeps GTM triggers working when a form is rebuilt and gets a new ID.
- Before each push, keys left in the dataLayer by an earlier push that this payload does not set are reset to `undefined`, so a GTM variable never reads another form's values.

When PII consent is denied, contact keys, hashes and raw fields are removed and `ec_pii_redacted: true` is added.

## Privacy modes

`piiMode` controls contact data in the payload:

| Mode | Payload | Browser storage |
| --- | --- | --- |
| `plain` (default) | Normalized name, email, phone and address keys, as in v3.3.0 | Full form copy in localStorage and sessionStorage for thank-you page recall |
| `hashed` | No plain contact keys. Email and phone only as `ec_email_sha256` / `ec_phone_sha256` | sessionStorage only, with choice fields, allowlisted fields and the hashes. No plain contact data |
| `none` | No contact keys and no hashes | sessionStorage only, no contact data |

In `hashed` mode the tracker waits up to `hashWaitMs` (300 ms) for the hash before pushing, so an instant submit still gets its hashes. In `hashed` and `none` mode without a `rawFields` allowlist, `ec_raw_fields` keeps only select, radio and checkbox values.

Recommended for GA4-connected setups that do not need plain contact data in the browser:

```js
window.EC_FORM_CONFIG = {
  piiMode: 'hashed',
  rawFields: ['service', 'location'],
  strictSuccess: true
};
```

## Quick start in GTM

1. Open Google Tag Manager and create a **Custom HTML** tag.
2. Paste the full contents of `cf7-form-tracker.gtm.html`.
3. Trigger it on **All Pages (Page View)**, or **Consent Initialization** if your consent setup needs the tracker early.
4. Create a **Custom Event** trigger with regex `^cf7_form_(success|failed)$`, or event name `cf7_form_success` for leads only.
5. Create Data Layer Variables for the keys you need (`form_id`, `ec_detection_method`, `ec_event_id`, `ec_email`, `ec_phone`, ...).
6. Attach your GA4, Google Ads, Meta or server-side GTM tags to the trigger.
7. Preview, submit a real test form, confirm one `cf7_form_success`, then publish.

Retire any old click or thank-you trigger for the same form, otherwise each lead is counted twice.

## Platform notes

- Field keys come from CF7's `data-name` wrappers (`your-name`, `your-email`), so they match your form tags.
- CF7 internal fields (`_wpcf7*`), the honeypot, file uploads and reCAPTCHA tokens are excluded.
- Uses shared storage key `ec_form_data`. Do not run it on the same page as the universal tracker.

## Configuration

Edit the config block at the top of the GTM file, or define `window.EC_FORM_CONFIG` before the raw `.js` loads.

| Option | Default | What it does |
| --- | --- | --- |
| `debug` | `false` | `[EC]` console logs. Testing only |
| `piiMode` | `'plain'` | `'plain'`, `'hashed'` or `'none'`. See Privacy modes |
| `hashPii` | `false` | Add SHA-256 hashes of email and phone (HTTPS only). Forced on by `piiMode: 'hashed'` |
| `hashWaitMs` | `300` | Hashed mode: how long to wait for a hash before pushing |
| `rawFields` | `false` | `true` adds `ec_raw_fields` with every field. An array adds only the named fields. Keep plain values out of GA4 |
| `payload` | `'lean'` | `'lean'` sends only keys with a use. `'full'` sends the v3.3.0 key set |
| `failedPayload` | `'minimal'` | `'minimal'` keeps contact data, hashes and field values off failed events. `'full'` restores v3.3.0 behavior |
| `formNameField` | `'form_name'` | Hidden CF7 field whose value is sent as `form_name` |
| `dedupe` | `true` | Suppress a repeat success for the same form and person within 5 minutes |
| `strictSuccess` | `false` | Drop inferred successes (`pagehide`, `form_removed`) |
| `defaultCountryCode` | `"1"` | Phone fallback when the form has no country field |
| `countryCodeField` | `""` | Exact field name holding an ISO2 or dial code |
| `redactFields` | `[]` | Extra field names (strings or RegExps) to strip |
| `extraThankYouPatterns` | `[]` | Extra thank-you URL substrings |
| `queueOnDenied` | `true` | Queue events while consent is denied and replay them on grant |
| `submitWindowMs` | `15000` | How long after submit a success signal is accepted |
| `ttlMs` | `600000` | Freshness window for thank-you recall |

Password, card, SSN, IBAN, API key, secret, token, CSRF, nonce and similar field names are always redacted.

To reproduce the v3.3.0 payload exactly, set `payload: 'full'` and `failedPayload: 'full'` and keep `piiMode: 'plain'`. Only the safety fixes listed in `CHANGELOG.md` remain.

## Consent

Auto-detected: Google Consent Mode v2, OneTrust, Cookiebot.

For anything else, define this before the tracker loads:

```html
<script>
window.EC_FORM_CONSENT_CHECK = function () {
  return { analytics: true, pii: false };
};
</script>
```

| Consent state | Behavior |
| --- | --- |
| `analytics: false` | Nothing fires and nothing is stored. Events queue and replay on grant |
| `pii: false` | Events fire without contact keys, hashes or raw fields, and with `ec_pii_redacted: true` |

If no CMP or override is found, the tracker assumes full consent. Match that to your own legal requirements.

## PII warning

`ec_name`, `ec_email`, `ec_phone`, the address keys and `ec_raw_fields` are for Enhanced Conversions, Meta CAPI and first-party CRM workflows. **Do not send them to GA4.** GA4's terms prohibit PII in event parameters. Send `form_id`, `form_name`, `form_platform` and `ec_detection_method` to GA4 instead.

If nothing on the site needs plain contact data, use `piiMode: 'hashed'` (or `'none'`) so it never reaches the dataLayer or browser storage at all.

## Detection methods

Every success and failure event includes `ec_detection_method`.

Success:

```text
confirmation_event
confirmation_class
stored_recall
pagehide
```

Failure:

```text
validation_message
spam_blocked
mail_failed
error_class
```

## Debugging

```js
window._ecCf7Tracker.status()
```

Returns version, consent state, forms attached, the last event and the active config.

## Known limitations

- CF7 add-ons that replace the AJAX submit can change which events fire. Test on the live page.
- A CF7 mail failure still means the visitor submitted. Decide whether you count `mail_failed` as a lead.
- Cross-domain redirects cannot use same-domain recall.
- Bot submissions that pass the platform's own validation still count as success. Use `ec_fast_submit` to filter.
- Phone normalization is best-effort.

## Version

Current version: `3.4.0-cf7.1`. See `CHANGELOG.md`.

## License

MIT. See `LICENSE`.

## Author

Iyke Abel  
LinkedIn: https://www.linkedin.com/in/iykeabel/  
Email: abeliyke05@gmail.com
