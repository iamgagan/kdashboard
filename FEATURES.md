# kdashboard Feature Roadmap

This roadmap is grounded in the current project shape described in this repo:
a jailbroken Kindle/KUAL dashboard, native e-ink renderer, InsForge edge
functions, Telegram input, HealthKit sync, and lightweight data cards. It does
not assume uninspected implementation details beyond the documented repo
structure and the PW3 constraints in `AGENTS.md`.

## Research Signals

Existing Kindle/e-ink dashboard projects tend to cluster around a few durable
patterns:

- Server-side or backend-composed data is easier to maintain than doing heavy
  parsing on Kindle hardware.
- Weather, agenda, tasks, RSS/news, air quality, and home automation are the
  most common always-on cards.
- Older Kindle browsers are fragile, so native rendering or pre-rendered
  bitmaps are safer than complex HTML/CSS on-device.
- Refresh strategy matters as much as feature choice: dashboards should update
  only when visible state changes, on coarse intervals, or after explicit user
  actions.
- KUAL/WebLaunch-style installs work best when the Kindle-side surface is a
  thin launcher plus simple shell scripts, with most logic off-device.

## Quick Wins

### 1. Calendar Agenda Card

Show today's and tomorrow's events from an ICS or Google Calendar feed, with
all-day events collapsed into a short line.

Implementation note: Parse ICS in an InsForge edge function or scheduled job,
normalize to the dashboard timezone, and include only the next 24-36 hours in
the dashboard payload. Refresh every 15-30 minutes while awake; agenda changes
do not need sub-minute updates.

### 2. Morning Briefing Card

Render a compact "today at a glance" block: weather, first calendar event,
unfinished chores, meal plan, and one top news headline.

Implementation note: Compose this server-side from existing dashboard data so
the Kindle renderer only draws text. Keep the brief to 3-5 short lines to avoid
font overflow and unnecessary redraws.

### 3. Kindle Battery Reporting

Post the Kindle battery level with each device fetch and show it in a small
status strip; alert via Telegram when it drops below a threshold.

Implementation note: Read battery percentage from Kindle system power files in
`launch-dashboard.sh` or a small helper, send it as a header/query param to the
read endpoint, and store only the latest value. Refresh on every fetch is fine
because the value is tiny; Telegram alerts should be rate-limited.

### 4. Manual Refresh Status

Add a visible "last updated" timestamp and a stale-state indicator when the
payload is older than the expected refresh window.

Implementation note: Include `generated_at` and per-source timestamps in the
payload. Draw "STALE" only after a generous threshold, such as 2x the intended
polling interval, to avoid false alarms during Wi-Fi sleep or backend timeouts.

## Medium Effort

### 5. Multi-Page Dashboard

Split the dashboard into touch-flippable pages: Planner, Health, News/Markets,
and System.

Implementation note: Reuse existing Kindle touch input and hotspot patterns.
Keep each page mostly static and redraw only on page change or payload version
change. Store the selected page locally so a restart returns to the last page.

### 6. RSS Reading Queue

Show a short queue of saved articles or RSS items, with source, title, age, and
read/unread state.

Implementation note: Fetch RSS server-side with timeouts and printable-ASCII
sanitization. Poll feeds every 30-60 minutes in a scheduled job; the Kindle
should consume a compact cached result instead of fetching feeds directly.

### 7. Grocery-from-Meal-Plan Helper

Generate a grocery list delta from planned meals and recipes, then let Telegram
confirm additions.

Implementation note: Use existing recipes, meal plans, and Telegram flows.
Start deterministic by diffing recipe ingredients against the active grocery
list; add AI cleanup later for unit normalization. No e-ink refresh is needed
until the list changes.

### 8. Evening Review Prompt

Send a Telegram prompt at night for open chores, habit streaks, and missed
targets; update the backend from replies.

Implementation note: Implement as an InsForge scheduled function plus existing
Telegram parser actions. The Kindle only needs the next payload version after
the backend state changes.

### 9. Home Automation Status Strip

Show a small read-only strip for key home states such as front door, thermostat,
lights, or server health.

Implementation note: Prefer a Home Assistant webhook or REST sensor endpoint if
available. Keep this read-only at first and refresh every 5-15 minutes; avoid
frequent polling from the Kindle itself.

## Stretch

### 10. Photo of the Day / Screensaver Mode

Let Telegram upload or choose a daily image, dither it to Kindle-friendly
monochrome, and show it as a corner image or full-screen idle page.

Implementation note: Do all image resize, contrast, rotation, and dithering
server-side or in a local build script. Emit PGM/bitmap dimensions that match
the PW3 renderer expectations and avoid decoding heavy formats on-device.

### 11. Offline Snapshot Cache

Keep the last known-good payload and render it when Wi-Fi or the backend is
unavailable, with a visible stale marker.

Implementation note: Save the last successful JSON payload under the Kindle
documents or extension directory. On network failure, render cached data and
back off polling exponentially to reduce battery drain.

### 12. Lightweight Plugin/Card Registry

Define a small card contract so new backend data sources can be added without
rewriting the renderer layout every time.

Implementation note: Keep this deliberately small: stable card IDs, title,
lines, optional meter values, and priority. Avoid rich nested schemas until
several cards prove the abstraction is worth it.

## Power, Battery, and Refresh Tradeoffs

Kindle hardware is excellent for static dashboards and poor for constantly
changing dashboards. The practical goal is "ambiently current," not live.

- Prefer backend-composed, compact payloads over on-device API calls. This keeps
  TLS, parsing, RSS cleanup, image processing, and retries off the Kindle.
- Poll slowly by default. Planner/status data can usually refresh every 5-15
  minutes; weather, news, and RSS can refresh every 30-60 minutes; stocks can be
  cached unless the dashboard is explicitly used during market hours.
- Use version hashes to avoid unnecessary redraws. If the visible state has not
  changed, skip the framebuffer update and only update internal timers.
- Separate "fetch cadence" from "screen cadence." A background fetch can update
  cached JSON, but the e-ink screen should redraw only when visible content
  changes, a page is tapped, or a manual refresh is requested.
- Prefer partial redraws only when they are already reliable on the target
  Kindle. Full redraws are visually cleaner but cost more battery and create
  flash; partial redraws reduce disturbance but can ghost if overused.
- Treat Wi-Fi as a battery cost. Batch network requests through one endpoint,
  keep timeouts short, and back off when the backend is unavailable.
- Make stale data explicit. A Kindle dashboard should never burn battery trying
  to be real-time; it should show when the last successful update happened.
- Avoid animation entirely. E-ink-friendly interfaces should use page changes,
  high-contrast text, icons, meters, and coarse state transitions.

## Suggested Build Order

1. Calendar Agenda Card
2. Morning Briefing Card
3. Manual Refresh Status
4. Kindle Battery Reporting
5. Multi-Page Dashboard
6. RSS Reading Queue
7. Grocery-from-Meal-Plan Helper
8. Evening Review Prompt
9. Offline Snapshot Cache

That order turns kdashboard from a status board into a daily planner first, then
hardens the device loop, then expands into richer ambient context.

## References

- `terminalbytes/kindle-dashboard`: Kindle PW7 weather dashboard using
  server-side TypeScript, Playwright, Sharp image conversion, weather data, and
  battery display: https://github.com/terminalbytes/kindle-dashboard
- `niutech/kindle-dashboard`: Kindle 3 browser dashboard with date/time,
  to-do list, Google News, air quality, weather, and coarse refresh intervals:
  https://github.com/niutech/kindle-dashboard
- `HimbeersaftLP/KindleDashboard`: jailbroken Kindle/WebLaunch home automation
  dashboard with browser limitations called out: https://github.com/HimbeersaftLP/KindleDashboard
- `PaulFreund/WebLaunch`: KUAL-based Kindle Touch/Paperwhite URL launcher for
  full-screen web apps: https://github.com/PaulFreund/WebLaunch
- `KOReader/koreader`: mature e-ink reader project with plugin architecture,
  Kindle support, and e-ink-optimized UI principles:
  https://github.com/KOReader/koreader
- MobileRead Kindle developer/jailbreak ecosystem, commonly used for KUAL,
  MRPI, and device-specific jailbreak context: https://www.mobileread.com/forums/forumdisplay.php?f=150
