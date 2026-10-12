# AGENTS.md

## Project

Self-contained single-file HTML app for optimizing microbus delivery routes in Santiago de Cuba / Guantánamo. No build step, no package manager, no server-side code.

## Running

Open `rutas-optimicer-v1.html` directly in a browser, or serve statically:

```bash
python3 -m http.server 8000
```

## External services (require internet)

- **OSRM** (`router.project-osrm.org`) — road distance matrices and route geometry
- **Nominatim** (`nominatim.openstreetmap.org`) — geocoding for new destination search
- **unpkg CDN** — Leaflet 1.9.4 (CSS + JS)

If any service is unreachable, the app falls back to haversine distances with a warning banner.

## Architecture

Everything lives in one IIFE in `rutas-optimicer-v1.html` (~1600 lines of JS). Key parallel arrays:

- `DESTINOS` — destination names (index 0 is the depot/base)
- `COORDS` — `[lat, lon]` pairs, same order as `DESTINOS`

Adding/removing destinations requires updating both arrays in sync. Distance matrix is N×N and rebuilt from these arrays plus optional manual overrides in `matrixManual`.

## Conventions

- All UI text is in Spanish.
- No tests, linting, or type checking — verify changes manually in the browser.
- Config export/import uses JSON with a `version` field (currently `"23"`).
