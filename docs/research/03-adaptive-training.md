# Adaptive / auto-regulated training science (F6)

> Prior-art research brief, generated 2026-06-24 via a fan-out research workflow
> (parallel web research → adversarial verification → synthesis). Claims that
> failed independent verification are flagged inline. Treat as a literature
> survey to ground implementation, not as gospel — spot-check before shipping.

## Periodization & Intensity Distribution

### The three-zone model: a concrete classification primitive

Every adaptive engine needs a way to label intensity. The most implementable scheme is Seiler's three-zone model, which defines boundaries by two physiological thresholds (LT1/VT1 and LT2/VT2) rather than arbitrary percentages ([Frontiers/PMC4621419](https://pmc.ncbi.nlm.nih.gov/articles/PMC4621419/)):

| Zone | Lactate | Threshold anchor | Approx. %HRmax |
|------|---------|------------------|----------------|
| Z1 (LOW) | < 2 mmol/L | ≤ VT1 | ~60–81% |
| Z2 (MODERATE/threshold) | 2–4 mmol/L | VT1–VT2 | ~82–87% |
| Z3 (HIGH) | ≥ 4 mmol/L | ≥ VT2 | > 88–90% |

To compute a Training Intensity Distribution (TID), classify each session (or each minute/sample) into a zone and sum the time fractions.

**Critical implementation decision — counting method.** "Time-in-zone" (label each minute) and "session-goal" (label the whole session by its primary intent) yield materially different percentages; session-goal counting tends to look more polarized. The app **must pick one method and document it consistently**, because it directly affects whether a plan is reported as polarized or pyramidal. This counting ambiguity is one of the main reasons the literature disagrees on polarized-vs-pyramidal superiority (see Open Questions).

### Distribution archetypes: polarized vs pyramidal vs threshold

- **Polarized (POL):** ~75–80% Z1, 5–10% Z2, 15–20% Z3. The "80/20" shorthand bundles Z2+Z3 into the "20."
- **Pyramidal (PYR):** ~84–95% Z1, descending Z2 > Z3 (e.g., elite rowers 77.3% / 16.9% / 5.8%).
- **Threshold:** concentrates Z2 (e.g., 46/54/0).

A clean programmatic discriminator: **compute the Z2:Z3 ratio. If Z3 > Z2 it is polarized; if Z2 > Z3 it is pyramidal** ([PMC4621419](https://pmc.ncbi.nlm.nih.gov/articles/PMC4621419/)). Roughly **2 high-intensity sessions/week** is sufficient to drive adaptation without overtraining markers.

### What the evidence actually supports

The single RCT most cited for polarized superiority is **Stöggl & Sperlich (2014)** — a 9-week, 48-athlete (41 completers, baseline VO2peak 62.6 mL/kg/min) trial across four TID groups ([PMC3912323](https://pmc.ncbi.nlm.nih.gov/articles/PMC3912323/)):

| Group | TID (low/thr/high) | VO2peak | Time-to-exhaustion | Peak vel/power |
|-------|--------------------|---------|--------------------|----------------|
| POL | 68/6/26 | **+11.7%** (p<0.05) | +17.4% | +5.1% |
| HIIT | 43/0/57 | +4.8% (p<0.05) | +8.8% | +4.4% |
| THR | 46/54/0 | ns | +6.2% | +1.8% |
| HVT | 83/16/1 | ns | +8.0% | −1.5% |

**Important caveat — do not over-generalize from this one study.** The most rigorous **2024 Sports Medicine meta-analysis** (17 studies, n=437) finds the polarized advantage is real but *small and conditional* ([PMC11329428](https://pmc.ncbi.nlm.nih.gov/articles/PMC11329428/)):

- POL vs all other TID on VO2peak: **SMD = 0.24 [0.01, 0.48], p=0.040** (small, I²=0%).
- **POL vs pyramidal head-to-head: SMD = 0.08 [−0.39, 0.55], p=0.73 — NOT significant.**
- No significant difference on time-trial (SMD = −0.01), time-to-exhaustion (SMD = 0.30, p=0.24), or velocity/power at VT2 (SMD = 0.04).
- POL is significant **only** in two subgroups: interventions **< 12 weeks** (SMD = 0.40 [0.08, 0.71]) and **highly-trained/national athletes** (SMD = 0.46 [0.10, 0.82]). For ≥12-week blocks or developmental athletes, **no advantage**.

A complementary 2024 systematic review pooling 11 studies reports a real-world TID of **81.3 ± 8.0% low / 3.4 ± 3.2% moderate / 15.4 ± 6.3% high**, with running-specific VO2max gains of +8.5% and an "optimal band" of **75–80% LIT with 15–20% HIT** ([PMC11679080](https://pmc.ncbi.nlm.nih.gov/articles/PMC11679080/)).

**Engineering takeaway:** Recommend polarized for *short blocks and advanced runners*. For recreational/sub-elite at lower volume, **pyramidal and polarized are interchangeable** — both beat threshold-heavy. **Do not hard-code 80/20**; expose the model as a user/coach choice.

### Block periodization

Block periodization concentrates HIT. In Rønnestad's well-trained cyclists (12 wk), the BLOCK group did 5 HIT sessions in week 1 then 1 HIT/week in weeks 2–4, repeating; TRADITIONAL did 2 HIT/week throughout, **with total HIT and LIT volume matched**. Result: **VO2max +8.8% (BLOCK) vs +3.7% (TRAD)**; power at 2 mmol/L lactate +22% vs +10% ([PubMed 23134196](https://pubmed.ncbi.nlm.nih.gov/23134196/), [PMC12575440](https://pmc.ncbi.nlm.nih.gov/articles/PMC12575440/)).

**Honest caveat:** this evidence is **almost entirely from cyclists**, in small (n~20), short (4–12 wk), lower-quality studies. Whether the advantage transfers to runners — who carry far higher mechanical/impact recovery cost — is **not established**. Treat block periodization for runners as a reasonable, structured option, not a proven winner.

### Runner-facing structures (coach-derived, lower evidence grade)

The following weekly layouts are practitioner conventions, not products of controlled trials ([Track & Field News](https://trackandfieldnews.com/track-coach/running-periodization-part-3-block-and-undulating-periodization/)) — useful as templates, but flag them as such:

- **Block (runners):** 3–4 wk mesocycle. Week 1 = 5–6 hard sessions (concentrated load); weeks 2–3 = 1 hard + high-volume easy; week 4 = recovery. Block types: Accumulation → Transmutation → Realization.
- **Undulating (DUP):** vary intensity day-to-day, e.g., volume phase `easy/medium/easy/medium/easy/hard`; intensity phase `hard/medium/easy/medium/hard/easy`.
- **Standard recreational week:** Tue intervals, Thu tempo, Sat/Sun long run, rest easy aerobic.
- **Linear:** sequential base → build → peak → taper, shifting volume→intensity over the macrocycle.

DUP is well-validated in resistance training but lacks high-quality *running* RCTs; the patterns above are coach-derived.

---

## Training-Load Quantification

### Heart-rate-based: TRIMP family

**Banister TRIMP** (verified formula, sex-specific):

```
TRIMP = Duration(min) × %HRR × Y
%HRR  = (HRmean − HRrest) / (HRmax − HRrest)
Y(male)   = 0.64 × e^(1.92 × %HRR)
Y(female) = 0.86 × e^(1.67 × %HRR)
```

The exponential up-weights high-intensity work to mirror the blood-lactate curve, which differs by sex. **The female leading constant is 0.86, not 0.64** — verified against multiple authoritative sources ([Orthopaedic Manipulation](https://www.orthopaedicmanipulation.com/quantification-of-training-load/), [Intervals.icu forum](https://forum.intervals.icu/t/bannisters-trimp/10200), [veohtu](https://www.veohtu.com/trimp.html)). Some web pages render the female constant incorrectly as 0.64; ignore those.

**Edwards TRIMP** — zone-summation, integer weights 1–5 ([trainingimpulse.com](https://www.trainingimpulse.com/edwards-trimp)):

```
TRIMP = Σ (minutes_in_zone × zone_weight)
Z1 50–60% HRmax ×1 … Z5 90–100% HRmax ×5
```
Simple and HR-only, but zone limits/coefficients are physiologically arbitrary.

**Lucia TRIMP** — three VT-anchored zones, weights 1/2/3 ([Lucia's TRIMP](https://www.trainingimpulse.com/lucias-trimp-0)): below VT1 ×1, VT1–VT2 ×2, above VT2 ×3. Preferred in research because zones map to measured thresholds.

> Note: the exact Edwards 1993 zone boundaries vary slightly across secondary sources; the 50/60/70/80/90/100% scheme is the commonly reported version.

### Power/pace-based: TSS and rTSS

**TrainingPeaks TSS** (verified) ([TrainingPeaks](https://www.trainingpeaks.com/learn/articles/estimating-training-stress-score-tss/)):

```
TSS = (sec × NP × IF) / (FTP × 3600) × 100
    = IF² × duration_hours × 100        (simplified)
IF  = NP / FTP
```
By definition, 1 hour at FTP (IF=1.0) = 100 TSS. **Normalized Power (NP)** = 4th root of the mean of the 4th powers of a rolling 30-second moving average of power ([NP help](https://help.trainingpeaks.com/hc/en-us/articles/204071804-Normalized-Power)). The 30s-window / 4th-power detail is verified, though if exactness matters cross-check Coggan's *Training and Racing with a Power Meter*.

**Running TSS (rTSS)** substitutes Normalized Graded Pace for power ([rTSS explained](https://www.trainingpeaks.com/learn/articles/running-training-stress-score-rtss-explained/), [veohtu](https://www.veohtu.com/trimp.html), [stressfactor repo](https://github.com/andrewhao/stressfactor)):

```
rTSS = (Time_sec × NGP × IF) / (FTP_pace × 3600) × 100
IF   = NGP / FTP_pace      (paces in m/s)
```
1 hour at threshold pace ≈ 100 rTSS. **HR fallback:** `HRSS = (session-TRIMP / 1-hour-TRIMP-at-LT) × 100`.

> **Open issue:** TrainingPeaks does **not** publish its NGP grade-adjustment polynomial — it is proprietary. Strava's GAP curve and the stressfactor repo are approximations whose coefficients differ by vendor. You must pick and document an explicit grade model (rule of thumb: +1% gradient ≈ +3.3% cost; downhill returns ~55% of equivalent uphill).

### Subjective: session-RPE

The cheapest, sensor-free metric, and well-validated ([PMC5673663](https://pmc.ncbi.nlm.nih.gov/articles/PMC5673663/)):

```
sRPE load (a.u.) = RPE(CR-10) × duration_min     (collect ~30 min post-session)
Monotony = mean(daily TL) / SD(daily TL)         (over 7 days)
Strain   = weekly total TL × Monotony
```
sRPE correlates with Edwards TRIMP r≈0.56–0.97 and Banister TRIMP r≈0.50–0.77. **Recommendation: implement sRPE as the universal fallback load metric** so the engine works even without a wearable.

### ACWR — implement, but as a soft warning only

```
ACWR = acute(7-day load) / chronic(28-day load)
Coupled:   chronic includes the acute week (mathematically coupled — avoid)
Uncoupled: chronic excludes the acute week (preferred)
EWMA form: EWMA_today = Load_today×λ + (1−λ)×EWMA_yesterday,  λ = 2/(N+1)
           (N=7 → λ≈0.25; N=28 → λ≈0.069)
```
The commonly cited bands ([PMC12487117](https://pmc.ncbi.nlm.nih.gov/articles/PMC12487117/), [Science for Sport](https://www.scienceforsport.com/acutechronic-workload-ratio/)): <0.8 undertraining, **0.8–1.3 "sweet spot,"** 1.3–1.5 elevated, **>1.5 high-risk, >2.0 sharply elevated** (~17% injury risk that week in rugby). EWMA shows higher sensitivity than rolling averages at high bands.

**Mandatory caveat — the sweet spot is contested and should NOT be treated as truth:**

- **Mathematical coupling** produces spurious correlation in the coupled variant; use uncoupled ([Lolli et al.](https://www.researchgate.net/publication/320847733_Mathematical_coupling_causes_spurious_correlation_within_the_conventional_acute-to-chronic_workload_ratio_calculations)).
- Impellizzeri et al. show the ratio magnifies acute load **without added predictive value**, introduces discretization bias, and is less accurate than continuous models ([Conceptual Issues and Pitfalls](https://www.semanticscholar.org/paper/Acute:Chronic-Workload-Ratio:-Conceptual-Issues-and-Impellizzeri-Tenan/ede5743a426fd6429d28f8505500a3f771dbcf8b)).
- A 2020 study found **no protective sweet spot**: in pentathlon **83.3% of injuries occurred *inside* the 0.8–1.3 band** ([Frontiers 2020](https://www.frontiersin.org/journals/physiology/articles/10.3389/fphys.2020.01034/full)).
- The figure also has a dedicated 2019 BJSM rebuttal, "[the ACWR-injury figure and its sweet spot are flawed](https://www.researchgate.net/publication/333589357_The_acute-chronic_workload_ratio-injury_figure_and_its_'sweet_spot'_are_flawed)."

The sweet spot originated from team-sport (Gabbett) datasets; transfer to endurance running is **weakly supported**. **Implementation rule: compute ACWR and surface it as a gentle "load is spiking" warning, never as a hard gate or injury prediction.**

---

## Fitness–Fatigue / Performance Management Chart

### The Banister impulse-response model

Performance = baseline + (positive fitness term) − (negative fatigue term), via convolution of training impulses with a two-exponential transfer function ([arXiv 2505.20859](https://arxiv.org/html/2505.20859v1)):

```
P(t) = P0 + Σ_{i<n} w(i) × [ k1·e^(−(n−i)/τ1) − k2·e^(−(n−i)/τ2) ]
g(t) = k1·e^(−t/τ1) − k2·e^(−t/τ2)
```
where `w(i)` = daily load, `k1` = fitness gain, `k2` = fatigue gain, `τ1` = fitness decay, `τ2` = fatigue decay.

**Parameter values (verified ranges).** Original Banister: k1=1.0, k2≈1.8–2.0, τ1≈49–50 d, τ2≈11 d. Classic/textbook ranges: τ1 ≈ 40–45 d, τ2 ≈ 7–15 d ([Fellrnr](https://fellrnr.com/wiki/Modeling_Human_Performance), [arXiv PDF](https://arxiv.org/pdf/2505.20859)). Empirical per-athlete fits vary widely (e.g., a cycling fit gave k1=0.048, k2=0.117, τ1=38 d, τ2=1.9 d). **τ1 is fairly stable (~42 d, low inter-individual variation); τ2 varies strongly (7–22 d).** Universally: k2>k1 and τ2<τ1 — fatigue is larger but decays faster, which is the mathematical basis for tapering.

> The often-quoted "original Banister 1975 swimming" values could not be confirmed from the primary paper (PDF blocked); the values above come from secondary/review sources but are independently cross-confirmed.

### The Performance Management Chart (the practical version to ship)

Coggan's PMC drops the gain factors and replaces the convolution with **two EWMAs of daily TSS** ([Science of the Performance Manager](https://www.trainingpeaks.com/learn/articles/the-science-of-the-performance-manager/), [Fitness (CTL)](https://help.trainingpeaks.com/hc/en-us/articles/204071884-Fitness-CTL), [FasCat](https://fascatcoaching.com/blogs/training-tips/performance-manager-chart/)):

- **CTL (Fitness)** = EWMA of daily TSS, **τ = 42 days**.
- **ATL (Fatigue)** = EWMA of daily TSS, **τ = 7 days**.
- **TSB (Form)** = CTL_yesterday − ATL_yesterday.

**Implementation recipe (daily step):**

```
load = TSS_today        # 0 on rest days
λ_CTL = 1 − e^(−1/42) = 0.02353
λ_ATL = 1 − e^(−1/7)  = 0.13319
CTL[d] = CTL[d−1] + (load − CTL[d−1]) × λ_CTL
ATL[d] = ATL[d−1] + (load − ATL[d−1]) × λ_ATL
TSB[d] = CTL[d−1] − ATL[d−1]
```
Two mathematically equivalent forms exist: the exact `X = X_prev·e^(−1/τ) + TSS·(1−e^(−1/τ))` and TrainingPeaks' linear `(1/τ)` approximation. Seed CTL/ATL from a prior average daily TSS, or seed at 0 and warn that values stabilize only after ~6 weeks.

**TSB interpretation bands:** < −10 fatigued (productive overload); −10 to +10 neutral; > +10 fresh/tapered (race-ready, but sustained high TSB = detraining). Productive racing CTL is often cited as 100–150 TSS/day. **The main tunable parameter is ATL τ** (default 7; 4–5 d for younger/low-load, 10–12 d for masters/high-load); CTL stays at 42.

> The "42-day half-life" coaching phrase is **imprecise** — e^(−42/42)=0.368, not 0.5. It is a time constant, not a half-life; don't implement a half-life parameterization based on that phrasing.

### Honest limitations

- **Data hunger:** fitting the full impulse-response model reliably needs ~20–200 performance tests per athlete — impractical, which is exactly why the PMC drops k1/k2.
- **Variable fit:** r² typically 0.7–0.97 but with wide spread (e.g., 0.79±0.13).
- **Non-identifiable gains:** k1, k2 vary so much that generic values are "difficult, if not impossible, to rely on."
- **Linearity pathology:** because performance is linear in load, naive optimization recommends "train maximally every day, then stop" — contradicting real tapering. Nonlinear/variable-dose variants (Busso) add a load-dependent k2 to fix this.
- **PMC gives relative, not absolute, performance;** needs consistent data; >10% missing data is unreliable; optimal TSB/CTL is athlete-specific.

> **Unresolved for a running app:** TSS (pace/power) vs Banister TRIMP (HR) are **not interchangeable** as the daily impulse and change parameter scaling. Pick one primary impulse and convert consistently. WKO's exact production recursion (today's vs yesterday's TSS) is not confirmed from a primary engineering doc.

---

## Auto-Regulation & HRV-Guided Training

### Does HRV guidance work? (Yes, modestly)

The largest meta-analysis (6 RCTs, n=195) found HRV-guided daily training beats fixed plans for VO2max ([PMC7663087](https://pmc.ncbi.nlm.nih.gov/articles/PMC7663087/)):

- HRV-guided **ES = 0.402 [0.273, 0.531], p<0.0001** vs predefined **ES = 0.215 [0.101, 0.329]**; between-arm difference p<0.0001.
- **Amateurs ES = 0.36 > elite ES = 0.17** — the benefit is larger for recreational athletes.
- Most interventions ~8 weeks. **Effect shrinks as sample size grows** (regression coef −0.016, p≤0.0001), so headline effects likely overstate real-world benefit for trained users.

A second meta-analysis reported moderate effects on submaximal markers (Wmax SMD=0.66, aerobic performance SMD=0.71, power at VT1 SMD=0.62), with HRV-guided arms doing **fewer** moderate/high sessions yet matching outcomes ([MDPI Applied Sciences 10:8532](https://www.mdpi.com/2076-3417/10/23/8532)). Treat this second one as *medium-confidence* corroboration.

**Mechanism, not miracle:** the landmark Kiviniemi (2007) RCT showed HRV-guidance mainly **reduced non-responders** — VO2max *decreased* in ~50% of fixed-plan participants vs only ~11% of HRV-guided ([summarized in PMC7663087](https://pmc.ncbi.nlm.nih.gov/articles/PMC7663087/)). And in cardiac rehab, HRV-guided training **matched** traditional HIIT on VO2max (+9.4% vs +6.8%) while requiring **31 fewer minutes** of high-intensity work ([PMC10828341](https://pmc.ncbi.nlm.nih.gov/articles/PMC10828341/)) — a strong precedent for a "lower-burden" app mode.

### The decision rule to implement

The canonical operational rule (Vesterinen/Kiviniemi lineage) uses a smallest-worthwhile-change band on smoothed lnRMSSD ([PMC7663087](https://pmc.ncbi.nlm.nih.gov/articles/PMC7663087/), [Marco Altini](https://marcoaltini.substack.com/p/a-brief-history-of-heart-rate-variability)):

```
1. Familiarization (~4 wk): collect supine morning lnRMSSD → mean, SD.
2. SWC band = mean ± 0.5 × SD
3. Each morning: lnRMSSD_7d = 7-day rolling average
4. If lnRMSSD_7d WITHIN band → run the planned hard/moderate session
   If BELOW band            → substitute low-intensity or rest
```

Using the **7-day average (not raw daily)** damps noise, so several consecutive low days are needed to trigger backoff. Simpler historical variants exist (Kiviniemi 2007: train hard if today's HRV ≥ baseline).

> **Tunable, no consensus:** the 0.5×SD multiplier and 7-day window are the most-cited but not standardized; some protocols use raw daily lnRMSSD or 1-SD bands. Expose band width as a design parameter.

### Pace-zone generation: deterministic, no ML needed

Jack Daniels' VDOT equations are a fully formulaic rule engine (verified coefficients) ([VDOT calculator](https://running-calculator.com/vdot-calculator/), [sport-calculator](https://sport-calculator.com/calculators/running/jack-daniels-running-calculator), [MyProCoach](https://www.myprocoach.net/calculators/running-zones/)):

```
VO2(v)      = −4.60 + 0.182258·v + 0.000104·v²          (v in m/min)
%VO2max(t)  = 0.8 + 0.1894393·e^(−0.012778·t) + 0.2989558·e^(−0.1932605·t)   (t in min)
VDOT        = VO2(race velocity) / %VO2max(race time)
```
Invert to get training paces at fixed %VO2max targets: Easy ~59–74%, Marathon ~75–84%, Threshold ~83–88%, Interval ~95–100%, Repetition >100%. A cheap alternative: threshold pace ≈ 95% of average pace from a 30-min time trial.

### Planner architectures

Two implementable patterns, each with citable limits ([Sensai](https://www.sensai.fit/blog/ai-coaching-endurance-athletes-runners-cyclists)):

- **Rule/template engines:** zones from Critical Power/Pace or threshold tests; periodization via fixed formulas + CTL/ATL + ACWR. Deterministic and auditable, but "can't reason about context" (sleep, niggles, nerves).
- **LLM reasoning layers:** ingest HRV/sleep/load and regenerate the week in natural language, can explain and be questioned. Framing guardrail: "a reasoning layer over data, not a crystal ball"; GIGO-sensitive to wearable quality.

**Recommended guardrail pattern: LLM proposes, rule engine validates** — clamp paces to VDOT zones, enforce ACWR bounds, cap weekly progression. Note this whole section is *vendor/blog-level (medium-to-low confidence)*; **no peer-reviewed RCT validates an LLM-with-guardrails coach** against rule-based plans or human coaches.

A 2025 ML study shows a concrete personalization pipeline (gradient boosting for marathon-improvement prediction + SVM for zone-distribution assignment + k-means responder phenotyping; 27 features, 5-fold CV; n=120) — prediction MAE 6.15 min (pyramidal) / 7.28 min (polarized), with **~18% of users classified as non-responders** to standard plans, arguing for phenotype routing ([Scientific Reports 2025](https://www.nature.com/articles/s41598-025-25369-7)). **AI Endurance** is a production example of optimization-based generation (~30,000 iterations searching polarized inputs, individualized zones) ([AI Endurance](https://aiendurance.com/blog/which-training-really-works)) — cited as illustrative only (low confidence, vendor source).

---

## Open Questions (honest unknowns)

1. **Polarized vs pyramidal is unresolved.** The 2024 Sports Medicine meta-analysis finds no head-to-head difference (SMD=0.08, p=0.73), while Stöggl & Sperlich favor polarized — likely confounded by counting method (time-in-zone vs session-goal) and athlete level. **Let users/coaches pick the model; don't hard-code 80/20.**
2. **Optimal HIT fraction is debated:** pooled reviews cite 15–20%, but some studies show 18–26% producing larger VO2max gains. The safe upper bound for *runners* (higher impact cost than cyclists) is unknown.
3. **Block periodization evidence is cyclist-dominated;** running RCTs are scarce and small. The +8.8% vs +3.7% advantage may not transfer to impact-constrained runners.
4. **DUP lacks running-specific RCTs;** the undulation templates are coach-derived.
5. **Most TID/periodization evidence is short (4–13 wk).** Long-term, full-season sequencing is under-evidenced; polarized's edge vanishes at ≥12 weeks.
6. **Norwegian double-threshold** method is increasingly used by elites and overlaps with pyramidal but is not separately quantified here — needs further sourcing.
7. **ACWR sweet spot (0.8–1.3) is contested** and team-sport-derived; its transfer to running is weak and directly contradicted by some studies. Ship it as a soft warning only.
8. **NGP/grade-adjustment polynomial is proprietary;** you must choose and document an explicit GAP model.
9. **Banister/PMC parameters are highly individual** and need per-athlete fitting from longitudinal data; population defaults give only rough forecasts. The exact WKO production recursion is unconfirmed.
10. **HRV RCTs are small (n≤60), short (4–8 wk), amateur-dominated,** and effects attenuate with sample size — real-world benefit for trained users may be smaller than ES=0.40 suggests.
11. **No RCT validates LLM-with-guardrails coaching** against rule-based plans or human coaches; those claims are vendor-level.
12. **The HRV SWC band width (0.5×SD, 7-day window) is not standardized** — a tunable design choice.
13. **TSS vs TRIMP as the daily impulse** are not interchangeable for a running app and change parameter scaling — pick one primary metric.

---

## Sources

- https://pmc.ncbi.nlm.nih.gov/articles/PMC4621419/
- https://pmc.ncbi.nlm.nih.gov/articles/PMC3912323/
- https://pmc.ncbi.nlm.nih.gov/articles/PMC11329428/
- https://pmc.ncbi.nlm.nih.gov/articles/PMC11679080/
- https://pubmed.ncbi.nlm.nih.gov/23134196/
- https://pmc.ncbi.nlm.nih.gov/articles/PMC12575440/
- https://onlinelibrary.wiley.com/doi/abs/10.1111/j.1600-0838.2012.01485.x
- https://trackandfieldnews.com/track-coach/running-periodization-part-3-block-and-undulating-periodization/
- https://www.veohtu.com/trimp.html
- https://www.orthopaedicmanipulation.com/quantification-of-training-load/
- https://forum.intervals.icu/t/bannisters-trimp/10200
- https://www.trainingimpulse.com/edwards-trimp
- https://www.trainingimpulse.com/lucias-trimp-0
- https://www.trainingpeaks.com/learn/articles/estimating-training-stress-score-tss/
- https://help.trainingpeaks.com/hc/en-us/articles/204071804-Normalized-Power
- https://www.trainingpeaks.com/learn/articles/running-training-stress-score-rtss-explained/
- https://github.com/andrewhao/stressfactor
- https://pmc.ncbi.nlm.nih.gov/articles/PMC5673663/
- https://science-cycling.org/wp-content/uploads/2019/07/Foster-presentatie-gecomprimeerd.pdf
- https://pmc.ncbi.nlm.nih.gov/articles/PMC12487117/
- https://www.scienceforsport.com/acutechronic-workload-ratio/
- https://www.frontiersin.org/journals/physiology/articles/10.3389/fphys.2020.01034/full
- https://www.researchgate.net/publication/320847733_Mathematical_coupling_causes_spurious_correlation_within_the_conventional_acute-to-chronic_workload_ratio_calculations
- https://www.semanticscholar.org/paper/Acute:Chronic-Workload-Ratio:-Conceptual-Issues-and-Impellizzeri-Tenan/ede5743a426fd6429d28f8505500a3f771dbcf8b
- https://www.researchgate.net/publication/333589357_The_acute-chronic_workload_ratio-injury_figure_and_its_'sweet_spot'_are_flawed
- https://arxiv.org/html/2505.20859v1
- https://arxiv.org/pdf/2505.20859
- https://www.trainingpeaks.com/learn/articles/the-science-of-the-performance-manager/
- https://help.trainingpeaks.com/hc/en-us/articles/204071884-Fitness-CTL
- https://www.trainingpeaks.com/coach-blog/a-coachs-guide-to-atl-ctl-tsb/
- https://fascatcoaching.com/blogs/training-tips/performance-manager-chart/
- https://fellrnr.com/wiki/Modeling_Human_Performance
- https://pmc.ncbi.nlm.nih.gov/articles/PMC7663087/
- https://www.mdpi.com/2076-3417/10/23/8532
- https://pmc.ncbi.nlm.nih.gov/articles/PMC10828341/
- https://marcoaltini.substack.com/p/a-brief-history-of-heart-rate-variability
- https://running-calculator.com/vdot-calculator/
- https://sport-calculator.com/calculators/running/jack-daniels-running-calculator
- https://vdoto2.com/calculator/
- https://www.myprocoach.net/calculators/running-zones/
- https://www.sensai.fit/blog/ai-coaching-endurance-athletes-runners-cyclists
- https://www.nature.com/articles/s41598-025-25369-7
- https://aiendurance.com/blog/which-training-really-works
- https://aiendurance.com/blog/the-ai-endurance-training-zones
- https://pmc.ncbi.nlm.nih.gov/articles/PMC12880663/
