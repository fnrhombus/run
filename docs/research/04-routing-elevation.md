# Route generation + elevation-aware routing (F7)

> Prior-art research brief, generated 2026-06-24 via a fan-out research workflow
> (parallel web research → adversarial verification → synthesis). Claims that
> failed independent verification are flagged inline. Treat as a literature
> survey to ground implementation, not as gospel — spot-check before shipping.

## Overview

This briefing covers route generation and elevation-aware routing for a runner-focused training app (feature F7). It synthesizes how production routing engines actually generate target-distance loops, the formal optimization problems behind them, how to weight routes by grade and surface quality, and how to persist a user's per-road preferences against the moving target of OpenStreetMap (OSM) data.

The bottom line up front: do **not** try to solve the "perfect scenic loop of distance D" exactly — it is NP-hard. Production engines (GraphHopper, OpenRouteService) use a fast geometric heuristic (place waypoints on a circle, route through them, penalize reused edges). Elevation-awareness is a *separate* concern layered on via per-edge slope weighting. For a commercial app, self-hosted GraphHopper open-source is the recommended core: it is the only permissively-licensed engine that ships both native round-trip generation and a precise, declarative slope-aware weighting model.

## Round-trip / loop generation: how it actually works

### The GraphHopper algorithm (the one to copy)

GraphHopper's `round_trip` is a **heuristic, not an exact optimizer**. The mechanism, verified directly against the current source in [`RoundTripRouting.java`](https://github.com/graphhopper/graphhopper/blob/master/core/src/main/java/com/graphhopper/routing/RoundTripRouting.java), is:

1. **Choose a waypoint count.** `pointCount = Math.min(20, 2 + (int)(distanceInMeter / 50000))`. Defaults: `distanceInMeter = 10_000` (10 km), `seed = 0`, `maxRetries = 3`.
2. **Project waypoints around the start.** Each waypoint is projected from the previous point using `DistanceCalcEarth.projectCoordinate(lat, lon, distance, heading)`.
3. **Snap each to the road graph.** If a projected point fails to snap (`locationIndex.findClosest`), its distance is multiplied by `0.95` and retried, up to `maxRetries` times.
4. **Chain shortest paths** through the ordered waypoints, then append the start snap at the end to close the loop.
5. **Penalize already-used edges** so the return leg diverges from the outbound leg. (The commonly-cited penalty factor of ~5 is consistent with GraphHopper's diversity mechanism but was *not* re-verified line-by-line in the current source — treat the exact constant as illustrative, not authoritative.)

The waypoint geometry comes from [`MultiPointTour.java`](https://github.com/graphhopper/graphhopper/blob/master/core/src/main/java/com/graphhopper/routing/util/tour/MultiPointTour.java), which spreads points evenly around 360°:

- `heading(0) = initialHeading` (caller-supplied `headings`, else a seeded random 0–359° angle).
- `heading(i) = initialHeading + 360.0 * i / allPoints` for `i > 0`.
- `distance(i) = slightlyModifyDistance(overallDistance / (allPoints + 1))`.

`slightlyModifyDistance` (in the parent `TourStrategy`) applies a uniform ±10% jitter, i.e. the per-segment length lands uniformly in `[0.9d, 1.1d]`, with the sign of the perturbation chosen by a coin flip.

> **Caveat on the "circle of radius D/(2π)" framing.** It is tempting to describe this as "waypoints on a circle of radius `target_distance / (2π)`." That radius framing is *inferred* from the even 360° spread plus the per-segment distance — it is **not quoted verbatim** from the source. What the code literally does is chain `projectCoordinate` calls segment-by-segment. Use the circle as intuition, not as the exact formula.

### Net behavior and what you control

The achieved loop length is **approximate, not exact**. The two knobs that govern it are the waypoint count and the ±10% per-segment jitter. Reproducibility comes from the seed: the same seed yields the same tour; changing the seed yields a different loop for the same distance. This is exactly the API surface to expose to users ("give me another route").

### The HTTP/API surface

For GraphHopper (confirmed against the [API docs](https://github.com/graphhopper/graphhopper/blob/master/docs/web/api-doc.md) and [core routing docs](https://github.com/graphhopper/graphhopper/blob/master/docs/core/routing.md)):

- `algorithm=round_trip`
- `round_trip.distance` — approximate loop length in meters
- `round_trip.seed` — integer; change for a different tour
- `headings` — north-based clockwise angle 0–360°, forces the *initial* direction of the loop
- `heading_penalty` — time penalty in seconds for not obeying the heading (default 300 s; also used to discourage U-turns at via-points)
- `ch.disable=true` — **mandatory** for heading/round-trip. Contraction Hierarchies (the CH speed-up index) cannot encode headings, so the flexible/landmark mode must be used.
- `elevation=true` — returns 3D points (changes point encoding to include altitude)

GraphHopper's flexible mode uses A\* with a landmark (ALT) heuristic (triangle-inequality-based), which still returns optimal point-to-point routes and handles headings cleanly — whereas CH does not. The "ALT is the reason heading works and CH doesn't" framing is well-established in GraphHopper's design but is a *medium-confidence* synthesis rather than a single quoted line; treat it as the correct mental model, not gospel about internals.

## The formal problem (and why nobody solves it exactly)

If you wanted the *principled* version of "a loop of length ≤ D that maximizes scenery/quality," that is the **Orienteering Problem (OP)**: given a rooted graph with node rewards and edge costs, find a path/cycle from the root with total length ≤ budget `D` maximizing collected reward. It is essentially Knapsack + TSP.

- OP is **NP-hard**. The seminal paper is *The Orienteering Problem*, Naval Research Logistics vol. 34 (1987). **Attribution correction:** this paper is by **Golden, Levy, and Vohra** — *not* Vansteenwegen, who is associated with the later 2011 *EJOR* OP survey. (Original FINDINGS mis-credited the 1987 paper; the verification flagged this as mixed. The NP-hardness fact itself is solid; only the authorship was wrong.) See [Golden, Levy, Vohra, NRL 1987](https://onlinelibrary.wiley.com/doi/abs/10.1002/1520-6750(198706)34:3%3C307::AID-NAV3220340302%3E3.0.CO;2-D).
- OP is also APX-hard (no PTAS unless P=NP). Constant-factor approximations exist for rooted orienteering in metric/undirected settings ([Blum, Chawla, Karger et al., JACM](http://www.cs.cmu.edu/~shuchi/papers/orienteering-jacm.pdf)). For loop routes, the start = finish (rooted cycle) variant applies.

> **Open question — do not cite a specific constant.** The exact constant-factor ratio for *rooted cycle* orienteering in the Blum et al. paper could not be extracted from the PDF (commonly cited as a 3- or 4-approximation for undirected/metric rooted orienteering). **Verify the precise constant before quoting it** in code comments or documentation.

When the reward/scenery lives on **edges** (the natural model for running — "good roads," scenic paths, soft trail), the relevant model is the **Arc Orienteering Problem (AOP)**, also NP-hard. Verified approximation ratios ([ScienceDirect, *Op. Res. Letters* / *Transportation Research Part E* 2014](https://www.sciencedirect.com/science/article/abs/pii/S002001901400218X)):

- `O(log² m)`-approximation in directed graphs (`m` = number of arcs)
- `(6 + ε + o(1))`-approximation in undirected graphs for general profits
- `(4 + ε)`-approximation for unit-profit instances

Verbeeck, Vansteenwegen & Aghezzaf (Transportation Research Part E, 2014) extended AOP and applied it directly to **cycle trip planning** — the closest published analog to scenic-loop generation for an athletic app.

### Practical takeaway

In production, **nobody solves OP/AOP exactly for live round-trip queries.** The dominant, battle-tested pipeline (and the one to replicate) is:

1. Compute `N` waypoints (`N ≈ 2 + dist/50km`, capped ~20) spread around the start, offset by an initial heading.
2. Snap each to the road network; shrink distance by `0.95` on snap failure.
3. Chain shortest paths (A\*+ALT) through the ordered waypoints back to start.
4. Penalize already-used edges so outbound ≠ return.
5. Expose a seed to regenerate alternate loops.

Accept that the length is approximate and tune via waypoint count + per-segment jitter.

## Elevation-aware & quality-aware weighting

Round-trip generation and elevation-awareness are **orthogonal**. The `round_trip` algorithm itself knows nothing about hills — grade preferences are imposed by the *weighting model* used during the shortest-path legs. This is the part you will spend the most tuning effort on. Below are the four self-hostable engines, ranked by how well their weighting fits a runner app.

### GraphHopper custom models (recommended — most precise and auditable)

GraphHopper's [custom model](https://github.com/graphhopper/graphhopper/blob/master/docs/core/custom-models.md) is a declarative JSON ruleset with three blocks:

- `speed` — `[{if, multiply_by | limit_to}]`
- `priority` — `[{if, multiply_by}]`
- `distance_influence` — a number; **default 70** (seconds per km). Higher favors shorter routes; lower favors faster routes.

Internal weight: `edge_distance / (speed * priority) + edge_distance * distance_influence + turn_penalty`. Crucially, `multiply_by` on **speed** affects both weight and time, while `multiply_by` on **priority** affects weight only (not time) — use priority to express "I prefer this road" without lying about pace.

Slope encoded values usable in `if` conditions (verified against the docs):

- `average_slope` = signed `100 * elevation_change / edge_distance`; the sign **flips in the reverse direction** of the edge.
- `max_slope` = unsigned max segment slope along an edge — use for long edges where a local pitch exceeds the average.

Plus `road_class` (FOOTWAY, CYCLEWAY, PRIMARY…), `surface` (PAVED, DIRT, GRAVEL, SAND), `hike_rating` (0–6 from OSM `sac_scale`), `smoothness`, `lit`, `curvature`.

Example "favor gentle, runner-friendly roads":

```json
{
  "priority": [
    { "if": "average_slope < 5", "multiply_by": "1.2" },
    { "if": "road_class == FOOTWAY || road_class == CYCLEWAY", "multiply_by": "1.3" }
  ],
  "speed": [
    { "if": "average_slope > 10", "multiply_by": "0.6" }
  ],
  "distance_influence": 70
}
```

POST `/route` accepts `custom_model` directly. This is the most precise *and* the most auditable option of the four.

### BRouter (most tunable energy model; no native round-trip)

[BRouter](https://brouter.de/brouter/profile_developers_guide.txt) profiles are a full scripting language (`.brf` files), so the cost function is the deepest customization of favored/avoided roads of the four. Elevation variables: global `uphillcost`, `downhillcost`, `uphillcutoff`, `downhillcutoff`; per-way `costfactor`, `uphillcostfactor`, `downhillcostfactor`; noise-filter buffers `elevationpenaltybuffer` (default 5 m), `elevationmaxbuffer` (default 10 m), `elevationbufferreduce` (default 0 slope%).

Slopes within `[downhillcutoff, uphillcutoff]` are treated as flat (`downhillcutoff` typical ≈ 1.5%). A noise filter first cuts `10 * cutoff` per km from elevation change; the remainder accumulates in a buffer and converts to ElevationCost. **Worked example (verified):** with `uphillcost = 60`, a residual 0.25% slope = 2.5 m/km → `2.5 * 60 = 150 m` of added equivalent-length cost.

BRouter is fully self-hostable, free, ships preset profiles (e.g. `trekking.brf`), and uses SRTM elevation baked into its `rd5` data tiles. **No native round-trip** in the core router — loops require manual extra waypoints or a front-end like BRouter-Web ([round-trip feature request #236](https://github.com/nrenner/brouter-web/issues/236)).

> **Open question:** Concrete default values for `uphillcost`/`downhillcost`/`uphillcutoff` are **not** in the dev guide — they live in individual `.brf` profiles. Inspect a specific profile (e.g. `trekking.brf`) to get real defaults before tuning.

### Valhalla (simplest grade knob; MIT; no native round-trip)

[Valhalla](https://valhalla.github.io/valhalla/sif/elevation_costing/) bakes elevation into routing tiles via the Skadi library (worldwide DEM at tile-build time). Bicycle costing exposes `use_hills` in `[0, 1.0]` (0 = avoid hills even if longer; 1.0 = strong cyclist indifferent to hills); penalties apply to descents too, since downhill implies later climbing. Grade is computed by sampling edges (~every 60 m) and weighting via a linear combination, with upslopes weighted more heavily; weighted grade modulates both speed and cost. Pedestrian costing: `type=foot|wheelchair`, with `max_grade`/`max_distance` (`max_grade` is primarily for wheelchair).

**License is confirmed MIT** — the [`COPYING` file](https://github.com/valhalla/valhalla/blob/master/COPYING) is the MIT License. The earlier "some sources say Apache 2.0" hedge is **refuted**; it is MIT, clean for a commercial app. **No `round_trip` parameter exists** — loops must be synthesized by your app via DIY waypoints.

> **Open question:** The exact weighted-grade-to-penalty coefficients are described only graphically; reading `src/sif/bicyclecost.cc` / `pedestriancost.cc` is required to reproduce them precisely.

### OpenRouteService (built-in round-trip; coarse grade; GPLv3 copyleft)

[ORS](https://giscience.github.io/openrouteservice/api-reference/endpoints/directions/routing-options) is a fork of GraphHopper 4.0 and is, besides GraphHopper, the only engine here with a documented built-in round-trip option: POST `directions` `options.round_trip` with `length` (m), `points` (waypoint count), `seed`. Profile shaping via `options.profile_params.weightings`: `steepness_difficulty` (0–3 = Novice…Pro, cycling-oriented), `green {factor}` (prefer green areas), `quiet {factor}`. Plus `avoid_features` (`[highways, tollways, ferries, fords, steps]`), `avoid_polygons`, `avoid_borders`. Elevation via `elevation=true`.

**License is GPLv3** — the [LICENSE file](https://github.com/GIScience/openrouteservice/blob/main/LICENSE) is GNU GPL v3. This is **copyleft**: self-hosting is fine, but *distributing a modified ORS engine* triggers source-disclosure obligations. Stricter than Apache/MIT — a real consideration for a commercial app. (The "LGPL-3.0" mention in some sources reflects bundled libraries; the engine itself is GPLv3.)

> **Open question:** The public ORS API caps round-trip at ~100 km, but that appears to be an *API-tier restriction*, not an engine limit. Confirm against a local deployment config whether self-hosted ORS removes the cap.

### Capability matrix

| | GraphHopper | BRouter | Valhalla | OpenRouteService |
|---|---|---|---|---|
| Native round-trip | **Yes** (`round_trip`+seed) | No (manual/front-end) | No (DIY waypoints) | **Yes** (`options.round_trip`) |
| Grade/elevation weighting | Per-edge `average_slope`/`max_slope` in declarative model (most precise) | Scriptable up/downhill cost + noise buffer (most tunable) | Single `use_hills` 0–1 (bike), `max_grade` (foot) — simplest | `steepness_difficulty` 0–3 (coarse, cycling) |
| Favored/avoided roads | road_class/surface rules | Arbitrary per-tag scripting (deepest) | `use_roads`/`avoid_polygons` | `avoid_features` + green/quiet |
| Self-host | Yes | Yes | Yes | Yes |
| License | **Apache-2.0** | OSS | **MIT** | **GPLv3 (copyleft)** |
| Hosted paid API | Yes | — | — | Yes |

**Recommendation:** Self-hosted **GraphHopper open-source** — the best balance of native round-trip, precise auditable slope-aware custom models, and a permissive (Apache-2.0) license for a commercial runner app. Keep BRouter in mind if you later need a genuinely physiological energy-cost model; its scripting is unmatched, but you would have to build round-trip generation yourself.

> **Open question / version watch:** `distance_influence` default is reported as 70 in current core docs, but custom-model behavior changed across GraphHopper 4.x–6.x. Pin and verify per the exact engine version you deploy.

## Persisting per-road preferences (favorite / hated roads)

This is the trickiest data-modeling problem in F7, because **OSM way IDs are not stable.** Per the [OSM wiki](https://wiki.openstreetmap.org/wiki/Overpass_API/Permanent_ID), IDs "may change at any time… e.g. if an object is deleted and re-added. Also, every split of a way leaves one half with a different ID." On a split, exactly **one** child segment keeps the original ID and history; the rest get new IDs. In [JOSM](https://josm.openstreetmap.de/wiki/Help/Action/SplitWay), the surviving segment is the one with the *most nodes* (not necessarily the geometrically longest), and expert mode lets the editor choose. A single street is also routinely split into many ways for speed limits, surface changes, etc.

**Consequence:** do not key a "favorite/hated road" record on way ID alone. The way ID is a *soft hint/cache*, never an authoritative anchor for a whole road.

### Recommended persistence schema

Anchor preferences to **snapped geometry**, re-resolve to the current way at query time:

```
{
  snapped_lat, snapped_lon,
  snapped_offset_along_way_m,
  captured_way_id,        // cache/hint only
  captured_osm_version,   // cache/hint only
  highway_value,          // for re-expansion
  name,                   // for "same-street" re-expansion
  preference,             // favorite | hated
  weight,
  created_at
}
```

**Re-resolution after an OSM refresh:**
1. Try the cached `way_id`. If it still exists, verify the stored point lies within ~10–15 m of that way's geometry.
2. If the way was split (cached ID now covers only part of the original road), re-snap the stored point to the nearest current way; accept the match if perpendicular distance < ~10–15 m **and** the `highway`/`name` tags are consistent.
3. If the preference was meant for a whole street, expand to adjacent ways sharing the same `name` tag.

This survives splits/merges/renumbering because geometry is the stable anchor. It mirrors OSM's own "Permanent ID" philosophy: identify features by stable tag combinations / geometry, not numeric ID. (Wikidata QID tags are stable but exist only for notable features — not usable for generic residential streets.)

### Snapping a tap to a way

Snapping is a nearest-segment projection problem. The standard baseline is the HMM map-matching model of [Newson & Krumm (2009)](https://www.semanticscholar.org/paper/Hidden-Markov-map-matching-through-noise-and-Newson-Krumm/e573b7076a8a6e1ec8044351e4bd149194ec2e19):

- **Emission probability** (point → segment): `p(z|x) = (1/(σz·√(2π)))·exp(-0.5·(d_gc/σz)²)`, where `d_gc` is the great-circle distance from the tap to the candidate projection.
- **Transition probability** (between consecutive points): `p = (1/β)·exp(-|d_route − d_gc|/β)`.

The measurement-noise parameter `σz ≈ 4.07 m` (derived as `1.4826 × MAD`) is **robustly corroborated** by multiple implementations (Mapzen, [Valhalla Meili](https://valhalla.github.io/valhalla/meili/algorithms/)).

> **Caveat on β.** The transition parameter `β` is **only loosely pinned down**. The "~5 m" figure is *not* well-established: Mapzen's [data-driven map-matching](https://www.mapzen.com/blog/data-driven-map-matching/) adopted `β = 3`, not ~5. Treat β as "order of a few meters, tune empirically," not a fixed constant. For a **single tap** (no trajectory), only the emission/nearest-projection term matters anyway: pick the nearest way by perpendicular distance within a search radius (commonly 50–200 m); β is irrelevant.

### Ready-made snapping endpoints

[OSRM](http://project-osrm.org/docs/v5.24.0/api/) provides HTTP endpoints:

- `GET /nearest/v1/{profile}/{lon,lat}.json?number={n}` — snaps one coordinate; returns waypoints with `location`, `distance`, `name`, `hint`, and `nodes`.
- `GET /match/v1/{profile}/{coords}?radiuses=...` — trajectory matching; `radiuses` = std-dev of GPS precision (default 5 m, consistent with σz ≈ 4–5 m); returns `confidence` in `[0,1]`.

**Critical gotcha:** OSRM's `nodes` field returns the two adjacent **OSM node IDs**, *not* a way ID. To recover the way ID you must either run a routing graph that preserves way IDs, or post-process via Overpass: `way(bn:<node_id>); out tags;`.

## OSM tag model for run-quality weighting

### Routable & access tags

Only `highway=*` (or `junction=*`) ways are routable ([OSM tags for routing](https://wiki.openstreetmap.org/wiki/OSM_tags_for_routing)). Pedestrian-relevant highway values: `footway, path, pedestrian, track, residential, living_street, service, unclassified, tertiary/secondary/primary, cycleway, steps`.

Access hierarchy (specific overrides general): `access=*` → `foot=*` / `bicycle=*` ([Key:access](https://wiki.openstreetmap.org/wiki/Key:access)). For a runner:

- **Hard-block:** `foot=no` / `access=no`
- **Prefer:** `foot=designated`, `highway=footway`/`path` with `foot=yes`/`designated`
- **Soft-avoid:** `foot=private` / `customers`
- `oneway` does **not** restrict pedestrians (only cyclists/cars).

### Surface, smoothness, tracktype → run comfort

[`surface=*`](https://wiki.openstreetmap.org/wiki/Key:surface) — map to a paved/unpaved binary plus a comfort score:

| Surface | Suggested runner weight (1.0 = best) |
|---|---|
| asphalt / concrete / paved | 1.0 |
| paving_stones / compacted / fine_gravel | ~0.9 |
| sett / gravel | ~0.7 |
| dirt / ground / grass | ~0.6 |
| sand / mud | ~0.3 |

`surface` is frequently **absent** — fall back to highway-class defaults (e.g. `residential` → assume asphalt).

[`smoothness=*`](https://wiki.openstreetmap.org/wiki/Key:smoothness) (8 ordinal levels, best→worst): `excellent > good > intermediate > bad > very_bad > horrible > very_horrible > impassable`. For running, `excellent`/`good`/`intermediate` are ideal; `bad`/`very_bad` acceptable trail; `horrible`+ avoid.

[`tracktype=*`](https://wiki.openstreetmap.org/wiki/Key:tracktype): `grade1` (solid/paved) → `grade5` (soft/unimproved). Map `grade1–2` → good, `grade3` → ok, `grade4–5` → soft/avoid when wet.

These give finer signal than `surface` alone; combine all three with highway-class fallback.

### Getting OSM data

- **[Geofabrik](https://download.geofabrik.de/):** pre-cut continent/country `.osm.pbf`, updated daily, ODbL 1.0. Note: Geofabrik strips usernames/changeset IDs (GDPR). `.osm.pbf` is the most space-efficient format.
- **[BBBike](https://extract.bbbike.org/extract.html):** arbitrary polygon/bbox (max ~24M km² or 1500 MB), many formats (pbf, GeoJSON, GeoPackage, PMTiles).
- **[Overpass API](https://wiki.openstreetmap.org/wiki/Overpass_API):** live, read-only, tag-filtered queries. Practical bbox limit ~0.5°×0.5°, rate-limited — use for tag-filtered subsets, not whole countries. Also the tool for `way(bn:NODE_ID)` re-resolution.
- **SliceOSM** (OSMUS) for on-demand custom extracts.

## Suggested architecture for F7

1. **Routing core:** self-hosted GraphHopper open-source (Apache-2.0), `ch.disable=true` profile, fed a Geofabrik `.osm.pbf` with elevation enabled.
2. **Loop generation:** native `round_trip` with `round_trip.distance`, `round_trip.seed` (expose "shuffle" = new seed), optional `headings` to bias direction.
3. **Run-quality weighting:** a GraphHopper `custom_model` combining `average_slope`/`max_slope` rules with `road_class`/`surface`/`smoothness` priorities; tune `distance_influence`.
4. **Personal preferences:** a "favorite/hated road" store keyed on snapped geometry (not way ID); fold each preference into the custom model's `priority` block at query time after geometry re-resolution. Re-snap via GraphHopper's own location index (or OSRM `/nearest` + Overpass `way(bn:...)` if you need way IDs).
5. **Mobile (Expo) client:** call the backend for routing; keep snapping server-side unless you build a local R-tree (see open question).

## Open questions

- **Rooted-cycle orienteering approximation constant.** The exact constant for rooted *cycle* orienteering in Blum et al. (commonly cited 3- or 4-approx) could not be extracted from the PDF — verify before quoting.
- **GraphHopper edge-penalty factor.** The "factor of 5" for round-trip edge diversity was historically reported but not re-verified in current master; current releases may use an alternate-route/via-point scheme instead.
- **Elevation × round-trip interaction.** Biasing waypoint *headings* toward hills (for hill-repeat workouts) or away from them is **not** handled by the `round_trip` algorithm itself — it would require pre-computing a hilly/flat heading from a local DEM and passing it as `headings`. Needs separate design work.
- **BRouter concrete defaults.** Real `uphillcost`/`downhillcost`/`uphillcutoff` values live in `.brf` profiles, not the dev guide — inspect `trekking.brf`.
- **Valhalla grade coefficients.** Exact weighted-grade-to-penalty function is only described graphically; read `bicyclecost.cc`/`pedestriancost.cc`.
- **Self-hosted ORS round-trip cap.** The ~100 km cap appears to be an API-tier limit, not an engine limit — confirm against a local config.
- **Newson & Krumm β.** Not robustly pinned; implementations diverge (Mapzen β=3). Tune empirically rather than hardcoding ~5 m. (σz ≈ 4.07 m is solid.)
- **Preference granularity.** Way-level vs. node-pair (edge) level anchoring — node-pairs survive renumbering even better but are harder to present to a user as "a street." Product decision + testing against real edit churn.
- **OSM way-ID churn rate.** No quantitative source for how often a typical residential street is split/renumbered per year — would inform how aggressively to run the geometry-verification re-resolution.
- **On-device snapping for Expo.** Whether to ship a local R-tree over `.osm.pbf`-derived geometry vs. calling a backend `/nearest` — tradeoff unresolved.

## Sources

- https://github.com/graphhopper/graphhopper/blob/master/core/src/main/java/com/graphhopper/routing/RoundTripRouting.java
- https://github.com/graphhopper/graphhopper/blob/master/core/src/main/java/com/graphhopper/routing/util/tour/MultiPointTour.java
- https://raw.githubusercontent.com/graphhopper/graphhopper/master/core/src/main/java/com/graphhopper/routing/util/tour/TourStrategy.java
- https://github.com/graphhopper/graphhopper/blob/master/docs/web/api-doc.md
- https://github.com/graphhopper/graphhopper/blob/master/docs/core/routing.md
- https://docs.graphhopper.com/openapi/routing/postroute.md
- https://docs.graphhopper.com/openapi/routing/getroute
- https://www.graphhopper.com/blog/2017/08/14/flexible-routing-15-times-faster/
- https://github.com/graphhopper/graphhopper/blob/master/docs/core/custom-models.md
- https://github.com/graphhopper/graphhopper/blob/master/LICENSE.txt
- https://github.com/graphhopper/graphhopper
- https://onlinelibrary.wiley.com/doi/abs/10.1002/1520-6750(198706)34:3%3C307::AID-NAV3220340302%3E3.0.CO;2-D
- https://ideas.repec.org/a/wly/navres/v34y1987i3p307-318.html
- http://www.cs.cmu.edu/~shuchi/papers/orienteering-jacm.pdf
- https://www.sciencedirect.com/science/article/abs/pii/S002001901400218X
- https://www.academia.edu/94829455/An_extension_of_the_arc_orienteering_problem_and_its_application_to_cycle_trip_planning
- https://brouter.de/brouter/profile_developers_guide.txt
- https://brouter.de/brouter/costfunctions.html
- https://github.com/poutnikl/Brouter-profiles/wiki/Glossary
- https://wiki.openstreetmap.org/wiki/BRouter
- https://github.com/nrenner/brouter-web/issues/236
- https://valhalla.github.io/valhalla/sif/elevation_costing/
- https://valhalla.github.io/valhalla/meili/algorithms/
- https://github.com/valhalla/valhalla/pull/3234
- https://github.com/valhalla/valhalla/blob/master/COPYING
- https://github.com/valhalla/valhalla
- https://github.com/GIScience/openrouteservice
- https://github.com/GIScience/openrouteservice/blob/main/LICENSE
- https://giscience.github.io/openrouteservice/api-reference/endpoints/directions/routing-options
- https://openrouteservice.org/restrictions/
- https://wiki.openstreetmap.org/wiki/Overpass_API/Permanent_ID
- https://wiki.openstreetmap.org/wiki/Permanent_ID
- https://josm.openstreetmap.de/wiki/Help/Action/SplitWay
- https://community.openstreetmap.org/t/id-split-line-into-two/63845
- https://www.microsoft.com/en-us/research/wp-content/uploads/2016/12/map-matching-ACM-GIS-camera-ready.pdf
- https://www.semanticscholar.org/paper/Hidden-Markov-map-matching-through-noise-and-Newson-Krumm/e573b7076a8a6e1ec8044351e4bd149194ec2e19
- https://www.mapzen.com/blog/data-driven-map-matching/
- https://link.springer.com/article/10.1007/s42979-022-01340-5
- http://project-osrm.org/docs/v5.24.0/api/
- https://wiki.openstreetmap.org/wiki/OSM_tags_for_routing
- https://wiki.openstreetmap.org/wiki/Key:access
- https://wiki.openstreetmap.org/wiki/OSM_tags_for_routing/Access_restrictions
- https://wiki.openstreetmap.org/wiki/Key:surface
- https://wiki.openstreetmap.org/wiki/Key:smoothness
- https://wiki.openstreetmap.org/wiki/Key:tracktype
- https://download.geofabrik.de/
- https://extract.bbbike.org/extract.html
- https://wiki.openstreetmap.org/wiki/Overpass_API
- https://wiki.openstreetmap.org/wiki/Downloading_data
