# World Business 20

Top 20 global business stories, ranked and refreshed automatically every day.
Next.js (App Router) + Prisma + Postgres, deployable to Vercel with a built-in
daily cron job.

## How it works

```
NewsAPI / GNews  →  clean + normalize  →  dedupe  →  score + rank
                                                          ↓
                                         Postgres (top 20 for the day)
                                                          ↓
                                              Frontend dashboard
```

- `lib/newsSource.js` — fetches real articles from NewsAPI and (optionally) GNews.
- `lib/dedupe.js` — collapses the same story reported by multiple outlets.
- `lib/rank.js` — scores each article on recency, source credibility, and
  business/economic impact, then keeps the top 20.
- `app/api/cron/refresh/route.js` — runs the whole pipeline and saves the
  result. Called automatically once a day by Vercel Cron (see `vercel.json`),
  or manually from `/admin`.
- Every article keeps its real, original `sourceUrl` — the app never
  fabricates a link. If a provider fetch fails, that provider is skipped and
  logged; the app never falls back to fake data.

## 1. Get your API keys (required before anything works)

1. **NewsAPI** (required): free key at https://newsapi.org/register
2. **GNews** (optional, improves coverage): free key at https://gnews.io

## 2. Get a database

Any Postgres works. Easiest free options:
- https://neon.tech
- https://supabase.com
- https://railway.app

Copy the connection string it gives you.

## 3. Local setup

```bash
cp .env.example .env
# open .env and paste in: DATABASE_URL, NEWSAPI_KEY, CRON_SECRET, ADMIN_SECRET

npm install
npx prisma migrate dev --name init   # creates the tables
npm run dev                          # http://localhost:3000
```

The homepage will show an empty state until you run a refresh. Trigger one by
visiting `/admin`, entering your `ADMIN_SECRET`, and clicking
**Trigger manual refresh** — or call the endpoint directly:

```bash
curl -X POST http://localhost:3000/api/cron/refresh -H "x-cron-secret: YOUR_CRON_SECRET"
```

## 4. Deploy (Vercel)

1. Push this folder to a GitHub repo.
2. Import it at https://vercel.com/new.
3. In **Project Settings → Environment Variables**, add everything from
   `.env.example` with your real values.
4. Deploy. `vercel.json` already tells Vercel to call `/api/cron/refresh`
   every day at 06:00 UTC — edit the `schedule` cron string there if you want
   a different time.
5. After the first deploy, run `npx prisma migrate deploy` once (or trigger
   one manual refresh from `/admin` — the tables must exist first via
   migrate).

## Where things live

| What | Where |
|---|---|
| Frontend dashboard | `app/page.js` + `components/` |
| Hidden admin panel | `app/admin/page.js` (protected by `ADMIN_SECRET`) |
| Fetch pipeline | `app/api/cron/refresh/route.js` |
| Public read API | `app/api/news/route.js`, `app/api/news/history/route.js` |
| Database schema | `prisma/schema.prisma` |
| Where to add API keys | `.env` (never committed — see `.env.example`) |

## Notes on scope

- Summaries are short, in-app rewrites of the description/snippet each
  provider returns — never the full original article — with a prominent
  "Read original source" link to the publisher's own page.
- The admin panel's protection is a shared-secret header, adequate for a
  single-operator project. If you need multi-user auth, put a proper auth
  provider (e.g. NextAuth) in front of `/admin`.
- Rate limits: both NewsAPI's and GNews's free tiers allow far more requests
  than a once-a-day cron needs; retry/backoff isn't included since one
  request/provider/day won't hit them.
