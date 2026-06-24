# USGS 3DEP lidar elevation data (F7, user-requested)

> Prior-art research brief, generated 2026-06-24 via a fan-out research workflow
> (parallel web research → adversarial verification → synthesis). Claims that
> failed independent verification are flagged inline. Treat as a literature
> survey to ground implementation, not as gospel — spot-check before shipping.

## USGS 3DEP Lidar Elevation Data: Implementation Briefing

This briefing covers what the USGS 3D Elevation Program (3DEP) offers, the exact products and access methods relevant to a running/training app, and how to wire them in. All findings below were independently verified; verdicts are noted inline where anything was less than fully confirmed.

### TL;DR for the "run" app

- For **single-point elevation** (e.g. snapping a GPS fix or marker to ground), use the keyless [EPQS](https://apps.nationalmap.gov/epqs/) service — one HTTP GET per point, no API key, US-only.
- For **route/track elevation profiles at scale** (the core need for a training app), do **not** hammer EPQS point-by-point. Read the **Cloud Optimized GeoTIFF (COG) DEMs** directly from the public AWS bucket with GDAL/rasterio HTTP range requests, or batch through the [3DEPElevation ImageServer](https://elevation.nationalmap.gov/arcgis/rest/services/3DEPElevation/ImageServer).
- The best-available data is **1-meter lidar-derived DEM** where it exists (now covering ~98% of the US), falling back to **1/3 arc-second (~10 m)** seamlessly.
- Everything is **U.S. Government public domain** — free, no key, no use restrictions, attribution merely requested.

---

## 1. What 3DEP Is and How Complete It Is

3DEP is the USGS program that has produced near-national high-resolution lidar elevation coverage. As of the end of FY2024, 3DEP terrestrial elevation data was **available or in progress for ~98.3% of the Nation**, roughly eight years after the first full production year in 2016, with the baseline phase expected to complete around 2026 ([USGS CIR 1553](https://pubs.usgs.gov/publication/cir1553/full), [What is 3DEP?](https://www.usgs.gov/3d-elevation-program/what-3dep)). Alaska is covered primarily by IfSAR rather than lidar.

The baseline standard is **QL2** (2 points/m²). **Next Generation 3DEP** (launched 2023) targets **QL1** (8 points/m²) with higher refresh frequency and adds inland bathymetry. A frequently-cited planning mix for new collections is roughly **60% QL1 / 40% QL2**, with QL1 prioritized for the Western U.S. and Eastern coastal areas — note this 60/40 figure is a *planning estimate*, not a verified national acquired-coverage split, and the exact acquired QL1/QL2 ratio as of 2025–2026 is not published as a single authoritative number ([Refreshing the 3DEP Baseline – LIDAR Magazine](https://lidarmag.com/2025/02/09/refreshing-the-3dep-baseline/)).

**App relevance:** You can assume high-resolution lidar-derived elevation for essentially anywhere a US user runs, with graceful fallback to 10 m elsewhere. Plan your UX so an occasional "no high-res data here" case (and Alaska's coarser data) is handled, but it will be rare.

---

## 2. Quality Levels — Exact Numbers

The USGS Topographic Data Quality Levels table is fully confirmed. NPS = nominal pulse spacing, NPD = nominal pulse density, RMSEz = vertical accuracy of the source lidar, DEM cell = matching raster resolution ([Topographic Data Quality Levels – USGS](https://www.usgs.gov/3d-elevation-program/topographic-data-quality-levels-qls)).

| QL | NPS (max) | NPD (min) | RMSEz | DEM cell |
|----|-----------|-----------|-------|----------|
| QL0 | ≤ 0.35 m | ≥ 8 pts/m² | 5 cm | 0.5 m |
| QL1 | ≤ 0.35 m | ≥ 8 pts/m² | 10 cm | 0.5 m |
| QL2 | ≤ 0.71 m | ≥ 2 pts/m² | 10 cm | 1 m |
| QL3 | ≤ 1.41 m | ≥ 0.5 pts/m² | 20 cm | 2 m |

Key points: QL0 and QL1 share point density but QL0 has 2× the vertical accuracy (5 cm vs 10 cm). **QL2 is the minimum acceptable QL for new USGS/NGP collections** ([Lidar Base Specification: Collection Requirements](https://www.usgs.gov/ngp-standards-and-specifications/lidar-base-specification-collection-requirements)).

**App relevance:** The 1 m DEM you'll consume corresponds to QL2-or-better source data with ~10 cm RMSEz vertical accuracy. This is the theoretical floor; real-world point-query accuracy is coarser (see §6).

---

## 3. Vertical Accuracy: RMSEz, NVA, VVA

Vertical accuracy is specified as RMSEz and reported at the 95% confidence level following [ASPRS 2014](http://florida.asprs.org/images/documents/ASPRS_Positional_Accuracy_Standards_Edition1_Version100_November2014.pdf):

- **NVA (Non-Vegetated Vertical Accuracy)** = RMSEz × 1.96, applicable in open terrain where errors approximate a normal distribution. So QL1/QL2 at 10.0 cm RMSEz ⇒ **~19.6 cm NVA at 95% confidence**.
- **VVA (Vegetated Vertical Accuracy)** is reported using the **95th-percentile method**, *not* the 1.96 multiplier.

QL2 vertical accuracy was changed from 9.25 cm RMSEz to **10.0 cm RMSEz** to align with the ASPRS 10-cm vertical accuracy class ([Adopt updated accuracy standards](https://www.usgs.gov/ngp-standards-and-specifications/adopt-updated-accuracy-standards), [LBS Revision History](https://www.usgs.gov/ngp-standards-and-specifications/lidar-base-specification-revision-history)).

> Caveat: The ~19.6 cm NVA figure is *derived* (RMSEz × 1.96), which the ASPRS standard confirms as the correct formula. The exact published NVA/VVA threshold strings in the LBS accuracy table were not read line-by-line, so treat the specific VVA centimeter value as approximate.

**App relevance:** For a running app, do not advertise sub-decimeter elevation precision to users. The honest, real-world figure to internalize is the EPQS service RMSE of ~0.53 m (§6), not the lidar's 10 cm RMSEz.

---

## 4. Products You Can Actually Consume

### Seamless DEMs (raster, bare-earth)

| Product | Spacing | Coverage |
|---------|---------|----------|
| 1/3 arc-second | ~10 m N/S (variable E/W) | CONUS + HI + PR + territories + limited AK |
| 1 arc-second | ~30 m | National (CONUS complete) |
| 2 arc-second | ~60 m | Alaska only *(consistent with USGS practice; not separately verified)* |
| **1-meter Seamless (S1M)** | 1 m | Highest-res seamless; COG tiles, production ongoing |

The **S1M** product is delivered as 10 km × 10 km cloud-optimized GeoTIFF tiles, added as available ([About 3DEP Products & Services](https://www.usgs.gov/3d-elevation-program/about-3dep-products-services), [Download Data & Maps from TNM](https://www.usgs.gov/tools/download-data-maps-national-map)).

### Project-based 1-meter DEM

The standard 1 m DEM: 1 m pixel, **UTM projection in meters, NAD83 horizontal datum, NAVD88 vertical datum**, single "elevation" band in meters, derived exclusively from QL2-or-better lidar ([USGS 3DEP 1m – Earth Engine catalog](https://developers.google.com/earth-engine/datasets/catalog/USGS_3DEP_1m)). Important gotcha: surfaces are **seamless *within* a collection project but not necessarily *across* projects** — tiles spanning UTM zones can show slight elevation discontinuities at project boundaries.

### Lidar Point Cloud (LPC)

Classified LAS/LAZ — the raw 3D points DEMs are derived from. The current spec is **Lidar Base Specification 2025 rev. A (released June 10, 2025)**, which requires **LAS 1.4-R15** (Point Data Record Format 6–10), with the minimum classification scheme in Table 5 and **no points allowed to remain in Class 0** ([LBS Revision History](https://www.usgs.gov/ngp-standards-and-specifications/lidar-base-specification-revision-history), [LBS Deliverables](https://www.usgs.gov/ngp-standards-and-specifications/lidar-base-specification-deliverables)). Increasingly distributed as COPC / EPT for cloud streaming.

**App relevance:** For "run," the **gridded DEM is what you want**, not the point cloud. Point clouds are overkill (gigabytes per project) for sampling elevation along a GPS track. Use the 1 m DEM where available, falling back to 1/3 arc-second.

---

## 5. Access Method #1 — EPQS (single point)

The **Elevation Point Query Service** returns elevation for one lon/lat point via a keyless GET. Verified live:

```
GET https://epqs.nationalmap.gov/v1/json?x=-105.0&y=39.7&wkid=4326&units=Meters&includeDate=true
```

Verified response:
```json
{"location":{"x":-105,"y":39.7,"spatialReference":{"wkid":4326,"latestWkid":4326}},
 "locationId":0,"value":"1594.441284180","rasterId":51787,"resolution":1,
 "attributes":{"AcquisitionDate":"2/6/2021"}}
```

- Params: `x`, `y` (coords); `wkid` = 4326 (lon/lat) or 3857; `units` = `Meters`|`Feet`; `includeDate` = `true`|`false`. Output via `/v1/json` or `/v1/xml`.
- `value` is elevation **as a string** — parse it. `resolution` tells you which source tier was hit (`1` = 1 m lidar). Null/NoData is returned outside coverage.
- **No API key, no documented rate limits, US-only, one point per request** ([EPQS](https://apps.nationalmap.gov/epqs/)).

**App relevance:** Great for one-off lookups (drop a pin, "what's the elevation here?"). **Do not** use it to build a 1,000-point elevation profile of a run — that's 1,000 sequential HTTP calls with no batch endpoint and unknown throttling behavior. Use COG reads (§7) for profiles.

---

## 6. EPQS Accuracy (be honest with users)

EPQS interpolates from the 3DEPElevation dynamic service: it samples **1 m lidar-based DEMs where available, falling back to 1/3 arc-second (~10 m)**. The service has an **overall vertical RMSE of ~0.53 m**, and the values are **interpolated, not official surveyed elevations** — accuracy degrades in high-relief terrain ([How accurate are EPQS elevations? – USGS FAQ](https://www.usgs.gov/faqs/how-accurate-are-elevations-generated-elevation-point-query-service-national-map)).

**App relevance:** ~0.53 m RMSE is the number to design around for elevation-gain calculations. Naively summing per-point deltas along a noisy DEM-sampled track **massively overestimates cumulative gain** — you must smooth/threshold (e.g., ignore deltas below a noise floor of ~1–3 m, or smooth the profile) before computing total ascent. This is the single most important practical caveat for a training app.

---

## 7. Access Method #2 — COG DEMs on AWS (best for route profiles)

3DEP DEMs are distributed as **Cloud Optimized GeoTIFFs** in a public, no-auth AWS bucket — ideal for HTTP range-request reads.

- Bucket: `prd-tnm.s3.amazonaws.com` (public, anonymous).
- Path pattern: `StagedProducts/Elevation/{res}/TIFF/current/{tile}/USGS_{res}_{tile}.tif` where `res` = `1` (1 m), `13` (1/3 arc-second), or `1` (1 arc-second); tiles are 1×1 degree cells like `n58w063`.
- Seamless 1/3 arc-second VRT mosaic (HEAD-verified 200, anonymous): `https://prd-tnm.s3.amazonaws.com/StagedProducts/Elevation/13/TIFF/USGS_Seamless_DEM_13.vrt`
- COG = single-band Float32 with internal tiling + overviews; GDAL `/vsicurl/` and rasterio fetch only the needed byte ranges ([What are COGs? – USGS FAQ](https://www.usgs.gov/faqs/what-are-cloud-optimized-geotiffs-cogs)).

**App relevance:** This is the right backend for elevation profiles. Open the COG (or VRT mosaic) over `/vsicurl/`, then sample all points of a run's GPS track in a single windowed read. For a mobile-facing product, do the sampling **server-side** (a small service using GDAL/rasterio or PDAL) and return a clean profile to the app — Expo/React Native can't run GDAL on-device. Cache results per route. Mirror tiles for hot regions into your own bucket/CDN if you scale.

---

## 8. Access Method #3 — TNM Access API (discovery/bulk download)

The **National Map Access API** is the single programmatic API to discover and download all TNM products. Live-verified keyless:

```
GET https://tnmaccess.nationalmap.gov/api/v1/products?datasets=Lidar%20Point%20Cloud%20(LPC)&bbox=-105.3,40.0,-105.2,40.1&prodFormats=LAS,LAZ
```

- Base: `https://tnmaccess.nationalmap.gov/api/v1/`; endpoints `/products` (search) and `/datasets` (list dataset names). GET or POST, no key.
- Key params: `bbox` (minX,minY,maxX,maxY in WGS84), `polygon`, `datasets`, `prodFormats` (GeoTIFF, IMG, LAS, LAZ), `dataType`, `start`/`end`, `offset`, `max`, `outputFormat=JSON`.
- Response: `items[]` with `title`, `sourceId`/`downloadURL`, `format`, `sizeInBytes`, `boundingBox`, `publicationDate`, `lastUpdated` ([TNM Access API](https://apps.nationalmap.gov/tnmaccess/), [USGS FAQ: TNM API](https://www.usgs.gov/faqs/there-api-accessing-national-map-data)).

> Caveat: The exact `datasets` enum strings (e.g. `'Lidar Point Cloud (LPC)'`, `'National Elevation Dataset (NED) 1/3 arc-second'`, `'Digital Elevation Model (DEM) 1 meter'`) come from prior USGS conventions and JS-rendered docs — **validate them against a live `GET /api/v1/datasets` call before hardcoding**. The endpoint and keyless access are confirmed working.

**App relevance:** Use TNM Access for *discovery and bulk pre-fetch* (which 1 m tiles cover a region, when they were collected), then read pixels from COG (§7). It's a catalog/download API, not a per-point elevation API.

---

## 9. Access Method #4 — 3DEPElevation ImageServer (WMS/WCS/REST)

A dynamic ArcGIS ImageServer exposing the seamless multi-resolution bare-earth DEM. Live `?f=json`-verified:

- REST base: `https://elevation.nationalmap.gov/arcgis/rest/services/3DEPElevation/ImageServer`
- `pixelType` F32, single band, native CRS **EPSG:3857 (wkid 102100)**, native cell ~1 m, `exportImage` max **8000×8000 px**.
- `identify` returns elevation at a point (EPQS-like). WMS GetCapabilities and WCS are both enabled (same service path, `service=WMS` / `service=WCS`); WCS pulls raw float elevation grids.
- Server-side raster functions include Hillshade Gray, Multidirectional Hillshade, Aspect, Slope, Elevation Tinted Hillshade, and Contour variants ([3DEPElevation ImageServer](https://elevation.nationalmap.gov/arcgis/rest/services/3DEPElevation/ImageServer)).

> Correction (verification = *mixed*): a prior claim of "11 raster functions" is wrong — the live service lists **14** (including "None"). The named functions otherwise match.

**App relevance:** Use WCS `GetCoverage` to pull a raw float DEM clip for a bounding box without managing tiles, or the rasterFunctions to generate **hillshade/slope map tiles** for your map UI server-side. For an Expo map, the Hillshade/Tinted-Hillshade outputs are an easy way to add terrain shading.

---

## 10. Access Method #5 — Raw Point Clouds on AWS (probably not needed)

Two us-west-2 buckets, organized by USGS project name:

- **Public EPT (anonymous):** `s3://usgs-lidar-public` — Entwine Point Tiles (streamable octree of LAZ); listing live-verified (e.g. `AK_BrooksCamp_2012/boundary.json`). Stream with PDAL/Entwine.
- **Raw LAZ (Requester-Pays):** `s3://usgs-lidar` — raw LAZ (LAS 1.4), broader coverage than EPT.
- Discovery via [usgs.entwine.io](https://usgs.entwine.io/) ([AWS Open Data Registry](https://registry.opendata.aws/usgs-lidar/)).

**App relevance:** Only relevant if you ever need surface analysis beyond bare-earth elevation (e.g., tree canopy, trail-surface roughness). For elevation profiles, skip these and use the gridded COG DEMs.

---

## 11. Licensing

All 3DEP / National Map elevation data is **U.S. Government public domain** — free, no copyright, no use restrictions; you may use, copy, distribute, adapt, and display it without permission. A citation is *requested but not required*: "Data available from U.S. Geological Survey, National Geospatial Program." (For map services: "Map services and data available from U.S. Geological Survey, National Geospatial Program.") The AWS registry lists the lidar license as "US Government Public Domain." Narrow exceptions to public-domain status apply only to some 2010–2016 US Topo road/ortho layers — **not** to 3DEP elevation/lidar ([TNM Terms of Use](https://www.usgs.gov/faqs/what-are-terms-uselicensing-map-services-and-data-national-map), [About 3DEP Products & Services](https://www.usgs.gov/3d-elevation-program/about-3dep-products-services)).

**App relevance:** You can bundle, cache, derive products from, and serve this data commercially with zero licensing cost or attribution obligation. Add the requested attribution string in your app's About/Credits screen as good practice.

---

## 12. Recommended Architecture for "run"

1. **Backend elevation service** (Node/Python with GDAL/rasterio) that takes a GPS polyline and returns a sampled elevation profile. Sample from 1 m COG where `resolution=1`, fall back to 1/3 arc-second.
2. **Single-point lookups** in the app (drop-a-pin) → call EPQS directly (keyless, simple), or proxy through your backend to add caching.
3. **Elevation-gain computation:** smooth or threshold the profile (account for ~0.53 m service RMSE) before summing ascent — do not sum raw per-sample deltas.
4. **Map terrain shading** (optional): pull Hillshade/Tinted-Hillshade tiles from the 3DEPElevation ImageServer rasterFunctions server-side.
5. **Tile pre-fetch/caching:** use TNM Access API to enumerate tiles for popular regions; cache COG reads or mirror tiles to your own CDN.
6. **Datum awareness:** elevations are NAVD88 meters in CONUS; if you ever compare against barometric/GPS (WGS84 ellipsoidal) altitude from the device, apply a geoid correction — they are not directly comparable.

---

## Open Questions

- **Exact `datasets`/`prodFormats` enum strings** for the TNM Access API are inferred from prior conventions and JS-rendered docs; validate via a live `GET https://tnmaccess.nationalmap.gov/api/v1/datasets` before hardcoding.
- **National QL1 vs QL2 acquired-coverage split** as of 2025–2026 is not published authoritatively; the 60/40 figure is a planning estimate only.
- **S1M downloadability via TNM Access** as a distinct dataset tag, and its national coverage percentage as of mid-2026, are unconfirmed.
- **Exact NVA/VVA threshold values** in the LBS accuracy table were derived (RMSEz × 1.96) rather than read directly; the specific VVA centimeter value is approximate.
- **EPQS/3DEPElevation rate limits**: none are documented, but this is an unproven negative. Assume bulk per-point querying may be throttled and prefer COG reads for volume.
- **Native vertical datum returned by EPQS/3DEPElevation** is expected to be NAVD88 (via a geoid model) for CONUS but was not explicitly stated on fetched pages — confirm before doing precise datum math.
- **EPT bucket currency**: whether `usgs-lidar-public` (EPT) is kept as current as the `prd-tnm` staged COG DEMs is unclear; the staged COG DEMs are the more authoritative/current source for gridded elevation.
- **LBS 2025 rev. A QL changes**: confirmed it standardizes LAS 1.4-R15 and Class-0 prohibition, but whether it altered any QL pulse-density/accuracy thresholds or added topobathy QLs vs v2.1 was not exhaustively checked (the QL table in §2 is from the current published QL page and is verified).

## Sources

- https://lidarmag.com/2025/02/09/refreshing-the-3dep-baseline/
- https://www.usgs.gov/3d-elevation-program
- https://pubs.usgs.gov/publication/cir1553/full
- https://www.usgs.gov/3d-elevation-program/what-3dep
- https://www.usgs.gov/3d-elevation-program/topographic-data-quality-levels-qls
- https://www.usgs.gov/ngp-standards-and-specifications/lidar-base-specification-collection-requirements
- https://www.usgs.gov/ngp-standards-and-specifications/vertical-accuracy-assessment-using-ground-points
- https://www.usgs.gov/ngp-standards-and-specifications/adopt-updated-accuracy-standards
- http://florida.asprs.org/images/documents/ASPRS_Positional_Accuracy_Standards_Edition1_Version100_November2014.pdf
- https://www.usgs.gov/ngp-standards-and-specifications/lidar-base-specification-revision-history
- https://www.usgs.gov/3d-elevation-program/about-3dep-products-services
- https://www.usgs.gov/tools/download-data-maps-national-map
- https://data.usgs.gov/datacatalog/data/USGS:4f34caac-f28f-4ea0-8d82-eafb2b8f9a5d
- https://developers.google.com/earth-engine/datasets/catalog/USGS_3DEP_1m
- https://data.usgs.gov/datacatalog/data/USGS:77ae0551-c61e-4979-aedd-d797abdcde0e
- https://www.usgs.gov/ngp-standards-and-specifications/lidar-base-specification-online
- https://www.usgs.gov/ngp-standards-and-specifications/lidar-base-specification-deliverables
- https://www.usgs.gov/ngp-standards-and-specifications/lidar-base-specification-data-processing-and-handling-requirements
- https://apps.nationalmap.gov/epqs/
- https://epqs.nationalmap.gov/v1/json?x=-105.0&y=39.7&wkid=4326&units=Meters&includeDate=true
- https://epqs.nationalmap.gov/v1/docs
- https://www.usgs.gov/faqs/how-accurate-are-elevations-generated-elevation-point-query-service-national-map
- https://apps.nationalmap.gov/tnmaccess/
- https://tnmaccess.nationalmap.gov/api/v1/docs
- https://tnmaccess.nationalmap.gov/api/v1/products?datasets=Lidar%20Point%20Cloud%20(LPC)&bbox=-105.3,40.0,-105.2,40.1&prodFormats=LAS,LAZ
- https://www.usgs.gov/faqs/there-api-accessing-national-map-data
- https://elevation.nationalmap.gov/arcgis/rest/services/3DEPElevation/ImageServer
- https://www.usgs.gov/news/technical-announcement/new-elevation-map-service-available-usgs-3d-elevation-program
- https://www.usgs.gov/faqs/what-are-cloud-optimized-geotiffs-cogs
- https://prd-tnm.s3.amazonaws.com/StagedProducts/Elevation/13/TIFF/USGS_Seamless_DEM_13.vrt
- https://prd-tnm.s3.amazonaws.com/StagedProducts/Elevation/1/TIFF/current/n58w063/USGS_1_n58w063.xml
- https://registry.opendata.aws/usgs-lidar/
- https://www.usgs.gov/news/technical-announcement/usgs-3dep-lidar-point-cloud-now-available-amazon-public-dataset
- https://usgs.entwine.io/
- https://planetarycomputer.microsoft.com/dataset/3dep-seamless
- https://www.usgs.gov/faqs/what-are-terms-uselicensing-map-services-and-data-national-map
