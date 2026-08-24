# Site Scout — Retail Trade-Area Assessor

A single-file, zero-backend web tool that scores any US location 0–100 for retail site quality — the way real site-selection teams think about it. Enter an address or click the map; the tool pulls Census demographics for the surrounding tracts, maps nearby anchor retailers, and explains exactly why the area scored what it did.

**Live data sources (all free, no API keys required):**

| Source | Used for |
|---|---|
| US Census ACS 5-year API | Income, education, age, household mix, homeownership, home values |
| FCC Area API | Point → Census tract lookup (15 sample points across a 2.5-mi ring) |
| Census TIGERweb | Tract boundaries (drawn on map) + land area for density |
| OpenStreetMap Overpass | Anchor brand locations (Costco, Chick-fil-A, Trader Joe's, 25 brands) + food/retail POI density |
| OSM Nominatim | Address geocoding |

## Deploy to GitHub Pages (5 minutes)

1. **Create the repo.** On github.com → New repository → name it (e.g. `site-scout`) → Public → Create.
2. **Add the file.** Click "uploading an existing file" (or *Add file → Upload files*) and upload `index.html` and this `README.md`. Commit to `main`.
3. **Enable Pages.** Repo → *Settings → Pages* → under "Build and deployment", Source = **Deploy from a branch**, Branch = **main**, folder = **/ (root)** → Save.
4. **Wait ~1 minute**, then open `https://<your-username>.github.io/site-scout/`.

That's it — no build step, no secrets, no server. To update, just edit `index.html` in the GitHub web editor and commit.

**Test locally first (optional):** `python3 -m http.server` in the folder, then open `http://localhost:8000`. (Opening the file directly via `file://` also works for most browsers since all APIs are CORS-enabled.)

## How to use it

1. Type an address (or any place name) and hit **Assess** — or just click a spot on the map.
2. Pick a **scoring profile**. The same signals are re-weighted per retail archetype:
   - **General retail vitality** — balanced weights
   - **Premium grocer** — education + income dominate (the Whole Foods / Trader Joe's logic)
   - **Family QSR** — family households + road access + co-tenancy (the Chick-fil-A logic)
   - **Coffee/café** — density + education + access (the Starbucks logic)
   - **Value retail** — *inverts* income/education percentiles (the Dollar General logic: underserved = opportunity)
3. Read the **score stamp** and the **"Why this score"** table: each signal shows its raw value, its national percentile, its weight in the chosen profile, and the points it contributed. Switching profiles re-scores instantly with no new API calls.
4. The map shows 1/3/5-mile rings, dashed tract boundaries, colored anchor-brand pins (green = premium tier, blue = mid, orange = value), and small gray dots for all other food/retail POIs within 2 miles.

## Scoring model

`score = Σ (signal percentile × profile weight)`, rescaled if any signal is unavailable.

| Signal | Type | How it's computed |
|---|---|---|
| Household income | Quantitative | Household-weighted median across sampled tracts, vs. national tract benchmarks |
| Bachelor's+ share | Quantitative | Pooled bachelor's+ / population 25+ |
| Population density | Quantitative | Pooled tract population / pooled land area (TIGERweb) |
| Family households | Quantitative | Family HH / total HH |
| Homeownership | Quantitative | Owner-occupied / occupied units |
| Anchor co-tenancy | Quali-quantitative | Unique catalog brands ≤5 mi; premium-tier brands count 2× ("the Chick-fil-A effect") |
| Road access | Proxy (qualitative) | Best OSM road class within 0.4 mi (highway=100 … local-only=30) — a stand-in for AADT |
| Retail vitality | Quantitative | Count of food/retail POIs ≤2 mi |

Percentile benchmarks are approximate national tract-level distributions (ACS ~2023) hard-coded in `BENCH` — edit them there if you want metro-relative scoring later.

## Known limitations (by design, v1)

- **ACS margins of error** — tract-level 5-year estimates are noisy in small tracts; treat single-digit differences as meaningless.
- **OSM completeness** — anchor coverage for national chains is good but not perfect; a missing pin means missing OSM data, not proof of a void.
- **Road class ≠ traffic counts** — real AADT would need state DOT data (NCDOT publishes it; good Phase 4 add).
- **Rings, not drive times** — isochrones need an OpenRouteService key (free tier); wire-in point is `sampleTracts()`.
- **Shared free endpoints** — Overpass/Nominatim are community servers (~1 req/sec fair use). Fine for personal use; if it ever errors, wait a few seconds and retry. The keyless Census API allows 500 calls/day per IP; paste a free key into `ACS_KEY` at the top of the script to lift it.
- **US only** (Census data).

## Roadmap (Phases from our build plan)

- [x] **Phase 1–3**: map + geocoding + ring sampling + ACS demographics + POI overlay + profile scoring + explainability
- [ ] **Phase 4a**: drive-time isochrones (OpenRouteService)
- [ ] **Phase 4b**: daytime population (Census LODES, precomputed static layer)
- [ ] **Phase 4c**: growth trend (compare two ACS vintages)
- [ ] **Phase 4d**: real AADT layer for your state, shareable URLs (`?lat=&lon=&profile=`)

## Architecture (for future edits)

Everything lives in `index.html`:

- `ANCHORS` / `BRAND_RX` — the brand catalog and its Overpass regex (add brands here)
- `BENCH` — percentile benchmark breakpoints
- `PROFILES` — archetype weights (must sum to 1.0; `inv:` lists signals to invert)
- `sampleTracts → fetchACS → fetchTractGeo → fetchPOIs → fetchRoads` — the data pipeline
- `aggregate()` / `scoreArea()` — pure functions: population-weighted aggregation, then weighted-percentile scoring
- `drawMap()` / `render()` — Leaflet layers and the explainability panel

*Research tool, not underwriting. Signals inspired by documented site-selection practices of US retailers (Costco, Chick-fil-A, Whole Foods, Aldi, Dollar General, et al.).*
