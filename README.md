# NYC Hourly Surface Smoke Map

An interactive Mapbox GL JS map visualizing hourly near-surface wildfire smoke
concentration over the New York City area, built from NOAA's HRRR-Smoke
forecast model.

## Overview

This project pulls hourly smoke data from NOAA's High-Resolution Rapid
Refresh (HRRR) model, specifically the near-surface smoke mass density
field (`MASSDEN`).
The geodata is cropped to the NYC area, converted into a time series
of contour polygons categorized by health-relevant smoke density.

**Date range covered:** 2026-07-13 to 2026-07-23 (264 hourly timesteps)
**Spatial extent:** NYC area (~lat 40.30–41.10, lon -74.50 to -73.50)

## Data Source

- **Model:** [NOAA HRRR-Smoke](https://rapidrefresh.noaa.gov/hrrr/) — a
  3km-resolution, hourly-updated numerical weather model that includes a
  wildfire smoke tracer.
- **Field used:** `MASSDEN` (near-surface smoke mass density, `heightAboveGround`
  level 8m), forecast hour 00 (analysis) from each hourly model run.
- **Archive:** [NOAA HRRR on AWS Open Data](https://registry.opendata.aws/noaa-hrrr-pds/)
  (bucket `noaa-hrrr-bdp-pds`, free, no authentication required). NOAA's live
  NOMADS server only retains a ~2 day rolling window, so historical dates
  require the AWS (or Google Cloud) archive instead.
- **Format:** GRIB2. Each hourly file's `.idx` sidecar index was used to
  byte-range-fetch only the `MASSDEN` message, avoiding full ~150MB CONUS
  downloads per hour.

## Pipeline

1. **Download** — byte-range fetch the `MASSDEN` message from each of the
   264 hourly HRRR files (11 days × 24 hours) via the `.idx` index files on
   S3. Output: `hrrr_smoke_raw/*.grib2`, one small single-message file per
   hour.
2. **Crop & stack** — read each file with `pygrib`, crop the full CONUS grid
   (1799×1059, Lambert Conformal projection) down to the NYC bounding box
   using a fixed row/column index window (computed once, reused for every
   hour since all HRRR files share the same grid), convert units from kg/m³
   to µg/m³, and stack all hours into a single `xarray.Dataset` with a
   `time` dimension.
3. **Contour & categorize** — for each hour, generate filled contour
   polygons at four smoke-density thresholds (see Legend below) using
   `matplotlib.contourf`, then compute the actual mean smoke value of the
   grid cells inside each polygon (not the threshold boundary itself) so the
   original data is preserved rather than overwritten by the category edges.
   Outputs a geojson file containing all 264 hours' polygons
   in one file.
4. **Display** — a single-page Mapbox GL JS app (`index.html`) loads the
   merged GeoJSON once and uses `setFilter` on `hour_index` to switch
   between hours .

## GeoJSON Feature Properties

Each polygon feature carries:

| Property          | Type            | Description                                                                                                                              |
| ----------------- | --------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `hour_index`      | integer (0–263) | Position in the 264-hour sequence; drives the time slider filter                                                                         |
| `time`            | string          | Human-readable UTC timestamp, e.g. `"2026-07-13T00:00"`                                                                                  |
| `smoke_mean_ugm3` | number          | Actual mean smoke concentration (µg/m³) of the real grid cells within this polygon — preserved from source data, not a category boundary |
| `smoke_level`     | string          | One of `clear`, `low`, `medium`, `high` — see Legend                                                                                     |

## Legend / Smoke Level Categories

Thresholds are derived from the EPA's PM2.5 Air Quality Index breakpoints
(as revised May 2024), collapsed from six official categories into four for
readability:

| Level  | µg/m³ range  | Corresponds to EPA AQI category            | Color       |
| ------ | ------------ | ------------------------------------------ | ----------- |
| Clear  | 0 – 9        | Good                                       | transparent |
| Low    | 9 – 35.4     | Moderate                                   | `#ffeda0`   |
| Medium | 35.4 – 125.4 | Unhealthy for Sensitive Groups + Unhealthy | `#fd8d3c`   |
| High   | 125.4+       | Very Unhealthy + Hazardous                 | `#bd0026`   |

## Map Features

- **Time slider** — scrub through all 264 hours manually.
- **Play / Pause button** — autoplay through hours sequentially (loops at
  the end), pauses automatically if the slider is dragged manually.
- **Legend** — static color key for the four smoke levels.
- **Title & description banner** — states what the map shows and links to
  the NOAA HRRR-Smoke and AWS Open Data source pages.

## Credits

- Data: [NOAA HRRR-Smoke](https://rapidrefresh.noaa.gov/hrrr/), via
  [AWS Open Data](https://registry.opendata.aws/noaa-hrrr-pds/)
- Map rendering: [Mapbox GL JS](https://docs.mapbox.com/mapbox-gl-js/)
