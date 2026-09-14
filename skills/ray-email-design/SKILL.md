---
name: ray-email-design
description: Designs consistent, on-brand, bulletproof HTML email templates (table layout, inline CSS, Outlook VML buttons, dark mode, real plain-text part) and saves them as email_html templates in the Ray notification API (ray.gege.mn). Use when creating transactional email templates such as welcome, verification code, password reset, magic link, receipt, invoice, invite or digest emails, when restyling or standardizing a project's existing emails into one brand look, or when redesigning emails imported into Ray from Resend, SendGrid, Postmark, react-email or MJML. Builds one frozen layout shell from the repo's brand, generates templates that reuse it, previews locally, then saves drafts, test-sends and publishes through Ray's MCP tools, SDK or REST API. Not for the Ray distributed computing framework.
---

# Ray email design

Produce **visually consistent** transactional emails and save them to Ray. Consistency comes from
one **frozen HTML shell** (wrapper, header, footer) that every template body is injected into, so
the chrome is byte-identical across the set.

**Ray constraints (don't fight them):**
- The API stores **raw HTML only**: `channelKind: "email_html"`,
  `content: { subject, bodyHtml, bodyText }`. The visual designer is dashboard-only, so this skill
  writes its own email-safe HTML.
- Variables: Ray renders a Mustache subset (sections, dotted paths). **This skill deliberately
  uses flat `{{name}}` vars only** (`^[a-zA-Z_][a-zA-Z0-9_]*$`). No `{{{raw}}}`, helpers,
  fallbacks or partials exist. For loops such as digests, see the `ray-integration` skill.
- Param values are HTML-escaped in `bodyHtml` (safe inside `href="{{url}}"`). They are inserted
  as-is in `subject` and `bodyText`, and a line break in the rendered subject fails the send.
- **Every top-level `{{var}}` in `subject`, `bodyHtml`, `bodyText`, `logTitle` or `logDescription`
  is a required send param.** Keep the variable surface small (see "Constants vs variables" in
  `references/email-html-rules.md`).

**How to talk to Ray:** prefer the Ray MCP tools when connected (`whoami`, `list_channels`,
`list_templates`, `create_template`, `update_template_draft`, `publish_template`,
`test_send_template`, `send_notification`, `read_docs`). Otherwise use the SDK `@gege-mn/ray` or
curl. Setup and the full API are in the `ray-integration` skill. Read `$RAY_API_KEY` from the
environment. Template writes need the `write` scope.

## Project artifacts: `.ray/email/` (committed)

```
.ray/email/
  layout.html   # FROZEN shell: bulletproof email HTML with <!-- PREHEADER --> and <!-- CONTENT --> slots.
                # Brand constants (logo, colours, footer legal) baked in as literals.
                # Per-recipient values stay {{vars}} (default: only {{unsubscribe_url}}).
  brand.md      # tokens (logo URL, palette, font stack, footer text, width), tone and CTA rules,
                # and the constant-vs-variable split agreed at setup.
```

If `.ray/email/layout.html` is missing, run **Brand setup**. If it exists, reuse it verbatim.

## 1. Brand setup (first run only)

1. **Derive, don't interrogate.** Scan for existing emails (`emails/`, `templates/`, `mail/`,
   MJML, react-email) and brand config (Tailwind theme, CSS custom properties, design tokens,
   logo). Propose: logo URL, primary and accent colours, font stack, header layout, footer text +
   legal address, width (default 600px).
2. **The logo must be a hosted `https://` URL.** Per-send CID attachments exist in Ray, but a
   shared template shell can't rely on them. If only a local file exists, ask for its hosted URL.
3. **Confirm the constant-vs-variable split** and show the user the exact list. This list is
   their required-param surface.
4. **Build the shell** per `references/email-html-rules.md`, write `.ray/email/layout.html` and
   `brand.md`, and tell the user to commit them.

## 2. Generate (one template or a batch)

- **new**: from a brief ("password reset email"), write the body copy and structure.
- **restyle**: from an existing email, keep the subject and meaningful copy, strip the old chrome,
  and re-flow the content into the shell. (`ray-integration`'s migration imports HTML as-is; this
  imposes the brand.)

Inject each body at `<!-- CONTENT -->` and its literal inbox-preview line at `<!-- PREHEADER -->`.
Never re-derive the shell.

## 3. Required outputs per template

- `subject` (1-998 chars, may contain `{{vars}}`).
- `bodyHtml`: the shell plus the body, fully inlined.
- `bodyText`: a genuine plain-text version with the **same vars** and URLs spelled out.
- `logTitle` (1-500) and `logDescription` (1-2000): both **required**. They are the in-app feed
  and log entry (e.g. "Password reset requested"), not the subject.
- A short, stable `name` (unique per workspace, including archived) and an optional `folder`.

## 4. Verify

1. **Local preview (always).** Fill every var with a plausible sample (`first_name` becomes
   "Alex", `reset_url` a fake `https://...`, `unsubscribe_url` becomes `#`). Write
   `/tmp/ray-preview/<name>.html`, open it, and iterate here.
2. **Real-inbox check (offer it).** Ray's test-send delivers only the **published** version. Pick
   the case:
   - **Draft, or an update to a live template**: `send_notification` / `POST /send` with inline
     `content` (the draft's subject, bodyHtml and bodyText), sample `params`, the user's own
     `recipient`, and an idempotency key. It's a real send (1 notification of quota, no feed
     entry without `externalUserId`) and renders exactly like the template.
   - **Published template**: `test_send_template` / `POST /templates/{id}/test-send`, marked
     `isTest`, with no feed, webhooks or monthly quota (daily allowance applies).

## 5. Save and publish

- **Idempotent upsert by name**: `list_templates` (`includeArchived=true`). If a match exists, use
  `update_template_draft` / `PATCH` with the **full body** (`name`, `channelKind`, `content`,
  `logTitle`, `logDescription`, `publish: false`). Otherwise `create_template` / `POST` with
  `publish: false`. Re-runs update the draft instead of failing on a duplicate name.
- **Print a param manifest** per template: Ray's returned `requiredParams`, which ones are
  per-recipient (e.g. `unsubscribe_url`), and a sample send payload.
- **Publish only on explicit confirmation** (`publish_template` / `POST /templates/{id}/publish`).
  Publishing switches live sends to this version immediately.

Exact bodies and examples: `references/save-to-ray.md`.

## References
- `references/email-html-rules.md`: the bulletproof email-HTML standard and the frozen shell.
- `references/save-to-ray.md`: template create, upsert, preview-send, test-send and publish.
- Everything else (auth, channels, sending, MCP setup) is in the `ray-integration` skill, or
  `https://ray.gege.mn/docs/templates.md`.
