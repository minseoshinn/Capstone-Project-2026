# Energy-Based Workload Quantification for Last-Mile Delivery

A system that measures how physically hard a courier's day actually was, from the
phone already in their pocket, and uses that measurement to reorder deliveries so
fatigue does not concentrate.

Capstone Design Project, Department of Industrial and Management Engineering,
Hankuk University of Foreign Studies. January – June 2026. Advisor: Prof. Bernardo
Nugroho Yahya. Four-person team; my contribution was the integrated activity and
energy model and the client stack (courier app, dashboard, heatmap) with its server
integration.

The application interface is in Korean, since it was built for and tested with a
Korean courier. Code, comments, and this document are in English.

---

## The problem

Logistics systems assign work by parcel count and distance. Neither reflects physical
load. The same 100 parcels are a different day depending on floor count, elevator
availability, parcel weight, and heat. Field interviews with a Coupang Quick-Flex
courier and a CJ Logistics sub-terminal manager established the operative constraint:
territory and daily volume are fixed by address block, so redistributing work is not
available. **Delivery order is the courier's own discretion, and it is the lever.**

## What the system does

1. Classifies courier activity from smartphone IMU data (waiting, elevator, stairs up,
   stairs down, level walking, running, cycling).
2. Converts each activity window into metabolic energy, personalized on sex, age,
   height, and weight.
3. Accumulates energy into a Workload Stress Index and issues rest recommendations
   against physiological and heat-exposure limits.
4. Recommends a delivery order that minimizes peak fatigue rather than total distance.

---

## Method

### Energy model

```
E(J) = MET × C_load × W × Δt × 4184
C_load = 1 + 0.5·L/W        (effective mass = W + 0.5·L)
```

MET values from the 2011 Compendium of Physical Activities (Ainsworth et al.); load
coefficient from Soule & Goldman (1969), reflecting that a load carried in the hands
costs more than the same mass on the torso. `C_load` is not applied to stair-ascent or
cart segments, where the Compendium MET already includes the load.

Δt is the 2-second sensor interval. 4184 J/kcal (NIST/CODATA thermochemical calorie).

### Workload Stress Index

```
WSI = E_total / E_limit
E_limit = 165 W/m² × BSA × T
BSA = 0.007184 · W^0.425 · H^0.725      (DuBois & DuBois, 1916)
```

165 W/m² is the sustainable average work metabolic rate from the ISO 8996 metabolic
rate classification. Bands: < 0.8 normal, 0.8–1.0 caution, 1.0–1.2 warning, ≥ 1.2
danger. Alerts fire only on upward band transitions, and are suppressed for the first
15 minutes, because T is small early in a shift and a single heavy parcel would
otherwise push WSI past 1.

### Activity classifier

XGBoost, 7 classes, 53 features spanning time domain, frequency domain, and vertical
drift. Trained on HHAR plus 1,480 windows self-collected with the Phyphox app under
matched with-box and without-box conditions.

Separating stair ascent from level walking was the hard part. The features that
resolved it are integrated z-axis acceleration (net positive drift on ascent, negative
on descent), skewness and kurtosis of z magnitude, positive/negative z peak asymmetry,
z-axis dominant frequency as a vertical cadence proxy, and spectral entropy. HHAR
reports raw acceleration including gravity while Phyphox reports linear acceleration
with gravity removed, so a gravity normalization step aligns the two scales.

**0.904 ± 0.057 accuracy** under leave-one-subject-out over 9 subjects (min 0.789,
max 0.963), which estimates generalization to a courier the model has never seen.

### Metabolic rate regressor

Random Forest on Empatica E4 wrist accelerometry from the WEEE dataset, labeled
against indirect-calorimetry VO₂ as MET = VO₂/3.5, validated leave-one-participant-out
over 16 participants. A domain router sends static windows to a population mean rather
than the regressor, because the wrist signal carries no metabolic information at rest
and regressing on it only overfits.

### Heat exposure

WBGT from live OpenWeather conditions using Stull (2011) for the natural wet-bulb
term, with solar correction to globe temperature. Work/rest recommendations from the
NIOSH tables, gated on the mean MET intensity of the remaining route. Below the heat
threshold the system falls back to a time-and-load rule: rest inserted at whichever
comes first, a single-bout limit (90 min moderate, 60 min heavy) or a cumulative
metabolic limit (350 or 240 MET·min).

### Delivery sequencing

Total energy is close to invariant when volume and territory are fixed, so the
objective is peak fatigue, not the sum. Three heuristics are implemented and compared:

| Heuristic | Complexity | Idea |
|---|---|---|
| Ascending load | O(n log n) | high-intensity buildings while fresh |
| Nearest neighbour | O(n²) | Haversine, within parking subgroups |
| High/low interleave | O(n log n) | alternate heavy and light to spread recovery |

Scored under Visentin's (2018) fatigue accumulation and recovery model,
`actualE = baseE · (1 + α · fatigue)`, with 65% decay of prior fatigue at each step and
a 30% ceiling on the penalty. Buildings sharing a parking location are grouped so
reordering does not destroy route efficiency.

Cart and split-carry recommendations follow the NIOSH lifting equation
(RWL ≈ 23 kg, Waters et al. 1994), Snook & Ciriello (1991) maximum acceptable carry
weights, stair-carry limits (Mital et al. 1993), and MOEL Notice 2020-12.

---

## Field data

Four days shadowing a Coupang Quick-Flex courier across a university campus and
adjacent apartment and low-rise residential blocks. **141 delivery records and 35
building records** (floor count, elevator presence, GPS coordinates, parcel count and
weight) collected as a full enumeration rather than a sample.

Predicted energy was then computed over **431 deliveries across four delivery dates**,
totalling 6,877 kcal, mean 16.0 kcal per delivery against a median of 10.5, mean MET
3.53 (range 2.07–7.12).

The analysis separates two load regimes that parcel count cannot distinguish:

| | Deliveries | Mean energy | Mean MET | Mean floors |
|---|---|---|---|---|
| Stairs | 376 | 14.8 kcal | 3.65 | 1.2 |
| Elevator | 55 | 24.2 kcal | 2.71 | 6.2 |

Stair deliveries carry higher instantaneous intensity; elevator deliveries cost more
per visit, because those buildings are taller and accumulate cart movement, round
trips, and long indoor carries. Stair cost scales steeply with height: 11.2 kcal at
one floor, 36.0 at two to three, 67.7 at four to five.

---

## System

```
FastAPI inference server (15 endpoints)
  ├── courier PWA          real-time WSI, rest schedule, delivery list,
  │                        grouped multi-parcel visits, photo confirmation
  ├── admin dashboard      per-courier daily/weekly/monthly energy and WSI,
  │                        building load ranking, alert history,
  │                        predicted vs measured comparison
  └── energy heatmap       Naver Maps: per-building energy bubbles, route,
                           heat overlay, measured footprint
```

Local storage plus on-device fallback inference, so workload accrual continues through
connectivity loss. Repeated parcels to the same building or adjacent complex are
grouped into a single visit, to avoid double-counting travel when summing per-parcel
energy.

Field data is not committed to this repository. Courier identifiers are pseudonymized,
and only body mass, height, age, sex, delivery records, and IMU signals were collected
— no heart rate, blood pressure, or other physiological or sensitive health data.

---

## Limitations

Stated plainly, because they define what this system does not yet establish.

**The routing improvement is self-scored.** Improvement is currently measured by the
same cost function the routes were optimized against. Independent validation requires
a within-subject crossover on identical volume, with distance, elapsed time, vertical
ascent, TRIMP, and peak %HRR as outcomes, paired Wilcoxon tests, and effect sizes.
Designed, not yet run.

**No physiological ground truth for the energy estimate.** Predicted and measured
energy diverge systematically in both directions: measured exceeds predicted when
sensors accumulate during waiting or the classifier misfires; predicted exceeds
measured on stair carries, where the formula applies a load-inclusive MET of 8–12
while the wrist signal registers only 3–5. Resolving this needs HR-derived VO₂ or a
portable indirect calorimeter, then Bland–Altman agreement analysis and per-activity
calibration coefficients.

**Single courier, single region.** One courier identifier, one campus and its adjacent
blocks. The system logic generalizes through personal parameters, but the validation
does not.

**Phone placement.** Couriers carry phones in hand, pocket, or bag, and the classifier
was validated on a limited set of placements.

---

## Sources

Ainsworth et al. (2011) Compendium of Physical Activities · Soule & Goldman (1969)
load coefficient · DuBois & DuBois (1916) body surface area · ISO 8996 metabolic rate
classification · Stull (2011) wet-bulb approximation · ISO 7243 WBGT · NIOSH work/rest
tables (Pub. 2017-127) and heat exposure criteria (2016) · KOSHA GUIDE W-12-2017 and
M-46-2012 · Waters et al. (1994) NIOSH lifting equation · Snook & Ciriello (1991) ·
Mital et al. (1993) · Visentin (2018) fatigue accumulation and recovery · Harris–Benedict
(revised) BMR · Gashi et al. (2022) WEEE dataset · Stisen et al. (2015) HHAR

## Contact

Minseo Shin — hh7352913@gmail.com
Department of Industrial and Management Engineering, Hankuk University of Foreign Studies
