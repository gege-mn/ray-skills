# Migrating templates and send calls into Ray

Goal: import a project's existing notification templates into Ray, then replace the old provider's
send calls. Ray's own template rules: `https://ray.gege.mn/docs/templates.md` (or
`read_docs` with slug `templates`).

## Contents
- Before you start
- Process
- Syntax conversion (Handlebars, Liquid, Mustachio, merge tags, react-email)
- Non-email steps and workflow features
- Replacing send calls
- Creating templates (MCP, SDK, curl)

## Before you start

- You need a `write`-scoped key in `RAY_API_KEY`. Preflight with `whoami` / `GET /me`: `scopes`
  must include `"write"`, otherwise writes return 400 "write scope required". If the key is unset
  or read-only, stop and ask.
- Call `list_channels` / `GET /channels` to see which channel kinds exist. A template's
  `channelKind` must match a configured channel to be sendable. Channels are created in the
  dashboard, not by you.
- **Don't send production messages or invent recipients.** The only deliveries allowed during a
  migration are render checks to an address the user gives you.

## Process

1. **Inventory.** Find templates in the repo (`emails/`, `templates/`, `mail/`, `*.html`, `*.hbs`,
   `*.liquid`, `*.mjml`, react-email `*.tsx`) and in the old provider. If the templates live only
   in the provider (SendGrid dynamic templates, Postmark, Knock, Novu, Courier), ask the user to
   export them or give you API access. Don't scrape dashboards.
2. **Map and show.** For each template, show `source → { name, channelKind, subject, detected
   variables, conversions applied, anything unsupported }` and **wait for approval**.
3. **Create or update idempotently.** Names are unique per workspace, including archived
   templates. First `list_templates` / `GET /templates?includeArchived=true`. If the name exists,
   `update_template_draft` / `PATCH /templates/{id}` with the **full body** (`name`,
   `channelKind`, `content`, `logTitle`, `logDescription`). Otherwise use `create_template` /
   `POST /templates`. Unarchive first if the match is archived.
4. **Trust Ray's `requiredParams`.** The create/patch response lists the variables Ray detected.
   Diff it against your scan. An unexpected entry usually means a leftover non-Ray `{{ }}` or a
   conversion miss.
5. **Publish only when confirmed.** Use `publish: true` on create/patch, or `publish_template`.
   Sends and test-sends use only the published version.
6. **Render check (optional, on request).** For a published template, use `test_send_template` /
   `POST /templates/{id}/test-send` with `{ channelConfigId, recipient, params }` to the user's
   address. It is marked `isTest` with no feed, no webhooks and no monthly quota. For a draft, use
   a normal `send_notification` with the draft's inline `content` to that address, with an
   idempotency key.
7. **Report** each template's `id`, `requiredParams`, published state, and every guess
   (derived `bodyText`, `logTitle`) or skipped feature.

## Required fields you may have to derive

- `subject` (email): from the provider template, `<title>`, or the filename. It can contain
  variables, but a param with a line break fails the send.
- `bodyText` (email, required string): write a real plain-text version with the same variables
  and URLs spelled out. Don't just strip tags.
- `logTitle` (1-500) and `logDescription` (1-2000) are required on every template. They are the
  in-app feed and log text (e.g. "Password reset requested"), not the subject. They may use
  variables, and those variables become required too.

## Syntax conversion

Ray supports: `{{x}}`, `{{a.b}}`, `{{#x}}...{{/x}}` (loops over arrays, renders once if truthy),
`{{^x}}...{{/x}}` (renders if falsy or empty), `{{.}}` (current item). Nothing else: no helpers,
filters, `else`, `@index`, partials, layouts, or unescaped output. Unknown `{{...}}` is left as
literal text.

| Source | Example | Ray equivalent |
|---|---|---|
| Handlebars (SendGrid dynamic, SES, Mailgun, Novu legacy) | `{{#if vip}}A{{else}}B{{/if}}` | `{{#vip}}A{{/vip}}{{^vip}}B{{/vip}}` (boolean `vip`) |
| Handlebars | `{{#each items}}{{this.name}}{{/each}}` | `{{#items}}{{name}}{{/items}}`; `{{this}}` becomes `{{.}}` |
| Handlebars | `{{#unless x}}...{{/unless}}` | `{{^x}}...{{/x}}` |
| Handlebars | `{{@index}}`, `{{formatDate d}}`, `{{insert name "default=Hi"}}` | Precompute in `params` (`position`, `dateLabel`, `greeting`) |
| Postmark Mustachio | `{{#each items}}`, `{{{@content}}}` layouts | Sections as above; inline the layout HTML into each template |
| Liquid (Knock, Novu, OneSignal) | `{{ data.name \| default: "there" }}` | `{{#name}}{{name}}{{/name}}{{^name}}there{{/name}}`, or default it in code |
| Liquid | `{% if a %}...{% endif %}`, `{% for i in items %}{{ i.x }}{% endfor %}` | `{{#a}}...{{/a}}`, `{{#items}}{{x}}{{/items}}` |
| Liquid filters | `{{ total \| money }}`, `{{ name \| upcase }}` | Format in code, pass the string |
| Courier | `{profile.name}`, `{data.orderId}` | `{{name}}`, `{{orderId}}` (flatten) |
| Mailchimp/Mandrill | `*\|FNAME\|*` | `{{FNAME}}` |
| SendGrid legacy substitutions | `-name-` or `%name%` | `{{name}}` |
| Triple-stash / raw | `{{{html}}}`, `{{& html}}` | Not supported. Move markup into the template and pass data. If HTML must come from code, render it in the app and send inline `content` without params |
| react-email (Resend) | `<Welcome name={name} />` | Render to static HTML with placeholder props (`name="{{name}}"`) via `@react-email/render`, plus the `plainText` option for `bodyText`. Props that drive `if`/`map` logic must become sections instead |
| MJML | `<mj-section>` | Compile to HTML first, then import the output |

Caveats:
- A section on an **array** loops. To guard a list with a heading, pass a separate boolean
  (`hasItems`) or put the heading inside a wrapper object.
- Dotted names like `{{subscriber.firstName}}` require the top-level `subscriber` param. Prefer
  flattening to `{{firstName}}` during migration.
- If a source uses `{{ }}` for something that isn't a send variable (client-side templating, CSS
  frameworks), it becomes a spurious required param or literal text. Flag it and ask.

## Non-email steps and workflow features

Workflow products (Novu, Knock, Courier) bundle channels and orchestration. Map them like this:

| Source concept | In Ray |
|---|---|
| Email step | `email_html` template |
| Push step (FCM/APNs via FCM) | `fcm_basic` template: `title`, `body`, `imageUrl?`, `data?` (strings) |
| Chat step: Slack / Discord / Telegram | `slack_text` / `discord_text` / `telegram_text` template |
| Webhook step | `webhook_json` template |
| In-app / inbox step | No template kind. Use `feed` (or the template's `logTitle`/`logDescription` + `showInFeed`) with `externalUserId`, read via `GET /notifications` |
| SMS step | `sms_text` template: `{ text }` (plain text, max 1600). Sent via a `twilio_sms` or `sendsms_mn` channel (see below) |
| WhatsApp step | **No Ray channel.** Report it as unsupported and leave it on the old provider |
| Multi-channel workflow | One `/send` with `deliveries[]` (up to 10 channels) for one user |
| Delay step | `notBefore` (can't be cancelled; for cancellable reminders, schedule in the app) |
| Digest / batch | Aggregate in the app, then send once using a `{{#items}}` section |
| Subscribers, device tokens, preferences, topics | Ray has **no registry**. Keep contacts, tokens and opt-outs in the app DB and pass `recipient` each send. FCM topics work as `{ topic }` |

### SMS (Twilio, Vonage, Plivo, MessageBird, workflow SMS steps)

- One `sms_text` template works on both `twilio_sms` and `sendsms_mn`. Content is `{ text }`,
  plain text, 1-1600 chars. Params are inserted verbatim (no markup, nothing escaped). Convert
  variables to Ray Mustache as for email; there is no `subject`, but `logTitle`/`logDescription`
  are still required.
- Twilio `client.messages.create({ from, to, body })` becomes a send with `recipient:
  { phoneNumber: to }` (E.164, e.g. `"+97699112233"`; no country code is a 400) and the body as
  inline `content: { text }` or a template. `from` / `messagingServiceSid` live on the channel
  config (dashboard), not the send. Twilio credentials (Account SID, Auth Token) are entered in the
  dashboard, never through the API.
- sendsms.mn recipients are 8 Mongolian digits (`"99112233"`; `+976` is stripped). It sends one SMS:
  the **rendered** text must be ≤159 chars if all GSM-7 (plain Latin), ≤69 if it contains any
  Cyrillic or other non-GSM character, otherwise `/send` and test-send return 400. When moving
  long Mongolian messages, shorten them, bound param lengths, or keep a separate shorter template
  for that channel. Flag any source message that can't fit.
- Ray has no SMS suppression list or opt-out registry. Keep STOP/opt-out state in the app (Twilio
  still blocks numbers that replied STOP on its side).

## Replacing send calls

- **From address and sender identity** live on the channel config (`fromAddress`, `fromName`).
  There's no per-send `from`, `replyTo` or custom headers. Different senders need different
  channel configs (dashboard).
- **Recipients**: `to: "a@b.com"` becomes `recipient: { email }` (SMS: `recipient: { phoneNumber }`). `cc`/`bcc`/attachments go inside
  `recipient`. Several independent `to` addresses become `targets[]` (identical body) or one send
  each (personalized).
- **Idempotency**: add an `Idempotency-Key` (SDK `{ idempotencyKey }`) derived from the event on
  every send, even if the old code had none.
- **Status webhooks**: replace provider event webhooks (SendGrid Event Webhook, Postmark,
  Resend) with Ray delivery webhooks, verified with `verifyWebhookSignature`. Ray reports
  `delivered` = provider accepted, `failed_terminal`, `suppressed`, and doesn't track opens.

Example (Resend to Ray SDK):

```ts
// before
await resend.emails.send({ from: "Acme <no-reply@acme.com>", to: user.email, subject: "Welcome", react: <Welcome name={user.name} /> });

// after
await ray.send(
  {
    channelConfigId: process.env.RAY_EMAIL_CHANNEL_ID!, // from address configured on this channel
    templateId: process.env.RAY_WELCOME_TEMPLATE_ID!,
    params: { name: user.name },
    recipient: { email: user.email, name: user.name },
    externalUserId: user.id,
  },
  { idempotencyKey: `welcome-${user.id}` },
);
```

## Creating templates

MCP: `create_template` with the same fields as the REST body. SDK:
`ray.templates.create({ name, folder, channelKind, content, logTitle, logDescription, publish })`.

With curl, assemble the JSON with `jq`, never with string interpolation (HTML has quotes and
newlines):

```bash
jq -n --rawfile html dist/emails/welcome.html --rawfile text dist/emails/welcome.txt \
  '{ name: "welcome", folder: "auth", channelKind: "email_html",
     content: { subject: "Welcome, {{name}}", bodyHtml: $html, bodyText: $text },
     logTitle: "Welcome email sent", logDescription: "Sent to {{name}} after sign-up", publish: false }' \
| curl -sS -X POST https://ray-api.gege.mn/templates \
    -H "Authorization: Bearer $RAY_API_KEY" -H "Content-Type: application/json" -d @-
```

The response is `201 { id, requiredParams }`. A duplicate name returns `409 conflict`. Kind or
content problems return `400` with a message like `content: subject: ...`, which names the field
to fix.
