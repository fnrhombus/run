# Physiological master-equation + run calorie burn (F2/F3)

> Prior-art research brief, generated 2026-06-24 via a fan-out research workflow
> (parallel web research → adversarial verification → synthesis). Claims that
> failed independent verification are flagged inline. Treat as a literature
> survey to ground implementation, not as gospel — spot-check before shipping.

## Physiological Master-Equation and Run Calorie Burn (F2/F3)

This briefing covers the energetics, metabolic equations, power models, HR-response dynamics, environmental/drift corrections, RPE anchoring, and calorie-estimation pipeline you need to build the F2 (per-instant metabolic rate / power) and F3 (cumulative calorie burn) modules. Everything below has been cross-checked against an adversarial verification pass; claims that were softened, mixed, or could not be byte-verified are flagged inline.

---

## 1. Energetics of graded running: the Minetti cost-of-transport model

The foundation of grade-aware running energetics is [Minetti et al. (2002), *J Appl Physiol* 93:1039-1046](https://pubmed.ncbi.nlm.nih.gov/12183501/), which measured the metabolic cost of locomotion at extreme uphill and downhill slopes and fit 5th-order polynomials to the energy cost of transport.

### Running cost polynomial (verified)

The cost of running `Cr` in J·kg⁻¹·m⁻¹ as a function of gradient `i` (a **dimensionless decimal fraction**, rise/run; +uphill, −downhill; 10% grade ⇒ i = 0.10; **not** degrees, **not** percent):

```
Cr(i) = 155.4·i⁵ − 30.4·i⁴ − 43.3·i³ + 46.3·i² + 19.5·i + 3.6
```

- Valid range tested: **i ∈ [−0.45, +0.45]**, R² = 0.999.
- The constant **3.6** is the *fitted* level cost; the paper's *measured* level cost was **3.40 ± 0.24 J·kg⁻¹·m⁻¹**, reported as independent of speed.
- This polynomial and all six coefficients were **independently verified** against multiple sources ([Aaron Schroeder's reverse-engineering writeup](https://aaron-schroeder.github.io/reverse-engineering/grade-adjusted-pace.html), [Fellrnr](https://fellrnr.com/wiki/Grade_Adjusted_Pace)). Use this as your canonical, citable grade-cost function.

> Note on a documentation hazard: an earlier fetch of one PDF mirror returned a *rescaled* variant with constant = 1.0. That variant is **not** the published J/kg/m form. Use the coefficients above.

### Walking cost polynomial (use with caveat)

```
Cw(i) = 280.5·i⁵ − 58.7·i⁴ − 76.8·i³ + 51.9·i² + 19.6·i + 2.5    (J·kg⁻¹·m⁻¹)
```

Same gradient convention and range; constant 2.5 = fitted level minimum walking cost. **Caveat:** verification rated the walking coefficients *uncertain* — the form, range, and ~2.5 level constant are consistent with the paper and with corroborated minimum-cost data, but the six walking coefficients could not be byte-verified against the original (PDF hosts returned 403, journal full-text blocked). Treat `Cw` as likely-correct but verify against the primary paper before shipping it as a hiking/power-hike cost model. Use `Cr` for running segments and `Cw` for walking/power-hiking segments; the two curves converge at steep uphill (at +0.45, running cost was only ~9% above walking).

### Cost minima and curve shape (verified)

The cost-of-transport curve is **U-shaped and bottoms out on a moderate downhill, not on the flat**:

- **Running:** measured minimum Cr = **1.73 ± 0.36 J·kg⁻¹·m⁻¹ at i = −0.20** (avg speed 3.14 m/s, ~51% of level cost). Minimizing the fitted polynomial over [−0.45, 0] gives Cr ≈ 1.78 at i ≈ −0.18. Below −0.20 cost rises again (≈3.92 ± 0.81 at −0.45).
- **Walking:** minimum Cw = **0.81 ± 0.37 J·kg⁻¹·m⁻¹ at i = −0.10**.
- Uphill cost rises ~linearly above i ≈ 0.22–0.24.

### Mechanical-efficiency asymmetry (corrected)

The asymmetry of the curve reflects muscle mechanics, **but the original findings stated the running figures with walking values mixed in**. Verification correction:

- Positive-work (concentric) efficiency ceiling for slopes steeper than +0.15: **≈0.243 for walking, ≈0.218 for running** — so the ~0.25 ceiling is the *walking* value; running is ≈0.22.
- Eccentric (negative-work) efficiency for slopes steeper than −0.15: **≈−1.215 for walking, ≈−1.062 for running** — so the often-quoted −1.20 is the *walking* value; running is ≈−1.06.

Do not attribute the +0.25 / −1.20 figures to running. The practical takeaway (optimum mountain-path gradients ≈0.20–0.30 for both gaits) stands.

---

## 2. Grade-Adjusted Pace (GAP)

### Minetti-derived GAP (recommended default)

GAP scales the metabolic cost at the actual grade to its level-ground equivalent. The simplest peer-reviewed GAP factor is the cost ratio:

```
GAP_factor(i) = Cr(i) / Cr(0) = Cr(i) / 3.6
GAP_pace  = actual_pace  × GAP_factor(i)      # pace in s/m: slower-equivalent
GAP_speed = actual_speed × Cr(0) / Cr(i)      # speed in m/s
```

Because the Minetti model is speed-independent, the multiplier depends only on grade. Computed factors (Cr(i)/3.6) match published values:

| Grade | Factor | Grade | Factor |
|------:|-------:|------:|-------:|
| −10% | 0.598× | +10% | 1.658× |
| −5%  | 0.763× | +15% | 2.060× |
| 0%   | 1.000× | +20% | 2.502× |
| +5%  | 1.301× | +30% | 3.494× |
|      |        | +45% | 5.396× |

(Cross-checked against the [Running Writings GAP calculator](https://apps.runningwritings.com/gap-calculator/), [Marathon Handbook](https://marathonhandbook.com/elevation-grade-calculator/), and [Fellrnr](https://fellrnr.com/wiki/Grade_Adjusted_Pace).)

### Strava's production GAP (verified alternative)

Strava uses a big-data quadratic, **not** the raw Minetti quintic:

```
f(i) = 15.14·i² − 2.896·i + 1
```

The cost-delta part is `15.14·i² − 2.896·i`; the `+1` normalizes to 1.0 at flat. This was **independently confirmed** ([Schroeder](https://aaron-schroeder.github.io/reverse-engineering/grade-adjusted-pace.html), [Fellrnr](https://fellrnr.com/wiki/Grade_Adjusted_Pace)). It is symmetric-ish (vs Minetti's asymmetric quintic) and gives **more conservative downhill credit** (~18% faster-than-actual adjusted pace downhill). Note that the Strava form already returns a relative multiplier, whereas Minetti needs division by 3.6.

**Recommendation:** ship Minetti as the citable default; offer Strava's quadratic as an "empirical / tuned on millions of runs" alternative.

### Implementation guardrails

- **Clamp i to [−0.45, +0.45].** The quintic extrapolates poorly and blows up beyond the tested range.
- Speed independence is a real limitation: the Minetti-only multiplier carries **no speed correction**. The Running Writings calculator blends Minetti 2002 with Black et al. 2018 to add speed/economy dependence, but the **exact blend weights are not published** (proprietary calculator code) — treat any speed-dependent GAP you build as your own model, not a citation.

---

## 3. ACSM/VO₂ metabolic equations and running economy

### Running equation (verified)

```
VO2 (mL·kg⁻¹·min⁻¹) = 0.2·S + 0.9·S·G + 3.5
```

- `S` = speed in **m·min⁻¹**; `G` = fractional grade.
- Terms: horizontal = 0.2 mL·kg⁻¹·m⁻¹ × S; vertical = 0.9 mL·kg⁻¹·m⁻¹ × S × G; resting = 3.5 (= 1 MET).
- Output is **GROSS VO₂** (includes resting).
- The 0.2 horizontal cost is twice the walking value (flight phase); 0.9 vertical is half the walking value.
- Unit conversions: mph × 26.8 = m/min; km/h × 16.667 = m/min.
- Example: 161 m/min (6 mph) at 0% ⇒ 35.7 mL·kg⁻¹·min⁻¹ (~10.2 METs).

Confirmed across [ACSM references (TTU)](https://www.depts.ttu.edu/ksm/_documents/grad/acsm_comps/6c-23-2013_HFI_Metabolic_Calculations.pdf) and corroborating sources.

### Walking equation (companion, verified)

```
VO2 = 0.1·S + 1.8·S·G + 3.5
```

Valid roughly 50–100 m/min (1.9–3.7 mph). Use below the running threshold.

### Equation-selection threshold (medium confidence)

Use the **running** equation at **≥134 m/min (5.0 mph / 8.0 km/h)**, or **≥80.4 m/min (3.0 mph)** if the gait is genuinely a run (flight phase present). Below that, use walking. The two equations diverge, so a naive single-speed switch creates a small discontinuity — **consider blending in the 80–134 m/min band**. (This threshold is from a search synthesis of ACSM lab references, not a single primary citation; treat as a sensible default rather than gospel.)

### Running economy (medium confidence)

The 0.2 mL·kg⁻¹·m⁻¹ horizontal coefficient **is** the canonical running-economy figure (~200 mL·kg⁻¹·km⁻¹), giving the rule of thumb **net run cost ≈ 1 kcal·kg⁻¹·km⁻¹**, nearly speed-independent on level ground. Real individual economy varies **±10–15%** (elites notably more economical), so this is a population prior, not per-athlete truth — see §8 on personalization.

### Known ACSM limitation (downhill)

The vertical term `0.9·S·G` goes linearly negative on downhills, but the true O₂ cost of downhill running is **U-shaped with a minimum near −10% to −15%** (per Minetti). For negative grades, prefer the Minetti cost curve over the ACSM vertical term, or clamp.

---

## 4. VO₂ → kcal conversion

### Caloric equivalent of O₂ (verified)

- Standard exercise approximation: **1 L O₂ ≈ 5.0 kcal**.
- Mixed-substrate: ~4.825 kcal/L.
- RQ-dependent endpoints (verified): **4.686 kcal/L at RQ 0.70 (pure fat)** → **5.047 kcal/L at RQ 1.00 (pure carbohydrate)**, near-linear in between.
- Usable linear fit (Lusk-derived): `kcal/L = 3.815 + 1.232·RER` (gives 4.679 at 0.70, 5.047 at 1.00). Mid value RQ ≈ 0.85 ⇒ ≈4.86 kcal/L.
- For steady running with unknown RER, assume **RER ≈ 0.90–0.95** (≈4.92–4.98 kcal/L) or simply use 5.0.

### Conversion pipeline

```
VO2 (L/min)  = VO2 (mL·kg⁻¹·min⁻¹) × mass(kg) / 1000
kcal/min     = VO2 (L/min) × caloric_equivalent   # 5.0 or RER-adjusted
```

MET shortcut: 1 MET = 3.5 mL·kg⁻¹·min⁻¹ ≈ 1 kcal·kg⁻¹·h⁻¹; `kcal/min = METs × 3.5 × mass(kg) / 200`.

### Weir equation (gold standard when VCO₂ available, verified)

```
EE (kcal/min) = 3.941·VO2(L/min) + 1.106·VCO2(L/min)
```

Confirmed (Weir 1949; protein term omitted; some sources round the VCO₂ coefficient to 1.11). Without gas exchange you cannot use Weir — fall back to `VO2 × caloric equivalent`.

> **Open question (precision):** the full per-0.01-RQ Lusk / Péronnet-Massicotte table was not retrieved verbatim — only endpoints and the linear fit are confirmed. Consult the original tables if you need sub-1% substrate-split precision.

---

## 5. Critical Speed / Critical Power (2-parameter model)

### Model form (verified)

Hyperbolic speed-time / linear distance-time:

```
t_lim = D' / (s − CS)            # s = speed (m/s), t_lim = time to exhaustion (s)
D = CS·t + D'                    # linear fit: slope = CS, intercept = D'
P = W'/t + CP                    # power analog
```

- **CS** = critical speed (asymptote, m/s); **D'** = curvature constant (meters, the finite distance reserve usable above CS).
- Fit from **2–3 recent maximal time trials of ~2–20 min** over different distances (e.g. 1200m/3k/5k). Trials >8k violate model validity.
- CP/CS is the heavy↔severe intensity-domain boundary; W'/D' is a fixed reserve depleted above it. Confirmed against [Running Writings](https://runningwritings.com/2024/01/critical-speed-guide-for-runners.html) and [a 2/3-trial comparison study](https://pubmed.ncbi.nlm.nih.gov/30427230/).

### Typical values & race-pace mapping (high confidence)

- CS for trained runners ≈ pace sustainable for ~30 min, between 5k and 10k race pace.
- Worked example (collegiate): CS = 5.005 m/s (3:20/km, 5:22/mile), D' = 260.1 m. D' ranges roughly 100–400 m between individuals.
- CS is **faster** than lactate threshold / MLSS / Daniels "T", **slower** than 5k pace / vVO₂max. Quick approx: CS ≈ 95–103% of 5k pace; a ±3% band is used for prescription.

### CP from Stryd power (medium confidence)

Stryd defines CP as the highest sustainable power (severe-domain boundary), conceptually identical to CS theory. A [2023 study](https://www.medrxiv.org/content/10.1101/2023.07.04.23292118) found CS and CP closely related since on flat terrain power ≈ ECOR·mass·v. **Open question:** the medRxiv PDF returned 403 in research, so the reported mean CS/CP/D'/W' values and agreement statistics (bias, limits of agreement) were not extracted — pull these from the full text before relying on them.

---

## 6. Running power models (GOVSS, Stryd, di Prampero)

### di Prampero energy-cost framework (medium confidence)

Underlies GOVSS and metabolic-power models:

- Flat constant-speed cost C ≈ **3.6–3.86 J·kg⁻¹·m⁻¹** (often cited 3.8).
- Air resistance: added cost ≈ `k·v²`, **k ≈ 0.01 J·s²·m⁻³·kg⁻¹** (a related ~0.0025 constant appears in mechanical-work formulations).
- Acceleration handled via **equivalent slope** ES = aₓ/g (forward accel over gravity), with equivalent mass EM = √(aₓ²/g² + 1).
- Metabolic power `P (W/kg) = C(ES)·EM·v`, where C(ES) is the Minetti-type slope cost evaluated at ES. See [di Prampero et al. (sprint running / metabolic power)](https://pubmed.ncbi.nlm.nih.gov/25549786/).

> **Open question:** di Prampero's air-resistance constant varies by source (~0.01 vs ~0.0025), and whether it is per-body-mass or absolute (plus frontal-area/drag assumptions) is unsettled. Pin this down before implementing a wind/air term.

### GOVSS structure (high confidence)

Sums slope cost (Minetti quintic) + air-resistance cost + kinetic/acceleration cost, converts to power via a **speed-dependent efficiency** (roughly 0.5 at low speed → 0.7 at 8.33 m/s / 30 km/h), then for aggregate scoring raises power to the **4th power over a 120 s rolling window** (lactate ∝ speed^~3.5 → rounded to 4; analogous to Coggan's Normalized Power). Threshold reference ≈ 10k / 1-hour effort; 100 points = 1 h at threshold. See [Ron George's GOVSS review](http://www.georgeron.com/2017/11/the-govss-running-power-algorithm-and.html).

> **Open question:** the exact efficiency-vs-speed regression endpoints (slope/intercept) are summarized, not given as a fitted equation.

### Stryd model (high confidence on structure; verify ECOR units)

External Energy Summation: estimates ground reaction forces from foot-pod IMU acceleration × mass, integrates COM velocity changes in 3 axes, sums vertical (potential) + horizontal/lateral (kinetic) power. "Form Power" ≈ step rate × mass × g × vertical oscillation. Captures **external energy only** (no limb-swing internal cost, no wind), assumes bilateral symmetry, apparent efficiency ~25%. See [Ron George's Stryd review](http://www.georgeron.com/2017/12/stryd-running-power-model.html).

Power-to-pace proxy:

```
Power (W) = ECOR × mass(kg) × velocity
```

**Units caveat (verification: mixed).** The physics and approximate magnitude are right, but the kcal/kJ labeling in circulation is internally inconsistent. ECOR is the near-constant energy cost of transport (~1 kcal·kg⁻¹·km⁻¹, the Margaria value), and Stryd-community sources cite ~1.04 for trained runners — **but Stryd's ECOR is normally expressed in kJ·kg⁻¹·km⁻¹ (~0.98–1.10 kJ/kg/km)**. Note that 1 kcal ≈ 4.18 kJ, so "1.04 kcal/kg/km ≈ 4.2 kJ/kg/km" conflates the two. **Decide on one unit system and convert carefully.** ([Stryd ECOR PDF](https://hetgeheimvanhardlopen.nl/wp-content/uploads/2017/02/18.-Run-efficient-lower-your-ECOR.pdf), [Stryd blog](https://blog.stryd.com/2019/12/06/how-to-analyze-running-power-data/).)

> **Open question:** Stryd's grade-adjustment and efficiency conversion are proprietary/undocumented; whether reported watts are mechanical-only or include internal-cost scaling is unclear. The ECOR ~1.04 figure is community-derived, not an official Stryd coefficient.

### Power vs running economy correlation (high confidence — modest)

[Aubry et al. (2019, PMC6317050)](https://pmc.ncbi.nlm.nih.gov/articles/PMC6317050/): in well-trained runners, power vs running economy **r = 0.6 (90% CI 0.2–0.8)**, form power vs RE r = 0.5 — i.e. **r² ≈ 0.31, only ~31% of RE variance explained**. RE ≈ 201.6 ± 12.8 mL·kg⁻¹·km⁻¹, unit cost 1.0 ± 0.1 kcal·kg⁻¹·km⁻¹, mechanical power ≈ 4.4 ± 0.5 W·kg⁻¹. **Implication:** power is a usable population-level metabolic proxy but **must be calibrated per-athlete** for accurate individual metabolic estimates.

---

## 7. HR-response dynamics (transients) for pace-from-HR control

This section underpins any feature that predicts required pace *before* HR responds, or that infers metabolic load from HR transients.

### First-order + dead-time plant (verified)

Standard minimal model: `HR(s)/speed(s) = k·e^(−Td·s) / (τs + 1)`.

Identified treadmill values (n=22, moderate-to-vigorous, [PMC11550391](https://pmc.ncbi.nlm.nih.gov/articles/PMC11550391/)):

- gain **k = 25.0 bpm per (m/s)**, time constant **τ = 47.7 s**, dead time **Td = 13.1 s**.
- Pure first-order (no dead time): k₁ = 28.57 bpm/(m/s), lumped τ₁ = 70.56 s.
- ODE form: `τ·dHR/dt = −ΔHR + k·u(t−Td)`.
- Cycle ergometer: k = 0.40 bpm/W, τ = 45.9 s, Td = 13.8 s.

### Two-time-constant / parallel structure fits better (verified)

A fast + slow component beats a single τ. **Parallel P1∥P1D was the best fit (~56.7% treadmill, 54.3% cycle)**: slow branch k = 7.0, τ = 141.5 s; fast branch k = 20.2, τ = 34.3 s, Td = 17.9 s. Second-order alternative: fast τ ≈ 18.6 s, slow τ ≈ 38.0 s. **Practical model:** fast component τ ≈ 18–35 s + slow drift component τ ≈ 140–180 s, input dead time ~13–18 s.

### Off-kinetics / recovery (verified)

Mono-exponential decay `HR(t) = HRend·e^(−t/τ) + HRrest`. HRRτ is a fitness/autonomic marker, much larger and more variable than on-kinetics: ~65.5 ± 12.1 s (single-exp, healthy); ~148 ± 82 s (healthy, peak VO₂ >22); ~376 ± 55 s (heart failure). Lower τ = fitter. Double-exponential sometimes fits submaximal recovery better. ([IJPP](https://ijpp.com/exponential-modelling-of-heart-rate-recovery-after-a-maximal-exercise/), [PMC4395265](https://pmc.ncbi.nlm.nih.gov/articles/PMC4395265/).)

### Grey-box (Cheng-Su) physics-informed model (high confidence)

A 2-state ODE model (HR + intensity) with **one identifiable fitness parameter λ** — the canonical implementable grey-box. Key constants: attractor gain α = 0.08 s⁻¹; HR_min(λ) = 35λ bpm (males), 35λ+5 (females); lactate coupling α₃ = 4 bpm/mM, lactate τ = 420 s, L_basal = 1 mM, L_max = 12 mM; vmax(λ) = 20λ km/h. ([PMC4395265](https://pmc.ncbi.nlm.nih.gov/articles/PMC4395265/).) Good fit for individualization with a single subject parameter.

### Hammerstein structure & inverse for feedforward pace (verified)

Static input nonlinearity → linear dynamics. **Its static nonlinearity can be inverted to linearize the plant**, enabling feedforward pace computation. Recipe: drive treadmill with a PRBS to identify the linear block via correlation analysis, fit the static nonlinearity, then place its inverse upstream to cancel it, leaving an ~LTI plant for H∞ / MPC control under speed/accel constraints. This inverse-static-nonlinearity is exactly the mechanism for predicting required pace before HR responds. ([PubMed 18002622](https://pubmed.ncbi.nlm.nih.gov/18002622/), [17946236](https://pubmed.ncbi.nlm.nih.gov/17946236/).)

### Achievable control performance (verified)

A first-order plant is invertible enough that simple LTI compensators hit **~2 bpm RMS HR tracking**: second-order compensator RMSE 1.98 ± 0.49 bpm, first-order 2.13 ± 0.35 bpm (n=20, 35-min constant-HR tracking). The **inverse static gain 1/k ≈ 0.035–0.0405 m/s per bpm** directly maps a desired HR change to a pace change. ([PMC10593204](https://pmc.ncbi.nlm.nih.gov/articles/PMC10593204/), [Frontiers Physiol 2018](https://www.frontiersin.org/journals/physiology/articles/10.3389/fphys.2018.00778/full).)

### Black-box ML (verified, but short-horizon)

Feedforward ANNs give ~3 bpm one-second-ahead HR prediction (MAE 3.31 with accel+HR inputs; 3.02 with cadence+HR; 4.38 at 30 s horizon) but **predict where HR is going, not the inverse (required pace)** and need current HR as input. For wearables, the review favors pre-trained ANNs, individually pre-adapted parameters, or parameter-reduced grey-box models — per-session optimization is too costly on-device.

### Central design fact

**Input-to-HR pure delay Td ≈ 13–18 s, plus a fast τ of ~18–48 s before HR settles.** Reactive feedback alone cannot beat this lag: a feedforward inverse model must lead the HR target by **at least Td + the dominant τ**. This motivates grey-box inverse / MPC over pure feedback for any pace-targeting-from-HR feature.

> **Open questions:** Wiener output-nonlinearity parameters for HR are scarce (most models are Hammerstein-only); published τ/Td are from steady-speed treadmill/cycle protocols (interval/variable-pace shifts unknown); no closed-form mapping from VO₂max to plant τ/k for cold-start individualization; analytic invertibility of the Cheng-Su model for real-time feedforward is unaddressed; no on-device latency/compute budgets reported.

---

## 8. Cardiac drift, heat, humidity, and decoupling

### Cardiovascular drift (verified)

Progressive HR rise + commensurate stroke-volume fall at fixed workload, with cardiac output ≈ constant. Onset ~10–20 min (5–10 min in heat ≥32°C); magnitude **+10–20 bpm (or ~10–15% of HR) over 30–60 min**. Two mechanisms: (1) Rowell — cutaneous blood flow displaces central volume, SV drops, HR compensates; (2) Coyle/González-Alonso — after ~15 min the HR rise *itself* (reduced filling time) is the primary SV-fall driver. **Model as a slow upward HR ramp added to steady-state baseline**, ~10–20 bpm/hr temperate, more in heat. ([Wikipedia](https://en.wikipedia.org/wiki/Cardiovascular_drift), [Fast Talk Labs / Coyle](https://www.fasttalklabs.com/fast-talk/cardiovascular-drift-with-dr-ed-coyle/).)

### Air temperature vs humidity — two separate channels (verified)

[Jenkins et al. 2023 (PMC10103870)](https://pmc.ncbi.nlm.nih.gov/articles/PMC10103870/), fixed-intensity cycling at 18/27/36°C matched for absolute humidity:

- **Air temp raises HR ~1 bpm per °C** (P<0.001); core temp barely moved.
- **Humidity does NOT meaningfully raise HR directly** (P=0.140) but sharply raises core temperature over time.

**Modeling implication — split two channels:**
1. **Instantaneous thermal HR offset ≈ 1 bpm/°C dry-bulb** (fast, direct).
2. **Cumulative humidity/wet-bulb effect** acting *slowly* via core-temp-driven drift.

A prolonged-running study (31°C, RH stepped 23→71%) found steady-state HR significantly higher only at **≥60% RH**, with core-temp rise rate climbing from +1.9°C (23% RH) to +2.4–2.5°C (52–71% RH) and SV declining as RH rose. ([PMC5079215](https://pmc.ncbi.nlm.nih.gov/articles/PMC5079215/).) **No clean per-%RH bpm coefficient exists** — see open questions.

### Dehydration amplifier (verified)

[Adams et al. 2014 systematic review (J Strength Cond Res)](https://www.researchgate.net/publication/277306972): HR rises **~3.3 bpm per 1% body-mass loss** during exercise in the heat (independent sources bracket this at 3–5 bpm/1%). Full fluid replacement roughly halves the drift (no-fluid 2.9% loss → HR +10%, SV −15%; matched fluid → HR +5%, SV maintained). **Implementation:** `HR_drift_bpm ≈ 3.3 × (%body_mass_lost) + baseline euhydrated drift`; estimate fluid loss from sweat-rate × duration − intake.

### Drift ↔ VO₂max erosion (high confidence, cycling-derived)

[Wingo et al.](https://pubmed.ncbi.nlm.nih.gov/15692320/): cycling at 35°C, HR rose ~12% from 15→45 min, SV fell 16%, **VO₂max fell 19%**; relative intensity climbed 63%→78% VO₂max at identical external work. Clamping HR limited the VO₂max fall to 7.5% vs 15%. A drifting HR at fixed pace signals rising metabolic strain — useful for fatigue estimation and pace-down recommendations. **Caveat:** derived from cycling in heat; the same proportionality for running in cooler conditions is unconfirmed.

### Physiological Strain Index (verified)

[Moran et al. 1998](https://journals.physiology.org/doi/full/10.1152/ajpregu.1998.275.1.R129) — a 0–10 real-time heat-strain score:

```
PSI = 5·(Tre_t − Tre_0)/(39.5 − Tre_0) + 5·(HR_t − HR_0)/(180 − HR_0)
```

Assumes max rectal temp 39.5°C (span 36.5–39.5) and max HR 180 (span 60–180); each term 0–5. **Caveat:** PSI requires *true core (rectal) temperature*. A wearable must substitute estimated/skin-derived core temp, and the accuracy of estimated-core PSI is uncharacterized — flag this as a known approximation.

### WBGT environmental scalar (verified)

```
Outdoor (with sun):  WBGT = 0.7·Tnwb + 0.2·Tg + 0.1·Tdb
Indoor (no sun):     WBGT = 0.7·Tnwb + 0.3·Tg
```

Tnwb = natural wet-bulb (dominant, humidity/evaporative limit), Tg = black-globe (radiant/solar), Tdb = dry-bulb. A single heat-load scalar that embeds humidity, radiation, and air temp — combine with the ~1 bpm/°C dry-bulb rule and humidity-driven core-temp drift for a composite environmental HR offset. ([Wikipedia](https://en.wikipedia.org/wiki/Wet-bulb_globe_temperature), [NC State ECONet](https://econet.climate.ncsu.edu/wp-content/uploads/2021/05/WBGT.pdf).)

### Aerobic decoupling (Pa:HR / Pw:HR) (verified)

% change in efficiency ratio between the two halves of a steady aerobic effort:

```
EF (per half)   = NGP (or NP) / avg_HR
Decoupling %    = (EF_first − EF_second) / EF_first × 100
```

Worked example: H1 180W/135bpm = 1.333; H2 178W/139bpm = 1.281; decoupling = 3.8%. Thresholds (Friel/TrainingPeaks): **<5% = strong/coupled** (cleared to leave base), 5–10% = moderate limitation, >10% = above aerobic threshold / insufficient durability. ([TrainingPeaks](https://help.trainingpeaks.com/hc/en-us/articles/204071724-Aerobic-Decoupling-Pw-Hr-and-Pa-HR-and-Efficiency-Factor-EF).) **Requires steady at/below-threshold effort and grade-adjusted pace (NGP) for free-run terrain** — for variable-pace app data the steady-state detection and grade-adjustment method materially affect the number.

> **Open questions:** no clean per-%RH or per-°C-wet-bulb HR coefficient exists (would need empirical fit); estimated-core PSI accuracy uncharacterized; generalizable euhydrated baseline drift (bpm/hr) not well quantified; decoupling thresholds derived under controlled steady efforts; the HR-drift↔VO₂max relationship is cycling/heat-derived.

---

## 9. RPE scales and physiological anchoring

### Borg 6–20 (verified)

Ordinal scale with verbal anchors (6 = no exertion … 20 = maximal), designed so **HR ≈ RPE × 10** in healthy young adults. Implement as integer input 6–20. ([Lumen](https://courses.lumenlearning.com/suny-fitness/chapter/borg-rating-of-perceived-exertion-rpe-scale/), [RPE Training](https://rpetraining.com/blog/borg-scale-rpe).)

### Empirical RPE→HR regression (medium confidence)

[Chen et al. 2013](https://pubmed.ncbi.nlm.nih.gov/24665812/) (dynamic exercise): `HR = 8.88·RPE + 38.2`. This **conflicts with the RPE×10 heuristic** (which overestimates at low RPE, underestimates at high RPE for that young-male sample). **Pick one explicitly and document it as a population prior, not individual truth.**

### ACSM intensity classification (high confidence)

| Intensity | %HRR/%VO₂R | %HRmax | METs | RPE |
|-----------|-----------|--------|------|-----|
| Light | 30–39% | 57–63% | — | 9–11 |
| Moderate | 40–<60% | 64–76% | 3–<6 | 12–13 |
| Vigorous | ≥60% | ≥77% | ≥6 | ≥14 |

**Use %HRR (Karvonen), not %HRmax, for the RPE-anchored zones** — they are not equal. ([ACSM table](https://www.researchgate.net/figure/Classification-of-physical-activity-intensity-recommended-by-ACSMs-guidelines_tbl3_353000465), [ACSM/ESSA 2024 consensus](https://www.jsams.org/article/S1440-2440(24)00559-0/fulltext).)

### Ventilatory-threshold anchoring (mixed confidence)

- VT1 (aerobic threshold): RPE ~10–12 (clinical mean ~10.8 ± 1.8).
- VT2 (respiratory compensation / ~lactate threshold): RPE ~13–16 (mean ~13.6 ± 1.8).
- In runners ([Cerezuela-Espejo et al., PMC6167480](https://pmc.ncbi.nlm.nih.gov/articles/PMC6167480/)): VT1 RPE 10–12, MLSS 12–13, VT2 15–16, MAS 18–20.

Runner physiological intensities (high confidence): VT1 = 77–81% HRmax / 68–74% HRR / lactate ~2.2–2.3 mM; MLSS = 85–87% HRmax / 79–83% HRR / ~3.3 mM; VT2 = 91–93% HRmax / ~5.3 mM. LT, LT+1.0, LT+3.0 mM predict VT1/MLSS/VT2 (ICC >0.82).

**Key fitness-invariance finding** ([PMC10524184](https://pmc.ncbi.nlm.nih.gov/articles/PMC10524184/), N=863): RPE at VT is stable (**12.5 ± 0.93**) across fitness, but **VT%VO₂R varies strongly (25.5–86.0%, U-shaped vs VO₂peak)**. **Implication: a fixed %VO₂max→RPE mapping is unreliable per-user; RPE is the more fitness-invariant threshold anchor.**

### Foster CR10 & session-RPE (high confidence)

Modified CR-10 (0 = rest … 10 = maximal). Collect a single whole-session rating ~30 min post-session.

```
Session load (AU) = sRPE(0–10) × duration(min)
Monotony          = mean daily load / SD of daily load (week, rest days = 0)
Strain            = weekly total load × Monotony
```

Session-RPE load is a **validated surrogate for HR-based TRIMP** (Edwards r = 0.52–0.97; Banister r = 0.49–0.99; Lucia r = 0.34–0.85), weakening for very high-intensity/intermittent work. Usable when HR data is missing. ([PMC5673663](https://pmc.ncbi.nlm.nih.gov/articles/PMC5673663/).)

> **Open questions:** no published closed-form RPE(6–20)→%VO₂max regression (fitness-dependent, only modestly correlated); the HR=RPE×10 vs HR=8.88·RPE+38.2 conflict must be resolved by choice; VT2/LT/MLSS/RCP are used interchangeably in popular sources but are distinct — decide which threshold the app targets; **Borg 6–20 → CR-10 conversion has no universally accepted formula** (a common practical mapping is CR10 ≈ (RPE − 6)/1.4 but this was *not* confirmed in primary sources — if the app collects Borg 6–20 but computes session-RPE load, this conversion is a known weak link).

---

## 10. Putting it together: the calorie-burn pipeline (F2/F3)

### Net cost shortcut (verified)

[Margaria et al. 1963](https://journals.physiology.org/doi/abs/10.1152/jappl.1963.18.2.367): net (above-rest) horizontal running cost ≈ **1 kcal·kg⁻¹·km⁻¹**, largely speed-independent (8–22 km/h), depending only on incline. Shortcut: `net kcal ≈ mass(kg) × distance(km)`. This is **net** (excludes resting). Consumer "kcal/mile" rules that scale only with weight/distance are crude — they ignore grade, ±20–30% individual economy variation, and the resting baseline.

### MET-based estimation (verified)

[Compendium of Physical Activities](https://pacompendium.com/running/) level-ground running METs: 5.0 mph = 8.5; 6.0 mph = 9.3; 7.0 mph = 11.0; 8.0 mph = 12.0; 9.0 mph = 13.0; 10.0 mph = 14.8; 12.0 mph = 18.5. `kcal/min = METs × 3.5 × mass(kg)/200`. Crude: lumps all individuals at a speed, ignores grade and economy.

### HR-based regression (Keytel, medium confidence)

[Keytel et al. 2005](https://www.semanticscholar.org/paper/Prediction-of-energy-expenditure-from-heart-rate-Keytel-Goedecke/2f647f62e650bf7df32546e541af3cf155297749) — verified verbatim:

```
Men   EE(kJ/min) = −55.0969 + 0.6309·HR + 0.1988·weight(kg) + 0.2017·age
Women EE(kJ/min) = −20.4022 + 0.4472·HR − 0.1263·weight       + 0.074·age
kcal/min = EE(kJ/min) / 4.184
```

Adding VO₂max raises explained variance from **R² = 73.4% → 83.3%**. **Caveat:** the VO₂max-augmented variant (e.g. Hexoskin's implementation with a −36.3781 male adjustment) differs slightly between published variants — verify against the primary *J Sports Sci* PDF before shipping. The plain-coefficient forms above are confirmed.

### Accuracy hierarchy (verified)

Most → least accurate for running EE:

1. **Indirect calorimetry** (gold standard) — not available in-app.
2. **Individually calibrated HR–VO₂** with speed/GPS and on/off (ramp-up vs recovery) modeling — individual calibration reduced error to <5% underestimation, ICC 0.79–0.93, vs group calibration ICC 0.64–0.79 ([PLOS One / PMC6837421](https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0224948)).
3. **Generic Keytel-with-VO₂max.**
4. **MET / ACSM speed-grade equations.**
5. **Raw accelerometer counts.**

### HR-based EE degrades at low intensity (verified)

[PulseOn wrist-PPG study (PMC5548984)](https://pmc.ncbi.nlm.nih.gov/articles/PMC5548984/): EE MAPE **6.7% at medium-heavy intensity** (between aerobic and anaerobic thresholds, r=0.97) but **16.5% below aerobic threshold** (r=0.77), because HR is a poor VO₂ proxy at low workloads (emotion, thermoregulation, hydration, posture). The model used a neural-net VO₂ from HR + GPS speed + demographics + maxHR, explicitly modeling on/off HR–VO₂ differences and RER 0.70–1.00. **Implication: weight HR-based EE less at low intensity; lean on speed/grade/MET there.**

### EPOC (medium confidence)

EPOC ("afterburn") is **~6–15% of net session O₂ cost** (~7% common figure), larger for HIIT. Prolonged EPOC needs ≥50 min at ≥70% VO₂max or supramaximal efforts. Example: HIIT EPOC 66.2 ± 14.4 kcal vs MICT 53.9 ± 12.6 kcal. **Add an intensity-scaled EPOC term (~6–15% of net cost, higher for intervals); ignoring it is a minor error for steady runs.** ([LaForgia et al.](https://pubmed.ncbi.nlm.nih.gov/17101527/), [Nature Sci Rep](https://www.nature.com/articles/s41598-024-59893-9), [ACE](https://www.acefitness.org/resources/pros/expert-articles/5008/7-things-to-know-about-excess-post-exercise-oxygen-consumption-epoc/).)

### Consumer-tracker reality check (verified)

Commercial wrist trackers showed **MAPE >10% for most VO₂max/EE evaluations** ([PMC6747132](https://pmc.ncbi.nlm.nih.gov/articles/PMC6747132/)). Root causes: group (not individual) HR–VO₂ coefficients, PPG/motion error, no grade/economy adjustment, GPS distance error (~−5%), inconsistent gross-vs-net usage, age-predicted maxHR error. **The differentiator for "run" is individual calibration + grade-aware cost + explicit gross/net handling.**

### Recommended F2/F3 architecture

**F2 (instantaneous metabolic rate, W/kg or kcal/min):**
1. Primary path (running, valid speed): `Cr(i)` from Minetti (clamp i to ±0.45) × speed → W/kg; or ACSM running VO₂ for ≥134 m/min, walking equation below.
2. Convert: VO₂ → kcal via 5.0 kcal/L (or RER-adjusted 0.90–0.95).
3. HR fusion: blend in an individually calibrated HR→VO₂ estimate, **down-weighted below aerobic threshold**; use HR transients (§7) only for control/strain, not as the primary low-intensity EE source.
4. Environmental/drift correction (§8): track cumulative drift but **do not double-count** it into metabolic EE — drift is largely a *cardiovascular* compensation at constant metabolic cost, so use it for strain/pacing, not as extra calories.

**F3 (cumulative calories):**
1. Integrate F2 over time.
2. **Decide gross vs net explicitly.** ACSM VO₂ and the 3.5 resting term are *gross*; Minetti `Cr` is essentially net mechanical/metabolic cost. To report *net active* calories, subtract resting (3.5 mL·kg⁻¹·min⁻¹ ≈ 1 MET); to avoid double-counting resting metabolism, specify in the master equation whether `Cr` is treated as net or gross.
3. Add an intensity-scaled EPOC term for the session total.

> **Critical master-equation caveat:** Minetti `Cr` is essentially a *net* cost while ACSM VO₂ is *gross*. Mixing them without a consistent gross/net convention is the single most likely source of systematic error. Pick one convention for the whole F2/F3 chain and document it.

> **Open questions:** exact Keytel SEE and whether the Hexoskin male adjustment matches the primary paper; accelerometry-only vs HR-based running EE head-to-head MAPE; a defensible EPOC coefficient for *recreational steady-state* running (the 6–15% range is broad); and the best in-app self-administered submaximal protocol (number of stages, speeds, resulting slope accuracy) for building the per-user HR–VO₂ calibration.

---

## Sources

- Minetti et al. 2002, J Appl Physiol (PubMed): https://pubmed.ncbi.nlm.nih.gov/12183501/
- Minetti et al. 2002 (full text): https://journals.physiology.org/doi/full/10.1152/japplphysiol.01177.2001
- Minetti 2002 PDF mirror (runscribe): http://runscribe.com/wp-content/uploads/power/Minetti2002.pdf
- Reverse-engineering Strava's GAP (Aaron Schroeder): https://aaron-schroeder.github.io/reverse-engineering/grade-adjusted-pace.html
- Fellrnr — Grade Adjusted Pace: https://fellrnr.com/wiki/Grade_Adjusted_Pace
- Running Writings GAP calculator: https://apps.runningwritings.com/gap-calculator/
- Marathon Handbook elevation grade calculator: https://marathonhandbook.com/elevation-grade-calculator/
- Energy cost of walking/running at extreme slopes (scispace): https://scispace.com/papers/energy-cost-of-walking-and-running-at-extreme-uphill-and-471htq48v2
- Metabolic energy expenditure level/uphill/downhill running (bioRxiv 2025): https://www.biorxiv.org/content/10.1101/2025.06.05.658094.full.pdf
- ACSM Metabolic Equations (TTU): https://www.depts.ttu.edu/ksm/_documents/grad/acsm_comps/6c-23-2013_HFI_Metabolic_Calculations.pdf
- ACSM Walking Metabolic Equation accuracy (WKU): https://digitalcommons.wku.edu/cgi/viewcontent.cgi?article=3509&context=ijesab
- ACSM running equation validity threshold (Quizlet synthesis): https://quizlet.com/124721545/lab-5-ascm-calculations-flash-cards/
- IDEA — How to Calculate Calories Expended: https://www.ideafit.com/personal-training/how-to-calculate-calories-expended/
- WikiLectures — Caloric equivalent: https://www.wikilectures.eu/w/Caloric_equivalent
- Respiratory quotient — Wikipedia: https://en.wikipedia.org/wiki/Respiratory_quotient
- Weir formula — Wikipedia: https://en.wikipedia.org/wiki/Weir_formula
- Indirect Calorimetry overview (ScienceDirect): https://www.sciencedirect.com/topics/medicine-and-dentistry/indirect-calorimetry
- AND Evidence Analysis Library — RQ application (Péronnet-Massicotte/Lusk): https://www.andeal.org/worksheet.cfm?worksheet_id=250229
- The science of critical speed (Running Writings): https://runningwritings.com/2024/01/critical-speed-guide-for-runners.html
- Critical Power / Critical Velocity (Scientist's Notebook): https://www.the-scientists-notebook.com/critical-power/
- CS and D' from 2 or 3 maximal tests (PubMed): https://pubmed.ncbi.nlm.nih.gov/30427230/
- CS vs CP in runners using Stryd power (medRxiv 2023): https://www.medrxiv.org/content/10.1101/2023.07.04.23292118
- Critical Power Definition — Stryd Help Center: https://help.stryd.com/en/articles/6879345-critical-power-definition
- di Prampero et al. — energy cost of sprint running / metabolic power (PubMed): https://pubmed.ncbi.nlm.nih.gov/25549786/
- Ron George — GOVSS running power model review: http://www.georgeron.com/2017/11/the-govss-running-power-algorithm-and.html
- Ron George — Stryd running power model review: http://www.georgeron.com/2017/12/stryd-running-power-model.html
- RunDida Running Power Calculator: https://rundida.com/tools/running-power/
- Stryd ECOR PDF (Run efficient: lower your ECOR): https://hetgeheimvanhardlopen.nl/wp-content/uploads/2017/02/18.-Run-efficient-lower-your-ECOR.pdf
- Stryd blog — How to analyze running power data: https://blog.stryd.com/2019/12/06/how-to-analyze-running-power-data/
- Aubry et al. 2019 — Running power vs running economy (PMC6317050): https://pmc.ncbi.nlm.nih.gov/articles/PMC6317050/
- Identification of HR dynamics: zeros and dead time (PMC11550391): https://pmc.ncbi.nlm.nih.gov/articles/PMC11550391/
- Two-phase response model feedback HR control (PMC10593204): https://pmc.ncbi.nlm.nih.gov/articles/PMC10593204/
- Modelling Heart Rate Kinetics — Cheng-Su grey-box (PMC4395265): https://pmc.ncbi.nlm.nih.gov/articles/PMC4395265/
- Exponential modelling of HR recovery (IJPP): https://ijpp.com/exponential-modelling-of-heart-rate-recovery-after-a-maximal-exercise/
- Oxygen kinetics and HR response during early recovery (PMC3270536): https://www.ncbi.nlm.nih.gov/pmc/articles/PMC3270536/
- Nonparametric Hammerstein MPC for HR regulation (PubMed 18002622): https://pubmed.ncbi.nlm.nih.gov/18002622/
- Modelling and control for HR regulation during treadmill exercise (PubMed 17946236): https://pubmed.ncbi.nlm.nih.gov/17946236/
- Measurement, Prediction, Control of HR Responses (Frontiers Physiol 2018): https://www.frontiersin.org/journals/physiology/articles/10.3389/fphys.2018.00778/full
- Cardiovascular drift — Wikipedia: https://en.wikipedia.org/wiki/Cardiovascular_drift
- New perspective on cardiovascular drift (ScienceDirect): https://www.sciencedirect.com/science/article/pii/S0024320521010961
- Cardiovascular Drift with Dr. Ed Coyle (Fast Talk Labs): https://www.fasttalklabs.com/fast-talk/cardiovascular-drift-with-dr-ed-coyle/
- Interaction of hyperthermia and HR on stroke volume (ResearchGate): https://www.researchgate.net/publication/44902038_Interaction_of_hyperthermia_and_heart_rate_on_stroke_volume_during_prolonged_exercise
- Delineating air temperature vs humidity for endurance exercise (Jenkins 2023, PMC10103870): https://pmc.ncbi.nlm.nih.gov/articles/PMC10103870/
- Systematic increase in relative humidity during prolonged running (PMC5079215): https://pmc.ncbi.nlm.nih.gov/articles/PMC5079215/
- Adams et al. 2014 — Dehydration and HR during exercise in the heat (ResearchGate): https://www.researchgate.net/publication/277306972
- Fluid replacement and glucose infusion prevent cardiovascular drift (GSSI): https://www.gssiweb.org/en/research/Article/fluid-replacement-and-glucose-infusion-during-exercise-prevent-cardiovascular-drift-
- Montain & Coyle 1992 — graded dehydration and cardiovascular drift: https://journals.physiology.org/doi/abs/10.1152/jappl.1992.73.4.1340
- Wingo et al. — cardiovascular drift and reduced VO2max (PubMed 15692320): https://pubmed.ncbi.nlm.nih.gov/15692320/
- Wingo et al. — VO2max after attenuation of cardiovascular drift (PubMed 16856352): https://pubmed.ncbi.nlm.nih.gov/16856352/
- Moran et al. 1998 — Physiological Strain Index (APS): https://journals.physiology.org/doi/full/10.1152/ajpregu.1998.275.1.R129
- Application of the Physiological Strain Index (DTIC ADA408496): https://apps.dtic.mil/sti/tr/pdf/ADA408496.pdf
- Wet-bulb globe temperature — Wikipedia: https://en.wikipedia.org/wiki/Wet-bulb_globe_temperature
- Verification of WBGT Index (NC State ECONet): https://econet.climate.ncsu.edu/wp-content/uploads/2021/05/WBGT.pdf
- Aerobic Decoupling and Efficiency Factor (TrainingPeaks): https://help.trainingpeaks.com/hc/en-us/articles/204071724-Aerobic-Decoupling-Pw-Hr-and-Pa-HR-and-Efficiency-Factor-EF
- Aerobic Endurance and Decoupling (TrainingPeaks blog): https://www.trainingpeaks.com/blog/aerobic-endurance-and-decoupling/
- Borg RPE scale verbal anchors (Lumen): https://courses.lumenlearning.com/suny-fitness/chapter/borg-rating-of-perceived-exertion-rpe-scale/
- Borg RPE Scale Guide (RPE Training): https://rpetraining.com/blog/borg-scale-rpe
- Chen et al. — Borg RPE 6-20 and HR (PubMed): https://pubmed.ncbi.nlm.nih.gov/24665812/
- HR regression equations with Borg 6-20 (ResearchGate table): https://www.researchgate.net/figure/HEART-RATE-HR-REGRESSION-EQUATIONS-WITH-BORG-6-20-RPE-AS-PREDICTOR_tbl3_261100868
- ACSM intensity classification table (ResearchGate): https://www.researchgate.net/figure/Classification-of-physical-activity-intensity-recommended-by-ACSMs-guidelines_tbl3_353000465
- ACSM/ESSA intensity terminology consensus 2024 (JSAMS): https://www.jsams.org/article/S1440-2440(24)00559-0/fulltext
- Lactate vs ventilatory thresholds in runners (PMC6167480): https://pmc.ncbi.nlm.nih.gov/articles/PMC6167480/
- ACE Ventilatory Threshold Testing: https://acewebcontent.azureedge.net/certifiednews/images/article/pdfs/VT_Testing.pdf
- Ventilatory threshold vs VO2R/HRR/RPE, large sample (PMC10524184): https://pmc.ncbi.nlm.nih.gov/articles/PMC10524184/
- Session-RPE method review (PMC5673663): https://pmc.ncbi.nlm.nih.gov/articles/PMC5673663/
- Validity/reliability of session-RPE (JSCR): https://journals.lww.com/nsca-jscr/fulltext/2013/01000/validity_and_reliability_of_the_session_rpe_method.38.aspx
- ACSM Metabolic Equations (SlideShare): https://www.slideshare.net/slideshow/met-calnew/26098363
- Estimating HR, EE, Performance with Wrist PPG During Running (PMC5548984): https://pmc.ncbi.nlm.nih.gov/articles/PMC5548984/
- Margaria et al. 1963 — Energy cost of running (J Appl Physiol): https://journals.physiology.org/doi/abs/10.1152/jappl.1963.18.2.367
- Running — Compendium of Physical Activities: https://pacompendium.com/running/
- Keytel et al. 2005 — EE prediction from HR (Semantic Scholar): https://www.semanticscholar.org/paper/Prediction-of-energy-expenditure-from-heart-rate-Keytel-Goedecke/2f647f62e650bf7df32546e541af3cf155297749
- Hexoskin EE calculation (Keytel implementation): https://support.hexoskin.com/calculation-of-the-energy-expenditure
- Heart Rate Calorie Calculator (Keytel coefficients): https://sport-calculator.com/calculators/general-fitness/heart-rate-calorie-calculator
- Individual vs group HR-VO2 calibration for AEE (PLOS One / PMC6837421): https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0224948
- LaForgia et al. — EPOC intensity and duration (PubMed 17101527): https://pubmed.ncbi.nlm.nih.gov/17101527/
- Acute interval vs continuous running EPOC (Nature Sci Rep): https://www.nature.com/articles/s41598-024-59893-9
- 7 Things to Know About EPOC (ACE): https://www.acefitness.org/resources/pros/expert-articles/5008/7-things-to-know-about-excess-post-exercise-oxygen-consumption-epoc/
- Validity of Wrist-Worn Activity Trackers for VO2max and EE (PMC6747132): https://pmc.ncbi.nlm.nih.gov/articles/PMC6747132/
