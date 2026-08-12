# Amphetamine pharmacokinetics (F5)

> Prior-art research brief, generated 2026-06-24 via a fan-out research workflow
> (parallel web research → adversarial verification → synthesis). Claims that
> failed independent verification are flagged inline. Treat as a literature
> survey to ground implementation, not as gospel — spot-check before shipping.

## Amphetamine Pharmacokinetics for the `run` Training App (F5)

This briefing covers what you need to model d-amphetamine (and its formulations) inside a training app: the core PK parameters, the one-compartment math, formulation-specific multi-pulse models, multi-dose superposition, the cardiovascular/thermoregulatory/sleep/appetite effects that interact with exercise, and how to calibrate an individual's parameters from observed feedback. Every quantitative claim below has been verification-checked; where the verifier flagged an issue (one mechanistic inaccuracy in the Adderall XR description, and several extrapolation/identifiability caveats) it is called out inline.

## Core d-amphetamine PK parameters

These are the numbers to seed your population priors. They are drawn from the FDA Adderall label and the Dolder et al. healthy-subject study, both independently verified.

| Parameter | Value (default) | Range / notes | Source |
|---|---|---|---|
| Elimination half-life t₁/₂ (d-isomer) | **~10 h** | 9.77–11 h (d), 11.5–13.8 h (l) | [FDA Adderall label](https://www.accessdata.fda.gov/drugsatfda_docs/label/2017/011522s043lbl.pdf) |
| Elimination rate kₑ = ln2/t₁/₂ | **0.069 h⁻¹** | 0.088 h⁻¹ in the Dolder fit (t₁/₂ 7.9 h) | derived |
| Absorption rate k01 (IR) | **1.3 h⁻¹** | 95% CI 0.84–1.95 | [Dolder et al. (PMC5594082)](https://pmc.ncbi.nlm.nih.gov/articles/PMC5594082/) |
| Lag time Tlag (IR) | **0.8 h** | — | [Dolder et al.](https://pmc.ncbi.nlm.nih.gov/articles/PMC5594082/) |
| Tmax (IR) | **3.3 h** | 95% CI 2.7–3.9; label says "~3 h" | [Dolder et al.](https://pmc.ncbi.nlm.nih.gov/articles/PMC5594082/) |
| Apparent volume Vz/F | **~195 L (~3 L/kg)** | StatPearls quotes 4 L/kg | [Dolder et al.](https://pmc.ncbi.nlm.nih.gov/articles/PMC5594082/), [StatPearls](https://www.ncbi.nlm.nih.gov/books/NBK507808/) |
| Apparent clearance CL/F | **~17 L/h (~0.28 L/h/kg)** | — | [Dolder et al.](https://pmc.ncbi.nlm.nih.gov/articles/PMC5594082/) |
| Oral bioavailability F | **~0.9** | treat F≈1 for oral models | [Wikipedia: Dextroamphetamine](https://en.wikipedia.org/wiki/Dextroamphetamine) |
| Plasma protein binding | **<20%** | secondary sources cite 15–40% | [StatPearls](https://www.ncbi.nlm.nih.gov/books/NBK507808/) |
| pKa | **9.9** (weak base) | drives urine-pH dependence | [StatPearls](https://www.ncbi.nlm.nih.gov/books/NBK507808/) |

The Dolder reference dose was 40.3 mg d-amphetamine sulfate ≈ 29.6 mg base, producing Cmax 120 ng/mL and AUC∞ 1727 ng·h/mL — every parameter in that row was verified against the paper's Table 1 as an exact match.

**Note on Vd:** the apparent 195 L (~2.8 L/kg) and the StatPearls "4 L/kg" are not the same quantity — 195 L is Vz/**F** (inflated by F<1 not being divided out). Use **Vd/F ≈ 3–4 L/kg** and do not mix the apparent and absolute values in one equation.

### Urine-pH dependence (a large, real covariate)

Because amphetamine is a weak base (pKa 9.9), renal elimination swings dramatically with urine pH:

- **Half-life:** ~7 h in highly acidic urine, up to ~34 h in highly alkaline urine. The acidic-end 7 h figure is well supported (FDA label + StatPearls). **Caveat:** the 34 h upper bound is a StatPearls figure for an *extreme* alkaline state; the same source lists a more typical "alkaline diet" range of ~16–31 h. The primary FDA label itself only gives the 9.77–13.8 h range, not pH-specific half-lives — so treat 7–34 h as a plausible-extremes envelope, not as a calibrated lookup table.
- **Renal clearance** can exceed GFR in acidic urine, implying active tubular secretion; alkaline urine reduces ionization and renal elimination. ([StatPearls](https://www.ncbi.nlm.nih.gov/books/NBK507808/))
- **Urinary recovery of unchanged drug** ranges from 1% to 75% depending on pH (verified verbatim against FDA labels).

The often-quoted ">2/3 excreted at urine pH < 6.6 vs < 1/2 at pH > 6.7" thresholds could not be re-confirmed verbatim from a fetched primary label section — treat them as illustrative, not authoritative.

**App implication:** individual clearance should be a **wide-variance parameter**. Urine pH (and its drivers: diet, ascorbic acid/bicarbonate intake, and possibly exercise-induced acid–base shifts) is a legitimate candidate input feature explaining a meaningful chunk of inter- and intra-individual variability.

### Mass balance and linearity

- At normal urine pH: **~50% of dose** excreted as α-/4-hydroxy-amphetamine derivatives, **~30–40%** as unchanged amphetamine (verified verbatim against FDA labels). Metabolism is via CYP2D6, FMO3, and DBH.
- **PK is dose-proportional**: Cmax and AUC0-∞ rose ~3-fold from 10 mg to 30 mg ([FDA Adderall label](https://www.accessdata.fda.gov/drugsatfda_docs/label/2017/011522s043lbl.pdf)). This is the key fact that licenses **linear superposition** for multi-dose modeling.

## The one-compartment model to implement

A **one-compartment model with first-order absorption, first-order elimination, and a lag time** adequately describes oral d-amphetamine. The Dolder study explicitly fit this form for both d-amphetamine and lisdexamfetamine.

Single-dose oral (Bateman) equation, for t > tlag:

```
C(t) = (F · D · ka) / (V · (ka − ke)) · ( e^(−ke·(t − tlag)) − e^(−ka·(t − tlag)) )
```

with `ke = ln2 / t₁/₂`. For IR d-amphetamine use `ka = 1.3 h⁻¹`, `ke = 0.069–0.088 h⁻¹`, `tlag = 0.8 h`.

**Honesty caveat:** the one-compartment adequacy is *inferred* from the Dolder authors' model choice, not from a published head-to-head one- vs two-compartment adequacy test (no AIC/objective-function comparison for oral d-amphetamine was located). For oral dosing this is almost certainly fine; two-compartment behavior would only matter for capturing an IV-bolus distribution phase, which is out of scope for this app.

## Formulation-specific models

All amphetamine products are a fixed **3:1 d:l salt ratio**. For a running app, modeling the **d-isomer alone** (the more potent CNS stimulant, ¾ of the dose, ke = ln2/10 h) is a reasonable simplification. If you need enantiomer-level fidelity, model d and l as separate species — l consistently has a longer t₁/₂ (e.g. Adderall XR adults: d=10 h, l=13 h).

### Immediate-release (Adderall IR, Dexedrine)

Single Bateman pulse with the IR parameters above (ka 1.3 h⁻¹, tlag 0.8 h, Tmax ~3 h). Clinical duration ~4–6 h.

### Vyvanse / lisdexamfetamine (prodrug)

The d-amphetamine curve from LDX is **nearly identical in Cmax and AUC to IR d-amphetamine, just shifted right** (Dolder, verified: Cmax 118 vs 120 ng/mL, AUC∞ 1817 vs 1727, both n.s.; t₁/₂ 7.9 h for both). LDX parameters: ka(apparent) ≈ 0.78 h⁻¹, tlag ≈ 1.5 h, Tmax 4.6 h.

- The **rate-limiting step is enzymatic hydrolysis of the lysine–amphetamine bond in red-blood-cell cytosol by an aminopeptidase**, *not* GI absorption or gut pH (verified against the Pennick prodrug-activation paper, PMC4257105, and Wikipedia). Intact LDX is absorbed fast (LDX t₁/₂ 0.4–0.9 h) but d-amphetamine appears later and persists longer.
- Oral bioavailability **~96.4%**, dose-proportional up to 250 mg ([Wikipedia: Lisdexamfetamine](https://en.wikipedia.org/wiki/Lisdexamfetamine)). Longest clinical duration: ≥13 h (children), ≥14 h (adults).
- **Practical shortcut:** treat Vyvanse as IR d-amphetamine with a slower lumped `ka ≈ 0.6–0.8 h⁻¹` plus ~0.7 h extra lag. A formally correct sequential prodrug model (LDX compartment → d-amph compartment with conversion rate `kc`) is possible, but **`kc` is not published as a fitted constant** — you would have to estimate it yourself. The shortcut is what the data actually support.

### Adderall XR (two-pulse beads)

Model as **two IR Bateman pulses offset by ~4 hours** (roughly 50:50 split). A single 20 mg XR dose reproduces the profile of IR Adderall 10 mg taken twice 4 h apart (verified). Adult t₁/₂: d-amph 10 h, l-amph 13 h. Fasted Tmax ~5–7 h; a high-fat meal prolongs Tmax by ~2.5 h without changing extent.

> **Correction (verifier-flagged):** the delayed beads in Adderall **XR** are **time-based** (Microtrol, ~4 h delay), **not pH-dependent/enteric**. The original finding mislabeled them as enteric; the pH-dependent bead mechanism belongs to **Mydayis**, not Adderall XR. Model XR with a fixed ~4 h second-pulse lag, not a pH trigger.

### Mydayis (triple-bead, pH-dependent)

Model as **three Bateman pulses with staggered lags**: an IR bead (~0 h), a delayed-release bead releasing at **pH 5.5**, and a delayed-release bead releasing at **pH 7.0**. This is the longest single-dose ER profile (verified against the FDA clinical pharmacology review and DailyMed):

- Adult Tmax: d-amph **7.0 h** fasted, l-amph 7.5 h. Half-life: d-amph 10–11 h, l-amph 10–13 h. Linear over 12.5–50 mg.
- High-fat meal prolongs Tmax by ~5 h (d-amph 7.0 → 12.0 h).
- **Alcohol dose-dumping risk:** in vitro, 20–40% alcohol increases amphetamine release rate — worth a user-facing warning flag if you track alcohol intake.

**Caveat on the multi-pulse lags:** per-bead release fractions and exact inter-pulse lag times are **not published as fitted PK parameters**. The 4 h offset (XR) and the staggered Mydayis lags are *design intent*, not fitted values — they are reasonable starting points to tune against observed curves, not ground truth.

## Multi-dose superposition

Because amphetamine PK is linear/dose-proportional, total concentration is the sum of single-dose curves shifted by each dosing time:

```
C_total(t) = Σ_i  C_single(t − t_i)   for t ≥ t_i
```

For multi-bead ER products, **first build the single-dose curve as the sum of per-bead Bateman pulses, then apply superposition across days.**

Steady-state accumulation factor for identical doses every τ hours:

```
R = 1 / (1 − e^(−ke·τ))
```

For once-daily dosing (τ = 24 h, ke = 0.069 h⁻¹): R ≈ **1.23** — modest accumulation (verified: e^(−1.656)=0.191, 1/(1−0.191)=1.236). ([superposition reference, PMC11351987](https://pmc.ncbi.nlm.nih.gov/articles/PMC11351987/))

**Caveat:** superposition fails if PK becomes nonlinear (saturable processes). Amphetamine is documented dose-proportional across the therapeutic range, so this is safe there. For LDX specifically, whether the RBC aminopeptidase saturates at supratherapeutic doses is **disputed** — sources call it high-capacity and effectively non-saturable in normal use, but no Km/Vmax is published, so linear superposition could in principle break at very high LDX doses.

## Physiological effects relevant to training

### Cardiovascular — distinguish chronic average from acute peak

This is the single most important modeling subtlety. The two regimes differ by ~4–5×:

| Regime | ΔSBP | ΔDBP | ΔHR | Source |
|---|---|---|---|---|
| **Chronic therapeutic** (resting, meta-analysis n=10,583) | +1.93 mmHg | +1.84 mmHg | **+3.71 bpm** | [meta-analysis (PMID 40152309)](https://pubmed.ncbi.nlm.nih.gov/40152309/) |
| **Acute single dose** (d-amph 40 mg, peak) | +27 mmHg | +18 mmHg | **+18 bpm** | [Dolder et al.](https://pmc.ncbi.nlm.nih.gov/articles/PMC5594082/) |

Both rows are verified exact matches. Implementation: for **chronic stable dosing** use a small flat additive resting offset (~+2 mmHg BP, ~+4 bpm HR); for **acute dosing** use the large Emax-type magnitudes **time-locked to the PK curve** (peak near Tmax). Do not use the small chronic numbers for an acute single dose.

**Open question:** there is no single published concentration→effect (Emax/EC50) curve linking mg dose → plasma ng/mL → bpm/mmHg. The ~4–5× gap between chronic and acute is real but not parameterized in one source; you will need to bridge it heuristically or fit it from user data.

### HR during exercise — offset, not slope (tentative)

The available evidence suggests stimulants raise the **resting/submaximal HR baseline by a roughly constant offset** rather than steepening the effort→HR slope, and do **not** raise maximal HR. Submaximal HR is ~+8 to +13 bpm on stimulant medication with VO₂, RER, and RPE unchanged ([methylphenidate-in-sport summary](https://www.researchgate.net/publication/288465098_Methylphenidate_and_its_use_in_sports_competitions_in_patients_with_ADHD)).

**Soften this — it is the weakest-supported modeling claim here (confidence: medium).** The cleanest rest→submax→max graded data is for **caffeine** (which *lowers* submaximal HR — opposite sympathetic direction to amphetamine), and the amphetamine data is submaximal-only. **No human amphetamine graded-exercise study fitting an HR–power slope vs baseline offset across the full rest-to-max range exists.** Whether amphetamine clamps or raises HRmax is unresolved. A defensible starting model: an **additive HR offset that attenuates toward zero as intensity approaches HRmax**, flagged in the app as an assumption to be calibrated, not a fact.

### Thermoregulation and heat-stroke risk

Amphetamine increases heat **dissipation** (lowering core temp early in exercise and extending time-to-exhaustion) but **lets muscle temperature climb to dangerous levels before exhaustion** by masking fatigue/thermal cues. In a rat treadmill study with explicit heat-balance ODEs, 2 mg/kg extended time-to-exhaustion from 14.8 to 17.3 min and raised the heat-dissipation coefficient, with core temp at exhaustion unchanged (40.0 vs 39.9 °C) but estimated **muscle temp higher (+0.7 °C, 41.9 vs 41.1 °C)** — a tissue-damage range ([Morozova 2016, PMC5027360](https://pmc.ncbi.nlm.nih.gov/articles/PMC5027360/)).

Heat-balance model form: `dTc/dt = Pc − η·(Tc−Tm) − ηa·(Tc−Ta)`; `dTm/dt = Pm − η·(Tm−Tc)`.

**App implication:** amphetamine + hot ambient conditions should **raise a heat-risk flag during exercise even when the user feels fine** (delayed perceived fatigue is the danger). **Heavy caveat:** the quantitative thresholds come from a **rodent ODE model and athlete case reports** — no controlled human core-temp-during-exercise dataset on amphetamine was located. Use the *direction* (override-the-safety-switch) with confidence; treat the specific temperatures as extrapolated.

### Sleep

Amphetamine dose-dependently delays sleep onset, increases wakefulness, and reduces REM/deep sleep for several hours post-dose ([stimulant/polysomnography review](https://www.tandfonline.com/doi/full/10.1080/08039488.2020.1833984)). Given d-amph t₁/₂ ~10 h, a morning IR dose still has meaningful plasma levels at bedtime.

**App implication:** penalize predicted sleep latency/efficiency for doses whose PK tail is still elevated near the user's bedtime, with the suppression window tracking the PK decay. **Caveat (confidence: medium):** the dose-response numbers are largely **preclinical (mg/kg)**; human "minutes of added latency per mg at a given pre-bed interval" is not well quantified. Also note that in *stabilized* ADHD treatment, sleep can actually *improve* vs unmedicated baseline (symptom-control effect) — so the penalty should apply to acute/peri-peak exposure, not blindly to all dosing.

### Appetite

Appetite suppression (hypothalamic NE + dopaminergic D1/D2 signaling) is dose-dependent, **strongest in the first ~2 h post-dose**, and develops tolerance over weeks–months ([PMC6832849](https://pmc.ncbi.nlm.nih.gov/articles/PMC6832849/)). Model suppression amplitude tracking the early PK window (near Tmax) with a tolerance-decay factor for chronic users. **Caveat (confidence: medium):** mechanism is well established but the human dose-response is mostly preclinical.

## Population PK and individual calibration

### Population structure

Use a **one-compartment model, first-order absorption with lag**, with:
- **Allometric weight scaling**: exponent ~0.75 on CL/F, ~1.0 on V/F. A pediatric popPK of LDX (n=1365 concentration points) confirmed body weight as a covariate on both CL/F and V/F, plus ethnicity (Japanese vs non) on CL/F ([ScienceDirect popPK](https://www.sciencedirect.com/science/article/pii/S1347436720304134)).
- **Age effects**: children eliminate amphetamine faster (t₁/₂ ~1–2 h shorter); adolescents show ~21–31% higher Cmax/AUC than adults at the same dose. (The verbatim scaling formula was summarized in sources but not extracted — treat age scaling as a known direction, not a precise equation.)
- A useful sanity anchor: Cmax scales ~linearly with dose, ≈3 ng/mL per mg at the 40 mg whole-body level.

### Individual calibration via MAP-Bayesian / MIPD

You can personalize CL/F, V/F, and ka from a population prior using **Maximum A Posteriori Bayesian estimation** (Model-Informed Precision Dosing). Start from priors (population means + inter-individual variance ω + residual error σ) and minimize:

```
Σ (obs − pred)² / σ²  +  Σ η² / ω²
```

Observations can be plasma levels **or PD biomarkers** (resting/submaximal HR and BP deltas) when drug levels are unavailable. Hybrid ML/PK approaches can outperform pure MAP-Bayes by selectively flattening priors when individual data conflicts ([PMC8520755](https://pmc.ncbi.nlm.nih.gov/articles/PMC8520755/), [PMC8020855](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC8020855/)). Open-source tooling exists: **posologyr** (R, [PMC8879752](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC8879752/)) and **PKPy** (Python).

**Important caveats (confidence: medium):**
- **PD-only calibration confounds PK with PD sensitivity** — observing only an HR delta cannot separate "more drug in plasma" (PK: concentration scale) from "more sensitive response" (PD: Emax/EC50). You need either multiple time points or a few anchored/known doses to disentangle them.
- MAP-Bayes is well-validated for narrow-therapeutic-index drugs (vancomycin, aminoglycosides, digoxin, phenytoin, lithium). **Amphetamine is not a standard TDM target, and no study validates calibrating amphetamine PK specifically from wearable HR/BP.** The method is methodologically sound but **unproven for this exact use case** — present any in-app personalization as an estimate, not a clinical figure.

## Open questions and honest limitations

- **pH-specific half-lives** (7 h acidic / 34 h alkaline) come from secondary sources (StatPearls); the primary FDA label gives only the 9.77–13.8 h range and the 1–75% urinary-recovery range. The 34 h figure is an extreme, not a typical value.
- **The ">2/3 at pH<6.6 vs <1/2 at pH>6.7" excreted-fraction thresholds** were not re-confirmed verbatim from a primary label section.
- **No formal one- vs two-compartment model comparison** for oral d-amphetamine exists; adequacy is inferred.
- **LDX→d-amph conversion constant kc** is not published as a fitted value; the lumped-ka shortcut is what's actually supported.
- **LDX aminopeptidase saturability** at supratherapeutic doses is disputed; no Km/Vmax found.
- **ER per-bead release fractions and inter-pulse lags** (XR ~4 h, Mydayis pH 5.5/7.0) are design intent, not fitted PK lags.
- **The chronic→acute CV concentration–effect curve** (mg → ng/mL → bpm/mmHg) is not published in any single source; the ~4–5× gap must be bridged heuristically.
- **Amphetamine HR offset-vs-slope across full exercise range** has no human study; the baseline-shift hypothesis is borrowed from caffeine (opposite direction) and submaximal-only data.
- **Whether amphetamine raises or clamps HRmax** is unresolved.
- **Human exercise core-temperature thresholds** are extrapolated from rodent ODE models and case reports.
- **Sleep-latency and appetite dose-response** are largely preclinical (mg/kg); human per-mg numbers are not well quantified.
- **Wearable-based amphetamine PK calibration** is methodologically plausible but untested and subject to PK/PD identifiability confounding.

## Sources

- FDA Adderall label (2017): https://www.accessdata.fda.gov/drugsatfda_docs/label/2017/011522s043lbl.pdf
- FDA Adzenys XR-ODT label (2017): https://www.accessdata.fda.gov/drugsatfda_docs/label/2017/204326s002lbl.pdf
- DailyMed — Adderall (Dextroamphetamine/Amphetamine): https://dailymed.nlm.nih.gov/dailymed/fda/fdaDrugXsl.cfm?setid=f92497b5-baea-4760-b615-45a0b6184402
- DailyMed — Adderall XR: https://dailymed.nlm.nih.gov/dailymed/drugInfo.cfm?setid=aff45863-ffe1-4d4f-8acf-c7081512a6c0
- Adderall XR prescribing information (drugs.com): https://www.drugs.com/pro/adderall-xr.html
- DailyMed — Mydayis: https://dailymed.nlm.nih.gov/dailymed/drugInfo.cfm?setid=141a7970-3f06-44ea-9ab7-aeece2c085fc
- Mydayis Clinical Pharmacology Review (FDA): https://www.fda.gov/media/142061/download
- ADDERALL XR Pharmacology — RxReasoner: https://www.rxreasoner.com/monographs/adderall/pharmacology
- StatPearls — Dextroamphetamine-Amphetamine (NBK507808): https://www.ncbi.nlm.nih.gov/books/NBK507808/
- Dolder et al. — PK/PD of Lisdexamfetamine vs D-Amphetamine in Healthy Subjects (PMC5594082): https://pmc.ncbi.nlm.nih.gov/articles/PMC5594082/
- Dolder et al. (Frontiers in Pharmacology 2017, full text): https://www.frontiersin.org/journals/pharmacology/articles/10.3389/fphar.2017.00617/full
- Pennick — Lisdexamfetamine prodrug activation by RBC peptidase (PMC4257105): https://pmc.ncbi.nlm.nih.gov/articles/PMC4257105/
- Lisdexamfetamine Dimesylate: Prodrug Delivery, Amphetamine Exposure and Duration of Efficacy (PMC4823324): https://pmc.ncbi.nlm.nih.gov/articles/PMC4823324/
- Single-Dose PK of Lisdexamfetamine in Normal vs Impaired Renal Function (PMC4949011): https://pmc.ncbi.nlm.nih.gov/articles/PMC4949011/
- Population PK of d-amphetamine in pediatric ADHD (ScienceDirect): https://www.sciencedirect.com/science/article/pii/S1347436720304134
- Wikipedia — Dextroamphetamine: https://en.wikipedia.org/wiki/Dextroamphetamine
- Wikipedia — Lisdexamfetamine: https://en.wikipedia.org/wiki/Lisdexamfetamine
- Wikipedia — Amphetamine (PK section): https://en.wikipedia.org/wiki/Amphetamine
- Effect of amphetamines on blood pressure — meta-analysis (PubMed PMID 40152309): https://pubmed.ncbi.nlm.nih.gov/40152309/
- Effect of amphetamines on blood pressure — meta-analysis (PMC11951410): https://pmc.ncbi.nlm.nih.gov/articles/PMC11951410/
- Morozova et al. 2016 — Amphetamine enhances endurance by increasing heat dissipation (PMC5027360): https://pmc.ncbi.nlm.nih.gov/articles/PMC5027360/
- Low doses of caffeine reduce heart rate during submaximal cycle ergometry (PMC2164943): https://pmc.ncbi.nlm.nih.gov/articles/PMC2164943/
- Methylphenidate and its use in sports competitions (ResearchGate): https://www.researchgate.net/publication/288465098_Methylphenidate_and_its_use_in_sports_competitions_in_patients_with_ADHD
- Impact of stimulants on sleep / polysomnography (Tandfonline review): https://www.tandfonline.com/doi/full/10.1080/08039488.2020.1833984
- Amphetamine dose-dependently affects binge intake (PMC6832849): https://pmc.ncbi.nlm.nih.gov/articles/PMC6832849/
- Amphetamine treatment for obesity — mechanism overview (Bolt Pharmacy): https://www.boltpharmacy.co.uk/guide/amphetamine-treatment-for-obesity
- Upholding or Breaking the Law of Superposition in Pharmacokinetics (PMC11351987): https://pmc.ncbi.nlm.nih.gov/articles/PMC11351987/
- Basic Pharmacokinetic Principles and One-Compartment Model Equations: https://pkineticdrugdosing.com/documents/basic-pharmacokinetic-principles-and-one-compartment-model-equations.html
- Hybrid ML/PK approach outperforms MAP Bayesian estimation (PMC8520755): https://pmc.ncbi.nlm.nih.gov/articles/PMC8520755/
- Bayesian estimation of PK parameters (PMC8020855): https://www.ncbi.nlm.nih.gov/pmc/articles/PMC8020855/
- Posologyr open-source Bayesian dose individualization (PMC8879752): https://www.ncbi.nlm.nih.gov/pmc/articles/PMC8879752/
