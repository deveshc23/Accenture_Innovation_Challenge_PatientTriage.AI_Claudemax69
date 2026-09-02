# PatientTriage.ai Console

A single-file, offline-first clinical decision-support console for Emergency
Department (ED) triage. It scores an arriving patient's acuity (ESI 1–5),
explains exactly why in plain language, tracks the patient through the
waiting room with an automatic safe-wait / re-score timer, models what
happens to wait times under surge load, and keeps a tamper-evident,
hash-chained audit log of every recommendation and every clinician decision.

Deployed as a demo instance for **District Hospital, Kanpur Dehat**
(Tier 1, HL7 v2 ADT), running fully on-prem with no data egress.

Current build: **v0.9.4** · rules **r2026.08.14** · model **gbm-mono-3.1**

---

## 1. What this is (and isn't)

- It is a **decision-support tool**: it recommends an ESI level and shows its
  work. It never silently decides for the clinician.
- It **cannot lower** a patient's acuity on its own. Only a licensed
  clinician can downgrade a level, and only with a documented reason code —
  that action is written to the audit chain against their licence number.
- The recommendation engine is **deterministic first, model second**: four
  rule-based layers set a acuity *floor*; a learned model may only make the
  patient *more* urgent than that floor, never less.
- The dataset loaded in the Waiting Room is a **deliberately enriched demo
  cohort** (30 patients, ~10 of them true ESI‑2 against a ~10–19% real-world
  base rate) — built to exercise the hard cases in a short demo, not to
  represent true ED prevalence.

---

## 2. Layout

The app is a single `.html` file — no build step, no server, no external
runtime dependency beyond two Google Fonts. Everything (styles, triage
engine, UI, audit log, simulator) lives in one `<style>` block and one
`<script>` block.

Four workspaces, switchable from the left rail or number keys `1`–`4`:

| # | View | Purpose |
|---|------|---------|
| 1 | **Triage console** | Score one patient, see the reasoning trace, accept or override |
| 2 | **Waiting room** | Live ranked queue with the safe-wait timer and deterioration watch |
| 3 | **Surge model** | Simulate arrival load vs. treatment capacity, ranked vs. first-come-first-served |
| 4 | **Audit & model** | Hash-chained decision log, model performance, ablation, confusion matrix, regulatory notes |

---

## 3. Triage console

### 3.1 Intake

Enter (or load a simulated arrival for) a patient:

- Age, sex
- Free-text presenting complaint — **bilingual**: English, Hindi
  (Devanagari), and Hinglish transliteration are all recognised by the same
  matcher (e.g. "seene mein dard", "छाती में दर्द", and "chest pain" all hit
  the same flag). Negation is handled explicitly — "no chest pain" does
  *not* fire the chest-pain rule.
- Vitals: HR, RR, SpO₂, SBP, Temp, AVPU
- Known history (chips — diabetes, COPD, anticoagulated, pregnant, dialysis,
  immunosuppressed, prior MI, etc.)
- Things observed at the desk (chips — airway compromise, apnoea, massive
  bleed, sudden-onset headache, visible injury, burns to face/neck…)

Press **Score patient** (or `Enter`) to run the engine.

### 3.2 The engine

`triage(patient)` runs four deterministic layers and one learned model, then
combines them with a strict invariant:

```
floor = min(LayerA, LayerB, LayerC, LayerD)      ← deterministic rules only
final = min(floor, model.acuity)                 ← model may only tighten, never loosen
```

ESI convention: **1 = most acute … 5 = least acute.** "Stricter" always
means a smaller number.

| Layer | Name | What it checks |
|---|---|---|
| **A** | Life-saving intervention | AVPU = Unresponsive, airway compromise, apnoea, SpO₂ < 85%, SBP < 70 (adult), massive haemorrhage → instant ESI‑1 |
| **B** | Red-flag patterns | ~24 named clinical patterns (ACS features, stroke/FAST positive, infant fever <3mo, sepsis, haemorrhagic shock, obstetric emergency, respiratory failure, airway burns, etc.), each traceable to a protocol line |
| **C** | Risk modifiers | Context the raw vitals can't see: atypical MI in diabetics/older women, occult geriatric sepsis, immunosuppression, anticoagulation + head injury, dialysis, COPD masking hypoxia, falls ≥65, syncope, road-traffic mechanism |
| **D** | Vital-sign scoring | A NEWS2-style aggregate (EWS), scored against the patient's **own age band** — not a single adult scale |

A **learned model** (a monotonic gradient-boosted classifier, represented
here as its learned scoring surface) computes a risk probability from
band-relative vital deviations plus a handful of text flags. Every feature
is a deviation *outside* the patient's own age band, so the model is
monotone in risk by construction — worse physiology can never score safer.
The model can only pull the final acuity number down toward ESI‑1; it has
no expressible path to raise it above the deterministic floor.

**Confidence & abstention.** A three-part confidence score (data
completeness, inter-layer agreement, distance from the model's decision
boundary) decides whether the engine trusts its own recommendation. Below
0.50 confidence, the patient does not get handed a routine number — the
engine abstains and holds them at ESI‑3 in the clinician's rapid-review
lane. The hold is built from *information completeness*, not the confidence
value alone, specifically so it can never be lifted by the patient getting
sicker (a hold that deterioration could release would be a safety
regression wearing a safety label).

**Age bands** (thresholds are local-protocol configuration, not hardcoded
constants — a deploying hospital owns this table):

| Band | Age | HR | RR | SBP | SpO₂ target | Temp |
|---|---|---|---|---|---|---|
| Infant | <1y | 100–160 | 30–60 | 70–110 | ≥95% | 36.5–37.9 |
| Child | 1–4y | 90–150 | 22–40 | 75–115 | ≥95% | 36.5–37.9 |
| Child | 5–11y | 70–120 | 18–30 | 80–120 | ≥95% | 36.5–37.9 |
| Adult | 12–64y | 60–100 | 12–20 | 100–140 | ≥96% | 36.1–37.9 |
| Geriatric | ≥65y | 60–100 | 12–20 | 110–150 | ≥95% | 36.0–37.7 |

(COPD patients are scored against an SpO₂ target of 88%, not the general
band floor — scoring a chronic hypoxic patient against the adult target
manufactures false alarms.)

### 3.3 Recommendation, trace, and disposition

- **Recommendation** panel: final ESI level, colour (RED / YELLOW / GREEN),
  confidence, and a one-line clinical *advice* string (a test/priority
  suggestion — the engine never names a diagnosis).
- **Why — execution trace**: every rule that actually fired, in the order it
  ran, with the protocol reference it maps to. This is `min() over layers`
  made visible — there is no hidden step between the trace and the number.
- **Disposition**: the clinician either
  - **Accepts** the recommendation as-is, or
  - **Overrides** it — choosing a new final level, a **required reason
    code** (`TRANSIENT_PHYSIOLOGY`, `CLINICAL_JUDGEMENT`,
    `ADDITIONAL_HISTORY`, `RESOURCE_CONSTRAINT`, `DATA_ERROR`), and an
    optional evidence reference. Every override is attributed to the
    clinician's licence (RN‑4471), never to the model.
- Two collapsible reference panels: the **same numbers across five age
  bands** (the calibration thesis, illustrated), and the **vitals-in-band**
  breakdown for the current patient.

---

## 4. Waiting room

The live, auto-advancing queue (1 simulated minute per 2 real seconds,
pausable). Patients are ranked by acuity, then by risk score, then by wait
time — or toggle to strict first-come-first-served for comparison.

### 4.1 Safe-wait timer (the single timer)

Each colour has **one** timer — it is both the maximum safe wait *and* the
re-score trigger. There is no separate periodic re-score clock.

| Level | Base safe wait |
|---|---|
| RED | 20 minutes |
| YELLOW | 2 hours |
| GREEN | 5 hours |

When a patient's timer lapses, the nurse retakes vitals and re-scores them,
right there:

- If the vitals still support the same acuity band (or only shift inside
  it), that's logged as a routine **re-check** (`REMEASURE`) — no
  interruption.
- If the re-score crosses a colour band, or the EWS jumps by 3 or more,
  that's a **deterioration** — the patient is escalated, re-ranked, flagged
  in the UI, and a toast fires (`RETRIAGE_ESCALATION`).
- **If the patient is then admitted** (removed from the queue), nothing
  further happens for them — that's the intended outcome.
- **If the patient is *not* admitted** and stays in the room after a noted
  deterioration, that's logged separately (`DETERIORATION_NOTED`) and their
  **next** safe-wait window is shortened — the room is telling itself to
  look at that patient again sooner:

  | Level | 1st note | 2nd note | 3rd+ note |
  |---|---|---|---|
  | RED | 20m → **15m** | holds at 15m | 15m × 0.9ⁿ (compounding 10% off each further note) |
  | YELLOW | 2h → **1.5h** | holds at 1.5h | 1.5h × 0.9ⁿ |
  | GREEN | 5h → **4h** | holds at 4h | 4h × 0.9ⁿ |

  A patient with two prior notes is tagged `watch · N×` in the queue.

There is also an independent **scripted deterioration** trigger built into
the demo cohort (a couple of patients are seeded to worsen at a specific
elapsed-wait mark), used to guarantee the demo shows a re-triage event
without waiting for the random vitals drift to produce one.

### 4.2 Deterioration watch panel

Explains the two mandatory triggers in plain language:

- **T1** — safe-wait window elapsed (vitals retaken and re-scored at that
  moment)
- **T2** — re-scored, not admitted, deterioration noted (logged, and the
  next window for that patient is shortened)

### 4.3 Safe-wait policy panel

Shows the live per-colour base window and the shortening rule in one place.

### 4.4 Queue tiles

Live counts: waiting, RED/YELLOW/GREEN in room, breaching (past their
current safe-wait window), abstained (in the clinician's rapid lane), and
longest current wait.

---

## 5. Surge model

A standalone what-if simulator (does not affect the live waiting room):

- **Arrivals/hour** (10–45, drag to see normal → elevated → surge)
- **Treatment stations** (2–6, fixed capacity)
- **Arrival stream seed** (kept identical across both compared policies, so
  the comparison is apples-to-apples)

Run it to compare, over a simulated 4-hour window at 3 stations / 12 min per
patient:

- Arrivals, peak queue depth, median and 95th-percentile wait for
  high-acuity patients, safe-wait breach rate, and mean wait — **ranked
  order vs. strict first-come-first-served**, same arrival stream.
- A collapsible panel spells out **what changes under load and what does
  not**: no threshold, rule, or floor ever relaxes under surge — the design
  premise is that surge is exactly when unobserved deterioration is most
  dangerous, so relaxing a safety threshold under load would invert the
  entire point of the system.

---

## 6. Audit & model governance

### 6.1 Decision log

A **hash-chained, tamper-evident** append-only log. Every `SCORE`,
`ACCEPT`, `OVERRIDE`, `REMEASURE`, `RETRIAGE_ESCALATION`,
`SAFE_WAIT_BREACH`, and `DETERIORATION_NOTED` event is written as a record
containing:

- the model and ruleset versions active at the time,
- a SHA-256 hash of the inputs,
- the previous record's hash (`prev_hash`),
- its own hash = `SHA256(prev_hash + JSON(record))`.

Filter by **All / Decisions / Escalations**, expand/collapse every visible
row, and:

- **Verify chain** — recomputes every hash from the genesis record forward
  and confirms nothing has been altered.
- **Tamper test** — deliberately edits one record in place *without*
  rewriting the chain, then re-verifies, to demonstrate that verification
  fails at that record and at every record after it. This is what
  tamper-evidence means in practice, made visible rather than asserted.

### 6.2 Model & governance reference (tabs)

- **Ablation** — high-acuity miss rate for the full engine vs. risk
  modifiers removed vs. adult-calibrated-vitals-only vs. a "hidden-sick"
  subgroup, measured on a 500-encounter held-out cohort. Shows which layer
  is load-bearing, not just that the ensemble works.
- **Model** — held-out performance (AUROC, AUPRC, sensitivity at 90%
  specificity, Brier score), audited by age subgroup (paediatric / adult /
  geriatric) **before** deployment. Includes a live **monotonicity check**
  button: perturbs every loaded patient further from their own age band
  across 8 features and asserts the risk score and recommended level never
  move toward safety — enforced as a property of the model class, not
  hoped for and tested after the fact.
- **Confusion** — a 3×3 (ESI 1–2 / 3 / 4–5) confusion matrix over 500
  encounters, with under-triage (the errors that can kill) and over-triage
  called out separately, plus a worked reconciliation of why the headline
  over-triage number differs from the collapsed-band matrix.
- **Regulatory** — a plain-language position against DPDP Act 2023 (India),
  CDSCO MDSW guidance, FDA CDS guidance, and EU AI Act Annex III — including
  why on-prem, no-egress deployment matters for each.

---

## 7. Interaction reference

| Shortcut | Action |
|---|---|
| `1` / `2` / `3` / `4` | Switch to Console / Waiting room / Surge / Audit |
| `Enter` (outside the complaint field) | Score the current patient |
| `A` | Accept the current recommendation |
| `O` | Open the override dialog |
| `S` | Toggle surge mode |
| ◐ button | Toggle light / dark theme (also respects OS preference) |

The UI is fully responsive down to narrow viewports (the rail collapses to
a horizontal bar; the queue table drops secondary columns).

---

## 8. Design system

- Instrument-panel treatment: hairline rules, no drop shadows/elevation,
  near-square corners, tabular numerals throughout.
- Saturated colour (red / amber / green) is reserved **exclusively** for
  clinical acuity — never used as decorative chrome.
- The brand violet is chrome-only and never encodes patient state.
- Typefaces: IBM Plex Sans (UI), IBM Plex Sans Condensed (labels), IBM Plex
  Mono (numerics, hashes, timestamps).
- Full light/dark theme support via CSS custom properties, following
  `prefers-color-scheme` by default with a manual override.

---

## 9. Data & privacy posture

- All computation is client-side; the top bar's "on-prem · no egress"
  indicator reflects the deployment model, not a live network check.
- The 30-patient waiting-room cohort and single-patient presets are
  synthetic demonstration data, explicitly disclosed as an enriched sample
  (see the "About this cohort" note in the Waiting room view) — not real
  patient records.
- An optional external `ENGINE_DATA` object (`ED` in the source) can supply
  real measured ablation/model/confusion-matrix numbers from an offline
  `evaluate.py` run; in its absence the console falls back to illustrative
  static figures, which are clearly the same numbers referenced in the UI
  copy.

---

## 10. Running it

No install, no server, no dependencies beyond an internet connection for
the two Google Fonts (the app still functions without them — the font
stack falls back to system fonts).

```
open console_3.html      # or double-click it, or serve the folder statically
```

Everything — engine, UI, state, audit log — lives in this one file and runs
entirely in the browser tab. Reloading the page resets the session (the
audit chain is in-memory only, by design, for a demo instance).

---

## 11. File map

Since this is a single-file app, "file map" means section map — searchable
by the comment banners in `console_3.html`:

```
<style>                          — full design system + component styles
<script>
  SHA-256                        — pure-JS hash implementation (no crypto deps)
  AGE_BANDS / bandFor            — age-banded clinical reference table
  scoreVitals / vitalPoints      — NEWS2-style aggregate scoring
  LEX / textFlags / isNegated    — bilingual free-text matcher with negation
  layerA … layerD                — the four deterministic rule layers
  riskModel                      — monotonic learned-model surrogate
  confidence                     — completeness / agreement / margin → abstain
  triage()                       — combines layers + model under the floor invariant
  1. TRIAGE CONSOLE               — intake, recommendation, trace, disposition
  2. WAITING ROOM                 — queue, safe-wait timer, deterioration watch
  3. SURGE                        — arrival-load simulator
  4. AUDIT                        — hash chain, verify/tamper, governance tabs
  navigation / init               — view routing, keyboard shortcuts, theme
```

---

## 12. Recent changes

- **Safe-wait / re-score merge.** The waiting room previously ran two
  independent clocks per patient — a safe-wait breach threshold and a
  separate periodic re-score interval (which also varied under surge mode).
  These are now **one timer**: RED 20m / YELLOW 2h / GREEN 5h, doubling as
  the re-score trigger. A patient who is re-scored, not admitted, and found
  to have risen in priority now has that deterioration explicitly logged
  (`DETERIORATION_NOTED`), and their next safe-wait window shortens on a
  defined schedule (see §4.1).
