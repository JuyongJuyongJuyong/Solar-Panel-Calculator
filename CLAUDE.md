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
- **Tier 2** (ML building footprints, no LiDAR): Google Open Buildings + Microsoft Footprints (cross-validate both; flag low-agreement matches), NASA POWER / Open-Meteo satellite radiation (ensemble both).
- **Tier 3** (weakest building/power data): minimal-tap fallback, widest stated uncertainty range, manual overrides available.
- Elevation correction (Copernicus DEM GLO-30, free & global) and dust/aerosol correction (CAMS/MERRA-2, applies to Sahel + Indo-Gangetic Plain) apply in ALL tiers — don't gate these behind Tier 1.

## Core formulas
- Sun position: standard solar geometry (declination via Cooper's equation, hour angle → altitude/azimuth). Near-zero error — don't "simplify" this part.
- GHI → POA (tilted plane): Liu-Jordan isotropic model as baseline; Erbs correlation for diffuse/beam split. Upgrade path: Perez anisotropic model.
- Annual energy: `E = A × r × H × PR` (area × panel efficiency × annual tilted irradiation × performance ratio).
- PR should be computed from local temperature + aridity, not hardcoded at 0.75.
- Savings = E × self_consumption × tariff (self_consumption = 1.0 under net metering).
- ROI must include panel degradation (~0.5–0.8%/yr) and local tariff escalation — both currently missing, both required for an honest lifetime number.

## Required UX taps (in priority order — see PROJECT_SUMMARY.md §3 for why)
1. Power access status: grid-tied / generator-dependent / no power — highest-impact single input, do not skip.
2. Self-consumption bucket: mostly-out / mixed / mostly-home (skip if net-metered).
3. Shading level (icon), optionally cross-checked against Sentinel-2 NDVI.
4. Roof shape (flat/gable/unknown) — splits azimuth 50/50 for gable roofs.
5. Roof material — feasibility gate (e.g. thatch → "get a local structural check" message, not a number).

## Accuracy target (already validated by Monte Carlo — don't re-derive, just hit it)
- Tier 1: ~±11% energy, ~±11% savings (90% CI)
- Tier 3: ~±16% energy, ~±24% savings (90% CI)
- If a change measurably worsens these, treat it as a regression.

## Division of work
- Engine side (data pipeline, tiered routing, physics/formulas, accuracy validation): owner A
- UI side (map interaction, tap-based question flow, i18n, PDF report generation): owner B
- Keep changes scoped to your side's files to avoid merge conflicts; coordinate interface changes (data shape between engine and UI) via PR description before merging.

## Before merging any PR
- Run `/code-review` on the diff.
- Check: does every displayed number carry a source and an uncertainty range? Does any new assumption have a comment citing where it came from?
