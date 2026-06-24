# `run` — feature & design notes

A personal running/training app for the author and a few close friends (no
public release intended). This doc consolidates the design discussion: the
vision, the architecture, the model at its core, and the feature set. Where a
claim is backed by the prior-art research, it links to the relevant report in
`research/`.

**Companion docs**
- `scosche-rhythm24.md` — BLE protocol for the Scosche Rhythm 24 HR armband.
- `hardware.md` — DIY build / buy / skip decisions for every sensor.
- `stack.md` — tech-stack & architecture preferences (React, event sourcing,
  Azure).
- `research/` — cited prior-art research reports (one per backlog area).

## Why this project exists

- Existing apps **paywall features** and/or **compute pace badly** (jumpy GPS
  instantaneous pace is the usual culprit — see `hardware.md` for the fix).
- Goal: a better personal tool — especially at pace, calorie burn, and adaptive
  guidance, where the incumbents are weak — without their paywalls or weak pace
  math.

## Hardware context

- On hand: Bluetooth optical HR monitor (Scosche Rhythm 24), Android phone.
- The author can fab PCBs and program ESP32 / Pi Pico → most body sensors are
  DIY (IMU foot/belt pods, breathing band); a few are buy/skip. Full rationale
  in `hardware.md`. Candidate sensors: foot pod (BLE RSC `0x1814`), sacrum/belt
  IMU, respiration band (RIP), barometer/grade, SmO₂ (NIRS), core temp, CGM.

## Status legend

`idea` (noted) · `scoped` (design agreed) · `building` · `done`.
Flags: **lock-in** (commit regardless) · **research-first** (now backed by
`research/`).

---

## Feature index

| ID | Feature | Group | Status |
|----|---------|-------|--------|
| F0 | Unified event-sourced data store + open export | Architecture | `idea` · lock-in |
| F2 | Personal physiological "master equation" | Model | `scoped` (research in) |
| F1 | Haptic HR/pace/effort-zone coaching | Live coaching | `idea` |
| F10 | Running power from own IMUs | Live coaching | `idea` |
| F13 | Transparent "why" explanations | Live coaching | `idea` · principle |
| F17 | Audio coaching | Live coaching | `idea` |
| F11 | Calibration field tests | Calibration & readiness | `idea` |
| F9 | Daily readiness score | Calibration & readiness | `idea` · lock-in cand. |
| F4 | Near-24/7 wear: resting HR, max HR, sleep | Tracking & physiology | `idea` |
| F5 | Medication tracking + PK concentration model | Tracking & physiology | `scoped` (research in) |
| F3 | Diet tracking + energy balance | Tracking & physiology | `idea` (research in) |
| F8 | Auto-ingest weather (esp. humidity) | Tracking & physiology | `idea` |
| F12 | Heat-safety advisor | Tracking & physiology | `idea` · tailored |
| F14 | Health anomaly flags from RR data | Tracking & physiology | `idea` |
| F6 | Dynamic / adaptive training guidance | Training guidance | `scoped` (research in) |
| F7 | Elevation-aware route designer | Routes & terrain | `scoped` (research in) |
| F15 | Auto shoe-mileage tracking | Conveniences | `idea` |
| F16 | Locomotor-respiratory coupling training | Conveniences | `idea` |

---

# Architecture & cross-cutting principles

## F0 — Unified event-sourced data store + open export · `idea` · **lock-in**

The substrate everything else needs: a clean, timestamped, multi-channel
**time-series log of every sensor stream**, captured durably and exportable in
open formats (FIT / GPX / Parquet / CSV).

- Every model and feature is downstream of this — they're only as good as the
  logged data. A clean unified log is what makes the F2 model, the projections,
  and the analytics possible at all.
- Open-format export keeps the data portable for offline analysis and model
  fitting.
- Normalize sources into named channels (`hr`, `rr`, `speed`, `cadence`,
  `grade`, `respRate`, `smo2`, `medConc`, `mass`, …) on a common clock —
  generalizes the two-channel Scosche design in `scosche-rhythm24.md`.
- Implementation uses **event sourcing + CQRS** (see `stack.md`): the log *is*
  an append-only event stream; read models/projections (the query side) derive
  everything else.
- **Lock this in regardless.**

## CP1 — The app is a human-in-the-loop control system (F1/F2/F6)

HR-zone is the **setpoint**, the runner is the **actuator** (commanded via
haptics/audio), pace/effort is the control output.

- HR has **dead time + first-order lag**, so naïve PID on HR oscillates — the
  failure mode F1's dead-band guards. **Research confirms the plant**: a
  first-order-plus-dead-time response (a two-time-constant structure fits
  better), see [research/01 §7](research/01-physiology-master-equation.md).
- Right architecture: **feedforward** from grade (known instantly via
  barometer/map) through the **F2 model as the plant model**, with HR
  **feedback only to trim**. The research's Hammerstein model is **invertible
  for feedforward pace** — exactly this design, and its "central design fact."
- Endpoint: **Model Predictive Control** — use F2 to look ahead over a route's
  grade profile (F7) and plan a pace trajectory that holds HR in zone. MPC
  handles dead time by predicting, not reacting.

## CP2 — The master equation must be fully invertible (extends F2)

F2 must be **solvable for ANY variable** given the others:
- "What HR to hold to climb this hill at pace X?" · "What pace is sustainable at
  HR zone Y on this grade/temperature?" · "What **effort (RPE)** to maintain to
  hold target Z?" → the haptic/audio channel can cue a change in *effort*, not
  only pace.
- Favors a **grey-box / physics-informed** model (invertible + interpretable).
  The research backs this: the Cheng–Su grey-box and the invertible Hammerstein
  structure give exactly the both-directions solvability we want
  ([research/01 §7](research/01-physiology-master-equation.md)).

## CP3 — Capability gating (not full graceful degradation)

Friends may have **few or none** of these sensors.

**Decision:** features **gate on/off** by connected sensors. Have the sensor →
feature available; don't → hidden/disabled. **Do not** build alternate
derivation paths that reconstruct a feature's data from a different sensor set.
- e.g. no foot pod → no foot-pod metrics (don't synthesize from GPS); no HR
  strap → HR features off.
- Each feature declares the inputs it requires; the app shows/hides accordingly.
  Simple capability flags, not fallback estimators.
- F2 requires its inputs present to run. (Population *priors* for personal
  *coefficients* are still fine — that's calibration from few runs, §Model, not
  substituting for a missing live sensor.)

> **PARKED — possible future direction (deferred, not rejected).** Full graceful
> degradation — reconstructing a feature's data from a different sensor set — is
> technically possible. If revisited: capability *tiers* (phone-only
> GPS+baro+IMU → +HR → +foot pod/breathing/SmO₂) with per-variable fallback
> estimators (pace: GPS-only → GPS+foot-pod fusion; HRmax: age formula →
> observed; calorie burn: METs → HR→VO₂ → calibrated F2; grade: DEM →
> barometer), and F2 giving a best estimate from *any subset* of inputs with
> uncertainty that widens as inputs drop (Bayesian / latent-variable framing —
> a missing sensor becomes a prior, not a hard failure). *Future reviewer: this
> door is open if/when you want it.*

## CP4 — Transparency by default (see F13)

Every recommendation explains itself. Both a feature (F13) and a constraint that
favors the grey-box model (CP2): black boxes can't explain themselves.

---

# The model (F2) — the core of the app

## F2 — Personal physiological "master equation" · `scoped` · research in

Fit a personal model relating the runner's key variables so it can be **solved
for any one of them given the others** (CP2). Full prior-art and the concrete
equations to use are in
[research/01](research/01-physiology-master-equation.md); summary below.

- **Variables:** pace/speed, ambient temperature (& humidity, F8), grade, heart
  rate, breathing, subjective effort (RPE/Borg), and body mass (an input — see
  below). Medication concentration (F5) enters as a covariate.
- **Headline use case — feedforward pacing:** grade is known *immediately* while
  HR lags 10–30 s, so the model predicts the pace that keeps HR in zone
  **before** HR drifts; F1 cues "slow down" on the way *into* the hill (CP1).

### Concrete building blocks the research settled

- **Grade energetics:** Minetti cost-of-transport polynomial `Cr(i)` (verified);
  **clamp grade to ±0.45**. Grade-Adjusted Pace via Minetti (default) or
  Strava's quadratic (empirical alternative).
- **Metabolic baseline:** ACSM running VO₂ equation (verified); VO₂→kcal via the
  caloric-equivalent-of-O₂ pipeline.
- **Capacity & reserve:** the **Critical Speed / Critical Power** 2-parameter
  model (verified) — this is the CS + D′ pair below.
- **HR dynamics:** first-order + dead-time (two-time-constant better);
  **invertible Hammerstein** for feedforward — see CP1/CP2.
- **Heat/humidity:** treat temperature and humidity as **two separate channels**;
  cardiac drift, Physiological Strain Index, WBGT scalar, Pa:HR/Pw:HR aerobic
  decoupling all have usable forms in the report.
- **RPE anchoring:** use **%HRR (Karvonen), not %HRmax**; Borg 6–20 + Foster
  CR10 / session-RPE.

### Personal coefficients ("current strength level")

Roughly **5–8 coefficients**, four roles. MVP starts with ~5 (CS, D′, economy,
HRmax, RHR) + one HR time-constant for the control loop; add the rest as data
justifies.

| Role | Coefficient(s) | Notes |
|------|----------------|-------|
| Aerobic capacity & reserve | **Critical Speed (CS)**, **D′** | *This is "strength."* Verified 2-parameter CP model. |
| Efficiency | **Running economy** | O₂ cost per kg per km; distinct from capacity. |
| HR coupling | **HRmax**, **resting HR** (→ HR reserve); optionally HR–VO₂ slope | Use %HRR for zones. |
| Dynamics & environment | **HR time constant(s) + dead time**, **heat/humidity sensitivity** | Plant-ID params (CP1) + cardiac-drift rate. |
| Perceptual / fuel (optional) | **RPE gain + offset**, **substrate/fat-max** | For the "cue me on effort" use (CP2) and F3 fuel side. |

### Two kinds of personal parameters — don't conflate them

- **Fitness coefficients** (above) change over *weeks* — CS, D′, economy,
  HRmax. "How fit am I."
- **Daily-state inputs** are *measured, not fitted*: HRV/readiness (F9),
  medication concentration (F5), heat/humidity (F8), accumulated fatigue, and
  **body mass**. "What's my state today." The controller needs both.

### Body mass is an input, not a coefficient

- Fitness coefficients are **mass-normalized** (VO₂max mL/kg/min, economy
  mL/kg/km, CS a speed), so mass enters separately as a scalar on the energetic
  terms.
- **Calorie burn scales ~linearly with mass** (~1 kcal/kg/km flat); **hills
  scale harder** (∝ mass·g·height).
- **Consequence:** a weight change needs **no recalibration** — log a new weight
  (F3 wants daily weigh-ins anyway) and the model rescales burn/hill terms
  instantly; slow coefficients stay put. Use **total moving mass** =
  body + carried. Composition drift is absorbed by slow recalibration.

### Identifiability & personalization

More coefficients → more data to fit (ties to F11). Use a **hierarchical**
approach: start from population priors, let each person's own data pull their
coefficients off the average as it accumulates. A sensor-light friend runs on
population values with wide uncertainty.

### Modeling stance (research-informed)

- A **dynamic** model (state-space / ODE / Hammerstein), **not** a static
  regression — a static fit is wrong exactly during transitions (the hill case).
- **Grey-box / physics-informed** for invertibility + transparency (CP2/CP4);
  black-box ML only helps short-horizon and isn't needed for pace zones.
- Open items: exact two-time-constant vs single, the speed-dependence of GAP
  (no published blend), and per-user heat-sensitivity calibration.

---

# Live run coaching

## F1 — Haptic zone coaching · `idea`

Phone vibrates to tell the runner to speed up / slow down to hold a target zone
**without looking at the screen** (phone strapped to upper arm).

- **Input:** live HR vs. a target zone (use **%HRR** bounds — research/01 §9).
  Extends to **pace** and **effort** zones once F10/F2 exist (CP2).
- **Output:** distinct vibration patterns — speed-up / slow-down / in-zone
  (silence) — distinguishable by feel through a sleeve; stronger pattern for
  "way out" vs gentle for "drifting."
- **Design notes (control loop — CP1):** cue off **smoothed HR + rate-of-change**
  with a **dead-band** and a minimum re-alert interval, so it doesn't nag near a
  boundary or chase HR lag. Android: `expo-haptics` is preset-only; rich custom
  patterns need the native `Vibration` API — confirm against v56 docs. Must fire
  with the app backgrounded → **foreground service** during an active run.

## F10 — Running power from own IMUs · `idea`

Derive a real-time **running power** metric from the DIY foot/belt IMUs + grade
(Stryd sells this for ~$200; we're building the sensors anyway — `hardware.md`).
Power responds to grade *instantly* (no HR lag) → a **better pacing target than
HR for hills** (CP1). Model structures (GOVSS, di Prampero, Stryd) and the
power↔economy relationship are in
[research/01 §6](research/01-physiology-master-equation.md). Can drive a haptic
*power*-zone mode (F1).

## F13 — Transparent "why" explanations · `idea` · principle (CP4)

Every recommendation explains itself — *"ease off: grade hit 6% and you're 4 bpm
over zone."* Answers the anti-paywall, anti-black-box motivation; favors the
grey-box F2.

## F17 — Audio coaching · `idea`

Voice cues via earbuds as a richer complement to F1 — splits, pace, "ease off"
without looking at the screen.

---

# Calibration & readiness

## F11 — Calibration field tests · `idea`

Periodic structured tests (critical-speed / threshold) to fit F2's coefficients
and anchor F6. Without these, the master equation is uncalibrated. The CS/CP test
protocols and race-pace mapping are in
[research/01 §5](research/01-physiology-master-equation.md). Byproduct: a
race-time predictor (Riegel) for free.

## F9 — Daily readiness score · `idea` · **lock-in candidate**

Synthesize overnight HRV + RHR + sleep (F4), recent training load, and
medication state (F5) into one number that **drives F6's day-of decision**. The
HRV-guided decision rule to implement is in
[research/03](research/03-adaptive-training.md) (modest but real benefit). Smooth
it — don't react to single-day noise (same lesson as F1).

---

# Tracking & physiology

## F4 — Near-24/7 wear: resting HR, max HR, sleep · `idea`

Wear the HR monitor as continuously as the charge cycle allows; mine passive
data for baseline physiology.

- **Resting HR** from the lowest sustained HR — a strong fitness/recovery/illness
  signal; feeds F2/F9.
- **Max HR** — *caveat:* true HRmax only appears at near-max effort, never at
  rest. Keep an *observed* max from hard sessions; use an age formula only as a
  prior until a real max is seen; label which is which. Anchors zones for F1/F2.
- **Overnight HRV** — per-beat RR already available in the device's HRV mode
  (`scosche-rhythm24.md`); nighttime rMSSD is the standard recovery metric.
  Almost-free. Confirm 24/7 HRV mode is OK for battery.
- **Sleep** — sleep/wake + duration (rough staging) from HR+HRV+actigraphy.
  Promise timing/duration confidently, stages loosely.

**Constraints:** daily charge window (surface the gap); rely on the device's
**onboard FIT recording + periodic background sync** rather than streaming 24/7
(BLE drops then don't matter); skin tolerance. Onboard-storage full-behavior is
an open Q in `scosche-rhythm24.md`.

## F5 — Medication tracking + PK concentration model · `scoped` · research in

Log daily medication (author takes amphetamines daily) and model estimated
**current blood concentration** C(t), then learn how it correlates with
physiology and performance. Concrete PK parameters and per-formulation models
are in [research/02](research/02-medication-pk.md).

- **Log:** dose (mg), time, and **formulation** (IR / Adderall XR two-pulse /
  Mydayis / **Vyvanse prodrug** — each has its own model in the report).
- **PK model:** research recommends a **one-compartment** model (resolves our
  earlier open question), with multi-dose superposition. d-amphetamine clearance
  is **strongly urine-pH dependent** — a real covariate. **UI must be honest:
  model estimate, not a blood measurement.**
- **Why it matters — a confounder for everything:** raises HR/BP (bias in F1
  zones and F2 → enter as a **covariate**); suppresses appetite (F3); disrupts
  sleep (F4).
- **Author's observation — now tentatively supported:** the research finds the
  acute HR effect during exercise is an **offset, not a slope change**, matching
  the author's report that effort→HR feels unchanged on/off the drug while
  *baseline arousal* shifts. So concentration should modulate a resting/arousal
  term, not rescale the HR-effort curve. (Still "tentative" in the literature —
  validate against the author's own data.)
- **Personal calibration:** **MAP-Bayesian / MIPD** — start from population PK,
  refine to the individual as data accumulates (answers "can we calibrate?").
- **Safety (responsible, not alarmist):** stimulants + hard exercise raise
  cardiac load and impair thermoregulation (heat-risk → F8/F12). A tracking aid,
  **not medical advice**.

**Open questions:** which formulation(s)? · sensitive-data storage/privacy.

## F3 — Diet tracking + energy balance · `idea` · research in

Fold in diet/nutrition tracking and pair it with run burn to track **energy
balance (surplus/deficit)**.

1. **Photo food logging** → calories/carbs/protein via a vision model (latest
   Claude models are strong; see the `claude-api` skill before wiring the API).
   *Hard part is portion/volume, not identification* → include a quick
   user-confirm/adjust step; consider a fiducial for scale.
2. **Accurate run calorie burn.** Essentially F2 solved for energy cost. The
   **accuracy hierarchy and recommended pipeline** are in
   [research/01 §10](research/01-physiology-master-equation.md) (individual HR-VO₂
   calibration beats generic formulas; consumer METs×time is crude — a real
   differentiator).
3. **Energy-balance ledger.** Intake − expenditure (BMR via Mifflin-St Jeor +
   activity + run burn) → surplus/deficit *trends*. Body weight is both an
   **input** to burn (§Model) and the **output** F3 manages.

**Open questions:** first-class feature or companion app? · privacy of food
photos + body metrics.

## F8 — Auto-ingest weather (esp. humidity) · `idea`

Pull ambient conditions from a weather API — temperature is a direct F2 input.
**Humidity is a separate channel from temperature** and matters greatly for
thermoregulation (research/01 §8) → use wet-bulb / WBGT, not just °. Also: wind
(routing), AQI/pollen (breathing). Feeds F2 and F12.

## F12 — Heat-safety advisor · `idea` · tailored

Combine WBGT/humidity (F8) + current medication concentration (F5) + exertion
(Physiological Strain Index, research/01 §8) to flag genuinely risky heat.
Specific to this user: amphetamines impair thermoregulation (F5), so heat risk
is elevated. Informational.

## F14 — Health anomaly flags from RR data · `idea`

Use the 24/7 per-beat RR stream (F4) to flag irregular-beat patterns or an
unexplained RHR spike (illness / overtraining / arrhythmia-like).
**Informational, not diagnostic.**

---

# Training guidance

## F6 — Dynamic / adaptive training guidance · `scoped` · research in

Reject the rigid "interview → fixed 12-week plan, fall behind = tough luck"
model. Decide each session **day-of (or day-prior)** from current state, so a
missed day doesn't break the schedule. Full survey in
[research/03](research/03-adaptive-training.md).

- **Rolling horizon, not a frozen calendar:** keep a flexible long-range
  *skeleton* (goal + phase + weekly shape); only *commit* a workout the day
  before/of.
- **Auto-regulation** from *readiness* (F9). HRV-guided training is
  research-backed (modest but real); the report gives a concrete decision rule.
- **Engine — what the research says to ship:**
  - **Performance Management Chart (CTL/ATL/TSB)** from the Banister
    fitness-fatigue model — the practical, shippable version.
  - **ACWR as a soft warning only** (recent statistical critiques — don't gate
    hard on it).
  - Load via **TRIMP** (HR), **TSS/rTSS** (pace/power), and **session-RPE**.
  - **Don't hard-code 80/20** — expose intensity distribution (polarized /
    pyramidal / threshold) as a user/coach choice; for sub-elite, pyramidal and
    polarized are interchangeable and both beat threshold-heavy.
  - **Pace zones are deterministic — no ML needed.**
- **Planner architecture:** rule/template engine (workout library, sequenced by
  periodization rules, scaled to current fitness from F11); optionally
  **LLM-with-hard-guardrails** given the rich per-day context.

**Tensions / open questions:** goal races need *some* forward structure →
"goal-anchored skeleton + day-of commitment"; don't over-react to single-day
readiness noise; cold start before personal data → lean on population rules.

---

# Routes & terrain

## F7 — Elevation-aware route designer · `scoped` · research in

Generate routes within a user-defined area (typically a radius from home),
accounting for hills, with road-level preferences. Engine comparison, schemas,
and OSM tag model in [research/04](research/04-routing-elevation.md); elevation
data in [research/05](research/05-usgs-3dep-lidar.md).

- **Hill awareness, two modes:** *find flat* (minimize gain — slope-weighted
  graph) or *account for hills* (report the profile and, via F2 + GAP, give
  expected pace/effort/time; optionally adjust distance so effort matches the
  session).
- **Road preferences:** mark **favorite** / **hated** roads as per-road routing
  weights. The report gives a persistence schema (store OSM way ID **and**
  geometry — IDs change) and tap-to-way snapping endpoints.

**What the research settled:**
- **Don't solve the "perfect loop of distance D" exactly — it's NP-hard.** Use
  the geometric heuristic (waypoints on a circle, route through them, penalize
  reused edges).
- **Self-hosted GraphHopper (open-source) is the recommended core** — the only
  permissively-licensed engine with **both** native round-trip generation and a
  precise, declarative slope-aware weighting model. (BRouter = most tunable
  energy model but no native round-trip; Valhalla = simplest grade knob, MIT;
  ORS = built-in round-trip but coarse grade + GPLv3 copyleft.)
- **Elevation: USGS 3DEP** (US, public domain) — **COG DEMs on AWS** for whole
  route profiles, **EPQS** for single-point queries; 1 m where available. Use
  the DEM for *planned* grade, the barometer for *live* grade.
- **Map data:** OpenStreetMap (surface/highway/foot tags for run-quality
  weighting).

**Notes:** rank **multiple candidate loops** (flatness / favorites / variety);
ties to F6 (day's session auto-requests a matching route) and F3 (predicted
burn); later: surface preference, safety/lighting, avoid-recent-routes.

---

# Conveniences

## F15 — Auto shoe-mileage tracking · `idea`

Track mileage per shoe pair, warn at replacement mileage. Foot pods could
**auto-detect which shoes** are worn.

## F16 — Locomotor-respiratory coupling training · `idea`

Once the breathing band exists, coach a breath:step rhythm (e.g. 3:2) from the
respiration sensor + cadence. Niche but researched; sensors will be on hand.

---

# Research

All five prior-art areas have been researched and synthesized into cited reports
under [`research/`](research/) (see `research/README.md` for the index). They
back F2/F3 (physiology), F5 (PK), F6/F9/F11 (training), and F7 (routing +
elevation). The reports flag any claim that failed independent verification —
spot-check before shipping.
