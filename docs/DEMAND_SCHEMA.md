# Demand capture — event schema

**Introduced:** v5.5 (2026-08-22)

The map shows what exists. This layer records what people looked for and did
**not** find. That gap data is the part no one else has: HYROX can replicate a
directory of their own gyms, but not the record of athletes searching for
training access and coming up empty.

---

## Why this exists

Two things it is meant to answer:

1. **Where is there unmet HYROX training demand in Japan?**
   Which wards and prefectures do people search for, or geolocate in, where
   there is no gym within reach.
2. **Who is asking?** JP-language vs EN-language demand shapes who any future
   gym outreach should be aimed at, and in which language.

It is *not* general web analytics. Netlify's built-in analytics covers traffic.
This covers intent that went unmet.

---

## How it works

```
user action
    │
    ▼
logDemand(type, extra)
    │
    ├──▶ gymready_demand_log     localStorage, capped at 500, permanent archive
    │                            (read via exportDemand())
    │
    └──▶ gymready_demand_queue   localStorage, pending POST
             │
             ▼
        flushDemand()  ──POST──▶  DEMAND_ENDPOINT
```

`flushDemand()` runs after every event and once on `boot()` (to retry anything
queued from a previous visit).

**If `DEMAND_ENDPOINT` is empty**, nothing is sent anywhere. Events accumulate
in the visitor's own browser only, and are invisible to the site owner.
This was the state at v5.5 ship.

**The POST is `mode: 'no-cors'`** because Google Apps Script web apps do not
return usable CORS headers. That means the response cannot be read, so the
queue is cleared optimistically. `gymready_demand_log` is the safety net —
it retains a local copy regardless.

Payload shape: `{ "events": [ …event objects… ] }`

---

## Event types

### `search_zero`
A text search returned zero results.

Fired **1.4 seconds after the user stops typing**, and only when the query is
≥ 2 characters. The debounce exists so that typing one word letter by letter
does not fire five bogus events for its prefixes.

Only fires when the result set is genuinely empty — a search that finds gyms
logs nothing. Absence of results is the signal.

| Field | Meaning |
|---|---|
| `query` | Exactly what the user typed, trimmed |

### `near_me`
Geolocation succeeded. Logged **whether or not** gyms were found nearby —
a successful lookup near a well-served ward is still useful density data.

| Field | Meaning |
|---|---|
| `lat`, `lng` | Rounded to 2 decimal places (~1 km) |
| `nearest_km` | Distance to the closest gym after filters, 1 dp. `''` if none |
| `results` | Number of gyms matching current filters |
| `unserved` | `1` if `nearest_km` is null or > 15 km, else `0` |

### `near_me_blocked`
Geolocation failed.

| Field | Meaning |
|---|---|
| `code` | `1` permission denied · `2` position unavailable · `3` timeout |

`code: 3` is common on desktop, which relies on wifi triangulation rather than
GPS. Do not read desktop timeouts as users refusing consent — `code: 1` is
the refusal signal.

### `waitlist`
Someone filled in the inline form on the zero-result card. **The highest-value
event by a wide margin** — a search is a data point, a waitlist submission is a
person raising their hand.

| Field | Meaning |
|---|---|
| `area` | Free text, required. Ward, city or prefecture |
| `email` | Optional |
| `stations_needed` | Semicolon-separated, **always English** regardless of UI language |
| `query` | Search active when the form was submitted, if any |
| `lat`, `lng` | From geolocation if it ran, else `''` |

Station names are deliberately logged in English so the sheet stays analysable
in one vocabulary rather than splitting across scripts.

---

## Fields on every event

| Field | Meaning |
|---|---|
| `type` | Event type above |
| `ts` | ISO 8601 UTC timestamp |
| `sid` | Random 8-char session id, `sessionStorage`, dies with the tab |
| `lang` | UI language at the time: `en` or `jp` |
| `mobile` | `1` if viewport ≤ 767px |
| `filters` | `type=…;region=…;status=…;cov=…` — filter state when fired |
| `referrer` | `document.referrer`, first 120 chars |

---

## Privacy

- **No cookies, no fingerprinting, no third-party trackers.**
- `sid` is random per tab and not linked to any identity. It groups events
  within one visit and is gone when the tab closes.
- Coordinates are rounded to ~1 km — enough for ward-level demand mapping,
  not enough to locate anyone.
- Email is optional, collected only on the waitlist form, and the form states
  it is used solely to contact the person about gyms in their area.
- Geolocation always goes through the browser's own permission prompt.

If email collection ever expands beyond this, revisit whether a proper privacy
policy page is needed under Japan's APPI.

---

## Reading the data

### From a browser (own device only)

```js
exportDemand()              // CSV string, printed inline
copy(exportDemand())        // CSV onto the clipboard
console.table(JSON.parse(localStorage.getItem('gymready_demand_log')))
```

`copy()` returns `undefined` — that is normal, it does not return a value.

**This only ever shows your own device's events.** It is a debugging tool,
not a reporting one.

### Gotchas when analysing

- **Volume before conclusions.** Six zero-result searches in Sapporo is an
  anecdote with a number attached. Do not build a claim on thin data — it will
  cost more credibility than saying nothing.
- **Filter state matters.** A `search_zero` fired with `status=verified`
  active means "no *verified* gym matched", not "no gym exists". Always read
  `filters` alongside the result.
- **Own traffic pollutes.** Testing from your own devices writes real rows.
  Consider filtering by `sid` or timestamp when analysing early data.
- **Desktop `near_me` is coarse.** Wifi-based location can be city-level or
  worse. Weight `mobile: 1` rows more heavily for geographic claims.
- **`unserved` uses the 15 km threshold** (`NEAR_RADIUS_KM`). If that constant
  ever changes, historical rows keep the old definition. Note the change here.

---

## Configuration

The receiving end is a Google Apps Script web app bound to
`GymReadyMap_Japan_Database_2026` (Extensions → Apps Script → "GymReadyMap
Demand Webhook"). It appends to the `demand_log` tab and emails
`jam@gymreadymap.com` on `waitlist` events only.

The `?t=` token on the endpoint URL is **public** — it ships in this repo. It
blocks drive-by scanners hitting a bare `/exec`, nothing more. To rotate:
change `TOKEN` in the Apps Script, then `DEMAND_ENDPOINT` here, then
**Deploy → Manage deployments → ✏️ → Version: New version** (never "New
deployment" — that mints a different URL and orphans the live site).

---

## Status

- [x] Event capture and local archive
- [x] CSV export helper
- [x] `DEMAND_ENDPOINT` wired to an Apps Script webhook
- [x] `demand_log` tab on `GymReadyMap_Japan_Database_2026`
- [x] Email notification on `waitlist` events only
- [ ] Enough accumulated volume to say anything with confidence
