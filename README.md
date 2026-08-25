# Site Scout: Retail Trade-Area Assessment Tool

A single-file, zero-backend web tool that scores any US location 0–100 for retail site quality to simulate the way real site-selection teams think about it. 
Enter an address or click the map; the tool pulls Census demographics for the surrounding tracts, maps nearby anchor retailers, and explains exactly why the area scored what it did.

**Live data sources (all free, no paid API keys required):**

| Source | Used for |
|---|---|
| US Census ACS 5-year API | Income, education, age, household mix, homeownership, home values (auto-detects the newest vintage; currently 2020–2024, the most recent tract-level data that exists — released Jan 2026) |
| FCC Area API | Point → Census tract lookup (15 sample points across a 2.5-mi ring) |
| Census TIGERweb | Tract boundaries (drawn on map) + land area for density |
| OpenStreetMap Overpass | Anchor brand locations (~110-brand catalog: grocery, QSR, casual dining, big box, off-price, pharmacy, c-store, fitness, auto) + food/retail POI density |
| OSM Nominatim | Address geocoding |

## How to use it:

1. Type an address (or any place name) and hit **Assess** or just click a spot on the map.
2. Pick a **scoring profile**. The same signals are re-weighted per retail archetype:
   - **General retail vitality**: balanced weights
   - **Premium grocer**: education + income dominate (the Whole Foods / Trader Joe's logic)
   - **Family QSR**: family households + road access + co-tenancy (the Chick-fil-A logic)
   - **Coffee/café**: density + education + access (the Starbucks logic)
   - **Value retail**: *inverts* income/education percentiles (the Dollar General logic: underserved = opportunity)
3. Read the **score stamp** and the **"Why this score"** table: each signal shows its raw value, its national percentile, its weight in the chosen profile, and the points it contributed. Switching profiles re-scores instantly with no new API calls.
4. The map shows 1/3/5-mile rings, dashed tract boundaries, colored anchor-brand pins (green = premium tier, blue = mid, orange = value), and small gray dots for all other food/retail POIs within 2 miles.
   
## Scoring model:

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

Percentile benchmarks default to approximate 2020–2024 national tract-level distributions hard-coded in `BENCH`. 
Click **Recalibrate benchmarks** in the app to compute exact percentiles from all ~85,000 US tracts via the Census API (one-time, ~15–30s, uses ~51 API calls and only works if `ACS_KEY` is set): results apply immediately, persist in the browser, and a paste-ready `BENCH` block is printed to the console to hardcode them for every device. Density stays approximate (the ACS API carries no land area). Edit `BENCH` directly if metro-relative is needed rather than national scoring.

## Known limitations (by design, v1)

- **ACS margins of error** — tract-level 5-year estimates are noisy in small tracts; treat single-digit differences as meaningless.
- **OSM completeness** — anchor coverage for national chains is good but not perfect; a missing pin means missing OSM data, not proof of a void.
- **Road class ≠ traffic counts** — real AADT would need state DOT data (NCDOT publishes it; good Phase 4 add).
- **Rings, not drive times** — isochrones need an OpenRouteService key (free tier); wire-in point is `sampleTracts()`.
- **Shared free endpoints** — Overpass/Nominatim are community servers (~1 req/sec fair use). Fine for personal use; if it ever errors, wait a few seconds and retry. The Census API officially requires a free key now (api.census.gov/data/key_signup.html and needs to be pasted into `ACS_KEY` at the top of the script); keyless calls still work intermittently but shouldn't be relied on. If `api.census.gov` is blocked on network, requests automatically fall back to a public CORS proxy.
- **US only** (Census data).

## Architecture (for future edits)

Everything lives in `index.html`:

- `ANCHORS`: the ~110-brand catalog; one line per chain with a strict classifier regex (`rx`), a loose Overpass fetch pattern (`q`), and a `tier`. Add any chain by adding one line; `BRAND_RX` builds itself.
- `BENCH`: percentile benchmark breakpoints
- `PROFILES`: archetype weights (must sum to 1.0; `inv:` lists signals to invert)
- `sampleTracts → fetchACS → fetchTractGeo → fetchPOIs → fetchRoads`: the data pipeline
- `aggregate()` / `scoreArea()`: pure functions: population-weighted aggregation, then weighted-percentile scoring
- `drawMap()` / `render()`: Leaflet layers and the explainability panel

*Research tool, not underwriting. Signals inspired by documented site-selection practices of US retailers (Costco, Chick-fil-A, Whole Foods, Aldi, Dollar General, et al.).*
