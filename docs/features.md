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
