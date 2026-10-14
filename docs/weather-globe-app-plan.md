# Weather Globe — a Ventusky-grade desktop app on real Earth imagery

## Context

Goal: a native desktop weather app that matches Ventusky layer for layer (wind
particles, color fields, isolines, timeline, models, point forecasts) but draws them on
real Earth imagery, either Google's (Photorealistic 3D Tiles / satellite) or Apple's,
picked in Settings. Data has to be live. The UI should look like a well-made instrument,
not a template: professional, warm rather than corporate, smooth animation, themes you
can swap, and no italics anywhere.

Two problems shape the whole design:

1. **Ventusky has no API.** Its tiles are internal, hotlink-protected, and its terms
   forbid scraping. A scraper would break every time they change a build hash. The
   workaround: **fetch from the same places Ventusky does.** Ventusky shows public
   numerical weather models (GFS, ECMWF, ICON, GEM, HRRR…) plus radar, satellite and
   lightning feeds. All of these can be downloaded directly and legally. We rebuild the
   rendering ourselves.
2. **The Earth imagery is licensed.** Google's imagery is available through the Map
   Tiles API (Photorealistic 3D Tiles and 2D satellite tiles; needs an API key, has a
   free monthly allowance, attribution required). Apple's imagery is only available
   through MapKit, so you can't pull raw Apple tiles on Linux or Windows. Workaround
   below.

## Stack decision

**Tauri 2: a Rust core with a WebGL/CesiumJS rendering frontend.** I'm changing from
"pure native Rust GUI" on purpose:

- CesiumJS already streams Google Photorealistic 3D Tiles, does globe and 2D Mercator
  projections, handles the camera, and runs terrain/depth picking. Writing a 3D Tiles
  streamer in wgpu would take weeks. That alone breaks the time budget.
- Tauri still gives you a real native app: small binary (~10 MB, unlike Electron),
  system webview, native menus, tray, notifications and file dialogs. Everything heavy
  runs in Rust: downloads, GRIB2 decoding, interpolation, caching and scheduling.
- Fallback if pure-native is non-negotiable: `eframe` + `wgpu` with a custom globe
  shader and Google **2D** satellite tiles only (no photoreal 3D). The data core below
  stays the same.

Frontend: TypeScript + Svelte 5 (tiny runtime, built-in springs and transitions),
CesiumJS, and hand-written GLSL for the weather passes.

## Architecture

```
┌──────────────────────── Rust core (src-tauri) ────────────────────────┐
│ scheduler ── knows model run times (GFS 00/06/12/18Z, ~3.5h delay,     │
│              ECMWF 4×/day, ICON 4×/day, HRRR hourly), polls indexes    │
│ sources/*  ── one `Source` trait impl per provider (list below)        │
│ grib       ── pure-Rust `grib` crate decode → f32 grid                 │
│ field      ── regrid to equirect, quantize to u16, pack 2 vars/texture │
│ cache      ── SQLite index + files; ETag/Range; LRU by disk budget     │
│ point      ── point forecasts/meteograms (Open-Meteo, multi-model)     │
│ earth/*    ── imagery providers (Google key proxy, Apple snapshotter)  │
│ protocol   ── custom URI scheme wx://field/{model}/{var}/{lvl}/{run}/  │
│               {step}.bin → binary straight to the webview, no JSON     │
└───────────────────────────────────────────────────────────────────────┘
┌──────────────────────── Frontend (webview) ───────────────────────────┐
│ Cesium viewer (globe / 2D) + Earth provider                            │
│ FieldPass   – screen-space overlay: depth → lat/lon → sample field     │
│               texture → colormap LUT (works on 3D tiles AND ellipsoid) │
│ WindPass    – GPU particles, ping-pong float textures                  │
│ IsolinePass – marching squares in Rust → polylines (pressure, temp)    │
│ LabelPass   – city value labels with collision culling                 │
│ UI shell    – Svelte: timeline, layer rail, legend, forecast sheet     │
└───────────────────────────────────────────────────────────────────────┘
```

Field textures are tiny and quick to fetch. A global 0.25° grid is 1440×721, about 2 MB
as u16 per variable per time step. GFS supports server-side subsetting (NOMADS grib
filter, or byte-range reads on the `.idx` files in the AWS `noaa-gfs-bdp-pds` bucket),
so we download only the variable and level being shown, then prefetch the steps either
side of the timeline cursor. No backend server is required. An **optional** relay mode
(the same binary, headless, run from cron on a cheap VPS) can pre-bake textures for
several users. It's a config flag, not a separate project.

### Layer system is data, not code

Every Ventusky layer is one entry in `layers.toml`: source, variable, levels, units,
palette, isoline spacing, and whether particles are on by default. Adding a layer means
adding config. This is how we get full Ventusky coverage quickly.

```toml
[[layer]]
id = "temperature"   source = ["gfs","ecmwf","icon"]   var = "TMP"
levels = ["2m","950hPa","850hPa","700hPa","500hPa","300hPa","200hPa"]
palette = "ventusky_temp"   units = ["°C","°F","K"]   isolines = { step = 2 }
```

## Data sources (replacing the Ventusky scrape)

| Need | Source | Notes |
|---|---|---|
| Global models | NOAA GFS 0.25° (NOMADS / AWS), ECMWF IFS open data 0.25° (CC-BY-4.0), DWD ICON (opendata.dwd.de), CMC GEM | GRIB2, subset by var/level |
| Regional models | HRRR (US), ICON-D2 / ICON-EU, HRDPS (CA), AROME (FR, data.gouv) | Same pipeline, smaller bbox |
| Point forecasts, multi-model | Open-Meteo forecast, ensemble, historical APIs | Free non-commercial; meteograms, 14-day |
| Radar | RainViewer (free tier is limited), NOAA MRMS (US), DWD RADOLAN (DE) | Past ~2h + nowcast frames |
| Satellite | NASA GIBS: GOES-East/West, Himawari, Meteosat | ~10-min geostationary imagery |
| Lightning | Blitzortung websocket | Non-commercial, attribution |
| Air quality | Open-Meteo Air Quality (CAMS) | PM2.5, PM10, NO₂, O₃, SO₂, CO, dust, AQI |
| Waves, SST, currents | Open-Meteo Marine, NOAA WaveWatch III, RTOFS | |
| Tropical cyclones | NHC / JTWC feeds, GDACS | Tracks + forecast cones |
| Alerts | NWS CAP, MeteoAlarm (EU) | |
| Webcams | Windy Webcams API | Ventusky has webcams; so do we |
| Geocoding | Open-Meteo geocoding, Nominatim fallback | |

## Earth imagery (Settings → Earth)

| Option | How | Limits |
|---|---|---|
| **Google Photorealistic 3D** (default when a key is present) | Map Tiles API 3D Tiles in Cesium (direct key, or the Cesium ion asset) | Key + billing; on-screen attribution is required; no long-term tile caching |
| **Google Satellite 2D** | Map Tiles API 2D satellite as a Cesium imagery layer on the ellipsoid | Cheaper; used for 2D map mode |
| **Apple** (macOS only) | Rust calls `MKMapSnapshotter` through `objc2-map-kit` for satellite snapshots per tile region, served to Cesium via `earth://apple/{z}/{x}/{y}` | No Flyover 3D; Apple logo and legal text must be shown; not available on Linux/Windows (option is greyed out with an explanation) |
| **NASA** (no key, default fallback) | Blue Marble by day, Black Marble city lights by night, GIBS true-color | Free, public domain, lower resolution up close |

The overlay pass reconstructs each pixel's lat/lon from the depth buffer, so the colors
lie correctly on mountains and buildings with every provider. Switching providers
crossfades instead of popping.

## Ventusky parity: every detail

**Layers** (each one supports every model and level it has):
- Temperature: surface, plus 950→200 hPa; feels-like; anomaly vs. normal
- Precipitation: 1h / 3h / accumulated over the selected range; type (rain, snow, freezing rain, mixed); new snow; snow depth
- Radar (animated past frames + nowcast); satellite (visible / IR / water vapor)
- Clouds: total, low, mid, high; cloud base; visibility / fog
- Wind: 10 m, 100 m, gusts, 850 / 700 / 500 / 300 / 200 hPa (jet stream)
- Pressure: MSLP isobars with H/L markers; geopotential height
- Humidity, dew point, freezing level
- Storms: CAPE, thunderstorm probability, lightning (live strikes that fade out over time)
- Air quality: everything in the sources table
- Ocean: wave height, wind waves, swell height + period + direction, currents, SST
- Tropical cyclones: named tracks, intensity colors, forecast cones

**Map behavior:**
- **Wind particles:** thin, near-white streaks (alpha ~0.6). Speed controls both trail
  length and velocity. Trails fade over ~1 s by decaying the previous frame's texture.
  Particle count scales with zoom and screen area, and particles respawn randomly
  (~1–2 %/frame) so the flow never clumps. They run on top of any layer. On the ocean
  layers the same system follows currents or swell direction.
- **Color fields:** bilinear sampling in space and **linear interpolation in time**
  between forecast steps, so scrubbing the timeline is smooth instead of jumping.
  Palettes come from 1D LUT textures. `ventusky_temp` runs deep violet → blue → cyan →
  green → yellow → orange → red → dark maroon. Each layer has its own Ventusky-style
  palette, plus a colorblind-safe alternative for every one.
- **Projections:** 3D globe (orthographic feel at distance, perspective up close), 2D
  Web Mercator, and an animated morph between them (Cesium `morphTo2D`).
- **Isolines:** labeled pressure and temperature contours, label density tied to zoom.
- **City labels** showing the current layer's value (e.g. "14°"), culled by population
  and collisions, updated as the timeline moves.
- **Click anywhere** for a pin, a value readout, and a forecast sheet: hourly table,
  10–14-day outlook, meteogram, and model comparison.
- **Timeline:** hours and days, play/pause, speed control, "now" snap, model run age
  shown ("GFS 06Z, 4 h old").
- **Legend:** color bar; click the unit to cycle units.
- Search, geolocate, favorites.
- **Settings:** units (°C/°F, km/h, m/s, kt, mph, Beaufort, mm/in, hPa/inHg/mmHg), 12/24 h, time zone (local / UTC / clicked location), particle density, animation speed.
- URL-style state (`wx://…?layer=wind&lvl=850&t=…`) for sharing views.

## Ten additions beyond Ventusky

1. **Model split view.** Drag a vertical divider to compare ECMWF on one side and GFS
   on the other. A "spread" layer shows where the models disagree, i.e. how much to
   trust the forecast.
2. **Live sky.** Real sun position lights the globe, with the day/night terminator and
   city lights on the night side. Live satellite clouds are blended onto the Earth
   imagery, so the globe looks like the actual sky right now.
3. **Rain-in-minutes nowcast.** Uses radar motion vectors (optical flow in Rust over the
   last frames) to say "rain starts in 12 min, light, ~25 min" for favorite places.
   Sent as a native notification.
4. **Route weather.** Draw a road, flight or sailing route with a departure time. The
   app samples the forecast along it at each ETA and shows a strip of conditions.
5. **Best-time finder.** Choose an activity (run, cycle, drone, astro / golden-hour
   photography, sailing), set thresholds, and it highlights the good windows on the
   timeline.
6. **Vertical sounding.** On click: a Skew-T profile, a wind hodograph, and a
   cross-section along a dragged line, all from the pressure-level data we already
   download.
7. **Severe alerts.** CAP/MeteoAlarm polygons drawn on the globe. A native notification
   when a favorite place falls inside one.
8. **Time-lapse export.** Record the animated globe over a time range to MP4 or GIF
   (frames captured in the webview, encoded by Rust with `ffmpeg-sidecar`).
9. **Tray / menu-bar mini view.** Current conditions, a 6-hour sparkline and the
   nowcast, without opening the window.
10. **Command palette (⌘K / Ctrl+K)** with full keyboard control: layer by name, jump to
    a city, `t+24h`, switch models.

## UI and motion design

**Direction:** "precision instrument with a bit of warmth". References: Apple Weather
(condition-reactive modules, the precipitation map, spring motion), Flighty (dense data
that stays calm), Teenage Engineering (personality, exact grids, one accent color),
aviation EFB/Garmin displays (legibility over imagery), and Dieter Rams/Braun
(restraint). Apple's HIG, the Apple Design Resources kits and the WWDC motion sessions
supply spring values and layout rules. Apple is one input among several, not a template.

**Avoiding generic-AI patterns, as concrete rules:**
- No purple-blue gradients, no glassmorphism on every surface, no emoji icons, no
  sparkles, no gradient text, no stacks of rounded cards.
- The map is full-bleed. Controls hug the edges: a thin layer rail on the left, the
  timeline docked at the bottom, the forecast sheet sliding in from the right. The
  imagery is the hero; the chrome stays quiet.
- One accent color per theme. Everything else is neutrals. Weather palettes carry the
  color.
- Hairline 1 px dividers instead of shadows. Small radii (6 px). Tight 4 px spacing grid.

**Type:** UI in **Instrument Sans**, data in **Barlow Semi Condensed** with tabular
figures (an instrument-panel feel, and the numbers don't jitter while animating). Only
upright weights are bundled, `font-style: normal !important` is set globally, and a
lint check fails the build if any `italic` / `oblique` appears in the CSS.

**Themes** are token files (`themes/*.toml`: surfaces, text, accent, hairline, chrome
opacity, palette set), hot-reloaded, and users can add their own. Five ship with the app:
- **Graphite:** neutral dark, default
- **Paper:** warm off-white `#F3EFE6`, ink text, terracotta accent
- **Night Watch:** red-on-black for use at night without losing dark adaptation
- **Tide:** deep blue-grey with a sea-glass accent
- **Classic:** Ventusky-like chrome and palettes

Switching themes animates the token values over 300 ms instead of swapping instantly.

**Motion** (all of it respects the OS "reduce motion" setting):
- Springs instead of fixed-duration easing: damping ~0.85, response 0.35 s for panels
  and 0.5 s for sheets.
- Changing layers crossfades two field textures in the shader (250 ms). Particles keep
  flowing through the change.
- The timeline has inertial scrubbing that snaps to hours. Temperatures roll digit by
  digit with tabular figures.
- Camera fly-to uses a zoom-out / arc / zoom-in path. Selecting a city on the 3D globe
  ends on a slight tilt toward the photoreal buildings.
- Condition-reactive forecast sheet: a subtle animated sky band (sun, rain streaks,
  snow) behind the header, drawn in the theme's palette and never as a stock gradient.

## Building it fast

A couple of hours with an AI coding agent is realistic for a working core: the globe on
Google/NASA imagery, wind particles, temperature/precipitation fields, the timeline, the
point forecast sheet, and the theme system. Full parity and all ten additions are more
than that. Every layer after the first is config, and every feature after the core
reuses the same field pipeline. What keeps it fast:

- Cesium does all the globe, tile and camera work; we write only three shaders (field,
  particles, crossfade).
- One generic `Source` → `Field` → texture path; layers live in TOML.
- Open-Meteo covers every point forecast, so no point extraction is needed from GRIB.
- Port the well-known GPU wind technique (ping-pong particle state textures, from
  mapbox/webgl-wind) instead of inventing one.
- No backend to deploy; the relay is optional.
- Use only crates that already work: `reqwest`, `tokio`, `grib`, `rusqlite`,
  `image`, `objc2-map-kit`, `tauri-plugin-notification`.

## Verification

- Golden test per source: decode a fixed GRIB sample and compare the value at known
  points with the Open-Meteo API value for the same run (±0.5 units).
- Screenshot diff of the globe at fixed camera, time and layer for each theme and Earth
  provider (Playwright against the Tauri webview build).
- Timeline scrub must stay at 60 fps with particles on a mid-range GPU (frame-time
  overlay in a debug build).
- Lint gate: no `italic`/`oblique` in the CSS; every API key comes from Settings or the
  keychain and is never committed.
- Manual check: each Earth provider shows its required attribution, and the Apple option
  is hidden or explained on non-macOS systems.
