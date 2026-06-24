# Feature wishlist — `run`

Running / training app. Captured from design discussion. Status legend:
`idea` (just noted) · `scoped` (design agreed) · `building` · `done`.

Context driving the project:
- Existing apps either paywall features or compute pace poorly.
- Hardware on hand: Bluetooth optical HR monitor (Scosche Rhythm 24, see
  `scosche-rhythm24.md`), Android phone.
- Candidate sensors discussed: foot pod (BLE RSC `0x1814`), sacrum/belt IMU,
  respiration band (RIP), barometer/grade, SmO2 (NIRS), core temp, CGM.

---

## Features

<!-- New features appended below as the user describes them. -->

### F0 — Unified local-first data store + open export · `idea` · FOUNDATIONAL

Not a user-facing feature so much as the substrate everything else needs: a
clean, timestamped, multi-channel **time-series log of every sensor stream**,
stored on-device and exportable in open formats (FIT / GPX / Parquet / CSV).

- Every other feature (F2 master equation, F3 burn, F5 PK, F6 planner) is
  downstream of this — they're only as good as the logged data.
- Purest expression of the project's ethos: **own all your data, forever**, no
  cloud lock-in, no paywall.
- Design notes: normalize sources into named channels (`hr`, `rr`, `speed`,
  `cadence`, `grade`, `respRate`, `smo2`, `medConc`, …) on a common clock;
  generalizes the two-channel Scosche design (see `scosche-rhythm24.md`).
- **Lock this in regardless** — it's the foundation.

### F1 — Haptic HR-zone coaching · `idea`

Phone vibrates to tell the runner to speed up or slow down so they hold a
target heart-rate zone **without looking at the screen** (phone strapped to
upper arm).

- **Input:** live HR from the BLE monitor (Scosche `0x2A37`), compared against
  a target zone (lo/hi bpm bounds, or % of HRmax / HRR).
- **Output:** distinct vibration patterns — e.g. one buzz pattern = "speed up"
  (HR below zone), another = "slow down" (HR above zone), silence = in zone.
  Patterns must be distinguishable by feel alone through a sleeve.
- **Design notes / open questions:**
  - HR lags effort by 10–30 s; cue off a smoothed HR + rate-of-change so it
    doesn't nag during normal zone wobble. Add hysteresis / a dead-band and a
    minimum re-alert interval so it isn't buzzing constantly near a boundary.
  - Distinguish "drifting out" (gentle reminder) from "way out of zone"
    (stronger/urgent pattern)?
  - Android haptics via `expo-haptics` are limited to preset styles; rich
    custom vibration patterns need the native `Vibration` API
    (`Vibration.vibrate([pattern])`) — confirm against v56 docs.
  - Screen will be off / pocketed-equivalent on the arm — vibration must fire
    reliably with the app backgrounded/asleep (foreground service / keep-alive
    during an active run).
  - Future: same haptic channel could cue target *pace* zones once foot-pod
    fusion exists (see sensor notes), not just HR.

### F2 — Personal physiological "master equation" · `idea` · research-first

After enough data is collected, fit a personal model relating the runner's
key variables so it can be **solved for any one of them given the others**:

- **Variables:** pace/speed, ambient temperature, grade, heart rate,
  breathing (rate and/or ventilation), and a **subjective effort parameter
  (RPE / Borg)**.
- **Headline use case — feedforward pacing:** holding an HR zone on flat
  ground, then a hill begins. Because grade is known *immediately* (barometer/
  map) while HR lags 10–30 s, the model predicts the pace that will keep HR in
  zone **before** HR drifts, and the haptic channel (F1) proactively says
  "slow down" on the way into the hill instead of reacting after the fact.
- Many other use cases (predict HR for a planned pace, estimate effort cost of
  a route, detect abnormal readings / fatigue when actual diverges from
  predicted, set realistic targets in heat, etc.).

**Research must come first** — this is well-trodden exercise-physiology +
modeling territory; survey prior art before building. Domains to cover:
  - Grade Adjusted Pace / Minetti energetics of slope running.
  - ACSM metabolic / VO2 running equations; running economy.
  - Critical Power / Critical Speed; running "power" models (Stryd).
  - HR *dynamics* during exercise: first-order / state-space / Hammerstein-
    Wiener / ODE models of HR response to load (this is the key to the
    transient/hill case — a static fit won't capture lag).
  - Cardiac drift & heat: effect of temperature/dehydration on HR
    (Physiological Strain Index, Pw:HR aerobic decoupling).
  - RPE relationships: Borg vs %HRmax / ventilatory thresholds; session-RPE.
  - ML approaches to personalized HR/pace prediction and any published
    "running performance" multivariate models.

**Modeling design tensions to resolve in research (not yet decided):**
  - *Regression vs. AI:* a static regression maps instantaneous inputs→output
    and will be **wrong exactly during transitions** (the hill case) — running
    is a dynamical system (HR has lag + transients). Likely need a *dynamic*
    model (state-space / ODE / recurrent), not a static fit.
  - *"Solvable for any variable"* wants an implicit relation
    `F(pace, temp, grade, HR, breath, RPE) = 0` that can be rearranged. A
    transparent grey-box / physics-informed model is invertible and
    interpretable; a black-box neural net must be inverted numerically and is
    harder to trust. Leaning grey-box.
  - *Personalization:* physiology is individual — likely a population/base
    model + per-user calibration (hierarchical / Bayesian), refined as more of
    the runner's own data accumulates.
  - *Data requirements:* what to log, at what rate, and how much before the
    fit is trustworthy — defines the data pipeline this feature depends on.

### F3 — Diet tracking + energy balance · `idea` · partly research-first

Possible expansion (user not fully decided): fold in **diet/nutrition
tracking** and pair it with run calorie burn to track **energy balance
(surplus / deficit)** over time.

**Three parts:**

1. **Photo-based food logging.** Take a picture of a meal → estimate
   calories, carbs, protein. Likely a vision model (the latest Claude models
   are strong at this; see `claude-api` skill before wiring up the API).
   - *Known hard part:* portion/volume estimation is where photo calorie
     estimates go wrong, not food *identification*. Plan for a quick
     user-confirm/adjust step (portion size, was-it-eaten-all) rather than
     trusting a single number. Consider a fiducial/known-object for scale.

2. **Accurate run calorie burn — RESEARCH TASK (explicitly requested).**
   Estimate calories burned during a run "very accurately" from all available
   variables (pace, grade, HR, breathing, temperature, body mass, duration).
   - This is essentially a **sub-application of F2**: calorie burn = metabolic
     rate = a quantity the master equation should already produce (solve for
     metabolic cost). Worth researching together / sharing the data pipeline.
   - Approaches to survey: HR→VO2→kcal (with individual HR-VO2 calibration,
     not generic formulas), running-power→metabolic-cost, ACSM running
     equation, grade-adjusted energetics (Minetti), accelerometry-based
     estimates, and EPOC / afterburn. Note: most consumer apps' calorie
     numbers are crude (generic METs × time) — accuracy here is a real
     differentiator.

3. **Energy balance ledger.** Intake (from #1) minus expenditure
   (BMR/RMR via e.g. Mifflin-St Jeor + daily activity + run burn from #2) →
   running surplus/deficit. Surface trends, not just single-day noise.

**Open questions:**
  - Scope creep risk — is diet a first-class part of this app or a separate
    companion? Decide before building.
  - Privacy: food photos + body metrics are sensitive; where is this stored?
  - For "accurate" burn, an individual HR-VO2 calibration (or the F2 model)
    matters far more than picking a fancier off-the-shelf formula.

### F4 — Near-24/7 wear: resting HR, max HR, sleep · `idea`

Wear the HR monitor as continuously as the charge cycle allows and mine the
passive data for baseline physiology.

- **Resting HR (RHR).** Derive from the lowest sustained HR (typically during
  sleep / early morning), tracked as a *trend* — RHR is a strong fitness /
  recovery / illness signal (a spike often precedes feeling sick or means
  overtraining). Achievable from passive wear. Good input to F2/F3 recovery
  state and RMR baseline.
- **Max HR.** *Caveat:* true HRmax only appears during near-maximal effort and
  is rarely captured at rest — passive wear won't find it. Better plan:
  detect *observed* max from hard run sessions, keep a running maximum, and use
  an age-based formula (e.g. 208 − 0.7·age) only as a prior until a real max is
  seen. Be honest in the UI about which it is. Accurate HRmax/RHR then anchor
  the HR zones used by F1 and F2.
- **Overnight HRV.** We already get per-beat RR in the device's HRV sport mode
  (see `scosche-rhythm24.md`) — nighttime HRV (e.g. rMSSD) is the standard
  recovery-readiness metric. Strong, almost-free win given the data is already
  there. Confirm running HRV mode 24/7 is acceptable for battery.
- **Sleep.** Estimate sleep/wake and duration (and *rough* staging) from HR +
  HRV + the armband's motion/actigraphy. Research-backed but consumer accuracy
  is limited — promise sleep timing/duration confidently, stages only loosely.

**Open questions / constraints:**
  - *Battery & charging window:* needs a daily charge slot; plan for and
    surface the inevitable data gap. Does HRV mode drain faster?
  - *Onboard storage & sync:* 24/7 logging is a lot of data. Device records to
    a FIT file onboard ("hundreds of hours" claimed, full-storage behavior
    undocumented — open Q in the Scosche notes). Need a reliable background
    BLE sync cadence so storage doesn't fill and the phone DB stays current.
  - *Continuous capture:* live BLE drops are fine here — rely on the onboard
    FIT record for completeness, sync periodically rather than streaming 24/7.
  - Skin tolerance / rotation for all-day optical wear.

### F5 — Medication tracking + PK concentration model · `idea` · research-first

Log daily medication (user takes amphetamines daily) and model the estimated
**current blood concentration** over time via pharmacokinetics, then learn how
concentration correlates with the runner's physiology and performance.

- **Log:** dose (mg), time taken, and **formulation** — this matters a lot:
  immediate-release vs extended-release vs prodrug (lisdexamfetamine) have very
  different curves.
- **PK model:** first-order absorption + elimination; one- vs two-compartment
  TBD by formulation (research). Estimate plasma concentration C(t) from
  superimposed doses. Notes for the research pass:
  - d-amphetamine half-life is on the order of ~10–13 h but is **strongly
    urine-pH dependent** (acidic urine clears it much faster) — a real source
    of day-to-day and person-to-person variability.
  - Extended-release is dominated by absorption kinetics; **lisdexamfetamine is
    a prodrug** converted to active d-amphetamine by rate-limiting hydrolysis →
    model as prodrug→active conversion, not a simple bolus.
  - Population PK gives the curve shape; individual clearance varies (genetics/
    CYP, urine pH, etc.). **Be honest in the UI: this is a model-based estimate,
    not a blood measurement.**

- **Why it's valuable here — it's a confounder for almost everything else:**
  - **Raises resting and exercise HR / BP** → directly biases HR zones (F1) and
    the master equation (F2). Concentration should be a *covariate* in F2 so the
    model can separate "the drug raised my HR" from "I'm working harder."
  - **Suppresses appetite** → skews intake in the F3 energy-balance ledger.
  - **Disrupts sleep** → interacts with F4 sleep/recovery; timing of last dose
    vs. sleep onset is learnable from the data.
  - Once concentration is a known input, the app can *learn the runner's
    individual response* (HR offset per ng/mL, RPE shift, sleep impact, etc.).

- **Safety note (surface responsibly, not alarmist):** stimulants combined with
  intense exercise raise cardiovascular load and impair thermoregulation
  (higher core-temp risk, esp. in heat — ties to the temperature variable in
  F2). Worth a gentle caution in heat/high-concentration conditions. This is a
  personal tracking aid, **not medical advice**, and doesn't replace a doctor.

**Personal observation (user, to test against data — not assume):** the user
reports that during exertion the effort→HR relationship feels *unchanged* on
vs. off the drug; what differs is **baseline arousal/excitability at rest**
(harder to relax, and anecdotally a slightly *higher* resting HR during multi-
week abstinence). Hypothesis to validate: medication mainly shifts the resting/
arousal baseline, not the effort→HR slope. If true, F2 should let concentration
modulate a resting/arousal term rather than rescaling the whole HR-effort curve.
Treat as a hypothesis to confirm from the runner's own data, not a fixed prior.

**Open questions:**
  - Which formulation(s) does the user take? (drives the model choice)
  - Sensitive health data — storage/privacy handling (same concern as F3 diet).
  - Can we *calibrate* the personal PK from observed HR response, or only
    assume population parameters?

### F6 — Dynamic / adaptive training guidance · `idea` · research-first

Reject the rigid "interview → fixed 12-week plan, fall behind = tough luck"
model. Instead, decide each session **day-of (or day-prior)** based on current
state, so a missed day doesn't throw the whole schedule into disarray.

- **Core idea — rolling horizon, not a frozen calendar.** Keep a flexible
  long-range *skeleton* (goal + phase + rough weekly shape) but only *commit* a
  specific workout the day before / day of. Life happens; the plan absorbs it
  instead of breaking.
- **Auto-regulation is the key concept** (the thing other apps mostly lack):
  pick today's session from *readiness*, which we already have the inputs for —
  overnight HRV / RHR / sleep (F4), recent load, RPE, and medication state
  (F5). HRV-guided training is research-backed (often matches or beats fixed
  plans). This is a natural consumer of the rest of the app's data.
- **"Different plans with different priorities"** ⇒ a workout/template library +
  selection logic parameterized by goal (5k vs marathon vs general fitness vs
  return-from-layoff), phase, and the runner's current fitness.

**How automated plan generators actually work (for the research deliverable —
user said they don't know; document it):**
  - Most are **rule/template engines**: a library of workout types, sequenced by
    periodization rules, scaled to current fitness (from a recent race or a
    threshold/critical-speed test), with paces derived from threshold/CS.
  - Underlying training-science to survey:
    - *Periodization:* linear vs. block vs. **daily-undulating** (DUP);
      macro/meso/microcycles. The user wants the auto-regulated end of this.
    - *Load quantification:* TRIMP (HR), TSS/rTSS (pace/power), session-RPE
      load; **Acute:Chronic Workload Ratio** (note recent critiques) for
      ramp-rate / injury risk.
    - *Fitness–Fatigue (Banister impulse-response) & PMC* (CTL/ATL/TSB =
      fitness/fatigue/form). This is itself a *dynamical model* — same flavor as
      F2, and a strong candidate engine for "how much can I do today."
    - *Intensity distribution:* polarized 80/20 (Seiler) vs. threshold vs.
      pyramidal.
    - *Progression / safety rules:* sensible ramp limits, recovery weeks,
      taper.
  - Advanced approaches to note: optimization / RL planners, and **LLM-driven
    planning with hard guardrails** (the rules above as constraints) — fits an
    app that already has rich per-day context.

**Design tensions / open questions:**
  - Goal races still need *some* forward structure (you can't fully wing a
    marathon build) — resolve as "goal-anchored skeleton + day-of commitment,"
    not zero planning.
  - Don't over-react to single-day readiness noise — smooth, like F1's HR cue.
  - Cold start: how to guide before enough personal data exists (lean on the
    population rules, personalize as F2/F4 data accumulates).
  - Ties together the whole app: F2 (what pace/load is appropriate), F4
    (readiness), F5 (medication as a state variable), F3 (fueling for the
    session).

### F7 — Elevation-aware route designer · `idea`

Generate running routes within a user-defined area (typically a radius from
home), accounting for hills, with road-level preferences.

- **Area constraint:** routes confined to a region — most likely a radius from
  home (also support custom-drawn areas later).
- **Hill awareness, two modes:**
  - *Find flat* when a flat route is wanted — minimize total elevation gain
    (weight the road graph by grade).
  - *Account for hills* otherwise — report the elevation profile and, via the
    F2 model + grade-adjusted pace, give an **expected pace / effort / time**
    for the specific hills on that route (and optionally adjust target distance
    so effort matches the intended session).
- **Road preferences:** mark **favorite roads** (prefer) and **hated roads**
  (avoid / heavy penalty). Per-road weighting applied to the routing cost.

**How round-trip route generation actually works (note for build):**
  - This is **loop generation**, not A→B shortest path — generating a closed
    loop of a *target distance* from a start point is related to the
    (NP-hard) orienteering / arc-routing problem, so engines use heuristics.
  - Existing engines that already do round-trip + elevation-weighted routing:
    **GraphHopper** (round-trip routing + custom elevation weighting),
    **BRouter** (excellent custom profiles, elevation-aware, self-hostable),
    **Valhalla**, **OpenRouteService** (round-trip + avoid features). Prefer
    self-hostable (BRouter/GraphHopper) — cost + privacy, fits the no-paywall
    ethos.
  - **Map/road data:** OpenStreetMap (free; tags for surface, highway class,
    foot access — also lets us prefer footpaths / avoid busy roads later).
  - **Elevation data:** a DEM (SRTM/Copernicus) or terrain API; needed both to
    weight for "flat" and to build the grade profile that feeds F2.
    - **USGS 3DEP lidar (user-requested — investigate at research time):** the
      USGS 3D Elevation Program (3DEP) publishes very high-resolution
      lidar-derived DEMs (1 m where available) for the US, public domain. Worth
      digging into as a higher-quality US elevation source than SRTM for both
      route grade-weighting and the grade input to F2. Research should confirm
      current coverage, resolutions (1 m / 1‑3 arc-sec), access methods
      (The National Map downloads, point-query API, dynamic image services),
      formats (GeoTIFF / COG), and licensing. Combine with the barometer
      (F1/F2) — lidar DEM for *planned* route grade, barometer for *live*
      grade. **Deferred to end-of-conversation research.**

**Open questions / notes:**
  - Favorite/hated roads need stable identity — store by OSM way ID *and*
    geometry (way IDs change); snap user taps to the nearest way.
  - Likely want **multiple candidate loops ranked**, not one answer
    (by flatness, by how much they use favorites, by variety).
  - Ties to F6: the day's planned session ("flat easy 8k") can auto-request a
    matching route; ties to F3 for predicted burn on that route.
  - Possible later: surface preference (road vs trail), safety/lighting,
    avoid-repeating-recent-routes for variety.

---

## Proposed additions (Claude-suggested, user said "save everything")

### F8 — Auto-ingest weather (esp. humidity) · `idea`

Pull ambient conditions from a weather API automatically — temperature is a
direct input to F2, so don't make the user guess it.
- **Humidity matters more than dry temperature** for thermoregulation; use
  wet-bulb / heat index, not just °. Also useful: wind (route planning),
  AQI/pollen (breathing).
- Nearly free to add; makes F2 honest about hot, muggy days. Feeds F12.

### F9 — Daily readiness score · `idea` · lock-in candidate

Synthesize overnight HRV + RHR + sleep (F4), recent training load, and
medication state (F5) into one number that **drives F6's day-of decision**.
- This is the concrete glue that turns "adaptive training" into something that
  actually picks today's workout. All inputs already exist in the system.
- Smooth it — don't react to single-day noise (same lesson as F1).

### F10 — Running power from own IMUs · `idea`

Derive a real-time **running power** metric from the DIY foot/belt IMUs +
grade (what Stryd sells for ~$200; we're building the sensors anyway).
- Power responds to grade *instantly* — none of HR's 10–30 s lag — so it's
  arguably a **better pacing target than HR for the hill scenario** (F1/F2).
- Could drive a haptic *power*-zone mode (extends F1).

### F11 — Calibration field tests · `idea`

Periodic structured tests (critical-speed / threshold) to fit the personal
parameters of F2 and anchor F6.
- Without this the master equation is uncalibrated.
- Byproduct: a race-time predictor (Riegel) for free.

### F12 — Heat-safety advisor · `idea` · tailored

Combine wet-bulb/humidity (F8) + current medication concentration (F5) +
exertion to flag genuinely risky heat conditions.
- Specific to this user: amphetamines impair thermoregulation (F5 safety note),
  so heat risk is elevated vs. a typical runner. Informational, not alarmist.

### F13 — Transparent "why" explanations · `idea` · design principle

Every recommendation explains itself — e.g. *"ease off: grade hit 6% and you're
4 bpm over zone."*
- Directly answers the project's motivation (hating opaque, paywalled apps).
- Also a design constraint favoring the **grey-box** model in F2 — black boxes
  can't explain themselves.

### F14 — Health anomaly flags from RR data · `idea`

Use the 24/7 per-beat RR stream (F4) to flag irregular-beat patterns or an
unexplained RHR spike (illness / overtraining / arrhythmia-like patterns).
- Frame carefully as **informational, not diagnostic**. Real signal exists in
  data we'll already have.

### F15 — Auto shoe-mileage tracking · `idea`

Track mileage per shoe pair and warn at replacement mileage.
- Foot pods could **auto-detect which shoes** are worn. Trivial, genuinely
  useful.

### F16 — Locomotor-respiratory coupling training · `idea`

Once the breathing band exists, coach a breath:step rhythm (e.g. 3:2) using the
respiration sensor + cadence. Niche but researched; sensors will be on hand.

### F17 — Audio coaching · `idea`

Voice cues via earbuds as a richer complement to haptics (F1) — the phone's
already strapped to the arm. Can convey more than buzz patterns (splits, pace,
"ease off") without looking at the screen.

---

## Deferred research backlog

Per the user's instruction, all research is deferred to the **end of the
conversation** and run as one batch. Items accumulated so far:

1. **Physiological master equation (F2) + accurate run calorie burn (F3 #2).**
   Treat as ONE combined pass — calorie burn is the energy-cost output of the
   same model. Cover: grade energetics (Minetti), ACSM/VO2 running equations,
   critical power/speed, HR-response *dynamics* (state-space/ODE/Hammerstein-
   Wiener), cardiac drift & heat, RPE relationships, ML personalization,
   regression-vs-dynamic-model and invertibility/grey-box tradeoffs.
2. **Medication PK model (F5).** Amphetamine pharmacokinetics: half-life &
   urine-pH dependence, IR vs ER vs lisdexamfetamine (prodrug) kinetics, one-
   vs two-compartment, personal calibration from HR response, and the
   arousal-baseline-vs-effort-slope hypothesis the user raised.
3. **Adaptive training science (F6).** Periodization (linear/block/DUP),
   load models (TRIMP, TSS/rTSS, sRPE, ACWR + critiques), Fitness–Fatigue/PMC
   (CTL/ATL/TSB), intensity distribution (polarized 80/20), how auto plan
   generators work, HRV-guided/auto-regulated training evidence.
4. **Routing + elevation data (F7).** Round-trip/loop generation algorithms;
   self-hostable elevation-aware engines (BRouter, GraphHopper, Valhalla, ORS);
   OSM data model for road preferences.
5. **USGS 3DEP lidar elevation (F7, user-requested).** Coverage, resolutions,
   access methods/APIs, formats (GeoTIFF/COG), licensing.
