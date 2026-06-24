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
