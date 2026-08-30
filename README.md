# PatientTriage.ai

**Accenture Innovation Challenge 2026 · Round 2 · Problem Track 2**
Team Claudemax69 — IIT Kanpur · Devesh Choudhury, Chitransh Gangwar, Atharva Katiyar

A triage decision-support assistant for emergency departments. It recommends an
acuity level and a queue position, explains every recommendation, and never
takes the decision away from the clinician.

The signature behaviour is catching the patient whose vital signs are all
normal but who is nonetheless critically unwell.

> **Decision support only — not a medical device.** Every patient record in this
> repository is synthetic. No real patient data has been used at any stage.

---

## Headline results

Measured on 500 synthetic encounters. Ground truth is assigned at generation
time and is independent of the engine.

| Measure | This engine | Adult-calibrated vitals baseline |
|---|---|---|
| Under-triage | **4.0%** (ACS-COT ceiling is 5%) | 9.6% |
| Over-triage | **32.0%** (accepted band 25–35%) | 33.2% |
| High-acuity patients missed | **7.5%** (6 of 80) | **45.0%** (36 of 80) |

At essentially the same over-triage cost, six times fewer misses.

**Hidden-sick subgroup** — patients who were genuinely high-acuity with every
vital sign inside the band for their age (n = 21): this engine catches 16, the
baseline catches 7, and the engine without its risk-modifier layer catches 6.

**Learning layer** — AUROC 0.943, AUPRC 0.838, sensitivity 0.777 at 90%
specificity, Brier 0.058 on a 1,200-encounter held-out set. Monotonicity audit:
**0 violations** across 8 constrained features × 1,200 rows.

---

## Quick start

```bash
git clone <this-repo> && cd patienttriage-ai
pip install -r requirements.txt      # only needed for learn.py and the tests

python -m pytest tests -q            # 20 acceptance tests
python build_app.py                  # regenerates the nurse cockpit
python build_console.py              # regenerates the clinical console
open app/console.html                # or double-click it
```

The prototype is a single self-contained HTML file. No server, no build step,
no network. That is deliberate: it mirrors the Tier 0 deployment claim, where a
rural department runs the whole system offline on one tablet.

---

## 1 · Implementation approach

The core design decision is that **the safety guarantee lives in deterministic
code, and the model is constrained by it** — not the other way round.

Four rule layers each inspect the patient and return the acuity level they
demand. The strictest answer becomes an **acuity floor**. Everything above the
floor may raise urgency; nothing may lower it. Acuity is ESI 1–5 where 1 is most
urgent, so the combination is a `min()` and the invariant holds by construction:

```python
floor  = min(f.floor for f in fired_rules)      # engine/engine.py
acuity = min(floor, resource_estimate)
final  = min(acuity, model_suggestion)          # the model can only lower
                                                # the number, i.e. raise urgency
```

Three consequences follow, and they are the reason for the architecture:

1. **The catch does not depend on the model.** The risk-modifier layer — which
   escalates on age, diabetes, anticoagulation and pregnancy rather than on
   symptoms — is plain Python. It works with the network down.
2. **Adding information can only make the system more cautious.** Provable by
   inspection, so it holds on data we have never seen.
3. **Nothing can silently downgrade a patient.** There is no code path from a
   higher acuity to a lower one. Only an authenticated clinician can do that,
   and only with a reason code and re-measured evidence attached.

### Why hybrid rather than pure rules or a pure model

Rules carry the floor because they are auditable, versionable and defensible to
a regulator. The model carries language and ranking because hand-written
thresholds cannot parse a free-text complaint and cannot order forty patients
inside one colour band. Neither component is trusted with the other's job.

### The label decision

The model is trained on **outcomes** — ICU transfer, admission, death, or
intervention within 24 hours — and never on the acuity a nurse assigned.
Training on assigned acuity teaches a model to reproduce the existing miss:
case P01 was assigned low acuity by a competent nurse, and a model fitted to
that label would score well for repeating her.

---

## 2 · Solution architecture

```
              ┌──────────────────────────────────────┐
              │  DETERMINISTIC — no network call     │
  patient ───►│  L1  immediate life-saving check     │
              │  L2  red-flag symptom patterns       │
              │  L3  risk modifiers (who they are)   │──┐
              │  L5  vital-sign scoring, 5 age bands │  │
              └──────────────────────────────────────┘  │
                                                        ▼
                                          ┌─────────────────────────┐
                                          │  ACUITY FLOOR = min()   │
                                          └────────────┬────────────┘
                                                       │
                        MAY RAISE · MAY NEVER LOWER    ▼
                    ┌───────────────────────────────────────────┐
                    │  learned risk model (monotonic GBM)       │
                    │  language layer (free text, Hinglish)     │
                    └────────────────────┬──────────────────────┘
                                         ▼
                    ┌───────────────────────────────────────────┐
                    │  RED / YELLOW / GREEN + why panel         │
                    │  ...or "I don't know" — abstain and ask    │
                    └────────────────────┬──────────────────────┘
                                         ▼
                    ┌───────────────────────────────────────────┐
                    │  clinician accepts or overrides           │
                    │  every decision writes an audit record    │
                    └───────────────────────────────────────────┘

  RE-TRIAGE LOOP — every waiting patient is re-scored on a timer.
  Two triggers: worsening re-measured vitals, or a breached safe-wait window.
```

### Module map

| File | Responsibility |
|---|---|
| `engine/engine.py` | Age bands, the five layers, floor combination, confidence, abstention. Pure functions — no UI, no network, no I/O. |
| `engine/cohort.py` | Synthetic generator. A latent true acuity is chosen **first**; observations are then generated from it, so ground truth is independent of the engine. |
| `engine/seed_cohort.py` | The 20 named demonstration cases (P01–P20) referenced throughout the proposal. |
| `engine/audit.py` | Append-only, hash-chained audit log. Field set driven by the DPDP Act 2023. |
| `engine/evaluate.py` | Confusion matrix, baselines, ablations, subgroup analysis, discrete-event queue simulation. Writes `results.json`. |
| `engine/learn.py` | Gradient-boosted model with monotonic constraints; held-out evaluation and the monotonicity audit. Writes `ml_results.json`. |
| `tests/test_acceptance.py` | One named test per requirement in the brief. |
| `build_app.py` | Runs the engine and emits the nurse cockpit. |
| `app/index.html` | The nurse cockpit. Open it directly. |
| `build_console.py` | Runs the engine over the seed cohort — including an age-band probe and the scripted queue events — and injects the result into the console shell. |
| `app/console_shell.html` | Markup, stylesheet and rendering code for the console. Carries an injection point; not meant to be opened directly. |
| `app/console.html` | The clinical console. Four surfaces over one engine. Open it directly. |

### Age stratification

Five calibrated bands — infant, toddler, child, adult, geriatric — because a
single adult-calibrated model is a silent safety risk. Paediatric hypotension
uses `70 + 2 × age` rather than a fixed threshold. Removing the bands and
forcing every patient through the adult table raises paediatric **over**-triage
from 27.0% to 68.0%, because a child's normal heart and respiratory rates sit
above the adult range.

### Uncertainty and abstention

Confidence is computed from data completeness weighted by relevance to the
rules that actually fired, plus rule agreement and presentation ambiguity.
Below 0.50 the engine abstains: the patient does not silently join the queue,
and the unresolved differential is named — *"consider DKA — check glucose and
ketones."* It recommends a test, never a diagnosis.

**Low confidence never lowers acuity.** Uncertainty resolves upward.

---

## 3 · Dependencies

The engine and the prototype run on the **Python standard library alone**.
There is no framework, no database and no cloud service in the clinical path.

| Package | Needed for | Version |
|---|---|---|
| `scikit-learn` | `learn.py` — the gradient-boosted model | ≥ 1.4 (monotonic constraints) |
| `numpy` | `learn.py` | ≥ 1.26 |
| `pytest` | the acceptance suite | ≥ 8.0 |

The prototype itself has **zero** JavaScript dependencies — no React, no CDN,
no build tooling. `app/index.html` is one file with the engine's output
embedded, so it opens offline by double-clicking.

Tested on Python 3.12. No GPU required.

---

## 4 · Execution instructions

### Run the acceptance tests

```bash
python -m pytest tests -q
```

Each test is named for a requirement in the brief's Minimum Prototype
Expectations. A failing build is a non-compliant build.

| Test | Requirement it discharges |
|---|---|
| `test_a1_scores_at_least_twenty_records` | Scoring on 15–20+ simulated records |
| `test_a2_ambiguous_presentation_is_low_confidence` | At least one ambiguous presentation |
| `test_a3_paediatric_and_geriatric_present` | Paediatric and geriatric cases |
| `test_a3_age_bands_change_the_answer` | Age bands are load-bearing, not decorative |
| `test_a4_zero_history_patient_is_scored_safely` | Zero-history first-time patient |
| `test_a4_missing_history_never_lowers_acuity` | Missing data is unresolved risk |
| `test_a5_surge_does_not_relax_any_threshold` | Behaviour under 3× surge |
| `test_a6_every_score_carries_a_confidence_figure` | No score without a confidence indicator |
| `test_a7_override_up_is_captured` | Clinician override captured and logged |
| `test_a7_override_down_requires_evidence` | A downgrade is refused without evidence |
| `test_a7_log_is_hash_chained` | Tamper-evident audit trail |
| `test_a9_escalation_bias_sits_inside_the_acs_cot_bands` | Escalation bias demonstrated, not asserted |

### Reproduce every figure in the proposal

```bash
cd engine
python evaluate.py     # → results.json : matrix, baselines, ablations, queue sim
python learn.py        # → ml_results.json : model metrics, monotonicity audit
```

Both are seeded, so runs are reproducible.

### Run the clinical console

```bash
python build_console.py
open app/console.html
```

Four surfaces over the same engine: **triage console** (intake, execution
trace, disposition), **waiting room** (live ranking, safe-wait breaches, the
re-triage timer), **surge model** (discrete-event simulation, ranked against
first-come over an identical arrival stream), and **audit & model governance**
(the hash-chained log, ablations, held-out metrics, confusion matrix).

**Where its numbers come from.** Every figure shown for a seed patient is
`triage()`'s own output, computed by `build_console.py` and embedded at build
time — acuity, zone, confidence, the rules that fired, NEWS2, and the age-band
probe. The page does not re-derive them. Edit any input and there is no measured
record to show, so a JavaScript mirror of the engine takes over and the card
says so: the green **engine output** badge becomes an amber **what-if** badge.
The ablation, model and confusion-matrix panels read `results.json` and
`ml_results.json` directly.

That split is deliberate. A demo that silently recomputes its own headline
numbers in the browser is not evidence of anything.

**What to do in it:** select **P01** — every vital normal, RED on context
alone, three risk modifiers firing and the other three layers silent. Look at
the age-band panel beside it: the same observations scored in five bands, which
is the calibration claim made checkable. Select **P14** — zero history, the
engine abstains and names the test that would resolve it. Open the waiting room
and let it run until **P19** deteriorates and re-ranks herself. Press **Tamper
test** on the audit view: it edits one record in place and verification then
fails at that record and every record after it.

### Run the nurse cockpit

```bash
python build_app.py
open app/index.html
```

**What to do in it:** the queue advances on a timer. Select **P01** — every
vital normal, escalated to RED on context alone. Select **P14** — zero history,
confidence 0.38, the engine abstains and names the test that would resolve it.
Wait for **P19** to deteriorate and auto-escalate. Watch **P20** breach its
safe-wait window with nothing wrong. Press **Override down** on any patient and
note that it demands evidence. Press **Simulate 3× surge** and watch breach
counts climb while no threshold moves.

---

## Limitations, stated plainly

- **Ground truth is constructed, not clinical.** These figures show the
  architecture behaves as designed. They are not evidence that it helps
  patients.
- **The model is trained on synthetic data.** It proves the pipeline end to end
  — outcome labels, constrained training, the right evaluation metrics. Real
  training requires MIMIC-IV-ED and local recalibration, which is a deployment
  gate rather than a nice-to-have.
- **Missing history costs detection.** High-acuity miss rate is 2.8% for
  patients with a record and 11.4% for those without. We report the penalty
  rather than absorb it into a confident-looking score.
- **Queue ranking does not create capacity.** Mean wait across all patients is
  unchanged. The sickest move forward and someone less sick waits longer.
- **No prospective validation.** Phase 0 of the roadmap is silent retrospective
  scoring of a partner department's own historical records. If it does not beat
  their human baseline, the correct response is to stop, not to re-tune.

## Grounded in

ESI Handbook v5 (ENA) · AIIMS Triage Protocol, *J Emerg Trauma Shock* 2020, and
its prospective validation across 15,505 patients · NEWS2 with PALS/APLS
paediatric ranges · ACS-COT under/over-triage tolerances · Sax et al., *JAMA
Network Open* 2023 (mistriage across 5,315,176 encounters) · MIMIC-IV-ED,
PhysioNet · published external validation of the Epic Sepsis Model · FDA
Clinical Decision Support Software final guidance, January 2026 · Digital
Personal Data Protection Act 2023 (India).

## Licence

MIT — see [LICENSE](LICENSE).
