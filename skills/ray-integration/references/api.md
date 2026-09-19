# Ray API: condensed contract

Base URL `https://ray-api.gege.mn`, header `Authorization: Bearer $RAY_API_KEY` on every call.
This is a map, not the full docs. For exact rules, fetch `https://ray.gege.mn/docs/<slug>.md` (or
the `read_docs` MCP tool). The schemas are in `https://ray-api.gege.mn/openapi.json`.

## Contents
- Endpoints
- Channels, template kinds, content, recipients
- POST /send
- Templates
- Status, feed, clicks
- Delivery webhooks
- Errors and limits

## Endpoints

| Method | Path | Scope | Purpose (docs slug) |
|---|---|---|---|
| GET | `/me` | read | `{ tenantId, apiKeyId, scopes }` (`authentication`) |
| GET | `/billing/subscription` | read | `{ plan, subscription, usage: { monthlyQuota, used, remaining } }` (`rate-limits`) |
| GET | `/channels` | read | `{ channels: [{ id, name, kind, templateKind, recipientSchema }] }` |
| POST | `/send` | write | Send. **202** `{ sendId }` (`sending`, `idempotency`) |
| GET | `/sends/{id}` | read | Per-row delivery status, `?limit=1..200&cursor=` (`status-and-feeds`) |
| GET | `/templates` | read | List, `?folder=` (exact), `?includeArchived=true` (`templates`) |
| POST | `/templates` | write | Create. **201** `{ id, requiredParams }` |
| GET | `/templates/{id}` | read | Template + `published` and `draft` versions |
| PATCH | `/templates/{id}` | write | Write the draft (full body, see below) |
| POST | `/templates/{id}/publish` | write | Promote draft. `{ id, versionId, versionNumber }` |
| POST | `/templates/{id}/archive` · `/unarchive` | write | `{ id, archived }` |
| POST | `/templates/{id}/test-send` | write | Send the **published** version as a test. **202** `{ sendId }` (`scheduling-and-test-sends`) |
| GET | `/notifications` | read | End-user feed, `externalUserId` required (`status-and-feeds`) |
| GET | `/clicks` | read | `?sendId=` and/or `?campaignId=` gives `{ totalClicks, byUrl: [{ url, clicks }] }` (`click-tracking`) |
| GET, POST | `/tenant-webhooks` | read / write | List (`?includeArchived=true`) / create (`webhooks`) |
| GET, PATCH, DELETE | `/tenant-webhooks/{id}` | read / write | Get / update or rotate / archive |

No API key needed: `GET /healthz`, `GET /openapi.json`, `GET /docs` (API reference UI),
`GET /c/{token}` (click redirect), and provider callbacks under `/webhooks/...`.

## Channels, template kinds, content, recipients

| Channel `kind` | Template `channelKind` | `content` | `recipient` |
|---|---|---|---|
| `ses_email`, `smtp_email` | `email_html` | `subject` (1-998), `bodyHtml`, `bodyText` | `{ email, name?, cc?, bcc?, attachments? }` |
| `fcm_push` | `fcm_basic` | `title` (1-200), `body` (1-2000), `imageUrl?`, `data?` (string to string) | `{ deviceToken }` **or** `{ topic }` |
| `slack_webhook` | `slack_text` | `text` (1-40000, mrkdwn) | `{}` |
| `discord_webhook` | `discord_text` | `content` (1-2000, Discord markdown; mentions disabled) | `{}` |
| `telegram_bot` | `telegram_text` | `text` (1-4096, Telegram HTML: `b i u s a code pre`), `disableLinkPreview?` | `{ chatId }` (numeric string or `@channel`) |
| `twilio_sms`, `sendsms_mn` | `sms_text` | `text` (1-1600, plain text, no escaping) | `{ phoneNumber }`: Twilio E.164 (`"+97699112233"`); sendsms.mn 8 Mongolian digits (`"99112233"`, `+976` stripped) |
| `generic_webhook` | `webhook_json` | `title` (1-200), `body` (1-4000), `data?` (string to string) | `{}` |

- An `email_html` template works on both SES and SMTP, and an `sms_text` template works on both
  Twilio and sendsms.mn. The `email_html` `content.source` field is
  optional (`"raw"`). The API rejects `"designed"` and `mailyJson`, because the visual designer is
  dashboard-only.
- Email `cc`/`bcc`: max 50 each. `attachments`: max 20,
  `{ filename, content (base64), contentType?, disposition?: "attachment"|"inline", contentId? }`,
  about 10 MiB each and about 25 MiB total. This works on SES and SMTP.
- Escaping of param values: `bodyHtml` and Telegram `text` are HTML-escaped. Slack escapes
  `& < >`. Discord backslash-escapes markdown. Email `subject` (no line breaks allowed), `bodyText`,
  FCM, SMS and webhook fields are not escaped. `logTitle`/`logDescription` are never escaped, so render
  them as text.
- SMS length is checked **after rendering**, as a 400 from `/send` and test-send. Twilio: ≤1600
  (billed per ~160 GSM-7 / ~70 UCS-2 segment). sendsms.mn sends exactly one SMS: ≤159 chars if
  every character is GSM-7 (plain Latin), ≤69 if any is not (Cyrillic, emoji: UCS-2). Keep
  Mongolian templates short and bound param lengths. A Twilio recipient without a country code is
  a 400. Credentials (Twilio Account SID + Auth Token + sender number or Messaging Service SID;
  sendsms.mn API key + token) are set in the dashboard only.
- Only SES populates the suppression list (hard bounces and complaints via SNS). Suppressed rows
  return 202 but are not sent or billed. There is no SMS suppression list (Twilio itself blocks numbers
  that replied STOP).

## POST /send

```jsonc
{
  "channelConfigId": "<uuid>",           // single and fan-out modes
  "templateId": "<uuid>",                // XOR content: published, kind must match channel
  "content": { },                        // XOR templateId: inline, channel-shaped
  "params": { },                         // any JSON; depth <= 16, <= 10,000 nodes
  "logTitle": "...", "logDescription": "...", // inline content only (max 500 / 2000)
  "recipient": { },                      // single mode
  "targets": [{ "recipient": { }, "externalUserId": "u1" }], // fan-out, 1-1000
  "deliveries": [{ "channelConfigId": "...", "templateId": "...", "recipient": { }, "params": { } }], // 1-10
  "externalUserId": "user_123",          // 1-256 chars
  "showInFeed": true,                    // feed entry per recipient with an externalUserId; not with deliveries
  "feed": { "title": "...", "description": "..." }, // explicit feed entry / override
  "notBefore": "2026-09-20T09:00:00+08:00", // ISO 8601, Z or offset; past = now
  "priority": "high",                    // "high" (default) | "low" for bulk
  "trackClicks": false,                  // email only, Starter plan+ (ignored on Free)
  "campaignId": "spring-sale"            // groups click stats, 1-256 chars
}
```

- Header `Idempotency-Key: <event-derived string>`. Stored 24h with a body hash (key order
  doesn't matter). Same key and body returns the original 202. Different body returns 409. A 409
  "in progress" means retry shortly. 4xx responses aren't stored.
- Modes: single, fan-out, multi-channel, feed-only (only `feed` + `externalUserId` + optional
  `notBefore`). `deliveries[]` can't be combined with top-level `channelConfigId`, `templateId`,
  `content`, `recipient`, `targets`, `logTitle`, `logDescription` or `showInFeed`.
- With `feed` or `showInFeed`, entries need an `externalUserId`: top-level for single and
  `deliveries[]`, per target for `targets[]`. `feed` with no `externalUserId` at all is a 400.
  Multi-channel and feed-only sends create exactly one entry. Entries exist from acceptance (or
  `notBefore`), are independent of delivery outcome, and are never created by test sends.
- With inline content, every top-level variable in `content`/`logTitle`/`logDescription` needs a
  value (there are no `paramOverrides`).
- Fan-out validates every target first, so one bad recipient means 400 and nothing is sent. A
  fan-out of 2 or more targets fires only `send.completed` webhooks.
- Scheduled sends render at request time and can't be cancelled via the API.
- Quota: each delivery row and each feed entry counts as one notification. Suppressed rows, test
  sends, replays and failed requests don't count. Free plan: hard cap of 25,000 per month, then
  402. Paid plans have a soft cap. Headers: `RayQuota-Limit/Used/Remaining/Status`.

## Templates

`POST /templates` body: `name` (1-100, unique per workspace **including archived**, otherwise
409), `folder` (≤200, default `""`), `channelKind` (fixed forever), `content`, `logTitle` (1-500,
required), `logDescription` (1-2000, required), `paramOverrides?`
(`{ "<param>": { optional?, description? } }`), `publish` (default `false`).

- `PATCH /templates/{id}` takes the **same full body**. `name` and `channelKind` are required by
  the schema, but `name`/`folder` aren't changed and `channelKind` must match. It overwrites the
  draft or creates the next draft version. `publish: true` publishes in the same call. It returns
  `{ id, draftVersionId, requiredParams }`.
- `requiredParams` (from the create/patch response) is authoritative. Only top-level
  interpolations count, across `content`, `logTitle` and `logDescription`. `{{user.name}}` requires
  `user`. Section names and names used inside sections don't count.
- Syntax: `{{x}}`, `{{a.b}}`, `{{#s}}...{{/s}}` (array loops, truthy renders once),
  `{{^s}}...{{/s}}`, `{{.}}`. Unsupported: `{{{x}}}`, `{{& x}}`, partials, comments, helpers.
  Unknown `{{...}}` stays literal. An unbalanced section is a 400 on create/update. Interpolating
  an object or array is a 400 at send time.
- A send renders the **published** version. A never-published template gives 404 "template not
  found or not published". Archived templates can't be sent, edited or published.
- `test-send` body: `{ channelConfigId, recipient, params }`. It uses the published version, is
  marked `isTest`, creates no feed entry or webhook, doesn't use monthly quota, and ignores
  `Idempotency-Key`. Daily allowance: Free 50, Starter 250, Pro/Scale 500, then 402. To render an
  **unpublished draft**, make a normal `/send` with the draft's inline `content` to your own
  address.

## Status, feed, clicks

- `GET /sends/{id}` returns `{ sendId, createdAt, aggregate: { total, <status>: n }, rows: [...], nextCursor }`.
  Row statuses: `pending`, `claimed`, `failed_retryable` (not final), and `delivered`,
  `failed_terminal`, `suppressed` (final). `providerError` is `{ name, message }` (provider) or
  `{ reason, details }` (pipeline). Content is redacted after 30 days.
- `GET /notifications?externalUserId=...&limit=&cursor=&templateId=&channelConfigId=&after=&before=`
  returns `{ notifications: [{ id, sendId, channelConfigId, templateId, externalUserId, logTitle, logDescription, status, providerMessageId, createdAt, dispatchedAt }], nextCursor }`.
  The feed is read-only with no read/unread state. `status` is always `"delivered"`.
  Serve it through your own backend.
- `GET /clicks` requires `sendId` or `campaignId`. Counts are totals including scanner clicks, and
  there is no open tracking. Only absolute `http(s)` links in `bodyHtml` are rewritten, and
  `<a data-ray-no-track>` opts a link out.

## Delivery webhooks (Pro plan+; lower plans get 403 `forbidden`)

- Create: `{ name, url (https, public), events: ["notification.delivered" | "notification.failed_terminal" | "send.completed"] }`.
  This returns 201 with `secret` **once**. `PATCH` accepts `name`, `url`, `events`, `enabled`,
  `rotateSecret: true` (returns a new `secret`, and the old one stops immediately). `DELETE`
  archives.
- Payload fields are snake_case: `event`, `occurred_at`, `send_id`, `notification_log_id`,
  `channel_config_id`, `template_id`, `external_user_id`, `provider_message_id`,
  `provider_error`, `is_test`. `send.completed` has `channel_config_ids`, `recipient_count` and
  `totals`.
- Signature: `X-Ray-Signature: t=<unix>,v1=<hex HMAC-SHA256("<t>.<raw body>", secret)>`. Use a
  5-min tolerance and constant-time compare. Use `verifyWebhookSignature` from `@gege-mn/ray`.
  Respond 2xx within 10 s. Delivery is retried up to 10 times, then disabled. Delivery is
  at-least-once and unordered.
- The `generic_webhook` **channel** signs differently (only if a secret is configured):
  `X-Ray-Timestamp: <ISO>` and `X-Ray-Signature: sha256=<hex HMAC("<timestamp>.<raw body>")>`.
  The envelope is `{ id, title, body, data?, sentAt }`. Deduplicate on `id`.

## Errors and limits

Body: `{ "error": "<code>", "message": "<text>" }`. Branch on status + `error`.

| Status | `error` | When | Retry |
|---|---|---|---|
| 400 | `validation_error` | Bad body, recipient/content shape, missing params, kind mismatch, missing `write` scope | No |
| 401 | `unauthorized` | Missing, invalid or revoked key | No |
| 402 | `quota_exceeded` | Free monthly quota or daily test-send allowance (`limit`, `used`) | After upgrade or reset |
| 403 | `forbidden` | Plan lacks the feature (delivery webhooks) | After upgrade |
| 404 | `not_found` | Missing, archived, unpublished or other-workspace resource | No |
| 409 | `conflict` | Idempotency conflict, or template name taken | Only "in progress" |
| 413 | `payload_too_large` | `/send` > 45 MiB, others > 1 MiB | No |
| 429 | `rate_limit_exceeded` | Rate limited (`retry_after_seconds`, `Retry-After`) | Yes |
| 500 | `internal_server_error` | Server error (no message) | Yes, same Idempotency-Key |

API rate limits apply per IP and per workspace. Burst and sustained rates by plan: Free 20/10 rps,
Starter 120/60, Pro 240/120, Scale 600/300. One `/send` with 1000 targets is one request. Provider
pacing (SES/SMTP 840/min default, Telegram 1500/min, Slack 1/s per webhook, Twilio and sendsms.mn
300/min default) queues the work and never rejects it.

Provider errors on delivery rows: `providerError.name` is e.g. `TwilioError` or `SendsmsMnError`.
For SMS, provider 4xx except 429 is `failed_terminal` (bad number, country not enabled, STOP,
wrong credentials, no sendsms.mn balance); 429 and 5xx are retried. sendsms.mn timeouts are not
retried, because the SMS may already have been sent.
