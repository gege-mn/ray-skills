---
name: ray-integration
description: Integrates apps with the Ray notification API (ray.gege.mn), which sends transactional email (Amazon SES or SMTP), mobile push (FCM), SMS (Twilio or sendsms.mn), Slack, Discord, Telegram and HTTPS webhook notifications through one REST API, a hosted MCP server and the @gege-mn/ray TypeScript SDK. Use when a project sends or should send notifications or transactional email through Ray, when adding or debugging send code, idempotent sends, fan-out or multi-channel sends, scheduled sends, in-app notification feeds, delivery status, click tracking or signed delivery webhooks, when creating or publishing Ray templates, or when migrating templates and send calls from Resend, SendGrid, Postmark, Mailgun, SES, Twilio, Novu, Knock, Courier, OneSignal or Firebase into Ray. Also use when RAY_API_KEY, ray-api.gege.mn or @gege-mn/ray appears in a project. Not for the Ray distributed computing framework (ray.io, Ray Serve, Ray Tune).
---

# Ray integration

Ray (https://ray.gege.mn) is a multi-tenant notification delivery API. It is not the Ray
distributed computing framework. Ray relays through the workspace's own provider credentials over
9 channels: Amazon SES email, SMTP email, FCM push, Slack, Discord, Telegram, Twilio SMS, sendsms.mn SMS
(Mongolian numbers) and generic HTTPS webhook.

- REST base URL: `https://ray-api.gege.mn` (no `/v1`). Auth on every request:
  `Authorization: Bearer $RAY_API_KEY` (`ck_live_...`). Read the key from the environment. Never
  hardcode it or ship it to browser or mobile code.
- Scopes: `read` covers every `GET`. `write` covers `POST /send`, template writes and test-sends,
  and webhook writes. A missing `write` scope returns **400** `validation_error` "write scope
  required", not 403.
- Provider credentials (channel configs) are created only in the dashboard. Code discovers them
  with `GET /channels` and never creates them.

## Pick the integration path

1. **Doing it yourself now** (send a test, create or publish templates, check a send, manage
   webhooks): use the **Ray MCP server's tools** when they are connected. In Claude Code they
   appear as `mcp__ray__<tool>`, or `mcp__plugin_ray_ray__<tool>` when installed via the plugin.
   Tools: `whoami`, `get_usage`, `list_channels`, `send_notification` (takes `idempotencyKey`),
   `get_send_status`, `list_templates`, `get_template`, `create_template`,
   `update_template_draft`, `publish_template`, `archive_template`, `unarchive_template`,
   `test_send_template`, `list_feed_notifications`, `get_click_stats`, `list_webhooks`,
   `get_webhook`, `create_webhook`, `update_webhook`, `delete_webhook`, `read_docs`.
   If they are not connected, fall back to `curl` and suggest setup: see
   `references/mcp-and-sdk.md`. Claude Code:
   `claude mcp add --transport http ray https://ray-api.gege.mn/mcp --header "Authorization: Bearer $RAY_API_KEY"`
2. **Writing app code in JS/TS** (Node 18+, Bun, Deno, edge): use the SDK `@gege-mn/ray`.
3. **Other languages, or the SDK can't be installed**: use plain HTTP (`fetch`, `requests`, ...)
   with the same JSON bodies.

## Workflow

1. **Preflight**: `whoami` / `ray.me()` / `GET /me` returns `{ tenantId, apiKeyId, scopes }`.
   Confirm `write` is present before sending or editing templates.
2. **Discover channels**: `list_channels` / `ray.channels.list()` / `GET /channels` returns
   `{ channels: [{ id, name, kind, templateKind, recipientSchema }] }`. Use `id` as
   `channelConfigId`. `recipientSchema` (JSON Schema) is authoritative for `recipient`. Don't guess.
   Don't hardcode ids in code either: put them in env/config.
3. **Send** with an idempotency key derived from the business event, on every send:

```ts
import { Ray, RayError } from "@gege-mn/ray";

const ray = new Ray(); // reads process.env.RAY_API_KEY

const { sendId } = await ray.send(
  {
    channelConfigId: process.env.RAY_EMAIL_CHANNEL_ID!,
    templateId: process.env.RAY_ORDER_SHIPPED_TEMPLATE_ID!, // or inline `content`
    params: { name: user.name, orderId: order.id },
    recipient: { email: user.email, name: user.name },
    externalUserId: user.id, // optional: labels rows, keys the in-app feed
    showInFeed: true,
  },
  { idempotencyKey: `order-shipped-${order.id}` },
);
// Throws RayError { status, code, message, requestId? } on non-2xx.
```

Raw HTTP fallback: `POST https://ray-api.gege.mn/send` with headers `Authorization`,
`Content-Type: application/json` and `Idempotency-Key: order-shipped-<orderId>`. Expect **202**
`{ sendId }`. Anything else: read `{ error, message }` and branch on `status` + `error`, never on
`message`.

4. **Confirm delivery**: 202 means queued, not delivered. Check `get_send_status` /
   `ray.sends.get(sendId)` / `GET /sends/{id}` (rows go `pending` to `delivered`, `failed_terminal`
   or `suppressed`). In production, prefer delivery webhooks.

## Rules that are easy to get wrong

- **Idempotency**: pass an `Idempotency-Key` on every `POST /send`. Build it from the event
  (`password-reset-<userId>-<tokenId>`), not a fresh UUID per retry. Replays within 24h return the
  original `sendId`. The same key with a different body returns 409. Only `/send` honors it.
  Retry only on 429 (wait `Retry-After`), 5xx, network errors, and a 409 "already in progress",
  always with the same key.
- **Exactly one content source** per delivery: `templateId` (a *published* template of the
  channel's kind) or inline `content` (same shape as a template's content).
- **Exactly one mode**: `channelConfigId` + `recipient` (single), `channelConfigId` +
  `targets[]` (1-1000, body rendered once with one `params` set, so no per-recipient
  personalization), `deliveries[]` (1-10 channels for one person, each with its own
  `channelConfigId`/content/`recipient`; a delivery's `params` *replaces* top-level `params`), or
  feed-only (`feed.title` + `externalUserId`, no channel).
- **Ray keeps no recipient registry**: pass the email, FCM `deviceToken`/`topic`, Telegram
  `chatId` or SMS `phoneNumber` on every send. `externalUserId` only labels rows and keys the feed. It never picks who
  receives. Slack, Discord and webhook recipients are `{}`.
- **SMS**: `twilio_sms` and `sendsms_mn` share the `sms_text` kind (`{ text }`, plain text, max
  1600, params inserted verbatim). Twilio wants E.164 (`{ phoneNumber: "+97699112233" }`, no
  country code is a 400); sendsms.mn wants 8 Mongolian digits (`"99112233"`, `+976` is stripped).
  sendsms.mn takes one SMS per request, so Ray splits long text itself at spaces/line breaks into
  parts of ≤159 chars if all GSM-7 (plain Latin), or ≤69 if a part has any other character
  (Cyrillic, emoji); each part is sent in order, billed as one SMS and arrives as a separate
  message. No length 400 on sendsms.mn (only Twilio rejects >1600 rendered). Ray has no SMS
  suppression list.
- **Bulk/marketing**: `priority: "low"` so transactional mail goes first. For isolated
  throughput, use a separate provider credential. For personalized campaigns, send one request per
  recipient with a shared `campaignId`.
- **Templates**: variables are a Mustache subset (`{{x}}`, `{{a.b}}`, `{{#list}}...{{/list}}`,
  `{{^x}}`, `{{.}}`). No `{{{raw}}}`, partials or helpers. Param values are escaped per channel
  (HTML in `bodyHtml` and Telegram; Slack `& < >`; Discord markdown). Email `subject` and
  `bodyText` are not escaped, and a line break in the rendered subject is a 400. Every top-level
  variable is a required param. Edits go to a draft. Sends use the published version only.
- **Delivery webhooks** (Pro plan+): verify `X-Ray-Signature` against the **raw body** before
  parsing. Use `verifyWebhookSignature` from `@gege-mn/ray`:

```ts
import { verifyWebhookSignature } from "@gege-mn/ray";

export async function POST(request: Request) {
  const rawBody = await request.text(); // raw bytes: don't JSON.parse first
  const ok = await verifyWebhookSignature({
    payload: rawBody,
    headers: request.headers, // reads x-ray-signature
    secret: process.env.RAY_WEBHOOK_SECRET!, // or an array during secret rotation
  });
  if (!ok) return new Response("invalid signature", { status: 400 });
  const event = JSON.parse(rawBody); // notification.delivered | notification.failed_terminal | send.completed
  // Deduplicate: notification_log_id (notification.*) or send_id (send.completed). Ack fast.
  return new Response(null, { status: 204 });
}
```

  The scheme is `t=<unix>,v1=<hex HMAC-SHA256 of "<t>.<raw body>">`, keyed with the secret string
  as returned. Reject timestamps older than 5 min. The **generic_webhook channel** is different:
  `X-Ray-Signature: sha256=<hex>` over `"<X-Ray-Timestamp>.<raw body>"`; verify it with
  `verifyChannelWebhookSignature` (same options). See
  `https://ray.gege.mn/docs/channels/webhook.md`.

## Details: fetch, don't guess

This skill is a summary. For exact fields, limits and error messages, read the live docs:
the `read_docs` MCP tool (slug such as `sending`, `templates`, `channels/telegram`), or
`https://ray.gege.mn/docs/<slug>.md` (index: `https://ray.gege.mn/llms.txt`). The OpenAPI 3.1 spec
is `https://ray-api.gege.mn/openapi.json`.

- `references/api.md`: condensed endpoint, channel, content, recipient and error contract.
- `references/mcp-and-sdk.md`: MCP client setup, the tool, SDK and REST mapping, and fallbacks.
- `references/migrate-templates.md`: importing templates and send calls from other providers.
- Designing on-brand email HTML: the `ray-email-design` skill.
