<p align="center">
    <img src="doc/demo/logo.png" width="80px" />
    <h1 align="center">Cloud Mail Plus</h1>
    <p align="center">Enhanced Cloudflare Workers email service — native CF Email Service sending + External API + automatic D1 backup</p>
    <p align="center">
        **English** | [简体中文](README.zh-CN.md)
    </p>
    <p align="center">
        <a href="https://github.com/Leechuihui/cloud-mail-plus/blob/main/LICENSE">
            <img src="https://img.shields.io/badge/license-MIT-green" />
        </a>
    </p>
</p>

## Credits

This project is based on [maillab/cloud-mail](https://github.com/maillab/cloud-mail), an excellent Cloudflare Workers email platform. We've added the following features on top of the original. Thanks to the original author for their open-source contribution.

## New Features

### 1. Cloudflare Email Service Integration

Uses the native `send_email` Workers binding to send outbound email, replacing Resend as the primary sender.

- **CF First** (default): try Cloudflare Email Service, fall back to Resend on failure
- **Resend Only**: original behavior
- **CF Only**: Cloudflare only, no fallback

Benefits:
- No third-party API key needed (Cloudflare handles SPF/DKIM/DMARC automatically)
- Better deliverability (Cloudflare IP reputation vs self-hosted)
- Zero additional cost (included in Workers paid plan)

Configure in admin Settings > Email Provider.

### 2. External API

Allows other applications to send email and query delivery status via HTTP API, for all configured domains.

```bash
# Send
curl -X POST "https://your-domain.com/api/external/send" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{"from":"App <noreply@example.com>","to":"user@gmail.com","subject":"Hello","html":"<p>Hello</p>"}'

# Check status
curl "https://your-domain.com/api/external/status/9" -H "X-API-Key: YOUR_API_KEY"
```

Generate the API key in admin Settings > External API Key.

Full documentation: [External API Guide](docs/external-api-guide.md)

### 3. Email Delete + R2 Attachment Cleanup (Web UI + API)

**Web UI**: After selecting emails, the toolbar shows two delete buttons:
- 🗑️ Soft delete — marks as deleted, recoverable
- 🗑️ **Permanent delete** — removes email + R2/S3/KV attachments + stars, irreversible (with confirmation dialog)

**External API** also provides delete endpoints:

```bash
# Soft delete
curl -X DELETE "https://your-domain.com/api/external/email/123" -H "X-API-Key: KEY"

# Permanent delete (email + R2 attachments + stars)
curl -X DELETE "https://your-domain.com/api/external/email/123/permanent" -H "X-API-Key: KEY"

# Batch delete
curl -X POST "https://your-domain.com/api/external/email/batch-delete" \
  -H "X-API-Key: KEY" -H "Content-Type: application/json" \
  -d '{"emailIds":[1,2,3],"permanent":true}'
```

### 4. Email Export as .eml (Web UI + API)

Export emails as standard `.eml` files (RFC 5322), compatible with any email client (Outlook, Thunderbird, Apple Mail, etc.).

**Web UI**: Open email → click download icon 📥 → `.eml` file downloads automatically.

**External API**:
```bash
curl "https://your-domain.com/api/external/email/9/export" \
  -H "X-API-Key: KEY" -o email-9.eml
```

Includes: headers (From/To/Subject/Date), HTML + plain text body, inline images (CID), attachments.

### 5. New User Registration Notification

Automatically sends Telegram + admin email notifications when a new user registers (via CF Email Service). Uses existing Telegram Bot settings — no extra configuration needed.

### 6. Admin Password Reset

Forgot admin password? Reset via JWT secret:

```bash
curl -X POST "https://your-domain.com/api/reset-admin/<jwt_secret>" \
  -H "Content-Type: application/json" \
  -d '{"password":"newpassword"}'
```

### 7. Automatic D1 Backup to R2

Built-in Worker cron job exports the entire D1 database as gzipped SQL to R2 daily.

- Runs at 02:00 UTC daily
- Retains last 30 backups, auto-cleans older ones
- Zero external dependencies — fully handled inside Cloudflare
- Manual trigger: `POST /api/backup/<jwt_secret>`
- List backups: `GET /api/backup/<jwt_secret>/list`

### 8. AI Email Agent (Cloudflare Workers AI)

A conversational email assistant powered by [Cloudflare Workers AI](https://developers.cloudflare.com/workers-ai/) (`@cf/moonshotai/kimi-k2.5`). After signing in, click the **✨ Email Agent** pill in the header to open the side panel and chat with the AI.

**9 email tools** (the model decides which to call):

| Tool | Purpose | Requires confirmation |
|------|---------|----------------------|
| `listEmails` | List inbox / sent / drafts / trash | No |
| `searchEmails` | Search by subject / sender / date | No |
| `getEmail` | Read full email body + attachment list | No |
| `getAttachmentText` | Read text-like attachments (text/* / json / xml / csv) | No |
| `summarizeEmail` | 3-5 bullet summary + action items | No |
| `draftReply` | Draft a reply (saved to Drafts) | No |
| `draftNew` | Draft a new email | No |
| `sendDraft` | Send a draft | **Yes ✓** |
| `deleteEmail` | Delete email (soft / permanent) | **Yes ✓** |

**Auto-draft on inbound email** — when new mail arrives, the agent reads it and generates a reply draft saved to the Drafts mailbox (**never auto-sent** — explicit user confirmation required at send time).

**Safety guarantees**:
- Send + delete actions only execute when the user clicks Confirm on the yellow card at the bottom of the side panel
- Tool calls are scoped per user — one user's agent never sees another user's emails
- Step count capped per chat turn (max 8) to bound cost
- Auto-draft is even tighter (max 2 steps: read + draft)
- The model can decide an email needs no reply (noreply / spam / automated notifications) and skip drafting

**Configuration**: After login, go to **Settings** → scroll to the **✨ AI Email Agent** section → toggle "Enable AI agent" + (optional) "Auto-draft replies" + custom persona instructions.

**Model & cost**: Uses Cloudflare Workers AI Kimi K2.5. Free tier is 10,000 neurons/day; a single chat turn uses ~50–200 neurons, sufficient for normal use.

**Integration stack**: [AI SDK v6](https://sdk.vercel.ai/) + `@ai-sdk/vue` v3 (`Chat` class) + `workers-ai-provider` + Cloudflare Workers AI. Full bilingual i18n.

---

## Deployment

### Prerequisites

- [Cloudflare](https://dash.cloudflare.com) account
- [Node.js](https://nodejs.org/) 16.17+
- [pnpm](https://pnpm.io/) 8+ (recommended) or npm
- `jq`, `python3`, `openssl`, `curl` (required by the one-click script)
- Domain added to Cloudflare DNS

### One-Click Deploy (Recommended)

```bash
git clone https://github.com/Leechuihui/cloud-mail-plus.git
cd cloud-mail-plus
bash scripts/deploy.sh
```

The script handles:
- Wrangler login check (launches `wrangler login` if needed)
- Idempotent D1 / KV / R2 creation (existing resources are reused, not duplicated)
- 64-char JWT secret generation
- Optional AI Email Agent enablement (Workers AI kimi-k2.5 + auto-draft) — writes the `[ai]` binding, `EmailAgent` Durable Object, DO migration, and agent schema automatically
- `wrangler.toml` patching inside markers (re-runs replace, never duplicate)
- `wrangler deploy` (auto-builds the Vue frontend)
- D1 schema init via `/api/init/<jwt_secret>` (including agent tables)
- State saved to `.cloud-mail-deploy.env` (gitignored, contains the JWT secret)

**Subcommands:**

```bash
bash scripts/deploy.sh                  # interactive first-time deploy (asks whether to enable the AI Agent)
bash scripts/deploy.sh --with-ai        # enable AI Email Agent automatically (non-interactive)
bash scripts/deploy.sh --no-ai          # disable AI Email Agent automatically (non-interactive)
bash scripts/deploy.sh --redeploy       # rebuild + ship only, skip resource creation
bash scripts/deploy.sh --reset          # clear local state file, start over
bash scripts/deploy.sh --destroy        # tear down Worker + D1 + KV + R2 (irreversible)
bash scripts/deploy.sh --destroy --yes  # skip confirmations (CI / scripting)
```

> `--destroy` permanently deletes mailbox data, attachments, and backups. Back up critical data via `wrangler r2 object` before running.

**One-click AI Email Agent activation:**

```bash
bash scripts/deploy.sh --with-ai
```

After deployment:
1. Sign in to the Web UI (first run requires registering the admin account — email must match `admin` in `wrangler.toml`)
2. The yellow **✨ Email Agent** pill appears in the top header
3. **Settings** page has an **✨ AI Email Agent** section — toggle "Enable AI agent" + (optional) "Auto-draft replies" + custom persona
4. Click the header pill → side panel slides in → chat with the agent

What the agent provides:
- **9 email tools**: listEmails / searchEmails / getEmail / getAttachmentText / summarizeEmail / draftReply / draftNew / sendDraft / deleteEmail
- **Send + delete require confirmation** (a yellow card pops up at the bottom of the side panel — never auto-executed)
- **Auto-draft replies on inbound email** (saved to Drafts — never sent automatically)
- **Full bilingual i18n** (en / zh, follows the language toggle)
- **Model**: `@cf/moonshotai/kimi-k2.5` (Cloudflare Workers AI — billed by neurons, free tier is 10,000 neurons/day)

> **Integration**: AI SDK v6 (`ai`) + `@ai-sdk/vue` v3 (`Chat` class) + `workers-ai-provider` + Cloudflare Workers AI. The Worker route calls `streamText()` directly and pipes SSE back (no Durable Object — avoids the WebSocket/HTTP protocol mismatch between AIChatAgent and the Vue Chat client).

### Known deployment notes

- **`pnpm install`** — first deploy may fail with `Could not resolve "workers-ai-provider"` etc. The script auto-installs but the first run is slow.
- **`compatibility_flags = ["nodejs_compat"]`** — required (the `agents` dependency uses `node:async_hooks` / `node:diagnostics_channel`). The one-click script writes it automatically.
- **`/api/init/<jwt_secret>`** is a **GET** request, not POST.
- **Turnstile** — if `site_key` is unset you'll see "Verification module failed to load". After deploy, run:
  ```bash
  npx wrangler d1 execute cloud-mail --remote --command "UPDATE setting SET site_key='', secret_key='';"
  ```
  Then hard-refresh the browser (Cmd+Shift+R) to disable captcha.
- **PWA cache** — after redeploy the service worker may keep serving the old bundle. DevTools → Application → Service Workers → Unregister, then hard-refresh.

### Manual Deploy (Step-by-Step)

For finer control (custom domain, sharing an existing D1, etc.), follow the manual steps below.

1. **Clone**

```bash
git clone https://github.com/Leechuihui/cloud-mail-plus.git
cd cloud-mail-plus
```

2. **Create Cloudflare resources**

```bash
cd mail-worker
wrangler d1 create cloud-mail
wrangler kv namespace create cloud-mail-kv
wrangler r2 bucket create cloud-mail-r2
```

3. **Configure `wrangler.toml`** — fill in the IDs from step 2:

```toml
[[d1_databases]]
binding = "db"
database_name = "cloud-mail"
database_id = "<your-d1-id>"

[[kv_namespaces]]
binding = "kv"
id = "<your-kv-id>"

[[r2_buckets]]
binding = "r2"
bucket_name = "cloud-mail-r2"

[vars]
domain = '["example.com"]'
admin = "admin@example.com"
jwt_secret = "<random-string>"
```

4. **Enable CF Email Service** (optional) — onboard your domain in Cloudflare Dashboard > Email > Email Sending, then uncomment in `wrangler.toml`:

```toml
[[send_email]]
name = "EMAIL"
```

5. **Deploy**: `wrangler deploy`

6. **Initialize**: visit `https://your-worker.workers.dev/api/init/<jwt_secret>`

7. **Register** admin account using the email in your `admin` config

---

## CF Email Service API Notes

Gotchas discovered during integration (not well documented by Cloudflare):

| Field | Note |
|-------|------|
| `from` | Must be `{ name, email }` object, not `"Name <email>"` string |
| Attachment `type` | MIME type field is `type`, not `mimeType` or `contentType` |
| Attachment `disposition` | **Required** — `"attachment"` or `"inline"` |
| Delivery status | Synchronous success/failure only, no webhook callbacks (unlike Resend) |
| Recipient limit | Max 50 across to + cc + bcc |

---

## FAQ

### Sub-domain catch-all mail routing (main domain already bound to another mail service)

If your main domain (e.g. `example.com`) is already bound to another mail service (e.g. Google Workspace) and you cannot enable Email Routing on it in Cloudflare, use a sub-domain:

1. Enable Email Routing for the sub-domain `mail.example.com` in the Cloudflare Dashboard
2. Set catch-all → cloud-mail Worker
3. Add `"mail.example.com"` to `domain` in `wrangler.toml`
4. User mailboxes become `user@mail.example.com`

Note: Email Routing for the sub-domain and the main domain are independent and do not affect each other.

### IMAP/POP3/SMTP client support (Outlook/Thunderbird)

Cloudflare Workers cannot run TCP protocol services such as IMAP/SMTP. To send and receive mail in clients like Outlook, pair it with [Stalwart Mail Server](https://github.com/stalwartlabs/stalwart):

- Deployment guide: [stalwart-mail-deploy](https://github.com/AndrewYukon/stalwart-mail-deploy)
- Stalwart provides IMAP (993) + SMTP (465) to Outlook
- Cloud-Mail Plus provides sending via the External API (through CF Email Service, better reputation)
- A Mail Bridge component can tie the two together (see the [Cloudflare Workers strategy](https://github.com/AndrewYukon/stalwart-mail-deploy/tree/main/outbound-strategies/cloudflare-workers) in stalwart-mail-deploy)

---

## Original Features

All features from [maillab/cloud-mail](https://github.com/maillab/cloud-mail) are preserved:

- Multi-domain support
- Email send/receive (Cloudflare Email Routing inbound + Resend outbound)
- Attachments (R2/S3/KV storage)
- Responsive Web UI (Vue 3 + Element Plus)
- Multi-user + RBAC
- Telegram push notifications
- Turnstile CAPTCHA
- Email forwarding
- Dark mode
- i18n (Chinese/English)

---

## Tech Stack

| Component | Technology |
|------|------|
| Runtime | Cloudflare Workers |
| Backend framework | Hono.js |
| Database | Cloudflare D1 (SQLite) + Drizzle ORM |
| Cache | Cloudflare KV |
| File storage | Cloudflare R2 |
| Outbound mail | **Cloudflare Email Service** (primary) + Resend (fallback) |
| Inbound mail | Cloudflare Email Routing |
| Frontend | Vue 3 + Element Plus + Vite |

---

## Support the Project

If this project helps you, feel free to buy me a coffee ☕

<img src="doc/demo/Buy-me-a-coffee-WeChat.JPG" width="200" />

---

## License

MIT — same as the original project. See [LICENSE](LICENSE).
