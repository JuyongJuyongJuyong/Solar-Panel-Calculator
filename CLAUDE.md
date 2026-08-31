# Global Rooftop Solar Potential Calculator

Free, browser-only web tool estimating rooftop solar potential, savings, and CO2 impact.
No paid GIS software, no backend server, no typing where a tap will do.
Full background/reasoning: see PROJECT_SUMMARY.md in this repo.

## Non-negotiable constraints
- Zero-cost stack only: Leaflet.js, Turf.js, SunCalc.js, static hosting. No paid APIs in the default path.
- Minimize user input: prefer 1-tap icon choices over typed fields wherever possible.
- Never claim precision the data doesn't support — every result ships with an honest uncertainty range and cited data sources, not a single confident number.

## Architecture: tiered by what free data exists at the user's location
- **Tier 1** (open government LiDAR, e.g. US/NL/FR): use USGS 3DEP / AHN / IGN LiDAR HD for measured roof geometry, OpenEI Utility Rate DB (US) for real tariffs, PVGIS/NSRDB-grade irradiance. If the region has 1:1 net metering, self-consumption ratio is irrelevant to annual $ — don't ask for it.
- **Tier 2** (ML building footprints, no LiDAR): Overture Maps' unified building-footprint dataset (already merges Google Open Buildings + Microsoft Footprints + OSM — see Data-source implementation notes for how this is actually served), NASA POWER / Open-Meteo satellite radiation (ensemble both).
- **Tier 3** (weakest building/power data): minimal-tap fallback, widest stated uncertainty range, manual overrides available.
- Elevation correction (Copernicus DEM GLO-30, free & global) and dust/aerosol correction (CAMS/MERRA-2, applies to Sahel + Indo-Gangetic Plain) apply in ALL tiers — don't gate these behind Tier 1.

## Data-source implementation notes (researched — a live REST API isn't available for all of these; some need a one-time offline preprocessing step, not a runtime call)
- **Dust/aerosol correction — solved, no preprocessing needed**: use Open-Meteo's Air Quality API (free, no key for non-commercial use, ships `dust` and `aerosol_optical_depth` as hourly variables). Same request pattern as the irradiance ensembling calls already planned — this replaces raw CAMS/MERRA-2 file handling entirely. Verify CORS with one real browser fetch before relying on it.
- **Elevation — primary source + fallback pool, so the 1,000/day problem mostly disappears**:
  - **Primary: AWS/Mapzen Terrain Tiles** (`elevation-tiles-prod` S3 bucket, Registry of Open Data on AWS) — global elevation as Terrarium-encoded PNG tiles (`elevation = R*256 + G + B/256 - 32768`), served as plain static files (S3 + CloudFront), no API key, **no rate limit at all** — same cost/scale profile as the static-tile approach already used for building footprints, because it *is* the same technique. One tile download covers a whole area's elevation grid, not one point per request, so it scales far better than any per-point API. This should be tried as the default, not a fallback — it's the one option here with zero quota risk of any kind. Verify CORS with a real browser fetch before committing (public S3-hosted tiles are routinely fetched cross-origin by web map libraries, so this is low-risk, but confirm rather than assume).
  - **Fallback pool, in order, only if the primary has an issue** (e.g. a specific point needs cross-checking, or tile decoding isn't implemented yet): (1) Open Topo Data (`api.opentopodata.org`, free, SRTM 30m, 1,000 calls/day, 1/sec, CORS unconfirmed — test first), (2) Open-Elevation's public demo instance (free, open-source, no officially documented hard limit but informal ~1 req/sec community guidance and it's a best-effort demo server, not an SLA — treat as least reliable), (3) USGS EPQS (free, no key, but **US-only coverage** — useful as a Tier-1-US-specific cross-check, not a global fallback), (4) Google Elevation API as absolute last resort, with a Cloud Console Quota cap set low (e.g. 4,500/month) so it fails closed instead of billing.
  - **Quota-exhaustion strategy for whichever source is active — do NOT gate this by tier.** Elevation correction is explicitly meant to apply in all tiers (see Architecture above), and Tier 1 countries are exactly the ones getting static tiles first — restricting live lookups to "Tier 1 only" would cut off Tier 2/3 (the original target users) first, backwards from the mission. Instead:
    1. Cache lookups by rounded coordinate (~3 decimal places, ~111m grid) — nearby users during a traffic spike reuse the same cached value instead of each burning a call (matters much less once the tile-based primary source is in place, since a cached tile already covers a whole neighborhood).
    2. When every source in the pool is exhausted, don't block the user — skip elevation correction for that request and widen the stated uncertainty range accordingly (same honest-degradation principle used elsewhere in this doc), so every user still gets a result.
    3. Prioritize which regions get higher-precision LiDAR-based elevation next based on observed real traffic, not tier classification — traffic patterns from launch channels won't necessarily match the tier map.
- **Building footprints — offline extract + static hosting, not a live per-request API**: use the Overture Maps Foundation dataset instead of separately cross-validating Google Open Buildings and Microsoft Footprints — Overture already merges both plus OSM into one dataset with its own confidence field. Overture's official data is S3-hosted GeoParquet with no lightweight per-building REST endpoint, and DuckDB-WASM (the in-browser SQL engine for querying it) can't hit S3 directly. The validated pattern: one-time offline extract of the footprint subset for each supported region → host as a static Parquet/PMTiles file alongside the rest of the site → query client-side with DuckDB-WASM or a vector-tile library. This is a documented, real technique, not a hypothetical.
- **LiDAR-derived roof geometry (USGS 3DEP / AHN / IGN LiDAR HD) — same offline-tile pattern as building footprints**: point-cloud processing at request time is not viable in a browser. Pre-generate a roof-height layer for supported Tier-1 regions and serve it as static tiles the same way.
- **Sentinel-2 NDVI shading cross-check — not feasible free/anonymous, descope or substitute**: Sentinel Hub requires a paid subscription after a 30-day trial; no CORS-friendly free tier exists. Since this was already flagged as a "modest gain" nice-to-have (not required to hit the accuracy target), either drop it, or substitute NASA GIBS/MODIS NDVI (250–500m resolution — much coarser than Sentinel-2's 10m, but genuinely free, no key, CORS-open) if the cross-check is still wanted at lower fidelity. Required UX tap #3 below has been updated to reflect this.

## Core formulas
- Sun position: standard solar geometry (declination via Cooper's equation, hour angle → altitude/azimuth). Near-zero error — don't "simplify" this part.
- GHI → POA (tilted plane): Liu-Jordan isotropic model as baseline; Erbs correlation for diffuse/beam split. Upgrade path: Perez anisotropic model.
- Annual energy: `E = A × r × H × PR` (area × panel efficiency × annual tilted irradiation × performance ratio).
- PR should be computed from local temperature + aridity, not hardcoded at 0.75.
- Savings = E × self_consumption × tariff (self_consumption = 1.0 under net metering).
- ROI must include panel degradation (~0.5–0.8%/yr) and local tariff escalation — both currently missing, both required for an honest lifetime number.
- Roof area A: don't use `polygon_area × packing_factor`. Take the raw polygon coordinates from the UI's map draw step and run 2D bin-packing (standard panel rectangles, minus keep-out zones for vents/chimneys) to get the real usable panel count.
- Roof area/azimuth must be computed geodesically, not from screen pixels — Leaflet is Web Mercator, not equal-area, so use a geodesic area function on the raw lat/lng polygon (not projected coordinates); derive azimuth from the polygon's longest-edge bearing; correct self-intersecting polygons before either calc.

## Required UX taps (in priority order — see PROJECT_SUMMARY.md §3 for why)
1. Power access status: grid-tied / generator-dependent / no power — highest-impact single input, do not skip.
2. Self-consumption bucket: mostly-out / mixed / mostly-home (skip if net-metered).
3. Shading level (icon), optionally cross-checked against NDVI — Sentinel-2 (10m) isn't free/anonymous-access feasible (see Data-source implementation notes), so use free MODIS-based NDVI (NASA GIBS, 250–500m) if this cross-check ships, or drop it — it's a nice-to-have, not required for the accuracy target.
4. Roof shape (flat/gable/unknown) — splits azimuth 50/50 for gable roofs.
5. Roof material — feasibility gate (e.g. thatch → "get a local structural check" message, not a number).

## Accuracy target (already validated by Monte Carlo — don't re-derive, just hit it)
- Tier 1: ~±11% energy, ~±11% savings (90% CI)
- Tier 3: ~±16% energy, ~±24% savings (90% CI)
- If a change measurably worsens these, treat it as a regression.

## Division of work
UI is joint (both owners). Engine is split into two owners, and **each owner's slice contains both physics and math/statistics** — neither owner is "the physics person" or "the stats person" exclusively.

- **UI (joint — owner A + owner B)**: map interaction, tap-based question flow, i18n, PDF report generation. Both work in `/ui/`; to avoid stepping on each other, informally split by feature (e.g. one takes map/polygon-draw + PDF export, the other takes the tap-question-flow + i18n) rather than by strict file ownership — either can review/approve the other's UI PRs.
- **Engine — Radiation & Uncertainty (owner A)**, `/engine/radiation/`:
  - *Physics*: solar geometry (sun position — declination, hour angle → altitude/azimuth), GHI→POA transposition (Liu-Jordan/Erbs, Perez upgrade path), dust/aerosol correction to GHI (CAMS/MERRA-2), irradiance ensembling (NASA POWER/Open-Meteo) and the tiered irradiance-source routing that feeds it.
  - *Math/statistics*: whole-system Monte Carlo uncertainty propagation and accuracy validation (`monte_carlo.py`) — quantifying how confident the physics above actually is.
  - Output: kWh with its own uncertainty range.
- **Engine — System & Economics (owner B)**, `/engine/system/`:
  - *Physics*: performance ratio modeled from local temperature + aridity (thermal), elevation correction (Copernicus DEM), roof-polygon geometry (panel bin-packing, geodesic area/azimuth — see Core formulas) and the tiered roof/building geometry sourcing (LiDAR, footprint cross-validation) that feeds it.
  - *Math*: ROI (amortization with panel degradation ~0.5–0.8%/yr + local tariff escalation), savings/self-consumption modeling, CO2 impact, tariff data integration (OpenEI).
  - Output: $/CO2/90% CI, combining owner A's kWh with this area's physical/financial modeling.
- Interface contract: UI hands off the raw roof polygon (`[lat, lng]` array) to `/engine/system/` (for geometry) and irradiance context flows from `/engine/radiation/`; `/engine/radiation/` and `/engine/system/` both feed into the final kWh → $/CO2/CI output shown in UI. Coordinate any change to these data shapes via PR description before merging.
- Keep changes scoped to your area's files to avoid merge conflicts.

(Considered and dropped for now, not in current scope: an interactive sun-path shading calculator and a client-side real-time uncertainty slider — the former duplicates the existing shading-icon tap without enough accuracy gain to justify the build cost, the latter duplicates `/engine/radiation/`'s `monte_carlo.py` logic rather than adding new work. Revisit only if an area ends up under-loaded once the current scope lands.)

## Before merging any PR
- Run `/code-review` on the diff.
- Check: does every displayed number carry a source and an uncertainty range? Does any new assumption have a comment citing where it came from?

