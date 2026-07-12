# AGENTS.md

<!-- INSFORGE:START -->
## InsForge backend

This project uses [InsForge](https://insforge.dev): an all-in-one, open-source Postgres-based backend (BaaS) that gives this app a database, authentication, file storage, edge functions, realtime, an AI model gateway, and payments through one platform.

- **Project:** **kindle-dashboard** (API base `https://<your-project>.<region>.insforge.app`)
- **Skills:** these InsForge skills are installed for supported coding agents. Reach for them before implementing any InsForge feature instead of guessing the API:
  - `insforge`: app code with the `@insforge/sdk` client (database CRUD, auth, storage, edge functions, realtime, AI, email, and Stripe payments).
  - `insforge-cli`: backend and infrastructure via the `insforge` CLI (projects, SQL, migrations, RLS policies, storage buckets, functions, secrets, payment setup, schedules, deploys).
  - `insforge-debug`: diagnosing failures (SDK/HTTP errors, RLS denials, auth and OAuth issues) and running security or performance audits.
  - `insforge-integrations`: wiring external auth providers (Clerk, Auth0, WorkOS, Better Auth, etc.) for JWT-based RLS, or the OKX x402 payment facilitator.
  - `find-skills`: discovering additional skills on demand.
- **Credentials:** app code reads keys from `.env.local`; the CLI reads `.insforge/project.json`. Never hardcode or commit keys.

Key patterns:

- Database inserts take an array: `insert([{ ... }])`.
- Reference users with `auth.users(id)`; use `auth.uid()` in RLS policies.
- For storage uploads, persist both the returned `url` and `key`.
<!-- INSFORGE:END -->

## This deployment

Gagan's personal deployment of thecodedose/kdashboard on a Kindle Paperwhite
7th gen (PW3, 2015; 1072x1448, 8bpp mxcfb framebuffer, xres_virtual=1088,
line_length=1088).

Full project reference (setup story, gotchas, rebuild cheat sheet) lives in
Notion (private page, owner access only):
https://app.notion.com/p/39bc456b7c3d8149bc77e31cc93fddf8

- Function endpoints: `https://<your-project>.<region>.insforge.app/functions/<slug>`.
- Exception: the SSE events endpoint only works on the function host directly,
  `https://<your-project>.function2.insforge.app/kindle-dashboard-events`. Going
  through `.../functions/kindle-dashboard-events` on the main host times out.
- Secrets that must exist beyond the generated ones: `INSFORGE_API_KEY`
  (functions require it; the reserved `API_KEY` secret does NOT satisfy it),
  `DASHBOARD_TIMEZONE=America/New_York` (functions default to Asia/Kolkata,
  which made the dashboard show tomorrow's date), and optional
  `WEATHER_LATITUDE` / `WEATHER_LONGITUDE` / `WEATHER_UNIT` (default NYC / C).
- Telegram bot webhook is registered against the telegram-webhook function and
  locked to the owner chat ID (see AGENTS.local.md).

## Local divergence from upstream

- `msync()` calls on the /dev/fb0 mmap were removed from
  `kindle/native/src/kindle_dashboard.cpp`. On the PW3 kernel `msync(MS_SYNC)`
  on the framebuffer mapping blocks forever. Symptom: dashboard renders to the
  save-pgm but the screen never updates, and the device feels frozen because
  the renderer holds an exclusive EVIOCGRAB on the touch input. Do not
  reintroduce msync before an upstream fix lands.
- The MEAL PLANNER home tile was replaced with a WEATHER tile
  (`drawWeatherTile`). Weather comes from Open-Meteo in
  `functions/kindle-dashboard-data.ts` (`fetchWeather`), is null on failure,
  and is deliberately excluded from the payload version hash so weather
  changes do not trigger SSE refresh events. The meal planner detail screens
  and Telegram meal commands still exist; only the home tile is gone.
- The steps/calories rings were replaced with a wide STOCKS card
  (`drawStocksCard`, Yahoo Finance chart API, symbols from `STOCK_SYMBOLS`,
  default AAPL,NVDA,MSFT,SPY), and the 75-day challenge tile with two news
  cards (`drawNewsCard`): AI NEWS (TechCrunch AI RSS, `AI_NEWS_FEED`) and
  LATEST NEWS (BBC World RSS, `NEWS_FEED`). Like weather, all three are
  fetched per request with 6s timeouts, degrade to empty on failure, and are
  excluded from the version hash. The challenge/health screens and Telegram
  commands still work; they just have no home-screen entry point.
- Headlines are sanitized to printable ASCII server-side (`sanitizeHeadline`);
  double quotes become single quotes because the renderer's flat JSON key
  scanner can misread escaped quotes inside string values as object keys.
- The 5x7 bitmap font in `glyphRow()` only covers listed characters; anything
  else renders as a fallback box. Glyphs for ' ? $ & ( ) ; were added for
  headlines. Add new glyphs there if other characters show up as boxes.

## Build and test loop

- Cross-compile: `make -C kindle/native extension-zig ZIG=$(which zig)`
  (zig via Homebrew). Output package: `kindle/native/build/kindle-dashboard-kual.tar.gz`.
- Fast preview without the Kindle: `make -C kindle/native local`, then
  `build/kindle-dashboard-local --render <payload.json> --dump-pgm out.pgm
  --dump-size 1072x1448`, convert with `sips -s format png` and view. Fetch a
  live payload with the `x-dashboard-read-token` header for realistic previews.
- Deploying to the Kindle: macOS TCC blocks terminal access to
  `/Volumes/Kindle` ("Operation not permitted"), so the user copies files via
  Finder. Binary swaps go to `extensions/kindle-dashboard/bin/kindle-dashboard`.
- Logs to request from the user after a Kindle-side failure:
  `documents/kindle-dashboard-native.log`,
  `documents/kindle-dashboard-kual-action.log`, and the rendered screenshot
  `documents/kindle-dashboard-last-render.pgm`.
- "Refresh Once" renders a single static snapshot and exits; taps on it do
  nothing (and actually land on the hidden Kindle UI underneath). Interactive
  mode is "Start Dashboard", which keeps the loop alive with input grabbed.
  There, EXIT is a hotspot at the top right, starting ~66px down from the
  screen top (taps in the very top pixel rows miss it).
- Redeploy a single function with
  `npx @insforge/cli functions deploy <slug> --file functions/<slug>.ts`.
  `functions deploy` updates in place; if a bootstrap run reports
  SLUG_ALREADY_IN_USE, the function already exists and the remaining ones can
  be deployed individually.
