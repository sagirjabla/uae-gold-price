# AGENTS.md

UAE Gold Price — a bilingual (English + Arabic, RTL) SEO-focused live gold rate site for Dubai/UAE with a gold calculator. Built on TanStack Start (SSR) and deployed on Vercel.

## Architecture

- **Pages** (`src/routes/`): `index.tsx` (home), `gold-calculator.tsx`, `gold-price-chart.tsx`, `silver-rate.tsx`, `about.tsx`, `gold-rate.$slug.tsx` (all keyword landing pages), `sitemap[.]xml.ts` (server route).
- **Content** (`src/content/pages.ts`): `SITE` config (name, canonical URL) and `LANDING_PAGES` — one entry per keyword cluster (city / karat / unit / currency / product). Routing, sitemap, nav and link grids derive from this list. Add pages here rather than creating new route files.
- **Pure maths** (`src/lib/gold-math.ts`): karat purities, units (gram, tola 11.6638 g, sovereign 8 g, troy oz, kg), currencies, AED peg 3.6725, calculator logic. Shared by client and server.
- **Price service** (`src/lib/prices.server.ts`, server-only): fetches spot from gold-api.com and FX from open.er-api.com with module-level caches (public data only), records daily OHLC rows, seeds history once from Yahoo Finance COMEX futures (`GC=F`, `SI=F`). All DB calls go through `safeDb` so pages still render if the database is unavailable.
- **Server functions** (`src/lib/prices.functions.ts`): `fetchLiveRates`, `fetchHistory`. They dynamically import `prices.server` inside handlers so server code never enters the client bundle (TanStack import protection blocks `*.server.*` in client). Shared types live in `src/lib/price-types.ts`.
- **Cron endpoint** (`src/routes/api.cron.record-price.ts`): records the price and runs backfill if needed. Protected by `CRON_SECRET`. Triggered daily by Vercel Cron (`vercel.json`) and every 15 minutes by `.github/workflows/record-price.yml`.
- **Database**: Neon Postgres (Vercel integration, `DATABASE_URL`) via Drizzle `neon-http` (`db/schema.ts`, `db/index.ts`). Table `daily_prices` keyed by UAE date (Asia/Dubai). Migrations in `db/migrations/` — generate with `pnpm db:generate --name <name>`, apply with `pnpm db:migrate`; never edit applied migrations.
- **Components** (`src/components/site/`): header/footer, `rates.tsx` (hero, tables, FAQ, link grids, `useAutoRefresh`), `gold-calculator.tsx`, `price-chart.tsx` (hand-rolled SVG line chart with crosshair tooltip).

## SEO conventions

- Every route sets `head()` via `pageHead()` in `src/lib/seo.ts` (title, description, canonical, OG/Twitter, JSON-LD). Use `faqJsonLd` + `breadcrumbJsonLd` on content pages.
- One `<h1>` per page; descriptions include the live rate for CTR.
- CDN caching is set in the root route `headers` (`Vercel-CDN-Cache-Control` 120 s + SWR). Keep pages server-rendered.
- Canonical base URL is `SITE.url`; update it (and `public/robots.txt`) when a custom domain is added.

## Styling

Theme variables in `src/styles.css`: ivory background, `ink` (dark navy) for hero/footer bands, `gold`/`gold-deep` accents, `up`/`down` for price change. Fonts: Fraunces (headings) and Manrope (body) from Google Fonts. Use `tabular` class for numbers. Hero image is `public/img/hero-gold.png`, served as a static file from `/img/` (optimised, e.g. WebP).

## Notes

- Rates are purity-based (spot × purity), not jeweller board rates; copy says "indicative" — keep that disclaimer.
- Dates/times are formatted with explicit `Asia/Dubai` timezone to keep SSR and hydration consistent.
- Do not add `.dark` theme without defining dark variables (removed intentionally).

## i18n

- English at `/`, Arabic at `/ar/...`. Each page has an `ar.*` twin route file; shared views live in `src/components/pages.tsx`. Strings in `src/lib/i18n.ts`, keyword pages in `src/content/pages.ts` (`en` + `ar` labels).
- Every page emits canonical + hreflang (en, ar, x-default); sitemap lists both languages. Root shell sets `lang`/`dir` from the path.
- Set `VITE_SITE_URL` in Vercel and update `public/robots.txt` for your domain.
