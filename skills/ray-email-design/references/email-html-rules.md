# Email HTML rules + the frozen shell

The standard every generated email must meet, and the anatomy of `.ray/email/layout.html`.
Goal: **renders correctly in Gmail, Apple Mail, and Outlook (desktop/Windows)** — the three that
matter — while every template in the set shares byte-identical chrome.

## Non-negotiable rules

1. **Tables for layout**, not flexbox/grid. Wrap with `role="presentation" cellpadding="0"
   cellspacing="0" border="0"`. `<div>`-only layouts break in Outlook.
2. **All CSS inline** (`style="…"` on each element). No `<style>`-only styling for structure;
   Gmail strips `<head>`/`<style>` in some contexts. A `<style>` block is allowed **only** for
   `@media` (dark mode / mobile) and as a progressive enhancement — never rely on it.
3. **Fixed width ≤ 600px**, centred. Content column 600px; full-bleed background via an outer
   table at `width:100%`.
4. **Web-safe font stack** with graceful fallback, e.g.
   `font-family:-apple-system,'Segoe UI',Roboto,Helvetica,Arial,sans-serif;`. A web font is
   optional and **must** degrade to the stack (assume it won't load).
5. **Absolute `https://` image URLs** only (hosted logo). Every `<img>` needs `alt`, explicit
   `width`, and `style="display:block;border:0;"`. No base64 `data:` images (Gmail blocks them).
   Ray can attach CID inline images per send (`recipient.attachments[].contentId`), but a shared
   template shell must not depend on per-send attachments.
6. **Bulletproof buttons** — anchor styled as a button, wrapped in MSO/VML for Outlook (see below).
   Don't use a bare styled `<a>` alone; Outlook ignores the padding.
7. **Preheader**: a hidden element at the very top holding the inbox-preview text, followed by a
   zero-width spacer so body text doesn't leak into the preview. The shell has a
   `<!-- PREHEADER -->` slot that each template fills with a **literal** line, so it isn't a
   required param. Use a `{{var}}` only when the preview must vary per recipient.
8. **Dark-mode aware** — `<meta name="color-scheme" content="light dark">`,
   `<meta name="supported-color-schemes" content="light dark">`, and a `@media
   (prefers-color-scheme: dark)` block. Don't depend on it for legibility; pick colours that work
   on white if dark is ignored.
9. **Accessibility** — `<html lang>`, `role="presentation"` on layout tables, real `alt` text,
   sufficient contrast, semantic headings in the body.
10. **No JS, no external stylesheets, no forms.** They're stripped or flagged as spam.

## Variables (Ray's renderer)

- Ray's renderer is a Mustache subset (sections, inverted sections, dotted paths, arbitrary-JSON
  `params`; see the `ray-integration` skill). **This design skill deliberately stays flat:** use
  only `{{name}}` matching `^[a-zA-Z_][a-zA-Z0-9_]*$`, with no `{{a.b}}` and no sections or loops.
  `{{{raw}}}`, helpers and fallbacks (`{{x,fallback=y}}`) don't exist in Ray.
- Escaping: param values are HTML-escaped in `bodyHtml`, so `href="{{reset_url}}"` is safe. They
  are inserted as-is in `subject` and `bodyText`, and a line break in a subject value fails the
  send. Never put HTML in a param. Structure belongs in the template.
- Every top-level `{{var}}` in `subject`, `bodyHtml`, `bodyText`, `logTitle` or `logDescription`
  is a **required** send param.
- **Constants vs variables:** bake brand chrome (logo, colours, company name, footer legal,
  address) as **literals** in the shell so they're _not_ params. Keep only genuinely
  per-recipient values as `{{vars}}` — default just `{{unsubscribe_url}}`. The body adds its own
  content vars (e.g. `{{first_name}}`, `{{reset_url}}`).

## The frozen shell — `.ray/email/layout.html`

One file, built once at brand setup, reused verbatim for every template. The body is injected at
the `<!-- CONTENT -->` marker and the preview line at `<!-- PREHEADER -->`. `data-ray-no-track`
keeps the unsubscribe link out of Ray click tracking. Brand values below are **examples**. Replace
them with the confirmed brand profile and bake them in as literals.

```html
<!DOCTYPE html>
<html lang="en" xmlns:v="urn:schemas-microsoft-com:vml" xmlns:o="urn:schemas-microsoft-com:office:office">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="color-scheme" content="light dark">
  <meta name="supported-color-schemes" content="light dark">
  <!--[if mso]><style>* {font-family: Arial, sans-serif !important;}</style><![endif]-->
  <style>
    @media (prefers-color-scheme: dark) {
      .bg   { background:#0b0b0c !important; }
      .card { background:#161618 !important; }
      .text { color:#e8e8ea !important; }
      .muted{ color:#a0a0a8 !important; }
    }
    @media only screen and (max-width:600px) {
      .container { width:100% !important; }
      .px { padding-left:24px !important; padding-right:24px !important; }
    }
  </style>
</head>
<body class="bg" style="margin:0;padding:0;background:#f4f4f5;">
  <!-- preheader: shown in inbox preview, hidden in body; filled per template with a literal -->
  <div style="display:none;max-height:0;overflow:hidden;opacity:0;"><!-- PREHEADER --></div>
  <div style="display:none;max-height:0;overflow:hidden;">&#8204;&zwnj;&nbsp;&#847; &#847; &#847;</div>

  <table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0" class="bg" style="background:#f4f4f5;">
    <tr>
      <td align="center" style="padding:24px 12px;">
        <table role="presentation" width="600" cellpadding="0" cellspacing="0" border="0" class="container" style="width:600px;max-width:600px;">

          <!-- header (brand constant, baked literal) -->
          <tr>
            <td align="left" style="padding:8px 0 24px;">
              <img src="https://cdn.example.com/logo.png" alt="Acme" width="120" style="display:block;border:0;">
            </td>
          </tr>

          <!-- card -->
          <tr>
            <td class="card" style="background:#ffffff;border-radius:12px;">
              <table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0">
                <tr>
                  <td class="px text" style="padding:40px;color:#18181b;font-family:-apple-system,'Segoe UI',Roboto,Helvetica,Arial,sans-serif;font-size:16px;line-height:1.6;">

                    <!-- CONTENT -->

                  </td>
                </tr>
              </table>
            </td>
          </tr>

          <!-- footer (brand constants baked; unsubscribe stays per-recipient) -->
          <tr>
            <td class="px muted" style="padding:24px 40px;color:#71717a;font-family:-apple-system,'Segoe UI',Roboto,Helvetica,Arial,sans-serif;font-size:12px;line-height:1.6;">
              © Acme Inc · 123 Example St, City<br>
              <a data-ray-no-track href="{{unsubscribe_url}}" style="color:#71717a;text-decoration:underline;">Unsubscribe</a>
            </td>
          </tr>

        </table>
      </td>
    </tr>
  </table>
</body>
</html>
```

### Bulletproof button (use inside the body)

```html
<!--[if mso]>
<v:roundrect xmlns:v="urn:schemas-microsoft-com:vml" xmlns:w="urn:schemas-microsoft-com:office:word"
  href="{{cta_url}}" style="height:44px;v-text-anchor:middle;width:220px;" arcsize="14%" strokecolor="#1a73e8" fillcolor="#1a73e8">
  <w:anchorlock/>
  <center style="color:#ffffff;font-family:Arial,sans-serif;font-size:16px;font-weight:bold;">{{cta_label}}</center>
</v:roundrect>
<![endif]-->
<!--[if !mso]><!-- -->
<a href="{{cta_url}}" style="background:#1a73e8;color:#ffffff;display:inline-block;font-family:-apple-system,'Segoe UI',Roboto,Helvetica,Arial,sans-serif;font-size:16px;font-weight:bold;line-height:44px;text-align:center;text-decoration:none;width:220px;border-radius:8px;">{{cta_label}}</a>
<!--<![endif]-->
```

## Plain-text (`bodyText`)

Hand-write it; don't tag-strip. Same `{{vars}}` as the HTML, sensible line breaks, URLs spelled
out (`Reset your password: {{reset_url}}`), and the same unsubscribe line. It's what text-only
clients and many spam filters read.

## Self-check before preview

- [ ] No `{{a.b}}`, sections or fallbacks. Every var matches `^[a-zA-Z_][a-zA-Z0-9_]*$`.
- [ ] No leftover `<!-- CONTENT -->` / `<!-- PREHEADER -->` markers. Preheader text is filled in.
- [ ] All structural CSS is inline; `<style>` only holds `@media`.
- [ ] Every `<img>` has `alt`, `width`, `display:block`, absolute `https` src.
- [ ] CTA uses the MSO/VML bulletproof pattern.
- [ ] Preheader present. Footer has `{{unsubscribe_url}}` with `data-ray-no-track`.
- [ ] Looks legible on **white** even if dark-mode CSS is ignored.
- [ ] Shell is identical to `.ray/email/layout.html` (chrome not re-derived).
