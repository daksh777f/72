# Agrim: project context (for the SIH 2026 idea deck)

Give this whole file to the model as the single source of truth. Do not invent facts, numbers or results that are not written here. If something is not here, leave it out or mark it "planned".

## 1. Identity

- **Name:** Agrim (Devanagari: अग्रिम). Meaning: "in advance", "ahead". The whole product is about lead time.
- **Tagline:** See the storm before it arrives.
- **Event:** Smart India Hackathon 2026
- **Problem Statement ID:** 26072
- **Title:** AI/ML based Nowcasting of thunderstorm and lightning using atmospheric observation including multiple radars, satellite, lightning and model data
- **Organization:** Ministry of Earth Sciences (MoES). **Department:** India Meteorological Department (IMD)
- **Category:** Software. **Theme:** Disaster Management
- **Repo:** https://github.com/daksh777f/72 (branch `UI-edits`)
- **Team name / Team ID:** (fill in from the SIH portal)

## 2. The problem in plain words

Severe convective storms (thunderstorm, hail, lightning, cloudburst, downburst) form and hit within minutes to a few hours. The data that describes them (radar, satellite, lightning detectors, weather-model grids) sits in separate systems and formats. Forecasters and disaster managers need one fused, aligned picture, a forecast a few hours ahead, and a clear "when will it reach me" answer that can trigger warnings to districts.

## 3. What Agrim is

A full-stack, real-time nowcasting platform: a Python/FastAPI backend that ingests and fuses multi-source data and runs detectors and forecast models, plus a React/TypeScript operations console with a cinematic landing page.

### 3.1 Inputs (the four the problem statement names, plus stations)

| Input | Source | Key needed | Used for |
|---|---|---|---|
| Radar reflectivity | RainViewer (republished IMD radar network) | No | Hail, cloudburst, downburst-proxy, rain field, all-India radar layer |
| Lightning | Blitzortung.org community VLF network (MQTT) | No | Lightning hazard points; bumps hail severity when collocated within 25 km |
| Satellite IR | Copernicus Sentinel-3 SLSTR; EUMETSAT MSG SEVIRI (geostationary, wired as preferred path) | Free account | Cloud-top temperature (TIR-1), water vapour, MWIR channels |
| Weather model | ECMWF Open Data 0.25 deg HRES | No | Wind, temperature, humidity, pressure; wind steers hazard and rain advection |
| Station weather | Tomorrow.io | Free tier | Per-station temperature, humidity, wind, precipitation (demo stations) |
| GIS layers | ISRO/NRSC MOSDAC and Bhuvan WMS | No | 23 layers (7 base maps, 16 overlays: rivers, basins, drainage, landslide risk, roads, railways, airports, admin boundaries, etc.) |

Every source is behind a `USE_LIVE_*` flag and falls back to a synthetic source automatically. The API endpoint `/system/status` reports which sources are live and which are synthetic, and the UI shows it. Nothing synthetic is presented as live.

### 3.2 Pipeline (runs at startup and every 15 minutes, configurable)

1. **Ingest** in parallel, each source failure-isolated.
2. **Fuse** onto one multi-channel raster: `[tir1, wv, mwir, reflectivity_dbz, lightning_prob]`.
3. **Detect hazards** with documented physical thresholds (see 3.3).
4. **Nowcast** with pySTEPS (0-6 h) and DGMR (0-90 min); storm ETA from the optical-flow motion field.
5. **Serve** via FastAPI (cached per cycle) as GeoJSON, PNG overlays and grids.
6. **Warn**: per-district risk roll-up and optional Twilio SMS with a per-district cool-down.

### 3.3 Hazard detection (all thresholds in one settings file)

| Hazard | Rule | Threshold | Scope |
|---|---|---|---|
| Hail | Reflectivity and cold cloud top and lightning probability, collocated | >= 55 dBZ, <= 210 K, >= 0.30 | Per-city regions (full rule); all-India (reflectivity + strike proximity) |
| Lightning | Real strikes; IMD probability categories in city regions | any real strike = high severity | All-India and cities |
| Cloudburst | Rain rate from real radar via Marshall-Palmer Z = 200 R^1.6 | >= 15 mm/hr (IMD very-heavy-rain boundary); high >= 50, moderate >= 30 | All-India (one point per storm core); pySTEPS-extrapolated in city regions |
| Downburst (confirmed) | Radial-velocity couplet | delta-V >= 25 m/s | City demo regions only; velocity is synthetic (no free public Doppler feed exists) |
| Downburst potential (proxy) | Intense reflectivity core with sharp edge gradient | >= 50 dBZ and >= 15 dBZ per grid cell; source tagged `proxy` | All-India; always labelled a proxy, never velocity-confirmed |

Hazard points can be advected to a lead time by the ECMWF wind field. This is a standard simplified nowcasting technique (storms roughly follow the steering flow). It is labelled as advection, not as a re-detected forecast.

### 3.4 Models

- **pySTEPS** (baseline): Lucas-Kanade optical flow plus semi-Lagrangian extrapolation. 0-6 h at 10-minute steps. Calibrated in mm/hr. Drives cloudburst and storm ETA.
- **DGMR** (DeepMind, pretrained `openclimatefix/dgmr`): generative model, zero-shot on CPU, 0-90 min. **Caveat stated openly:** trained on UK Met Office radar, so output over India is unitless relative intensity. Shown as a comparison layer only, never used to trigger a warning.
- **SmaAt-UNet**: architecture and training scaffold implemented. **Not yet fine-tuned** (planned on SEVIR / Indian radar). The UI says "in training".

## 4. What we built that is new or different (the differentiators)

1. **Uses all four inputs the problem statement names, fused on one grid**, not a single-sensor demo.
2. **Two model families side by side** on the same scene: statistical optical flow (calibrated) and generative AI (relative intensity), with a comparison page.
3. **Physics-based, explainable hazards**: every alert traces to a documented threshold; no black box decides who is warned.
4. **Transparency by design**: live vs synthetic provenance panel and `/system/status`; replay page badges reflect the truth; limitations are stated in the UI and README.
5. **All-India view by default** (not a fixed demo city): real hail, lightning and cloudburst detected across the country from radar and strikes; district and state labels on every hazard point (about 134 district centroids).
6. **Rain simulation**: the radar rain-rate field rendered as a smooth intensity layer plus wind-slanted particle rain, ground splashes, real-strike lightning flashes and detected storm-core callouts; follows the lead-time slider; states its method ("radar echo advected by ambient wind (persistence + steering flow)").
7. **Storm-arrival countdown (ETA)** from the pySTEPS motion field, plus district SMS alerts with cool-down.
8. **Offline-safe India map**: official outline and state boundaries bundled locally (aligned to the same India bounding box the backend uses), so the console renders fully without external tile servers; fonts are bundled too.
9. **23 ISRO/MOSDAC GIS layers** for disaster context, with a same-origin proxy for Bhuvan (which sends no CORS headers).
10. **Resilience**: failure-isolated ingest, background pre-warm of 10 city regions so switching is instant, route-level code splitting on the front end.
11. **Historical replay** of persisted ingest cycles for after-action review and back-testing.
12. **Cinematic landing page** whose figures come from the live API (shows a dash when offline, never a fake number).

## 5. Console features (front end)

- Full-bleed MapLibre map centred on India; layer toggles: Radar, Satellite, Lightning, Hazards, Terrain, Rain Sim, Select Area, Legend.
- Lead-time slider (0-6 h for pySTEPS, 0-90 min for DGMR) with play; intensity trend chart.
- Left dock: telemetry, active hazard counts by type, data provenance (live/synthetic per source), GeoJSON export.
- Right dock: threat matrix (storm cells with ETA, bearing, speed), 6-hour rain-rate outlook buckets, legends, rain-simulation controls.
- Click a point or drag an area to get current and forecast weather stats.
- Pages: Hazards (table with district, metric, ETA, fly-to), Models (pySTEPS vs DGMR vs SmaAt-UNet), Replay.
- Weather overlays: temperature, humidity, wind (with arrows), pressure, rainfall, composite convective risk index.

## 6. Tech stack

Python 3.11, FastAPI, Uvicorn, NumPy, SciPy, OpenCV, Matplotlib, Pillow, pySTEPS, PyTorch (DGMR, SmaAt-UNet), xarray/cfgrib (ECMWF), paho-mqtt (Blitzortung), Twilio (SMS). Front end: React 19, TypeScript, Vite, MapLibre GL JS, GSAP, Lenis, Lucide icons. Bundled fonts: Bebas Neue, Instrument Serif, Manrope, JetBrains Mono, Tiro Devanagari Hindi.

## 7. Prototype status (be honest in the deck)

- End-to-end working prototype: ingest, fuse, detect, nowcast, serve, console, tests (`test_backend.py`).
- Runs on a single CPU server with no paid services; all live flags default to off (synthetic mode) so it always demos.
- Screenshots in `assets/` were captured against a synthetic all-India radar mosaic; recapture with live flags on before using them as "live" evidence.
- Not done / planned: IMD DWR and INSAT-3D/3DR direct feeds; validation of thresholds against IMD archives; SmaAt-UNet fine-tuning; DGMR calibration for India; multi-channel alerts beyond SMS.
- Do not claim: accuracy scores (CSI/POD/FAR), lives saved, latency figures, or number of radar stations. None have been measured.

## 8. Feasibility and risks (use on the Feasibility slide)

| Risk | Mitigation |
|---|---|
| Radar coverage gaps and latency | Failure-isolated sources, automatic synthetic fallback flagged in UI |
| No public Doppler velocity | Reflectivity-core downburst proxy, labelled; plug in IMD DWR when available |
| DGMR domain shift (UK training) | Comparison layer only; pySTEPS is the calibrated engine |
| Thresholds not yet validated on Indian events | Replay page and back-testing module; calibration with IMD archives is next |
| Satellite revisit gaps (Sentinel-3 polar orbit, 1-2 passes a day) | EUMETSAT MSG SEVIRI (15-minute geostationary) preferred path |

Roadmap: **Now** prototype on open data. **Next** IMD DWR and INSAT feeds, validation on archives. **Then** fine-tune SmaAt-UNet on Indian radar, multi-channel alerts, national deployment.

## 9. Impact (audiences)

- Disaster management (NDMA, SDMA, district administrations): earlier district-level warnings with a countdown.
- Farmers and rural communities: hail and lightning alerts by SMS in time to protect crops, livestock and people.
- Aviation and infrastructure: storm ETA and hazard cells for airports, grids, highways.
- IMD forecasters: one fused, explainable console plus replay for review.
- Benefits: social (fewer lightning and hail casualties), economic (lower crop and asset loss), operational (fewer blanket over-warnings because each alert traces to a threshold).
- Any numeric impact claim must come from a cited public source (for example IMD/NCRB lightning-fatality statistics). Do not invent one.

## 10. References (verify exact citations before printing)

Data: RainViewer API; Blitzortung.org; ECMWF Open Data (CC-BY-4.0); Copernicus Data Space (Sentinel-3 SLSTR); EUMETSAT MSG SEVIRI; ISRO MOSDAC and Bhuvan WMS; Tomorrow.io.

Methods: Pulkkinen et al. 2019, pySTEPS (Geoscientific Model Development). Ravuri et al. 2021, DGMR, "Skillful precipitation nowcasting using deep generative models of radar" (Nature). Trebing, Staręga, Mehrkanoon 2021, SmaAt-UNet (Pattern Recognition Letters). Veillette et al. 2020, SEVIR (NeurIPS). Marshall and Palmer 1948, raindrop size distribution (Z-R). IMD rainfall intensity categories.

Software: FastAPI, React, MapLibre GL JS, PyTorch, NumPy/SciPy.

## 11. Brand and visual identity

- Dark "storm-room" look: ink `#04070D` to `#0B111B`; white headlines; body `#C3CEDE`; single accent ice-blue `#8BD8FF`.
- Signal colours for data only: amber `#FFB43A` lightning; coral `#FF6A55` hail/downburst; blue `#7CC0FF` cloudburst; violet `#8A5CFF` and magenta `#FF5FC0` heavy rain.
- Fonts: Bebas Neue (headlines), Instrument Serif Italic (one accent phrase), Manrope (body), JetBrains Mono (tags and numbers), Tiro Devanagari Hindi (अग्रिम).
- Logo: an "A" shaped like an arrowhead inside an open radar ring, an amber storm-cell dot at the upper right.
- Available images in the repo: `assets/landing.png`, `assets/console.png`, `assets/rain-simulation.png`, `assets/hazards.png`, `assets/landing-hazards.png`, `logo.svg`, `logo.png`, and the background scenes in `nowcast/dashboard/public/landing/`.

## 12. Numbers you may use (all true for the current build)

0-6 h pySTEPS horizon; 10-minute forecast steps; 0-90 min DGMR horizon; 15-minute ingest cycle; 4 hazard types; 10 city forecast regions; 23 ISRO/MOSDAC GIS layers; about 134 district centroids; 5 data-source families; thresholds: 55 dBZ, 210 K, 0.30, 15 mm/hr, 50 dBZ proxy core.
