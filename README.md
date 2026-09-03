# GymReadyMap Japan

An independent, athlete-built map of HYROX-ready training gyms across Japan,
with station-by-station equipment data for all eight HYROX stations.

**Live:** [gymreadymap.com](https://gymreadymap.com/)

Listing and verification are free, permanently, and are never pay-to-play.

---

## What this is

The official HYROX gym finder lists paying affiliates and does not publish
comparative equipment data. GymReadyMap is independent and equipment-verified:
for each gym it records which of the eight stations you can actually train
(SkiErg, sled push, sled pull, row, burpee broad jump, farmers carry,
sandbag lunges, wall balls).

Since v5.5 it also records **what people searched for and did not find** —
see [`docs/DEMAND_SCHEMA.md`](docs/DEMAND_SCHEMA.md).

---

## Architecture

Deliberately boring. One file, no build step, no framework, no server.

```
Google Sheet (GymReadyMap_Japan_Database_2026)
        │  published as CSV
        ▼
   index.html  ──fetch──▶  parseCSV() ──▶ rowToGym() ──▶ gyms[]
        │                                                  │
        │                                          Leaflet map + sidebar
        │                                          + mobile bottom sheet
        │
        └──▶ localStorage cache (5 min TTL)
        └──▶ FALLBACK_GYMS (hardcoded, last resort)
```

**Stack**
- Leaflet.js 1.9.4 + Leaflet.markercluster 1.5.3 (CDN)
- CartoDB Dark Matter tiles (raster — requires an API key as of Aug 2026;
  key is in the `L.tileLayer` URL in `index.html`, domain-restricted)
- Barlow / Barlow Condensed / Noto Sans JP (Google Fonts)
- No bundler, no npm, no dependencies to install

**Hosting**
- GitHub repo `jammacabeo/gymreadymap`, auto-deploys from `main` to Netlify
- DNS managed at Netlify (nameservers delegated to `nsone.net`),
  domain registered at Namecheap
- Analytics: Netlify built-in

---

## Data flow

1. `boot()` calls `loadGyms()`.
2. `loadGyms()` checks `localStorage` (`hyrox_gym_cache_v2`, 5-minute TTL).
   Cache hit → return immediately.
3. Cache miss → `fetchCSVWithRetry()` (3 attempts, 1.5s × attempt backoff).
4. `parseGymCSV()` → `parseCSV()` (handles quoted fields, embedded commas,
   escaped quotes) → `buildColIndex()` → `rowToGym()` per row.
5. Rows without `name`, `lat` or `lng` are dropped.
6. On total failure, `FALLBACK_GYMS` renders and an error banner shows.

The **Refresh** button (`manualRefresh()`) clears the cache and refetches.

### Column mapping is deliberately tolerant

`COL_ALIASES` maps each internal field to several accepted header spellings.
`normHeader()` strips BOM, non-breaking and zero-width spaces, case, and all
separators — so `Ward JP`, `ward-jp` and `WARD_JP` all resolve to the same field.

This means **renaming a column in the sheet usually will not break the map**,
as long as the new name matches one of the listed aliases.

Station data is read two ways, preferring the first:
- Eight per-station boolean columns (`skierg`, `sled_push`, …), or
- A single `has` column, either `1,0,1,...` or `10110011`

### Bilingual fields

Any field with a `_jp` twin (`name_jp`, `ward_jp`, `address_jp`, `hours_jp`,
`notes_jp`, `tags_jp`) is resolved through `locFld()`, which returns the JP
value in JP mode and **falls back to English when the JP cell is blank**.
A missing translation never blanks the UI.

`parseGymCSV()` logs a JP column diagnostic to the console on every load —
it reports whether each `_jp` column was found and how many rows are filled.

---

## Key concepts in the code

| Thing | Where | Note |
|---|---|---|
| `I18N` | top of script | All UI strings, EN + JP. Add keys in pairs. |
| `t(key)` | | Lookup with EN fallback |
| `locFld(gym, field)` | | Localised gym field with EN fallback |
| `PREF_JP` | | All 47 prefectures, romaji key → kanji |
| `prefLabel(gym)` | | Strips `-to`/`-fu`/`-ken`/`-do` suffixes, returns localised |
| `PREF_REGION` | | Prefecture → macro region, accepts romaji *and* kanji |
| `getFiltered()` | | Single source of truth for what is displayed |
| `isMobile()` | | `window.innerWidth <= 767` |

**Do not rename `locFld` to `L`** — `L` is Leaflet's global namespace.

### Mobile

Below 767px the desktop sidebar, filter bar and view tabs are hidden, and a
bottom sheet takes over with three snap states (`peek` / `half` / `full`).
`initMobile()` is idempotent — it guards on `mobileInited` because resize
fires constantly as mobile browser chrome hides and shows, and re-running the
DOM moves would stack duplicate touch listeners.

---

## Editing and deploying

No local toolchain required. All work happens through the GitHub web UI.

**Feature work**
1. Edit `index.html` on a feature branch (`feat/...`)
2. Open a PR — Netlify posts a deploy preview URL as a comment
3. Test the preview on **desktop and phone** (geolocation needs HTTPS,
   and most real traffic is mobile)
4. Merge, delete the branch
5. Create a GitHub Release tagged `vX.Y` and update `CHANGELOG.md`

**CSS-only changes** may go straight to `main`.

**Content changes** (adding a gym, marking one verified) happen in the Google
Sheet and appear within 5 minutes — no deploy needed.

### Before merging, check

- Console is clean apart from the expected `[data]` and `[i18n]` logs
- JP mode renders: toggle to JP and confirm no romaji leaks into gym cards
- Mobile sheet drags and the search input does not zoom the page on tap
  (inputs are 16px on mobile specifically to prevent iOS zoom-on-focus)

---

## Repo layout

```
index.html                 Everything: markup, styles, logic, fallback data
CHANGELOG.md               Version history
README.md                  This file
docs/DEMAND_SCHEMA.md      Demand event types and field semantics
favicon.png
og-image.png
```

---

## Current state

- 134 gyms mapped, 13 verified
- Regions covered: Kanto, Kansai, Chubu

### Known issues

- **All gyms flagged `official`.** The sheet reports 134 official / 0 HYROX
  Ready, so the Type filter has a dead option and the blue "ready" marker ring
  never renders. Sheet data issue.
- **Gym data is client-side only.** Search engines can only index the homepage.
  Region and equipment-type static pages are blocked on having enough verified
  equipment data to make them worth generating.
- **`DEMAND_ENDPOINT` is empty** as of v5.5 — demand events stay in the
  visitor's browser until the webhook is wired up.

---

## Positioning

Independent and athlete-run. Not affiliated with, endorsed by, or speaking for
HYROX. "HYROX" is a trademark of its owner and is used here descriptively.

Avoid "HYROX Japan" as primary branding, and do not imply the ability to grant
or fast-track affiliation.
