# AGENTS.md

## Project

Self-contained single-file HTML app for optimizing microbus delivery routes in Santiago de Cuba / Guantánamo. No build step, no package manager, no server-side code.

## Running

Open `rute-optimizer.html` directly in a browser, or serve statically:

```bash
python3 -m http.server 8000
```

## External services (require internet)

- **OSRM** (`router.project-osrm.org`) — road distance matrices and route geometry
- **Nominatim** (`nominatim.openstreetmap.org`) — geocoding for new destination search
- **unpkg CDN** — Leaflet 1.9.4 (CSS + JS)
- **OpenStreetMap France tiles** and **Google Fonts** (Roboto; falls back to system fonts)

If any service is unreachable the app keeps working: offline, selection/activation/deletion/import still work using the saved road matrix (or straight line × `ROAD_DETOUR_FACTOR` 1.3 if it was never loaded). Only geocoding (add destination) and the destination editor are blocked (`.requires-online`). A failed OSRM leg is estimated individually (`estimateLeg`) instead of discarding the whole route. Nominatim calls go through `fetchNominatim()` (≥1.1 s apart, per its usage policy); OSRM calls use `fetchJsonWithRetry()` (retries 429/5xx).

## Architecture

Everything lives in one IIFE in `rute-optimizer.html` (~1900 lines of JS, numbered sections with Spanish comments). Key parallel arrays:

- `DESTINATIONS` — destination names (index 0 is the depot/base)
- `COORDS` — `[lat, lon]` pairs, same order as `DESTINATIONS`

Adding/removing destinations requires updating both arrays in sync, then calling `onDestinationsChanged()`. The N×N `distMatrix` is rebuilt from these arrays plus manual overrides in `manualMatrix`, then overwritten with OSRM table distances. Provinces come from the fixed `PROVINCE_BY_DESTINATION` map (destination name → province), not from a live lookup.

Data flow:

- `selected` (Set of indices) is the source of truth for the selection — never read it back from the DOM. `inactive` (Set of indices) holds deactivated destinations: hidden from the Ruta view and the map, never selectable.
- Every index-keyed structure (`selected`, `inactive`, `manualMatrix`, `distMatrix`) must be reindexed when a destination is removed — see `deleteDestination()`, which also snapshots state for the snackbar "Deshacer". After any change to the destination arrays call `onDestinationsChanged({ importedMatrix, keepRoadMatrix, fit })`.
- `updateRoute()` re-optimizes (`nearestNeighbor` + `improveTwoOpt` over the closed tour Base → … → Base) and re-renders summary, results and map. Vehicle parameter changes only call `renderResults()`.
- `realRoute` (OSRM geometry) is tagged with `routeKey()` = `coordsVersion` + order; `currentRealRoute()` ignores it when stale. While a route is pending the map shows only markers (no straight lines); straight dashed lines are the fallback only on OSRM error/offline.
- Road geometry is requested **per leg** (`fetchLeg`, `alternatives=3`) because OSRM returns no alternatives for multi-waypoint requests. `pickPreferredRoute()` minimizes `effectiveCost()` (distance penalized for km on unnamed roads and for turns) among alternatives within `MAX_DETOUR` (15 %) of the shortest. Legs are cached by coordinate pair in `legCache` and fetched with concurrency 3 (`mapLimit`) to be polite to the public OSRM server.

Fuel model (`computeConsumption`): each leg's km are split by road type from OSRM step speeds (`kmByRoadType`: urban/slow < 40 km/h, suburban 40–65, highway ≥ 65). `cityFactor` applies fully to urban km and half to suburban; highway uses the base efficiency ("rendimiento en carretera"). Load decreases linearly per delivery (return leg empty) and adds `LOAD_PENALTY` per road type; idle fuel = stops × idle minutes × L/h. Driving time = OSRM × `VEHICLE_TIME_FACTOR` + stops × minutes per stop. Vehicle parameters and their valid ranges live in `PARAMS`; `readParams()` validates them and invalid values block the results (shown as "—").

Persistence: working data (destinations, coords, provinces, selection, inactive, params, road matrix) auto-saves to `localStorage` under `rute-optimizer:data` via `scheduleSaveData()` and is restored on load through the same `parseConfiguration()` → `applyConfiguration()` path as file import. `parseConfiguration()` validates everything before any state changes; `applyConfiguration()` rolls back on failure. "Restaurar destinos originales" uses `ORIGINAL_DATA`.

UI shell: top app bar (help, theme toggle, `⋮` menu) + left navigation rail that becomes a bottom bar at ≤720px. Three views (`<section class="view" data-view="…">`) are switched by the hash router (`applyRoute()`): `#ruta`, `#destinos`, `#ayuda/<topic>`. Leaflet cannot measure a hidden map, so `refreshMapLayout()` runs when the Ruta view is shown.

The favicon is an inline SVG data URI in `<head>` matching the app-bar logo.

Help Center: a full page view. Articles are `<article class="help-article" data-topic="…">` inside `#helpContent`; the index and "related" links are generated from them. Any element with `data-help="<topic>"` navigates to `#ayuda/<topic>`, so new UI features should get an article and a `data-help` entry point. Keyboard shortcuts live in section 19 and are documented in the `shortcuts` article.

UI preferences (`theme`: light/dark/system, `legendVisible`, `groupByProvince`, `collapsedProvinces`) persist in `localStorage` under `rute-optimizer:prefs`; an inline script in `<head>` applies the theme before first paint. Destinations and selection are intentionally *not* persisted (export/import covers that).

## Conventions

- All UI text is in Spanish. Visual style follows Google/Material conventions: color tokens in `:root` with dark overrides in `:root[data-theme="dark"]` (never hard-code colors), inline SVG icons from the sprite at the top of `<body>`.
- Responsive breakpoints: 1180px, 900px (single column), 720px (bottom navigation), 600px (phone), 380px. Check changes at 360–390px wide.
- Source identifiers (JS variables/functions, HTML `id`s, CSS classes) are in English. Comments stay in Spanish.
- Destination/province names and the config JSON keys (`atributos`, `destinos`, `coordenadas`, `provincias`, `matrizManual`, `matrizDistancias`, optional `destinosInactivos` = list of names, optional `matrizVial` = whether `matrizDistancias` holds road distances; `atributos` may also include `cargaInicial`, `minutosParada`, `minutosRalenti`, `consumoRalenti`) are data, not code — keep them in Spanish.
- No tests, linting, or type checking — verify changes manually in the browser.
- Config export/import uses JSON with a `version` field (currently `"23"`). Keep the schema keys stable so previously exported files still import.
