# Global Rooftop Solar Potential Calculator — Project Summary

A complete reference on the solar panel calculator: what it is, why it can't be perfectly accurate, how accurate it can actually get, and where the hard limits are.

## 1. The idea

A free, browser-only web tool that estimates rooftop solar potential, annual savings, and CO2 impact — built with open-source libraries (Leaflet.js, Turf.js, SunCalc.js), no paid GIS software, no backend server. Originally scoped for low-income households and NGOs in developing countries; later expanded to work globally, including developed markets, using a tiered data strategy (see §4).

Core requirement from day one: **be honest about accuracy**. "Zero error, matches real-world values exactly" is not achievable with free data and no site visit — so the design goal became *transparent, minimized, well-understood uncertainty* rather than false precision.

## 2. Why perfect accuracy is impossible

Every input has a known error source:

| Input | Typical error | Source |
|---|---|---|
| Sun position (SunCalc-style astronomy) | ~0% | deterministic formula |
| Roof polygon geometry (once we have a footprint) | ~0% | pure math (Turf.js) |
| Building footprint detection (Google Open Buildings / Microsoft Footprints) | ~94% precision, ~70% recall | ML-derived from satellite imagery |
| Building height (for shadow modeling) | RMSE 1.5m (Oceania) to 8.9m (South America); Africa has no validation data at all | GlobalBuildingAtlas (2025) |
| Satellite-derived solar irradiance (GHI) | ~10% daily, worse hourly | East Africa validation studies |
| Tilt/transposition model | ~5–15% | isotropic vs. Perez sky models |
| Performance ratio (dust, heat, inverter losses) | ±10–15% | assumption unless modeled from local climate |
| Self-consumption ratio (savings calc) | biggest single lever — up to ±40%+ | pure behavioral guess unless asked |
| Grid-connection status (grid-tied vs. generator vs. no power) | if unasked: showed value could be **0.36x–6.9x** the real value | structural, not just noisy |
| Future weather over the system's 25-year lifetime | irreducible — see §6 | not a data problem, a forecasting-horizon limit |

## 3. Ranked list of accuracy improvements (impact-ordered, from simulation)

1. **Ask "grid-tied / generator-dependent / no power access"** (1 tap) — by far the largest fix; without it, savings shown could be off by up to 7x.
2. **Ask self-consumption bucket** ("mostly out / mixed / mostly home", 1 tap) — cuts savings uncertainty ~28% relative. In net-metering regions (see §4, Tier 1) this variable drops out entirely.
3. **Compute performance ratio from local temperature + aridity** instead of a fixed constant — biggest lever for the energy (kWh) number itself.
4. **Cross-validate building footprints** using two independent datasets (Google + Microsoft, already merged by VIDA/source.coop) — flag low-confidence matches.
5. **Ensemble multiple free irradiance sources** (NASA POWER + Open-Meteo satellite radiation) to reduce random error.
6. **Upgrade the transposition model** from isotropic to Perez (anisotropic) — moderate gain.
7. **NDVI (Sentinel-2, 10m, free) vegetation-proximity check** — cross-checks the user's self-reported shading answer; modest gain (~33% tighter on that specific component).
8. **Dust/aerosol correction** — not just West Africa. A 2025 study found Harmattan/Saharan dust cuts GHI estimation error by up to 75% locally when corrected for; separately, aerosol/haze over the Indo-Gangetic Plain (India/Pakistan/Bangladesh) has similarly large documented radiative effects (≥30 W/m² surface forcing). Same free data (CAMS, MERRA-2) covers both regions — this generalizes to most of South Asia and the Sahel, not just one region.
9. **Elevation correction, globally** — Copernicus DEM (GLO-30) is a free, 30m-resolution global elevation dataset (unlike national LiDAR, it covers every country, Africa included). Lets altitude-based temperature/atmospheric corrections apply everywhere, not just Tier-1 countries.
10. **Post-installation feedback loop** — let users/NGOs report real measured output back; recalibrates the model over time. Unlike everything else on this list, this has no theoretical ceiling — it just takes time to accumulate data.
11. **Panel degradation (~0.5–0.8%/yr) and local tariff escalation** — currently missing entirely from the ROI/payback math; both needed for an honest lifetime estimate (they partially offset each other).

## 4. Data architecture: tiered by what's actually free in each country

Rather than one global pipeline, the design auto-detects what free data exists at a given location:

- **Tier 1** (US, Netherlands, France, and other countries with open government LiDAR): sub-meter roof geometry (USGS 3DEP, AHN, IGN LiDAR HD), real utility tariffs (OpenEI Utility Rate Database, US), PVGIS/NSRDB-grade irradiance. In net-metering regions, self-consumption ratio — our biggest error source — stops mattering at all for the annual $ total.
- **Tier 2** (countries with decent ML building-footprint coverage but no LiDAR): Google/Microsoft Open Buildings + NASA POWER.
- **Tier 3** (poorest building/power data — much of the original target region): minimal-tap fallback, wider stated uncertainty, manual overrides. Still benefits from #8 and #9 above, since those are globally free.

Google's Solar API is the closest commercial benchmark (building-level shadow analysis) but is pay-per-request — useful as an accuracy reference, not usable given the free-only constraint.

## 5. UX principle: minimize typing, maximize taps

Given the target users (limited literacy, low-end phones, developing regions), almost nothing should require typing:

- Location: GPS permission (1 tap)
- Confirm auto-detected roof footprint (1 tap if needed)
- Shading level: icon selection (1 tap)
- Roof shape: flat / gable / unknown (1 tap)
- Self-consumption: mostly-out / mixed / mostly-home (1 tap) — skipped automatically in net-metering regions
- Power access: grid-tied / generator / none (1 tap) — **highest-impact addition found**
- Roof material: feasibility gate, not accuracy (1 tap) — flags roofs (e.g. thatch) that likely can't support panels at all

Everything else (tariff defaults, currency, language, tilt angle via the "tilt ≈ latitude" rule, azimuth via the polygon's longest edge) is inferred automatically from location.

## 6. The computed accuracy ceiling

We ran Monte Carlo simulations (not guesses) to quantify this, using a real test case (Kisumu, Kenya) and propagating every error source in §2.

**Baseline design** (before any of the fixes in §3): annual energy ±28%, annual savings ±47% (90% confidence interval).

**With every fix in §3 implemented**, split by data tier:

| | Tier 3 (no LiDAR / no net metering — most of the original target region) | Tier 1 (open LiDAR + net metering — US, NL, FR, etc.) |
|---|---|---|
| Annual energy (kWh) | ~±16% | ~±11% |
| Annual savings ($) | ~±24% | ~±11% |

Tier 1's savings figure collapses to match its energy figure specifically because net metering removes the self-consumption variable entirely — our single biggest error source disappears structurally in those regions.

For context: Google's own free "Project Sunroof" tool was documented underestimating a real installation's output by 47% in one field case. Professional bankable assessments (site visits, paid data, LiDAR survey) get to ~5–10%. Tier 1's ±11% is close to that professional floor; Tier 3's ±16–24% reflects the genuine data gap that still exists in the poorest-data countries.

**This is treated as the practical ceiling for this design.** Going further requires one of: (a) real-world calibration data accumulating through #10 above, (b) paid data or a site visit — which the free/no-server/no-typing constraints rule out, or (c) better free open data than currently exists, which isn't something more research can produce on demand.

## 7. What can never be fixed, and why

One part of the ±11–24% band is not a data-access problem at all: **the actual weather in any specific future year of the system's 25-year life is unknowable today, by anyone, at any price.**

This was tested directly. Google's Weather API only forecasts 10 days ahead — useless for a 25-year estimate. Even the best seasonal climate forecasting (NOAA/IRI ENSO-based outlooks, free) has real but fast-decaying skill: correlation ~0.89 at a 1-month lead, ~0.60 at 4 months, only ~0.14 (essentially useless) at 7 months. No forecast — free or paid, from Google or anyone — has meaningful skill on a multi-year horizon. This is a fundamental limit of the physical climate system, not a data source we haven't found yet.

What real-time weather data *can* do is different: after a system is installed, live data can power a monitoring feature ("your output is 15% below expectation this month because this is the region's rainy season") — useful, and connects naturally to the feedback loop in §3.10. It does not, however, shrink the upfront ±11–24% estimate given at the point of decision, because that number already honestly reflects not knowing the future.

## 8. Where things stand

Nothing has been built yet — everything above is design and validated-by-simulation research. The recommended next step is to implement the top-ranked items from §3 (especially #1 and #2), using the tiered architecture in §4 as the target structure, and treat §6's numbers as the accuracy target rather than searching for further improvements.
