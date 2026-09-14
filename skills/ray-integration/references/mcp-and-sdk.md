# Ray MCP server and TypeScript SDK

## Contents
- Connect the MCP server
- Tool, SDK and REST map
- SDK usage
- Fallback: raw HTTP

## Connect the MCP server

Hosted server: `https://ray-api.gege.mn/mcp`. It uses MCP Streamable HTTP, is stateless and
POST-only, and authenticates with the same API key as the REST API
(`Authorization: Bearer ck_live_...`). A missing or invalid key returns HTTP 401. Tools call the
REST routes, so scopes, rate limits, quota and validation are identical.

Check `/mcp` (Claude Code) or the client's MCP panel. If `ray` isn't listed, tell the user how to
add it and continue with HTTP in the meantime. **Never write an API key into a file that gets
committed.** Prefer env expansion or a user-level config.

Claude Code (key taken from the shell environment when the command runs):

```bash
claude mcp add --transport http ray https://ray-api.gege.mn/mcp --header "Authorization: Bearer $RAY_API_KEY"
```

The Claude Code plugin `ray@ray-skills` only installs these skills; it doesn't register the MCP
server, so run the command above as well.

Cursor (`.cursor/mcp.json`):

```json
{ "mcpServers": { "ray": { "url": "https://ray-api.gege.mn/mcp", "headers": { "Authorization": "Bearer ck_live_..." } } } }
```

VS Code (`.vscode/mcp.json`):

```json
{ "servers": { "ray": { "type": "http", "url": "https://ray-api.gege.mn/mcp", "headers": { "Authorization": "Bearer ck_live_..." } } } }
```

Windsurf (`~/.codeium/windsurf/mcp_config.json`):

```json
{ "mcpServers": { "ray": { "serverUrl": "https://ray-api.gege.mn/mcp", "headers": { "Authorization": "Bearer ck_live_..." } } } }
```

Local stdio bridge for any client, including those without remote-HTTP support:

```json
{ "mcpServers": { "ray": { "command": "npx", "args": ["-y", "@gege-mn/ray-mcp"], "env": { "RAY_API_KEY": "ck_live_..." } } } }
```

Codex (`~/.codex/config.toml`):

```toml
[mcp_servers.ray]
command = "npx"
args = ["-y", "@gege-mn/ray-mcp"]
env = { RAY_API_KEY = "ck_live_..." }
```

## Tool, SDK and REST map

| MCP tool | SDK (`@gege-mn/ray`) | REST |
|---|---|---|
| `whoami` | `ray.me()` | `GET /me` |
| `get_usage` | `ray.usage()` | `GET /billing/subscription` |
| `list_channels` | `ray.channels.list()` | `GET /channels` |
| `send_notification` (`idempotencyKey` arg) | `ray.send(body, { idempotencyKey })` | `POST /send` + `Idempotency-Key` |
| `get_send_status` | `ray.sends.get(id, { cursor })` | `GET /sends/{id}` |
| `list_templates` | `ray.templates.list()` | `GET /templates` |
| `get_template` | `ray.templates.get(id)` | `GET /templates/{id}` |
| `create_template` | `ray.templates.create(body)` | `POST /templates` |
| `update_template_draft` | `ray.templates.update(id, body)` | `PATCH /templates/{id}` |
| `publish_template` | `ray.templates.publish(id)` | `POST /templates/{id}/publish` |
| `archive_template` / `unarchive_template` | `ray.templates.archive(id)` / `.unarchive(id)` | `POST /templates/{id}/archive` / `unarchive` |
| `test_send_template` | `ray.templates.testSend(id, body)` | `POST /templates/{id}/test-send` |
| `list_feed_notifications` | `ray.notifications.list(query)` | `GET /notifications` |
| `get_click_stats` | `ray.clicks.get(query)` | `GET /clicks` |
| `list_webhooks` / `get_webhook` | `ray.webhooks.list()` / `.get(id)` | `GET /tenant-webhooks[/{id}]` |
| `create_webhook` / `update_webhook` / `delete_webhook` | `ray.webhooks.create/update/delete` | `POST` / `PATCH` / `DELETE /tenant-webhooks` |
| `read_docs` (`slug` optional) | none | `https://ray.gege.mn/docs/<slug>.md`, or `/llms.txt` when empty |

Tool arguments mirror the REST bodies. Read-only tools are annotated `readOnlyHint`, and
`delete_webhook` and `archive_template` are marked destructive, so confirm with the user before
calling those.

Using the MCP tools well:
- Call `whoami` then `list_channels` first. Use real `channelConfigId`s and each channel's
  `recipientSchema`.
- `send_notification` really delivers. Only send to recipients the user named. Always pass an
  `idempotencyKey`.
- Poll `get_send_status` after a send to report the real outcome (`delivered`,
  `failed_terminal` + `providerError`, `suppressed`).
- Use `read_docs` for any detail this skill doesn't cover instead of guessing.

## SDK usage

`@gege-mn/ray` has no runtime dependencies, uses fetch, and ships ESM + CJS for Node 18+, Bun,
Deno and edge runtimes. Install it with the project's package manager
(`npm i @gege-mn/ray` / `pnpm add` / `bun add`). If the install fails (for example the package
isn't available), use raw HTTP.

```ts
import { Ray, RayError } from "@gege-mn/ray";

export const ray = new Ray(process.env.RAY_API_KEY); // apiKey defaults to env RAY_API_KEY; { baseUrl } optional

export async function notifyPasswordReset(user: { id: string; email: string }, token: { id: string; url: string }) {
  try {
    const { sendId } = await ray.send(
      {
        channelConfigId: process.env.RAY_EMAIL_CHANNEL_ID!,
        templateId: process.env.RAY_PASSWORD_RESET_TEMPLATE_ID!,
        params: { resetUrl: token.url },
        recipient: { email: user.email },
        externalUserId: user.id,
      },
      { idempotencyKey: `password-reset-${user.id}-${token.id}` },
    );
    return sendId;
  } catch (err) {
    if (err instanceof RayError) {
      // err.status, err.code ("validation_error", "rate_limit_exceeded", ...), err.message, err.requestId
      if (err.code === "validation_error" || err.code === "not_found") throw err; // bug: don't retry
    }
    throw err; // 429 / 5xx / network: retry later with the SAME idempotencyKey
  }
}
```

Multi-channel for one user (push + email, one feed entry):

```ts
await ray.send(
  {
    externalUserId: user.id,
    feed: { title: "Order shipped", description: `Order ${order.id} is on its way.` },
    params: { orderId: order.id },
    deliveries: [
      { channelConfigId: PUSH_ID, templateId: PUSH_TPL, recipient: { deviceToken } },
      { channelConfigId: EMAIL_ID, templateId: EMAIL_TPL, recipient: { email: user.email } },
    ],
  },
  { idempotencyKey: `order-shipped-${order.id}` },
);
```

Webhook receiver: see `SKILL.md` (`verifyWebhookSignature`). Always pass the raw request body.
In Express, use `express.raw({ type: "application/json" })` on that route.

## Fallback: raw HTTP

Same bodies, any language:

```ts
const res = await fetch("https://ray-api.gege.mn/send", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${process.env.RAY_API_KEY}`,
    "Content-Type": "application/json",
    "Idempotency-Key": `order-shipped-${order.id}`,
  },
  body: JSON.stringify(body),
});
if (res.status !== 202) {
  const err = await res.json().catch(() => ({}));
  throw new Error(`Ray ${res.status} ${err.error}: ${err.message}`);
}
const { sendId } = await res.json();
```

```python
import os, requests
r = requests.post(
    "https://ray-api.gege.mn/send",
    headers={"Authorization": f"Bearer {os.environ['RAY_API_KEY']}", "Idempotency-Key": f"order-shipped-{order_id}"},
    json=body, timeout=15,
)
if r.status_code != 202:
    raise RuntimeError(f"Ray {r.status_code}: {r.text}")
send_id = r.json()["sendId"]
```

Verifying delivery webhooks without the SDK: HMAC-SHA256 over `"<t>.<raw body>"` with the secret,
compared in constant time to `v1` from `X-Ray-Signature: t=...,v1=...`, rejecting `|now - t| > 300`.
For Node, Web Crypto and Express versions, see `https://ray.gege.mn/docs/webhooks.md`.
