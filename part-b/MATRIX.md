# Impact vs Effort Matrix

This matrix places all six features from `SPECS.md` on an Impact vs Effort 2×2. Impact and effort scores both reference Part A frequency data and the technical implementation plans in the specs. Recommended sprint order follows the matrix discussion.

---

## The Matrix

|                   | Low Effort                                                | High Effort                                                  |
|-------------------|-----------------------------------------------------------|--------------------------------------------------------------|
| **High Impact**   | **Spec 2** — Search Filters<br>**Spec 4** — Captcha + Form State<br>**Spec 6** — Payment Recovery | **Spec 1** — Tatkal Queue<br>**Spec 3** — Berth Allocation<br>**Spec 5** — PNR + AI |
| **Low Impact**    | *(none)*                                                  | *(none)*                                                     |

Note on the empty quadrants: every problem documented in Part A is genuinely high-impact — that was the discovery threshold. Nothing made it into the six unless it materially affected a real user segment at meaningful frequency. So the Impact axis collapses to "all High" and the differentiation is on Effort. This is a healthy outcome — it means Part A discovery was disciplined.

---

## How I Scored Each Dimension

### Impact Scoring (1–5)

I scored Impact based on:
- **Number of users affected** — taken directly from Part A frequency analysis (e.g. ~36 lakh PNR checks/day, ~30 lakh+ captcha solves/day, twice-daily Tatkal load of 25–28 lakh users/min).
- **Whether the problem is in the core booking flow** — failures during search, booking, or payment compound (a captcha reset before payment costs the user the seat).
- **Severity of consequence** — losing money (Spec 6) > losing a seat (Spec 1) > losing form data (Spec 4) > taking longer to find a seat (Spec 2).
- **Trust damage** — measured by Part A's qualitative analysis. Spec 6 (payment timeout) was explicitly flagged in Part A as the highest trust-damage problem.

| Spec | Users affected | Core flow? | Severity | Trust damage | Impact score |
|---|---|---|---|---|---|
| 1 — Tatkal Queue | 25–28 lakh users/min, twice daily | Yes | Lose seat + Tatkal premium | High | **5** |
| 2 — Search Filters | Lakhs of filter sessions/hour | Yes (search) | Workaround exists (manual scroll) | Medium | **4** |
| 3 — Berth Allocation | Most multi-passenger bookings | Yes | Wrong berth + cancellation cost (25–50% fare) | High (especially for senior citizens, pregnant users) | **5** |
| 4 — Captcha + Form State | ~30 lakh+ captcha solves/day | Yes (login + booking) | 5 minutes of lost work per failure | Medium-High | **5** |
| 5 — PNR + AI | ~36 lakh PNR checks/day | Post-booking (high frequency) | No financial loss, but trust + third-party migration | High | **5** |
| 6 — Payment Recovery | "Lakhs of failed-payment refunds monthly" (Part A) | Yes | Money locked + lost seat | **Highest of all six** | **5** |

### Effort Scoring (1–5)

I scored Effort based on:
- **Number of system components touched** — counted from the Technical Implementation Plan in each spec.
- **Whether new infrastructure is required** — queue services, edge waiting rooms, ML pipelines all increase effort substantially.
- **Risk of breaking existing flows** — changes to session management, PRS integration, and payment plumbing carry higher regression risk.
- **External dependencies** — Railway PRS / CRIS coordination, Railway Board approvals, payment gateway changes, and government data-localisation reviews all add calendar time even when the code is small.
- **Frontend-only changes** are the cheapest; **backend session model changes** are mid; **third-party / cross-org dependencies** are most expensive.

| Spec | Components | New infra? | External deps | Frontend / Backend mix | Effort score |
|---|---|---|---|---|---|
| 1 — Tatkal Queue | 4+ | Queue service, edge waiting room | PRS, Railway Board | Heavy backend + new frontend | **5** |
| 2 — Search Filters | 2 | None | None | Mostly frontend | **2** |
| 3 — Berth Allocation | 4 | New real-time PRS read endpoint | CRIS / PRS | Backend + frontend | **4** |
| 4 — Captcha + Form State | 3 | Bot-detection vendor (Turnstile) | Vendor + MeitY review | Mostly frontend, some backend | **2** |
| 5 — PNR + AI | 5 | ML model + serving infra + data pipeline | CRIS data access (largest risk) | Heavy across the stack + content | **5** |
| 6 — Payment Recovery | 4 | None new (session model change) | Payment gateway behaviour, RBI compliance review | Backend session change + frontend recovery screen | **3** |

---

## Placement Justifications

### Spec 1 — Tatkal Virtual Queue — High Impact / High Effort

The impact is unambiguous: 25–28 lakh users per minute, twice daily, for years, with the most-affected segments (migrant workers, urgent travellers, students) having no fallback option. Part A's frequency table puts this at "daily, twice" — the highest possible frequency tier. The effort is also high: a new queue service, edge waiting-room infrastructure, PRS-level inventory holds, and — critically — Railway Board / CRIS approval to materially change how Tatkal's "first-come-first-served" mechanic is enforced. This is a Big Bet quadrant placement: it should be sprint-funded and roadmap-committed, but not attempted in a one-week sprint.

### Spec 2 — Search Filters — High Impact / Low Effort

Impact is high because every search session that uses filters hits at least one of three failure modes (Part A, Steps 4, 6, 9), and search volume is in the lakhs per hour during peak. The workaround (manual scrolling) caps the severity slightly compared to the "lose a seat" problems, hence Impact = 4 not 5. The effort is genuinely low: the fix is mostly a frontend change moving filter state into the URL, with small API tweaks to return better filter metadata. No new infrastructure, no PRS changes, no Railway Board sign-off, no external vendors. This is the cleanest Quick Win in the matrix and should be the first thing shipped — it earns trust with the user and validates the team's execution rhythm before tackling the harder bets.

### Spec 3 — Berth Allocation — High Impact / High Effort

The impact is highest among the safety-and-dignity problems (senior citizens, pregnant passengers, infants), and the cancellation-cost consequence makes this severe. Effort is high primarily because of the real-time PRS availability read endpoint (Spec 3's `POST /api/booking/preview-allocation`) — that endpoint may not exist today and getting CRIS to expose per-berth real-time availability is a non-trivial cross-org negotiation, similar in flavour to the CRIS data-access dependency in Spec 5. The frontend changes are bounded; the backend changes touch PRS-adjacent code which is highly regulated. Big Bet placement, second-tier priority after Spec 1 because the user segments are smaller in raw count.

### Spec 4 — Captcha + Form-State Preservation — High Impact / Low Effort

This is the matrix's highest-leverage Quick Win. Impact is severe — every booking and every login hits the captcha (~30 lakh+ solves/day per Part A), and form resets cost users 5 minutes of typing per failure on a high-anxiety flow. Effort is genuinely low because both halves of the spec are bounded: form-state preservation is a frontend refactor from server-rendered template to controlled SPA form (a well-trodden pattern), and bot detection is a vendor integration (Cloudflare Turnstile is free, privacy-respecting, and deployable in days). The MeitY data-localisation review for the vendor adds calendar time but not engineering effort. Ship this in the same sprint as Spec 2.

### Spec 5 — PNR + AI Confirmation Probability — High Impact / High Effort

Impact is high in raw volume (36 lakh checks/day) and strategically critical (recapturing the user surface from ConfirmTKT and RailYatri). Effort is high across multiple axes: a new ML model and serving infrastructure, a content-design effort to translate every railway code into plain language across 10 languages, a multi-state UI redesign with multiple fallback paths, and — the biggest risk — a CRIS data-access negotiation for 365+ days of historical PNR outcome data without which the model cannot be built. Big Bet placement, highest priority of the Big Bets after Spec 1 because the data dependency negotiation can run in parallel with engineering on Specs 2, 4, and 6.

### Spec 6 — Payment Recovery — High Impact / Low–Medium Effort

This is the most contested placement in the matrix. Part A explicitly flagged Spec 6's underlying problem as the **highest trust-damage problem of the six** — money debited, no information, week-long refund freeze. Impact is unambiguously top-tier. Effort is the harder call: the recovery-screen frontend is bounded (a single new route with a status poller), but the transaction-aware session model is a real backend change that touches session middleware, payment-intent state, and idempotency keys. I scored Effort = 3, putting it on the Low Effort side of the matrix because the change is bounded to one well-understood subsystem and uses standard patterns (idempotency keys, intent-bound sessions). It does not require external CRIS coordination or new ML infrastructure. **This is the Quick Win to prioritise above Spec 2 and Spec 4 if the team has bandwidth for one Quick Win** — the trust-damage data from Part A makes it the highest-ROI fix in the matrix.

---

## Recommended Sprint Order

**Sprint 1 (weeks 1–2): Quick Wins**

1. **Spec 6 — Payment Recovery.** Highest trust-damage problem per Part A. Bounded engineering scope. Ship the recovery screen and transaction-aware sessions together. Visible win for the most-affected users (Tatkal payers, slow-network users). This is the order's strongest signal of intent.

2. **Spec 4 — Captcha + Form State.** Pair with Spec 6 because they share the modern-form-state pattern. The form-state preservation work in Spec 4 directly benefits the recovery screen. Bot detection vendor work runs in parallel (procurement + MeitY review).

3. **Spec 2 — Search Filters.** Cleanest engineering job in the set. Use it as the team's "shipping rhythm" exercise to validate processes and CI/CD. Frontend-heavy.

**Sprint 2 (weeks 3–8): Big Bets — kick off in parallel**

4. **Spec 1 — Tatkal Virtual Queue.** Begin Railway Board / CRIS conversations on Day 1 of Sprint 2; this gates everything else for this spec. Engineering can start on the queue service and edge waiting room in parallel. Realistic ship target: 8–10 weeks.

5. **Spec 5 — PNR + AI.** Begin CRIS data-access negotiation on Day 1 of Sprint 2 (this is the bottleneck — model engineering is fast once data is available). Content design (plain-language code translations across 10 languages) can run fully in parallel with data negotiation. Realistic ship target: 12 weeks.

6. **Spec 3 — Berth Allocation.** Begin PRS real-time-availability endpoint negotiation alongside Spec 1 (likely the same CRIS counterparts). Engineering can start on the form and review-screen redesigns in parallel. Realistic ship target: 10 weeks.

**Sequencing logic:**

- **Quick Wins ship first** to build user-visible momentum and validate team execution. They also de-risk the next sprint by exercising the deployment pipeline before the Big Bets land.
- **Big Bets begin their cross-org negotiations on Day 1 of Sprint 2.** The negotiation calendar (CRIS, Railway Board, MeitY) is the long pole, not the engineering work. Starting these conversations early — even before the Quick Wins ship — would be even better, but is captured here as the first action of Sprint 2.
- **Specs 1 and 3 share PRS / CRIS counterparts.** Bundle the conversations.
- **Specs 4 and 6 share the modern form-state pattern.** Bundle the engineering work.
- **Spec 5 has the longest data-dependency tail** but the lightest user-facing dependency on other specs. It can ship later in the calendar without blocking anything else.

---

## What this matrix is *not*

- **Not a final commitment.** Effort estimates are calibrated to publicly-knowable IRCTC architecture; actual estimates should be revised after a one-day spike per spec by the engineering team that owns each subsystem.
- **Not a substitute for user research.** The Impact column is grounded in Part A's frequency analysis and segment-affected analysis, but pre-launch user testing on the wireframes (especially for Spec 5 plain-language translations and Spec 1 queue framing) is still required.
- **Not blind to organisational reality.** "Railway Board approval" is treated as a single line item here; in practice it is months of coordination. The matrix is correct on relative effort but conservative on absolute calendar time.

---

## Post-peer-review note

After peer review (see SPECS.md "Peer Review Updates"), the matrix placements remain unchanged but two effort estimates are stress-tested:

- **Spec 1 (Tatkal Queue)**: now explicitly depends on a parallel KYC-tiering effort to defend against account farming. The effort score stays at 5/5 (already maxed) but the calendar widens — the queue cannot ship as a meaningful fairness improvement without account verification, so its real ship date is gated on the slower of the two efforts.
- **Spec 5 (PNR + AI)**: the new production calibration monitoring requirement adds roughly one additional week of engineering. Effort score stays at 5/5. The CRIS data-access negotiation remains the dominant calendar factor.

The recommended sprint order is unchanged. The Quick Wins (Specs 2, 4, 6) ship first; the Big Bets (Specs 1, 3, 5) begin cross-org negotiations on Day 1 of Sprint 2 and ship on a longer calendar — now slightly longer than originally estimated for Specs 1 and 5.