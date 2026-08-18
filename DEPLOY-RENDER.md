# Deploying Mail Automation to Render

The whole app ships as **one Render Web Service plus one Render Postgres**.
[render.yaml](render.yaml) is currently set to the **free tier** for testing —
read [Free-tier limits](#free-tier-limits) before trusting it with real
campaigns, and [Going to production](#going-to-production) when you are ready.
The Express server serves the built React dashboard from the same origin, so
there is no second hostname, no CORS config, and no `VITE_API_URL` to keep in
sync — the dashboard's relative `/api/...` calls just work.

```
  https://<your-service>.onrender.com
        |
        |--  /            -> web/dist  (React SPA)
        |--  /assets/*    -> hashed JS/CSS, cached 1 year
        \--  /api/*       -> Express API + node-cron scheduler
                              |
                              \-- Render Postgres (private network)
```

---

## 1. Push the repo

```bash
git add -A
git commit -m "Add Render deployment"
git push origin main
```

## 2. Create the Blueprint

1. Render Dashboard → **New +** → **Blueprint**
2. Connect `github.com/ranjithui/mail-Automation`, pick the `main` branch
3. Render reads [render.yaml](render.yaml) and proposes:
   - Web service `mail-automation` (Free)
   - Postgres `mail-automation-db` (Free)
4. It will prompt for the `sync: false` variables. **Leave them all blank for
   now** and click Apply — the app boots fine without Google or Stripe, and
   Google credentials can be entered later in the UI.

First deploy takes ~5 minutes. `JWT_SECRET` and `ENCRYPTION_KEY` are generated
by Render automatically.

## 3. Verify

```bash
curl https://<your-service>.onrender.com/api/health
# {"ok":true,"database":"connected","time":"..."}
```

On free, the first request after an idle period takes **30-60 seconds** — that
is the instance waking up, not a hang.

Then open the site and register — the first account created becomes the
instance admin.

## 4. Wire up Google OAuth

Gmail sending needs an OAuth client pointed at the live callback URL.

1. Google Cloud Console → **APIs & Services** → enable the **Gmail API**
2. **Credentials** → Create OAuth client ID → *Web application*
3. Authorised redirect URI — exactly this, with your real hostname:
   ```
   https://<your-service>.onrender.com/api/mail-accounts/oauth/callback
   ```
4. Put the client ID/secret into the app: **Settings → Google OAuth** (they are
   stored in the database and take effect immediately), or set
   `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` in Render → Environment.

> `GOOGLE_REDIRECT_URI` needs no value on Render. The server derives it from
> `RENDER_EXTERNAL_URL`, which Render injects with the service's real address.

## 5. Stripe (optional — skip to stay on the Free plan)

Set in Render → Environment:

| Variable | Where it comes from |
| --- | --- |
| `STRIPE_SECRET_KEY` | Dashboard → Developers → API keys |
| `STRIPE_PRICE_STARTER` / `_PRO` / `_BUSINESS` | one recurring monthly Price each |
| `STRIPE_WEBHOOK_SECRET` | from the endpoint created below |

Add a webhook endpoint pointing at:
```
https://<your-service>.onrender.com/api/webhooks/stripe
```
Copy its signing secret into `STRIPE_WEBHOOK_SECRET`.

---

## How the deploy is wired

| Step | Command | Why |
| --- | --- | --- |
| Build | `npm install --include=dev && npm run db:generate && npm run build` | `--include=dev` is required: `NODE_ENV=production` would make npm skip `typescript`, `vite` and `prisma`, which are all build-time tools here |
| Start | `npm run db:deploy && npm run start` | `prisma db push` runs at **start**, not build — a build container cannot reach the database's private network address |
| Health | `/api/health` | pings Postgres; a 503 fails the deploy rather than shipping a broken instance |

### Schema changes

This project has no `prisma/migrations` directory, so deploys use
`prisma db push`, which syncs `schema.prisma` onto the database. It refuses to
run if a change would drop data — in that case create a real migration
(`npm run db:migrate`) and switch `db:deploy` to `prisma migrate deploy`.

<a name="free-tier-limits"></a>
## Free-tier limits

Three of these change behaviour, not just performance. Plan your testing around
them.

**1. The scheduler does not run on schedule.**
Free instances sleep after ~15 minutes with no HTTP traffic. `node-cron` — which
fires daily campaign sends and all three follow-up levels — is frozen while the
instance sleeps, and a sleeping instance is not woken by the passage of time,
only by an incoming request. So a campaign due at 09:00 goes out whenever
somebody next loads the site.

For testing, either trigger runs manually from the dashboard, or keep the
service awake by pinging `/api/health` every ~10 minutes from an external
uptime service. A free instance kept awake all month uses ~730 of your 750
free instance-hours, so one such service is the most you can do this way.

On wake, [recoverInterruptedRuns()](server/src/services/scheduler.ts) picks up
runs that were mid-flight when the instance went down, so nothing is lost — it
is only delayed.

**2. Attachments are erased.**
No disk can be mounted on free, so `UPLOAD_DIR=uploads` writes to the container
filesystem. Every redeploy, restart and wake-from-sleep wipes it. The database
rows survive, so a campaign will still reference an attachment whose file is
gone and fail at send time. Test attachments in one sitting, or leave them out
of free-tier testing.

**3. The database is deleted after 30 days.**
Render drops free Postgres instances 30 days after creation, with an email
warning first. Upgrade it or export anything you want to keep.

Also: the free Postgres allows 97 connections and the build container has less
RAM, so a build can occasionally OOM where it succeeds locally. Re-running the
deploy usually clears it.

<a name="going-to-production"></a>
## Going to production

Four edits to [render.yaml](render.yaml), then push:

```yaml
databases:
  - plan: basic-256mb              # was: free

services:
  - plan: starter                  # was: free  -> always-on, scheduler works

    disk:                          # add this block: attachments survive deploys
      name: uploads
      mountPath: /var/data
      sizeGB: 1

    envVars:
      - key: UPLOAD_DIR
        value: /var/data/uploads   # was: uploads
```

Roughly $7/mo for the web service plus $7/mo for Postgres. Existing data and
generated secrets are preserved across the plan change, so mailboxes stay
connected.

Migrating attachments is the one thing that does not carry over — files written
before the disk existed are already gone, so re-upload them after the switch.

### Rotating secrets

Never change `ENCRYPTION_KEY` after mailboxes are connected. It encrypts the
stored Gmail refresh tokens (AES-256-GCM); a new key makes every one of them
undecryptable and all connected mailboxes must be re-authorised.

### Custom domain

Add it under Render → Settings → Custom Domains, then set `APP_URL` and
`API_URL` to `https://your-domain.com` (they otherwise default to the
`.onrender.com` address) and add the new callback URL in Google Cloud Console.
