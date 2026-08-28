# Comment to DM

Instagram keyword automation: reply to comments containing your keywords and
automatically DM those people your link. Built to replace the one slice of
Manychat that actually gets used.

- **Auto-reply to comments** matching keywords, on all posts / specific posts /
  posts from now on
- **Auto-DM the commenter** with your link — no prior conversation needed
- **Inbox** — an activity log of every automation run, plus read-only DM threads

Single-tenant (one Instagram account). Instagram only; the provider boundary in
`lib/instagram/` is where TikTok would slot in later.

---

## How the DM works

Meta has a first-class mechanism for this called a **Private Reply**: you POST
to `/{ig-user-id}/messages` with `recipient: { comment_id }` and Meta delivers a
DM to the commenter with no existing conversation. Its limits are enforced by
Meta and shape the whole design:

| Rule | How this app handles it |
|---|---|
| One private reply per comment, ever | `comment_event.comment_id` is UNIQUE — duplicate webhooks cannot double-send |
| Must be sent within 7 days | Events carry `expires_at`; past it they are marked `dead`, not retried |
| Account must be Professional | Checked at login by Instagram itself |
| 2 messaging calls/sec per account | The retry sweeper paces itself at ~550ms |

---

## Setup

### 1. Instagram account

Convert to a **Professional** account (Business or Creator) in the Instagram
app: Settings → Account type and tools. Personal accounts cannot use this API.

### 2. Meta app

1. Create an app at [developers.facebook.com](https://developers.facebook.com) →
   **Other** → **Business**.
2. Add the **Instagram** product, then set up **Instagram Login** (not Facebook
   Login).
3. Under **App roles → Roles**, add your Instagram account as an
   **Instagram Tester**.
4. Accept the invite in Instagram: Settings → Apps and websites → Tester invites.

**No App Review is needed** while the app stays in Development Mode and you are
using your own tester account. That is the entire reason this is a weekend
project rather than a six-week one.

### 3. Database

Create a Postgres database at [neon.tech](https://neon.tech) and copy the
connection string.

### 4. Environment

```bash
cp .env.example .env.local
```

Fill in `INSTAGRAM_APP_ID` and `INSTAGRAM_APP_SECRET` from the Meta dashboard,
invent a `WEBHOOK_VERIFY_TOKEN`, paste your `DATABASE_URL`, and generate the
three secrets:

```bash
openssl rand -hex 32
```

`TOKEN_ENC_KEY` must be exactly 64 hex characters — it is the AES key for the
stored Instagram token.

### 5. Migrate

```bash
pnpm db:migrate
```

### 6. Deploy

```bash
pnpm dlx vercel deploy --prod
```

Set the same environment variables in the Vercel dashboard, with `APP_URL` set
to your deployed HTTPS URL.

### 7. Point Meta at the webhook

In the Meta dashboard → Instagram → **Webhooks**:

- Callback URL: `https://your-app.vercel.app/api/webhooks/instagram`
- Verify token: your `WEBHOOK_VERIFY_TOKEN`
- Subscribe to the **comments** and **messages** fields

Also add `https://your-app.vercel.app/api/auth/instagram/callback` under
**Instagram Login → OAuth redirect URIs**. It must match `APP_URL` exactly.

### 8. Use it

Open the app, click **Continue with Instagram**, create an automation, publish.

---

## Local development

```bash
pnpm dev
```

Webhooks need a public HTTPS URL, so comment events will not reach a
`localhost` server. Either use a Vercel preview deployment, or tunnel:

```bash
pnpm dlx ngrok http 3000
```

then set `APP_URL` to the ngrok URL and point the Meta webhook at it.

---

## Architecture

```
Instagram comment
      ↓  webhook (field: "comments")
POST /api/webhooks/instagram
      ↓  verify signature → INSERT raw event → 200 (fast)
      ↓  after() — runs once the response is already sent
  match automation → public comment reply → private-reply DM
      ↓
Vercel Cron (*/2 min) retries anything left pending or failed
```

The webhook handler acknowledges before doing any work, because Meta throttles
and eventually disables slow endpoints. Durability comes from the
`webhook_event` table plus the cron sweeper rather than from a queue service.

| Path | Role |
|---|---|
| `app/api/webhooks/instagram/route.ts` | Receives events. The critical file. |
| `lib/automation/matcher.ts` | Keyword and scope matching. Pure, fully unit-tested. |
| `lib/automation/processor.ts` | Fires the reply and the DM; writes the activity log. |
| `lib/instagram/client.ts` | Graph API calls. |
| `lib/instagram/oauth.ts` | Business Login and the 60-day token lifecycle. |
| `db/schema.ts` | Drizzle schema. |

### Cron jobs

| Schedule | Job | Why |
|---|---|---|
| every 2 min | `process-events` | Retries failed sends within the 7-day window |
| every 15 min | `sync-inbox` | Backfills DM threads and refreshes the post picker |
| daily 04:00 | `refresh-token` | Renews the 60-day token — **if this stops, everything silently stops** |

---

## Commands

```bash
pnpm dev          # dev server
pnpm test         # unit tests
pnpm typecheck    # tsc --noEmit
pnpm lint         # eslint
pnpm db:generate  # generate a migration after editing db/schema.ts
pnpm db:migrate   # apply migrations
pnpm db:studio    # browse the database
```

---

## Notes and gotchas

- **Keyword matching defaults to whole-word**, so `link` does not fire on
  `linkedin`. Switch to "anywhere in the comment" per automation if you want the
  looser behaviour.
- **Comment replies rotate** between the variants you write. Posting an
  identical reply every time is the fastest way to get flagged as spam.
- **Self-comments are ignored**, otherwise the bot replies to its own replies
  forever.
- **The Inbox is read-only.** Replying from here would run into Meta's 24-hour
  messaging window and its `human_agent` rules; the Instagram app has none of
  those problems.
- **DM history beyond 20 messages** exists only in your own database — the
  Instagram API caps `GET /me/conversations` at the 20 most recent messages per
  thread, which is why every incoming message is mirrored locally.
