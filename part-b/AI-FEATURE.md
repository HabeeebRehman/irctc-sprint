# AI Feature Specification: PNR Confirmation Probability + Plain-Language Explanation

This AI feature is the prediction layer that powers the redesigned PNR Status page in `SPECS.md` Feature Spec 5. The UI surface is specified there; this document covers the model, data, output presentation, fallback, and risks.

---

## Problem It Solves

This feature directly addresses **Problem 5 in `part-a/PROBLEMS.md`** — *PNR Status Page is Buried and Ambiguous About Chart Preparation*.

Specifically, it answers the question that every PNR-checking user actually has but that IRCTC's current page does not answer: *"Will my waitlisted ticket get a confirmed seat?"*

Today, IRCTC shows raw codes like `WL 12 GNWL` and forces users to either decode railway jargon themselves or migrate to third-party sites (ConfirmTKT, RailYatri) that do precisely this prediction using IRCTC's own publicly-checkable PNR data. IRCTC has effectively outsourced the most-asked product question to private third parties. This feature brings the prediction in-house, on the official surface, in plain language, with transparent confidence bounds.

The feature also partially addresses **Problem 1** (Tatkal anxiety) — a user who sees a clear, honest 30% confirmation probability for their post-Tatkal waitlist ticket can make a better decision faster than one staring at `WL 45 PQWL` and Googling acronyms.

---

## Proposed Feature — User Perspective

A user enters their 10-digit PNR. The result page shows, at the top:

> **Likely to confirm: 70%**
> *Chart prepares in 4 hours · 14:30 today*
>
> [============70%============]    (green progress bar)

Below the bar, a small (i) opens a modal that says, in plain language:

> *Based on 90 days of historical data for train 12951 NDLS → BCT in 3A. Of 124 similar bookings at waitlist position 12 in this period, 87 confirmed, 12 cleared to RAC, and 25 did not confirm. Predictions are estimates, not guarantees.*

Per-passenger, a row shows:

> *Passenger 1 (R., 67y) · Current: WL 12 · Plain: Waitlist 12 (General Quota) · Likely to confirm: 70%*

If probability is high (≥65%), the bar is green. Medium (35–65%) is amber. Low (<35%) is red and accompanied by an explicit prose cue: *"Likely will not confirm. Consider an alternate train."* Plus a one-tap link to alternate trains on the same route.

If the chart has already been prepared, no probability is shown — the page shows the final allocation.

If the model has insufficient data for this train/route/class combination, no probability is shown — only the plain-language translation, with a clear note: *"Confirmation probability not available for this train."*

---

## Model or API Choice

**Primary model: Gradient-boosted decision tree classifier (XGBoost or LightGBM).**

This is deliberately a "classical ML" choice rather than a deep learning or LLM choice, because:

1. **The problem is structured and tabular.** Inputs are categorical and numerical — train number, route, class, quota type, waitlist position, days-to-departure, day-of-week, season, historical clearance rates. XGBoost is the established best-in-class for this exact shape of problem.
2. **Calibration matters more than raw accuracy.** When we tell a user "70% likely to confirm," that number must mean something — across many such predictions, the actual confirmation rate must be close to 70%. Gradient-boosted trees calibrate well with isotonic regression or Platt scaling. Deep learning models and LLMs are notoriously poorly calibrated for probability outputs.
3. **Inference is fast and cheap.** Sub-10ms per prediction on commodity hardware. At 36 lakh PNR checks/day this matters — a 50ms-per-prediction LLM is operationally and economically infeasible at IRCTC scale.
4. **Feature importance is interpretable.** When the explanation modal says "based on historical clearance for this train at this position," that traceability is real, not post-hoc — XGBoost gives us per-prediction feature attributions via SHAP values.
5. **No data leaves IRCTC's premises.** Government data localisation rules effectively rule out hosted LLM APIs (OpenAI, Anthropic, Vertex) for the prediction itself. A self-hosted XGBoost model fits cleanly inside IRCTC infrastructure.

**Why not these alternatives:**
- *Deep learning (TabNet, FT-Transformer)*: marginal accuracy gains for this problem; significantly worse calibration and interpretability; harder to deploy and monitor at IRCTC scale.
- *LLM (GPT-4, Claude, Llama)*: catastrophically wrong tool. LLMs cannot reliably produce calibrated probabilities; they hallucinate confidence; they're 1000x more expensive to run; and the fundamental task (binary classification on tabular data) doesn't need language understanding.
- *Linear model (logistic regression)*: too simple. The interactions between train, position, days-to-departure, and seasonality are non-linear and interaction-heavy. XGBoost captures these natively.

**Secondary model — for the plain-language explanation only**: a small fine-tuned T5 or a templated NLG approach (preferred for v1). The "explanation" text — *"Based on 90 days of historical data, 87 of 124 similar bookings confirmed"* — is a fill-in-the-blank template populated from the XGBoost prediction's metadata. **No generative model is needed for v1.** Templates with i18n strings are sufficient and safer (no hallucination risk).

If, later, we want richer free-form explanations or multi-language nuance, a small fine-tuned T5 (220M parameters) running on-prem can replace templates. Not for v1.

---

## Training or Input Data

**Required input data per prediction (at inference time):**

| Feature | Source | Available today? |
|---|---|---|
| Train number | PNR record | Yes |
| Route (From → To stations) | PNR record | Yes |
| Class (1A / 2A / 3A / SL / 2S) | PNR record | Yes |
| Quota type (GN / TQ / PT / LD / SS / HP / DF) | PNR record | Yes |
| Current waitlist position | PNR record | Yes |
| Booking timestamp | PNR record | Yes |
| Journey date | PNR record | Yes |
| Days-to-departure (computed) | Derived | Yes |
| Day-of-week of journey | Derived | Yes |
| Month / season indicator | Derived | Yes |
| Festival / holiday flag | Calendar lookup | Easy to add |

**Required training data (historical):**

| Feature | Source | Available today? |
|---|---|---|
| Historical PNRs (anonymised) at booking time | CRIS / IRCTC archive | **Needs negotiation** |
| Final outcome of each PNR (confirmed / RAC / not confirmed) | CRIS chart-prepared records | **Needs negotiation** |
| At least 365 days of history; ideally 3 years | CRIS | **Needs negotiation** |

**The data dependency is the largest risk.** IRCTC and CRIS have this data — in fact ConfirmTKT and RailYatri have built businesses on accessible-but-not-easy-to-export versions of it. The model's existence is gated entirely on whether CRIS will expose anonymised historical PNR outcome data to an internal IRCTC ML team.

**Data volume estimate:** ~12 lakh bookings/day × 365 days × ~3 years = ~130 crore records. Of these, perhaps 30% are waitlisted at booking time (the prediction-relevant set) = ~40 crore training examples. This is comfortably enough for a robust XGBoost model — we are not data-limited if the data is accessible.

**Anonymisation requirements:**
- No passenger names, ages, or IDs in training data.
- PNRs themselves should be hashed before training (we don't need the actual PNR value, just the booking metadata).
- Booking source (web / app / agent / counter) can be retained as a feature.

**Retraining cadence:**
- Initial training on 365 days of data.
- Weekly retraining with rolling window (drop oldest week, add newest week).
- Monthly evaluation against held-out time-based test set (predict last week, compare to actuals).

---

## How Output Is Shown to the User

The full UI is specified in `SPECS.md` Feature Spec 5 and its Figma brief. Critical summary:

**Headline (top of page):**

```
Likely to confirm: 70%
Chart prepares in 4 hours · 14:30 today
[============70%============]   ← green probability bar
```

**Per-passenger (in the passenger table):**

```
Passenger 1 (R., 67y)
Current: WL 12  ·  Plain: Waitlist 12 (General Quota)
[======70%======]                  ← per-passenger bar
```

**Explanation modal (opens on tap of the (i) icon):**

```
Based on 90 days of historical data for train 12951
NDLS → BCT in 3A.

Of 124 similar bookings at waitlist position 12:
  ✓ 87 confirmed
  ◐ 12 cleared to RAC
  ✗ 25 did not confirm

Predictions are estimates, not guarantees.
```

**Colour and threshold logic:**
- **High (≥65%)**: green bar, headline reads *"Likely to confirm: 70%"*.
- **Medium (35–65%)**: amber bar, headline reads *"Uncertain — could go either way: 50%"*. Add a soft prompt: *"Watch this PNR — chart prepares in 4 hours."*
- **Low (<35%)**: red bar, headline reads *"Likely will not confirm: 25%"*. Strong prompt: *"Consider booking an alternate train."* with one-tap pivot.

**What the user is never shown:**
- A bare "30%" with no context.
- A confidence like "99%" — the model is calibrated; spurious near-certainty would be a calibration bug.
- A prediction for a chart-already-prepared PNR (the answer is already final).
- A prediction for a booking with insufficient training data — fallback applies.

---

## Confidence Threshold and Fallback

**Confidence threshold for showing the prediction:**

The model emits two outputs per prediction: the predicted probability (calibrated) and a *coverage* score representing how well-supported the prediction is by training data (essentially: how many similar bookings are in the training set within a learned similarity threshold).

| Coverage score | UI behaviour |
|---|---|
| High (≥150 similar historical bookings) | Show probability + bar + explanation modal with sample size. |
| Medium (50–149 similar bookings) | Show probability + bar + a small caveat: *"Based on limited data — confidence is lower."* |
| Low (<50 similar bookings) | **Hide the probability entirely.** Show only the plain-language code translation. Note: *"Confirmation probability not available for this train at this position."* |

*Threshold raised from the original (20 / 100) after peer review — see SPECS.md Peer Review Update 3. The original threshold would have shown point estimates from sample sizes too small to be calibrated, even with a "limited data" caveat. Honest fallback is preferable to spurious confidence.*

**Fallback when the prediction service is unavailable** (model server down, CRIS data feed delayed, network issue between IRCTC and the prediction service):

- The PNR Status page renders without the headline probability block.
- The plain-language code translation still works (it's not model-dependent).
- The chart-preparation timeline still works.
- A small muted banner is shown: *"Confirmation prediction temporarily unavailable. Status shown is current."*
- This degraded experience is **still a vast improvement over today's PNR page.** The whole feature degrades gracefully.

**Fallback when the chart is already prepared:**
- No prediction shown (there's nothing to predict — the answer is final).
- Final allocation is shown (`Confirmed: B4-45 Lower` or `Did not confirm — Waitlist 8 final`).

---

## Success Metrics

**Model quality metrics (measured weekly on held-out data):**

- **Calibration error (Expected Calibration Error)**: target < 5%. When we say "70%," roughly 70% of those should actually confirm.
- **Brier score**: target < 0.18.
- **AUC**: target > 0.85 (this problem has strong signal — train, position, days-to-departure are highly predictive).
- **Coverage rate**: ≥ 80% of PNR queries should fall in High coverage (≥100 similar bookings); the long tail of rare trains will have lower coverage and that's fine — fallback handles them.

**Product impact metrics (measured monthly):**

- **PNR check share — IRCTC vs third-party sites**: rises by 30 percentage points within 6 months. Measurable via direct vs referrer traffic mix on third-party PNR-prediction sites; or via internal user surveys.
- **Repeat PNR checks per booking**: drops by 30%. Users who get a clear "70% likely to confirm" check fewer times than users staring at `WL 12 GNWL`.
- **NPS on the PNR page**: rises to 80%+.
- **Notification opt-in rate** (which is downstream of PNR page engagement): 40%+ of waitlisted users opt in.
- **Explanation-modal open rate**: 15%+ of users tap to see the explanation. Lower than 5% would suggest the headline is sufficient; higher than 25% would suggest the headline is unclear.

---

## Limitations and Risks

**1. Calibration drift over time.**
Railway scheduling, demand patterns, and quota allocations change. A model trained on 2024 data will degrade in 2026. Mitigation: weekly retraining with rolling window; monthly calibration audits; alerting when ECE crosses 7%. **Production calibration monitoring** (added post-peer-review): a daily job computes ECE on the previous 7 days of predictions where the chart has now resolved. If ECE crosses 7%, predictions are auto-gated to a stricter coverage threshold or muted entirely until retraining catches up. This is the production safety net the v1 spec was originally missing.

**2. Distribution shift on rare events.**
Festivals, COVID-like disruptions, sudden train cancellations, kumbh-mela-scale events. The model has not seen these at sufficient scale; predictions during such events will be less reliable. Mitigation: festival-flag feature; manual override flag during named disruptions (the model can be muted with a banner *"Predictions paused during current schedule disruption"*).

**3. The model can be wrong, and users will rely on it.**
A user seeing "70% likely to confirm" may decide *not* to book a backup train. If the prediction is wrong, the user is stranded. Mitigation:
   - Prominent "estimate, not guarantee" framing in the explanation modal.
   - The low-probability state actively suggests booking alternates.
   - Notification system tells users when status changes — they don't have to keep checking.
   - Honest fallback when coverage is low.

**4. Bias risks.**
Model could under-predict for routes with less training data (often regional / non-metro routes). This would silently disadvantage non-metro travellers — exactly the segment Part A flagged as most affected by IRCTC's existing UX failures. Mitigation: per-route accuracy and calibration audit; surface coverage-low fallback aggressively rather than guessing; don't ship until per-route bias is bounded.

**5. Privacy.**
Even hashed PNRs and anonymised training data carry re-identification risk in a tabular dataset of journeys. Mitigation: differential-privacy-aware training (or at least k-anonymity audit); legal review with MeitY data localisation; no training data ever leaves IRCTC infrastructure.

**6. Adversarial gaming.**
Travel agents could probe the model's predictions and selectively book or cancel to manipulate waitlist clearance for clients. Mitigation: rate limit per device and per agent ID; the prediction itself doesn't change waitlist mechanics, so the gameable surface is small.

**7. Dependency on CRIS data access.**
The largest external risk. Without CRIS releasing 365+ days of PNR outcome data, the feature cannot be built. This must be negotiated formally as a precursor to any engineering work.

**8. User over-reliance and learned helplessness.**
If users come to treat the prediction as gospel, they may stop developing their own judgment about which trains have reliable waitlist clearance. This is a long-term concern; mitigation is honest framing and transparency about the model's basis. A user who reads the explanation modal once learns *why* the prediction is what it is, which builds calibrated trust rather than blind trust.

---

## Out of scope for v1 (parking lot)

These are obvious extensions to consider once v1 is shipped and stable:

- **Notification with prediction trajectory**: alert the user when their predicted probability changes by ≥10 points (e.g. "Your probability dropped from 70% to 50% as more bookings came in").
- **Cross-train recommendations**: when probability is low, automatically suggest the 3 trains most likely to have available seats on the same route and date.
- **Per-quota-type sub-models**: a single model for all quotas may underperform for niche quotas (LD, HP, DF). Specialist sub-models could improve coverage in the long tail.
- **Free-form explanation generation**: replace template explanations with a small fine-tuned T5 for richer, multi-language, context-aware explanations. Only if v1 templates prove insufficient.

---

## Summary

A self-hosted, calibrated XGBoost model trained on anonymised historical PNR outcomes from CRIS, exposed via a low-latency internal API to the redesigned PNR Status page. Output is presented as a colour-coded probability bar with a transparent explanation modal. Fallback to plain-language translation when coverage is low or the service is unavailable. The largest risk is data-access negotiation with CRIS, not modelling.

This feature converts IRCTC's most-frequent post-booking question from "go Google these acronyms" to "here's a calibrated, transparent estimate" — and recaptures the user surface that ConfirmTKT and RailYatri have, for years, owned by default.