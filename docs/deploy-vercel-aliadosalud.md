# Deploy OpenReply on Vercel (AliadoSalud / TOM-76)

Fork: [TOMIBU/openreply](https://github.com/TOMIBU/openreply)  
Upstream: [diwenne/openreply](https://github.com/diwenne/openreply)  
Linear: [TOM-76](https://linear.app/tomasbujan/issue/TOM-76/forkear-y-deployar-openreply-en-vercel-aliadosalud-para-reemplazar)

This document lists every environment variable needed for a Vercel web deploy for AliadoSalud, and the steps to create/link the project. **Do not put secret values in this repo or in this PR.** Tomi loads secrets in the Vercel dashboard (or equivalent) after the project exists.

Out of scope for TOM-76: custom domain, ManyChat migration, channel connection, loading secret values, running the always-on DM worker.

## Prerequisites (outside this PR)

1. **PostgreSQL** connection string reachable from Vercel (public host). Neon, Railway, or similar.
2. **Redis** that supports BullMQ blocking commands (not HTTP-only Redis). Public URL for Vercel.
3. **Email for magic-link login**: Resend API key + verified sender, **or** an SMTP `EMAIL_SERVER` URL (then Resend is not needed).
4. **Instagram connection path** (choose one later; not required to create the empty Vercel project):
   - **Zernio** (credentials go in the app Settings UI, not env vars), or
   - **Direct Meta app** (env vars in the Meta table below).
5. A separate **always-on worker** process (`npm run worker`) that shares `DATABASE_URL`, `REDIS_URL`, and `ENCRYPTION_KEY` with the web app. Vercel hosts the web app + crons only; DMs will not send until a worker is running elsewhere. That worker is **not** created in this ticket.

## Vercel project setup

1. Team: **aliadosalud**.
2. Link Git repo: `TOMIBU/openreply` (default branch).
3. Framework: Next.js (auto-detected).
4. Build: Vercel picks up `vercel-build` from `package.json` when present (`prisma generate && prisma migrate deploy && next build`). Ensure `DATABASE_URL` is set before a successful production build that runs migrations.
5. Existing root `vercel.json` already registers daily crons (`/api/cron/refresh-tokens`, `attach-next-reel`, `snapshot-followers`). No extra cron config is required in this PR.
6. After first successful deploy, set `NEXTAUTH_URL` to the default Vercel URL (`https://<project>.vercel.app`) if it was left as a placeholder.
7. Do **not** add a custom domain in this ticket.

Expected: the first deploy may fail until required env vars exist. That is OK for TOM-76; Tomi loads them next.

## Environment variables (web / Vercel)

Never commit values. Marked **secret** must be treated as credentials.

### Core (required for a working web app)

| Name | Required | Secret | Purpose | Where to get it |
| --- | --- | --- | --- | --- |
| `NEXTAUTH_URL` | Yes | No | Public HTTPS base URL (Auth.js, tracked links, webhooks targets). | After deploy: `https://<project>.vercel.app` (update if domain changes). |
| `NEXTAUTH_SECRET` | Yes | Yes | Auth.js session/crypto secret (min 16 chars). | Generate: `openssl rand -base64 32` |
| `CRON_SECRET` | Yes (for cron routes) | Yes | Protects Vercel cron API routes (`Authorization: Bearer …`). | Generate: `openssl rand -base64 32` |
| `ENCRYPTION_KEY` | Yes | Yes | 64 hex chars (32 bytes). Encrypts Instagram/provider tokens at rest. **Same value on web and worker.** | Generate: `openssl rand -hex 32` |
| `DATABASE_URL` | Yes | Yes | PostgreSQL connection string (Prisma + runtime). Public URL from Vercel. | Neon / Railway / other Postgres provider |
| `REDIS_URL` | Yes | Yes | Redis for BullMQ queues and rate limits. Must support blocking commands. | Redis Cloud / Railway Redis / compatible host |

### Email (required for sign-in)

Use **either** Resend **or** SMTP.

| Name | Required | Secret | Purpose | Where to get it |
| --- | --- | --- | --- | --- |
| `RESEND_API_KEY` | Yes if not using `EMAIL_SERVER` | Yes | Sends magic-link login emails via Resend. | [resend.com](https://resend.com) API keys |
| `EMAIL_FROM` | Recommended | No | From address; must be on a domain verified in Resend (when using Resend). | Your verified sender, e.g. `OpenReply <login@yourdomain.com>` |
| `EMAIL_SERVER` | Optional alternative | Yes | SMTP URL; if set, takes over and Resend is not needed. URL-encode special chars in user/password. | Your mail provider SMTP (`smtps://…:465` or `smtp://…:587`) |

### Optional access control

| Name | Required | Secret | Purpose | Where to get it |
| --- | --- | --- | --- | --- |
| `ALLOWED_EMAILS` | Optional | No (PII) | Comma-separated allowlist of emails that may sign in. Unset = open signup on the public URL. | Decide AliadoSalud team emails |

### Direct Meta Instagram (only if not using Zernio-only)

Leave unset if all accounts use Zernio (Zernio keys live in Settings, not env).

| Name | Required | Secret | Purpose | Where to get it |
| --- | --- | --- | --- | --- |
| `META_GRAPH_API_VERSION` | Optional (default `v25.0`) | No | Meta Graph API version string. | Meta docs / app dashboard |
| `INSTAGRAM_APP_ID` | Yes for direct Meta OAuth | No | Instagram Login app id (not the Facebook App ID on Basic settings). | Meta app → Instagram → API setup |
| `INSTAGRAM_APP_SECRET` | Yes for direct Meta OAuth | Yes | Instagram app secret. | Same page → Show |
| `FACEBOOK_APP_SECRET` | Yes for webhook signature verify | Yes | Facebook/app secret used (with IG secret) to verify webhooks. | Meta app → Settings → Basic → App secret |
| `WEBHOOK_VERIFY_TOKEN` | Yes for Meta webhook verify | Yes | Shared verify token Meta calls on subscribe. | Generate any strong random string; paste same value into Meta webhook config |

### Optional tuning (defaults are fine)

| Name | Required | Secret | Purpose | Default / notes |
| --- | --- | --- | --- | --- |
| `DB_POOL_MAX` | Optional | No | Postgres pool size. | Default `3` |
| `COMMENT_POLL_INTERVAL_MS` | Optional (worker) | No | Worker comment sweep interval. | `300000` (5 min) |
| `COMMENT_POLL_MAX_PER_SWEEP` | Optional (worker) | No | Max new comments acted on per sweep. | `30` |
| `COMMENT_POLL_LOOKBACK_HOURS` | Optional (worker) | No | How far back sweeps look. | `72` |
| `FOLLOW_RECHECK_DELAYS_MS` | Optional (worker) | No | Comma-separated delays for follow-gate rechecks. | `20000,40000` |
| `FOLLOW_RECHECK_ACK_MESSAGE` | Optional (worker) | No | Opt-in ack message text for follow recheck. | Unset = disabled |
| `HEALTH_STUCK_QUEUE_WAITING` | Optional | No | Health check threshold for stuck queue. | See `/api/health` |

Worker-only vars above are listed so Tomi knows the full surface; they belong on the worker host, not necessarily on Vercel.

## After Tomi loads env vars

1. Redeploy the Vercel project.
2. Confirm `https://<project>.vercel.app` loads and magic-link sign-in works.
3. Point Meta OAuth redirect / webhook (or Zernio webhook) at that URL — **later tickets**, not TOM-76.
4. Provision the always-on worker with the same `DATABASE_URL`, `REDIS_URL`, and `ENCRYPTION_KEY`.

## Config shipped in repo (no secrets)

- Root `vercel.json`: daily cron paths only (already upstream).
- `package.json` script `vercel-build`: generate Prisma client, run migrations, then `next build`.

No additional Vercel config files were required for this ticket beyond documentation.
