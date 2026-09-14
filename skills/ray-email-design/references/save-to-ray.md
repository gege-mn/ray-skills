# Saving email templates to Ray

The template write path this skill needs. For auth, channels, MCP setup and sending, see the
`ray-integration` skill. The live reference is `https://ray.gege.mn/docs/templates.md` (MCP
`read_docs` slug `templates`).

- REST base `https://ray-api.gege.mn`, header `Authorization: Bearer $RAY_API_KEY`.
- Template writes, test-sends and sends need a **write**-scoped key (`whoami` / `GET /me` must show
  `write` in `scopes`).
- All templates here use `channelKind: "email_html"`, and `content` is
  `{ subject, bodyHtml, bodyText }`.

## Operations

| Step | MCP tool | SDK (`@gege-mn/ray`) | REST |
|---|---|---|---|
| Find by name | `list_templates` | `ray.templates.list()` | `GET /templates?includeArchived=true` |
| Read versions | `get_template` | `ray.templates.get(id)` | `GET /templates/{id}` returns `published` + `draft` |
| Create draft | `create_template` | `ray.templates.create(body)` | `POST /templates` returns `201 { id, requiredParams }` |
| Update draft | `update_template_draft` | `ray.templates.update(id, body)` | `PATCH /templates/{id}` returns `{ id, draftVersionId, requiredParams }` |
| Email channel id | `list_channels` | `ray.channels.list()` | `GET /channels` (kind `ses_email` or `smtp_email`) |
| Preview a draft in a real inbox | `send_notification` | `ray.send(body, { idempotencyKey })` | `POST /send` with inline `content` |
| Test the published version | `test_send_template` | `ray.templates.testSend(id, body)` | `POST /templates/{id}/test-send` |
| Publish | `publish_template` | `ray.templates.publish(id)` | `POST /templates/{id}/publish` |
| Unarchive a name clash | `unarchive_template` | `ray.templates.unarchive(id)` | `POST /templates/{id}/unarchive` |

## Template body (create and update)

```json
{
  "name": "password-reset",
  "folder": "auth",
  "channelKind": "email_html",
  "content": {
    "subject": "Reset your password",
    "bodyHtml": "<!DOCTYPE html>... shell with body injected ...",
    "bodyText": "Reset your password\n\nOpen this link: {{reset_url}}\n\nUnsubscribe: {{unsubscribe_url}}"
  },
  "logTitle": "Password reset requested",
  "logDescription": "A password reset link was emailed.",
  "paramOverrides": {},
  "publish": false
}
```

- Lengths: `name` 1-100, `folder` ≤200, `subject` 1-998, `logTitle` 1-500 (required),
  `logDescription` 1-2000 (required), `paramOverrides[x].description` ≤500.
- `content` is validated server-side against `channelKind`. On a mismatch, the 400 message names
  the field (`content: subject: ...`). `content.source` may be omitted. `"designed"` and
  `mailyJson` are rejected.
- `PATCH` takes the **same full body**: `name` and `channelKind` are required by the schema,
  `name`/`folder` aren't changed, and `channelKind` can't change. It overwrites the existing draft,
  or creates the next version's draft if the latest version is published.
- `requiredParams` in the response is the authoritative list of detected vars. Use it for the
  manifest.
- `paramOverrides[name] = { optional: true }` skips the up-front required check, but a top-level
  `{{name}}` still needs a value (pass `null` or `""`) or the send fails. Avoid it in this skill.

## Idempotent upsert by name

Names are unique per workspace **including archived templates**. A second create with the same
name returns `409 conflict`.

```
templates = list_templates(includeArchived: true)
existing  = templates.find(t => t.name === name)

if existing?.archivedAt: unarchive_template(existing.id)    # ask first
if existing:  update_template_draft(existing.id, { name, folder, channelKind: "email_html", content, logTitle, logDescription, publish: false })
else:         create_template({ name, folder, channelKind: "email_html", content, logTitle, logDescription, publish: false })
```

## Real-inbox checks

Test-send only delivers the **published** version (an unpublished template returns 404 "template
not found or not published").

**Draft preview.** A normal send with the draft's content to the user's own address:

```json
POST /send                      (MCP: send_notification with idempotencyKey)
Idempotency-Key: preview-password-reset-<draftVersionId>

{ "channelConfigId": "<email channel id from list_channels>",
  "content": { "subject": "Reset your password", "bodyHtml": "<!DOCTYPE html>...", "bodyText": "..." },
  "params": { "reset_url": "https://app.example.com/r/sample", "unsubscribe_url": "https://app.example.com/u/sample" },
  "recipient": { "email": "you@yourco.com" } }
```

This returns `202 { sendId }`. Every top-level var needs a value. It counts as 1 notification, and
no feed entry is created without `externalUserId`. Check the outcome with `get_send_status`.

**Published template test.**

```json
POST /templates/{id}/test-send  (MCP: test_send_template)
{ "channelConfigId": "<email channel id>",
  "recipient": { "email": "you@yourco.com" },
  "params": { "reset_url": "https://app.example.com/r/sample", "unsubscribe_url": "https://app.example.com/u/sample" } }
```

This returns `202 { sendId }`. The row is marked `isTest`, with no feed entry, no webhooks and no
monthly quota. The daily allowance is Free 50, Starter 250, Pro/Scale 500, and exceeding it returns
402. Missing params return 400 `missing required params: ...`.

## Publish

Publish only after the user confirms: `publish_template` / `POST /templates/{id}/publish`
returns `{ id, versionId, versionNumber }`, or `PATCH` with `"publish": true`. Live sends switch
immediately, while scheduled sends keep their already-rendered content. Publishing with no draft
returns 400 `no draft to publish`.

## Param manifest (print after saving each template)

```
Template: password-reset  (draft saved, id=<uuid>, published: no)
Required params: reset_url, unsubscribe_url      # from requiredParams
Per-recipient (computed by your backend each send): reset_url, unsubscribe_url
Sample send:
  ray.send(
    { channelConfigId: "<email channel id>", templateId: "<uuid>",
      recipient: { email: user.email },
      params: { reset_url: "https://...", unsubscribe_url: "https://..." } },
    { idempotencyKey: `password-reset-${user.id}-${token.id}` },
  )
```

Ray stores no recipient registry. Callers pass `recipient` and `params` on every send. If you
enable `trackClicks` on sends, add `data-ray-no-track` to the unsubscribe and legal links in the
shell.
