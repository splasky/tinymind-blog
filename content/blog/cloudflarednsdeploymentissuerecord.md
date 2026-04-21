---
title: Cloudflare DNS / Deployment Issue Record
date: 2026-04-21T18:53:01.000Z
---


# Cloudflare DNS / Deployment Issue Record

* Date: 2026-04-22
* Goal: Switch production URL from `xxx.workers.dev` to custom domain `xxx.online`.

## 1. URL Updates in Project

Files that need changes (completed):

- `wrangler.toml` — `ORIGIN_URL`, `GOOGLE_REDIRECT_URI`
- `README.md` — Live / API Docs links
- `worker/openapi.json` — `servers[0].url`
- `worker/openapi-admin.json` — `servers[0].url`

Other places (`frontend/.well-known/`, `sitemap.xml`, `robots.txt`, `EMAIL_FROM`) were already using `xxx.online`, no changes needed.

## 2. DNS Issue: Route vs Custom Domain

### Symptoms

Clicking on Route `xxx.online` in Cloudflare dashboard shows **"This site can't be reached"** (DNS resolution fails, not 404 or 502).

### Wrong Configuration

```
workers.dev      xxx.workers.dev
Preview URLs     *-xxx.workers.dev
Route            xxx.online       ← Problem here
```

### Root Cause

Cloudflare Workers **Route** and **Custom Domain** are two different mechanisms:

| Type | Behavior |
| --- | --- |
| **Custom Domain** | Cloudflare automatically creates DNS record + SSL certificate, works immediately |
| **Route** | Just a URL pattern (e.g., `example.com/*`), **does NOT** automatically create DNS record - you must manually add a proxied (orange cloud) A/AAAA/CNAME record |

Route without corresponding DNS record → hostname cannot resolve → "can't be reached".

### Solution

Delete Route and use **Add Custom Domain**. Cloudflare will:

1. Automatically create proxied DNS record in `xxx.online` zone
2. Issue SSL certificate via Universal SSL (completes within 1-15 minutes)

Correct state should look like:

```
workers.dev      xxx.workers.dev
Preview URLs     *-xxx.workers.dev
Custom Domain    xxx.online       ← Should be this
```

### Verification

```bash
dig xxx.online +short
# Should return Cloudflare IP (e.g., 104.21.x.x / 172.67.x.x)
# No response = DNS record not created yet
```

## 3. Deployment Order (Important)

Correct workflow for switching domains:

1. **Set Custom Domain first** (not Route) and verify `dig` resolves to Cloudflare IP, browser can access. No worker redeployment needed - existing worker will respond on new hostname automatically.
2. **Google Cloud OAuth**: Go to Google Cloud Console and add `https://xxx.online/api/google/callback` to authorized redirect URIs. **If skipped, Google login will break after deployment.**
3. **Deploy worker**: Make new `ORIGIN_URL` and `GOOGLE_REDIRECT_URI` values take effect - internal URLs, OAuth callbacks, email links will point to new domain.

### Why "Deploy First" Can't Fix "can't be reached"

"can't be reached" is a DNS/network layer issue, unrelated to code. Worker responds regardless of hostname bound, so deploying new code doesn't help. Fix the domain binding first, then deploy.

## 4. Email Sending Domain (Related Note)

`EMAIL_FROM = "noreply@xxx.online"` sends via Resend. xxx.online must be verified in Resend Domains:

- SPF record: `v=spf1 include:_spf.resend.com ~all`
- DKIM record: CNAME provided by Resend dashboard
- (Recommended) DMARC record: `v=DMARC1; p=none;`

These DNS records are added in Cloudflare `xxx.online` zone. If emails go to spam/ bounce, check domain status in Resend dashboard.

## 5. Lessons Learned

- Cloudflare Workers custom domain binding → **Always use Custom Domain, never Route**, unless you have specific pattern-routing needs and manage DNS yourself.
- Domain switch checklist: OAuth redirect URIs, email sending domain DNS, sitemap/robots, openapi servers, README links, wrangler environment variables.
- Verify DNS (`dig`) and OAuth settings before deployment - saves time vs. fixing broken things after deployment.
