# Changelog

All notable changes to GymReadyMap Japan.
Versions are reflected in the header sub-line (`Station-by-station gym data · Japan · vX.Y`)
and should be tagged in GitHub on merge to `main`.

Format loosely follows [Keep a Changelog](https://keepachangelog.com/).

---
## [5.9] — 2026-09-03

Restores the basemap after CARTO began charging admission.

### Fixed
- CARTO started requiring an API key on its raster basemap endpoints in late
  August 2026, stamping a repeated "API KEY REQUIRED" watermark across every
  anonymous tile request. The map kept working — markers, clustering, sheet
  data and the demand pipeline were never affected — but every tile was
  defaced. The `dark_all` tile URL now carries a free `key` parameter,
  domain-restricted to `gymreadymap.com`, `www.gymreadymap.com` and
  `*.netlify.app`.

### Changed
- Attribution aligned with CARTO's published form: "OpenStreetMap" spelled out
  rather than "OSM", and the CARTO link points at `/attributions` rather than
  the homepage. Sections 6.b and 15.e of the Basemaps terms require these
  notices stay visible and CARTO is now actively enforcing that.

### Notes
- **This is a stopgap.** CARTO has said the raster basemaps are being retired
  and is considering stopping data updates to them, in which case Japanese
  cartography will silently freeze while the vector product moves on. The free
  tier is 5M tile requests/month, counted in tiles rather than map loads —
  nowhere near a constraint at current traffic.
- Migration candidate is `protomaps-leaflet`, which renders vector tiles inside
  Leaflet and would leave `markerClusterGroup`, `iconCreateFunction`, popups
  and every category colour untouched. A MapLibre GL move would keep Dark
  Matter's exact look but rewrites the entire marker layer.
- The `*.netlify.app` wildcard does match Netlify's
  `deploy-preview-N--sitename` host format, so PR previews render clean.
- The key is public by necessity — it ships in client-side HTML. Domain
  restriction, not secrecy, is what protects the quota.

---
## [5.8] — 2026-08-24
 
Stops the UI advertising a category that contains nothing.
 
### Changed
- Controls pointing at the **HYROX® Ready** set now hide themselves when that
  set is empty. `renderGyms()` computes the count once and calls the new
  `syncReadyUI()`, which toggles `.cat-empty` on every element tagged
  `js-ready-ui`: the Ready filter chip (filterbar, mobile chip bar, mobile
  drawer), the blue legend row, the Ready header stat, and the Official header
  stat. Header falls back to Mapped Gyms + Verified.
- Official is hidden alongside Ready because while Ready is zero, Official is
  just a second copy of Mapped Gyms.
- No flag, no migration. The first unaffiliated gym added to the sheet brings
  all four controls back on the next render.
- `h-official` is now derived as `gyms.length - readyCount` rather than a second
  `.filter()` pass, so the two stats cannot drift apart.
  
### Fixed
- `prefLabel()` was stripping the `-do` in "Hokkaido" as if it were a suffix,
  rendering the prefecture as "Hokkai" in both languages. The full string is now
  checked against `PREF_JP` before any suffix stripping is attempted.
  Regression from the suffix handling introduced in 5.4. `prefKey()` was never
  affected — it does not strip `-do` — so region assignment was always correct.
- `manualRefresh()` could drop Ready to zero while the Ready filter was still
  armed, leaving an empty map with no visible control to undo it.
  `syncReadyUI()` calls `clearAllFilters()` in that case.
  
### Notes
- Hiding uses a `!important` class rather than inline `style.display` so the
  `.hstat-sec` (1120px) and `.hstat` (767px) breakpoints added in 5.7 continue
  to govern layout untouched once the count goes above zero.
- The marker ring needed no change. `g.official?'#e8ff00':'#4da6ff'` is correct;
  the blue branch simply never executes against the current dataset.
- Closes the "dead Type filter" item logged under Known gaps in 5.5.

---
## [5.7] — 2026-08-24
 
Navigation layer. The map becomes a site.
 
### Added
- **Header nav** (`Map · Guide · About`) as plain text links across all three
  pages, replacing a header that had no way out of the map. Active state
  underlined in accent. Full EN/JP via new `nav_*` i18n keys.
- **`guide.html`** — standalone bilingual page covering the eight HYROX stations
  with equipment requirements and Japan availability, marker/filter/search
  reference, what verification means, and a 7-question FAQ marked up as
  `FAQPage` schema. Links the official HYROX Singles rulebook as the authority
  on weights and movement standards.
- **`about.html`** — verification methodology, independence and no-pay-to-play
  statement, data sources, contact. `AboutPage` + `Organization` schema.
- **App footer strip** in the desktop sidebar and the mobile bottom sheet:
  Guide / About / Contact links plus the HYROX® trademark disclaimer.
  Placed here rather than in `.map-legend`, which is `display:none` below
  900px and would have hidden the notice from most mobile traffic.
- **`sitemap.xml`** and **`robots.txt`**. Without these the two new pages are
  orphaned — nothing on the map linked to them and nothing told Google.
- Favicon and `apple-touch-icon` links on the content pages, which were built
  from scratch and inherited nothing from `index.html`. Declared as `/favicon.png`
  rather than the relative path `index.html` uses, so it resolves from any depth.
  
### Changed
- **Tagline** `Intelligence map for HYROX® races · Japan` →
  `Station-by-station gym data · Japan` (JP: `8種目対応ジムを探す · 日本`),
  now driven by a `tagline` i18n key rather than hardcoded. States the
  differentiator plainly instead of describing the format.
- Header breakpoints for the ~230px the nav adds: sub-line hides at 1200px,
  Official and HYROX® Ready counters hide at 1120px, and below 767px the header
  wraps to two rows — logo and nav on the first, language toggle and buttons on
  the second.
  
### Fixed
- `#h-official` / `#h-ready` sit on the stat *numbers*, not their wrappers, so
  collapsing them by id hid the digits and left orphan `OFFICIAL` and
  `HYROX® READY` captions. Now scoped to a `.hstat-sec` class on the wrapper.
- Logo, submit button and stat labels wrapping to multiple lines in the
  1000–1200px band. `white-space:nowrap` added; stat collapse moved from
  1000px to 1120px so counters drop before the squeeze rather than during it.
- Mobile header briefly rendered as three rows, costing map height on the
  layout that gets the most use. Reduced to two.
  
### Notes
- Content pages hold both languages in the DOM and toggle on the shared
  `gymready_lang` key, so language persists when crossing between the map and a
  content page. One URL per page, no `hreflang`.
  
### Known gaps
- Sheet still reports 134 official / 0 HYROX Ready. The Type filter, the blue
  legend entry and the blue marker ring remain non-functional. Data issue,
  carried over from 5.5, not addressed here.
- `demand_log` gid in the waitlist notification email still points at `#gid=0`.

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
