# Changelog

All notable changes to GymReadyMap Japan.
Versions are reflected in the header sub-line (`Intelligence map for HYROX® races · Japan · vX.Y`)
and should be tagged in GitHub on merge to `main`.

Format loosely follows [Keep a Changelog](https://keepachangelog.com/).

---
## [5.6] — 2026-08-22

Demand capture goes live. v5.5 built the collection layer but pointed it nowhere.

### Added
- **Apps Script webhook** receiving `flushDemand()` POSTs, appending to a new
  `demand_log` tab on `GymReadyMap_Japan_Database_2026`. Fixed 18-column
  schema, auto-created on first write.
- **Email notification to `jam@gymreadymap.com` on `waitlist` events only.**
  Zero-result searches are volume data, not inbox events.
- Shared-token gate (`?t=`) on the endpoint, plus a formula-injection guard —
  a search for `=IMPORTXML(...)` lands as text, not a live formula.

### Changed
- `DEMAND_ENDPOINT` populated. Events now leave the browser.

---

## [5.5] — 2026-08-22

Demand capture layer. The map records what people searched for and did **not** find.

### Added
- **Search box** in the desktop sidebar and inside the mobile bottom sheet.
  Live client-side filter across `name`, `name_jp`, `ward`, `ward_jp`,
  `prefecture`, `address`, `address_jp`, `tags`, `tags_jp`.
  Both scripts match: `Setagaya`, `世田谷` and `sled` all resolve.
- **"Near me" geolocation button** (both layouts). Sorts results by great-circle
  distance, shows km per result, flies the map to the user, and flags when no
  gym falls within `NEAR_RADIUS_KM` (15 km).
- **Zero-result demand card.** Rendered when nothing matches the current
  filters/search, or when "near me" finds nothing in range. Contains an inline
  waitlist form: area (required), stations the user cannot train (optional),
  email (optional).
- **Event logging** to `localStorage` under `gymready_demand_log` (archive,
  capped at 500) and `gymready_demand_queue` (pending POST).
  Event types: `search_zero`, `near_me`, `near_me_blocked`, `waitlist`.
  See `docs/DEMAND_SCHEMA.md`.
- **`exportDemand()`** console helper — returns the local archive as CSV.
- `data-i18n-ph` attribute support in `applyI18n()` for input placeholders.
- Full EN/JP strings for all new UI.

### Changed
- `getFiltered()` now applies `matchSearch()` and, when near-me is active,
  attaches `_dist` to each gym and sorts ascending.
- `renderGyms()` and `renderMobileSheet()` prepend the demand card when
  `demandCardNeeded()` is true, and render a distance chip when `_dist` is set.
- `clearAllFilters()` / `mobileClearFilters()` also reset `searchQuery`.
- `boot()` calls `flushDemand()` to retry events queued in a previous visit.

### Known gaps
- `DEMAND_ENDPOINT` is intentionally empty. Until it points at a live webhook,
  events reach nobody but the visitor's own browser.
- Geolocation untested on mobile. Desktop testing returned `code: 3` (timeout) —
  expected, as desktop relies on wifi triangulation.
- Sheet currently reports 134 official / 0 HYROX Ready, which makes the Type
  filter a dead control and means the blue "ready" marker ring never renders.
  Data issue, not a code issue.

---

## [5.4] — 2026-08-20

### Fixed
- `PREF_JP` expanded from four Kanto prefectures to all 47. Osaka, Aichi and
  every other non-Kanto gym now render in kanji in JP mode instead of romaji.
- `prefLabel()` strips `-to` / `-fu` / `-ken` / `-do` / `-prefecture` suffixes
  however they are separated, fixing "Osaka-fu" rendering literally.

### Added
- Full SEO/OG/Twitter meta and JSON-LD reframed Japan-wide (Tokyo, Osaka, Nagoya)
  rather than Tokyo-only.
- `doc_title` i18n key so `<title>` switches with UI language.

---

## [5.3] and earlier

Not retroactively documented. Broad strokes:

- Bilingual (EN/JP) UI layer with `locFld()` helper and graceful EN fallback
  for empty JP cells.
- Google Sheets published-CSV backend with 5-minute `localStorage` cache,
  3-retry fetch with backoff, and a hardcoded `FALLBACK_GYMS` array.
- Schema-tolerant CSV column mapping via `COL_ALIASES` + `normHeader()`.
- Leaflet map with marker clustering, custom station-count marker icons,
  desktop popup card, mobile bottom sheet + detail panel + filter drawer.
- Deep links (`?gym=ID`).
- Region derivation from prefecture (`PREF_REGION`), supporting both romaji
  and kanji sheet values.

---

## Versioning conventions

- Bump the minor version for any user-visible feature or fix.
- Update the header sub-line in `index.html` in the same commit.
- Tag `main` after merge: `v5.5`, `v5.6`, …
- One PR per version where practical; the PR description is the draft changelog entry.
