# ray-skills

Agent skills and a Claude Code plugin for **[Ray](https://ray.gege.mn)**, the notification delivery
API. (Ray the notification API is not related to the Ray distributed computing framework.)

Ray sends transactional notifications over **email (Amazon SES or SMTP), mobile push (Firebase Cloud
Messaging), SMS (Twilio or sendsms.mn), Slack, Discord, Telegram and your own HTTPS webhook**. It uses one REST endpoint
(`POST /send`) and your own provider credentials, and handles templates, idempotency, fan-out,
scheduling, retries, suppression, signed delivery webhooks, an in-app feed and click tracking.

With these skills, a coding agent (Claude Code, Cursor, Codex, and others) can wire Ray into your
app, send and debug notifications, migrate templates from another provider, and design on-brand
email templates.

## What's in the repo

| Path | What it is |
|---|---|
| `skills/ray-integration/` | Sending through Ray: MCP tools, the `@gege-mn/ray` SDK or raw HTTP. Covers idempotency, fan-out and multi-channel sends, feeds, delivery status, webhook verification, templates, and migrating templates and send calls from Resend, SendGrid, Postmark, Mailgun, SES, Twilio, Novu, Knock, Courier, OneSignal. |
| `skills/ray-email-design/` | Consistent, bulletproof HTML email templates: one frozen brand shell, local preview, real-inbox check, then save and publish to Ray. |
| `.claude-plugin/marketplace.json` | Claude Code plugin marketplace `ray-skills` with one plugin, `ray`. |
| `.claude-plugin/plugin.json` | The `ray` plugin: both skills. |

```
skills/ray-integration/
  SKILL.md                      # entry point: pick MCP / SDK / HTTP, send, key rules
  references/api.md             # condensed endpoint, channel, content and error contract
  references/mcp-and-sdk.md     # MCP client setup, tool/SDK/REST map, SDK and HTTP examples
  references/migrate-templates.md  # template and send-call migration playbook
skills/ray-email-design/
  SKILL.md                      # brand setup, generate, verify, publish
  references/email-html-rules.md   # bulletproof email-HTML standard + frozen shell
  references/save-to-ray.md        # template upsert, preview send, test-send, publish
```

## Install

### Any agent: `skills` CLI

```bash
npx skills add gege-mn/ray-skills            # pick skills and agents interactively
npx skills add gege-mn/ray-skills -g         # install for your user (all projects)
npx skills add gege-mn/ray-skills --skill ray-integration   # just one skill
```

### Claude Code: plugin marketplace

Inside Claude Code:

```text
/plugin marketplace add gege-mn/ray-skills
/plugin install ray@ray-skills
```

This installs both skills (as `/ray:ray-integration` and `/ray:ray-email-design`). It doesn't
register the MCP server, so a missing API key never leaves a broken server behind; add it with the
one-line command in [MCP server](#mcp-server) below. Update with
`/plugin marketplace update ray-skills`.

### Manual

A skill is a directory containing `SKILL.md`. Clone the repo and copy or symlink the skill folders
into your agent's skills directory:

```bash
git clone https://github.com/gege-mn/ray-skills ~/src/ray-skills

# Claude Code, all projects
ln -s ~/src/ray-skills/skills/ray-integration  ~/.claude/skills/ray-integration
ln -s ~/src/ray-skills/skills/ray-email-design ~/.claude/skills/ray-email-design

# Claude Code, one project
mkdir -p .claude/skills && cp -R ~/src/ray-skills/skills/ray-integration .claude/skills/
```

## Set up access

1. Create an API key at https://ray.gege.mn under **API keys**. Keep the `write` scope if the agent
   should send or edit templates. Configure at least one channel under **Channels**, because
   provider credentials can't be created through the API.
2. Put the key in your environment, never in code:

```bash
export RAY_API_KEY="ck_live_..."
```

## MCP server

Hosted endpoint: `https://ray-api.gege.mn/mcp` (Streamable HTTP, same API key as the REST API).
It exposes `whoami`, `get_usage`, `list_channels`, `send_notification`, `get_send_status`,
template tools (`list_templates`, `get_template`, `create_template`, `update_template_draft`,
`publish_template`, `archive_template`, `unarchive_template`, `test_send_template`),
`list_feed_notifications`, `get_click_stats`, webhook tools (`list_webhooks`, `get_webhook`,
`create_webhook`, `update_webhook`, `delete_webhook`) and `read_docs`.

Claude Code:

```bash
claude mcp add --transport http ray https://ray-api.gege.mn/mcp --header "Authorization: Bearer $RAY_API_KEY"
```

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

Local stdio bridge (any MCP client):

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

## SDK

TypeScript/JavaScript: [`@gege-mn/ray`](https://github.com/gege-mn/ray-node) (Node 18+, Bun,
Deno, edge; zero dependencies).

```ts
import { Ray } from "@gege-mn/ray";

const ray = new Ray(); // reads RAY_API_KEY
const { sendId } = await ray.send(
  { channelConfigId, templateId, params: { name: "Ada" }, recipient: { email: "ada@example.com" } },
  { idempotencyKey: "welcome-user_123" },
);
```

## Example prompts

- "Send a welcome email through Ray when a user signs up, with an idempotency key."
- "Post deploy notifications to our Slack channel through Ray."
- "Move our Twilio SMS login codes to Ray, and send Mongolian numbers through sendsms.mn."
- "Add push notifications for shipped orders via Ray and FCM, and clean up dead device tokens."
- "Migrate our SendGrid dynamic templates to Ray and replace the send calls."
- "Move our Knock workflows to Ray: email, push and in-app feed."
- "Design a password reset and a receipt email in our brand and save them to Ray as drafts."
- "Add a Ray delivery webhook endpoint and verify its signature."
- "Why did send 6a7b8c9d-... fail?" (uses `get_send_status` when the MCP server is connected)

## Docs

- Docs: https://ray.gege.mn/docs. Append `.md` to any page for markdown, e.g.
  https://ray.gege.mn/docs/sending.md
- Index for agents: https://ray.gege.mn/llms.txt. Everything in one file:
  https://ray.gege.mn/llms-full.txt
- OpenAPI 3.1: https://ray-api.gege.mn/openapi.json. API reference: https://ray-api.gege.mn/docs
- Using Ray with AI agents: https://ray.gege.mn/docs/ai-agents

## License

[MIT](LICENSE) © gege.mn
