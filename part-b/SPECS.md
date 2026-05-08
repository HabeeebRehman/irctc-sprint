# IRCTC Feature Specifications — Part B

These six specs are a direct continuation of `part-a/PROBLEMS.md`. Every spec begins from a documented pain point — the affected users, frequency, and exact failure step were established in Part A and are referenced here, not re-discovered.

The specs are ordered to match Part A's numbering. Recommended sprint order (which is different from this list order) is in `MATRIX.md`.

---

## Feature Spec 1: Tatkal Virtual Queue with Live Position & Wait Estimate

### Problem Statement

Twice every weekday — at 10:00 AM for AC Tatkal and 11:00 AM for non-AC — the IRCTC booking flow becomes unresponsive under a 25–28 lakh-per-minute load. The engineering overload is not the problem this spec solves; the problem is that **between the click on "Book Now" and the next visible screen, the user gets zero feedback** (Part A, Problem 1, Step 5). Migrant workers, daily-wage earners, students travelling for exams, and patients seeking urgent care — the segments most dependent on Tatkal — are forced to refresh blindly, which both costs them their place in any queue and worsens the underlying load.

### Current State (from Part A)

The Part A flow documents that at Step 5 (10:00:00 AM onwards), the user sees a white page or generic spinner with no queue position, no wait estimate, and no differentiation between "still processing" and "already failed." Outcomes are randomised across three failure modes — silent session expiry, generic toast, or a reload to a sold-out train list. The user has no signal to act on, so they refresh, pushing themselves to the back of any in-flight queue.

### Proposed Solution

When a user clicks "Book Now" on a Tatkal-eligible train at or after 10:00:00 AM, IRCTC issues them a **queue token** and shows a dedicated **Tatkal Queue screen** with three live values: the user's current position number, the number of users ahead of them, and a rolling wait estimate computed from server throughput. The user is told plainly: *"You are #45,820 in queue. Estimated wait: 4 minutes. Do not refresh — your place is held."* When their turn arrives, the page transitions automatically into the seat/passenger entry screen with a 90-second hold on Tatkal inventory specifically allocated to that queue token.

The queue itself is the engineering work — but the spec here is about the UX contract: the user is given a position, a number, and a promise. Every refresh, tab close, or back-button press is now a self-inflicted wound the user can clearly see they are about to make.

### Proposed User Flow — Step by Step

1. **9:58 AM** — User pre-fills search and lands on the train list page. Tatkal trains show a countdown to 10:00:00 AM as before.
2. **10:00:00 AM** — User clicks "Book Now." A POST request enters the Tatkal queue service and returns a `queue_token` and a starting position.
3. **10:00:01 AM** — User is shown the **Tatkal Queue screen**: queue position (e.g. `#45,820`), users ahead, estimated wait time, and a "What's happening?" expander explaining the queue model.
4. **10:00:02 AM onwards** — Position counter ticks down via a websocket / SSE connection (e.g. one decrement per ~50ms during peak throughput). Wait estimate updates every 5 seconds based on the rolling throughput rate.
5. **Closing the tab or refreshing** triggers a confirmation modal: *"Refreshing will release your place. You are currently #12,400."*
6. **When position = 0** — The user's `queue_token` is exchanged for a 90-second seat-entry hold. The Queue screen transitions automatically to the passenger details page with the chosen train's seat inventory locked for them for 90 seconds.
7. **If 90 seconds elapse** without form submission, the hold releases and the user is shown a clear timeout screen with a one-click "Rejoin queue" path (which puts them at the back of the *current* queue — not the original 10:00 queue).
8. **If the user's chosen Tatkal quota sells out before their position arrives** — they are notified at the queue screen with: *"Tatkal seats for 12951 sold out at #18,200 (you are #45,820). View other Tatkal trains on this route?"* with a one-click pivot to alternates.

### Technical Implementation Plan

**System components affected:**
- Booking entry endpoint (`POST /book`) — wrapped behind a queue admission gate
- New Tatkal queue service (Redis sorted set or a dedicated queueing tier such as Cloudflare Waiting Room / AWS API Gateway request throttling) keyed by `(train_no, journey_date, quota=TATKAL)`
- PRS (Passenger Reservation System) inventory hold mechanism — needs a 90-second soft-lock on a per-`queue_token` basis
- Frontend booking SPA — new Queue screen component and websocket/SSE client

**New data requirements:**
- `queue_token` table: token, user_id, train_no, journey_date, quota, position, issued_at, expires_at, status
- Rolling-throughput counter per queue (used to compute wait-time estimate)
- Per-train `tatkal_inventory_lock` table (token → seats reserved → expires_at)

**API changes:**
- New: `POST /api/tatkal/enqueue` → returns `queue_token`, initial position, estimated wait
- New: `GET /api/tatkal/position/:token` (HTTP fallback) and `WS /tatkal/position/:token` (live)
- New: `POST /api/tatkal/redeem/:token` → exchanges token for a 90-second inventory hold and returns the booking session
- Modified: existing `POST /book` requires a valid redeemed token during the Tatkal window for Tatkal quota

**Frontend changes:**
- New route: `/tatkal/queue/:token` with the Queue screen component
- Websocket client with HTTP polling fallback (poll every 3s if WS fails)
- Refresh / close / back-button warning modal
- Seamless handoff from Queue screen → passenger details page (no full reload)

**Third-party services:**
- Likely a managed waiting-room service (Cloudflare Waiting Room or equivalent) at the edge to absorb the 25–28 lakh/min spike before it hits IRCTC origin. The Queue screen can be hosted at the edge during the spike and rehydrated to the IRCTC SPA on admission.

### Wireframe

See **Figma brief: Tatkal Queue Screen** at the end of this spec. Saved export: `assets/wireframes/01-tatkal-queue-screen.svg`.

### Success Metrics

- **Refresh rate during 10:00–10:05 AM window**: drops from current (effectively 100% — every user refreshes at least once) to under 10%.
- **Tatkal "money debited but no ticket" complaints**: drop by 40% within 8 weeks of launch (because the queue prevents the half-state failures that produce these).
- **Tatkal booking completion rate** (clicks on "Book Now" → confirmed booking): rises from current ~5–8% (estimated from public complaints) to 25%+.
- **Customer-care call volume** between 10:00 AM and 11:30 AM: drops by at least 30%.
- **User-reported clarity** (in-app NPS micro-survey shown post-Tatkal-attempt): "Did you understand what was happening during the wait?" — target 80%+ Yes.

### Edge Cases and Constraints

- **Network drop during queue wait**: queue token is stored client-side (localStorage) AND server-side; on reconnect, the user resumes from their actual server-side position, not from where the client thinks they were.
- **User opens multiple tabs** to "improve odds": only one active token per user_id; second tab shows the same position as the first. Enforce server-side.
- **Travel agents with multiple bookings**: agent IDs get a small token quota (e.g. 5 concurrent tokens per agent, configurable) — not unlimited, to prevent agent monopolisation.
- **Railway PRS itself goes down** (a real risk during peak): the queue gracefully shows *"Railways system temporarily slow. Holding your place."* rather than dumping users to a generic error page. Token does not expire while PRS is the bottleneck.
- **Government / Railway Board approval**: queue model needs sign-off from CRIS and Railway Board because it materially changes how Tatkal "first-come-first-served" is enforced. Risk: this is not a unilateral IRCTC change.
- **Graceful degradation**: if the queue service itself fails, fall back to current behavior (direct booking attempt) with a clear banner: *"Queue system unavailable. Booking will proceed with standard load handling."*
- **Account-farming and fake user IDs** (added post-peer-review): the per-user token limit is only as strong as IRCTC's user-id uniqueness. Adversaries can create thousands of fake accounts to bypass the cap, relocating the abuse from the booking endpoint to the account-creation endpoint. Mitigation: pair the queue with KYC-tier rate limiting — verified-Aadhaar accounts get full priority during the Tatkal window; unverified accounts get heavily throttled token issuance. Surface this contract to legitimate users at sign-up: *"Verify your Aadhaar to get standard Tatkal access."* This is out-of-scope for v1 of the queue itself but **must be roadmapped together** — shipping the queue without account-farming defenses would be cosmetic.

### Figma Brief — Tatkal Queue Screen

**Frame size:** Two frames — `Mobile 390×844` (primary; most Tatkal users are on mobile web) and `Desktop 1440×900`. Build mobile first.

**Visual hierarchy (top to bottom):**

1. **Top status bar** — IRCTC logo (left, 32px), "Tatkal Queue" title (centre), small countdown chip on right showing current IST time (e.g. `10:01:47 AM`). Background: deep IRCTC blue (`#1A237E`). Height 56px.

2. **Hero block — Queue position** — full-width card, white background, rounded 12px, shadow. Inside, very large number `#45,820` (display font, 64px on mobile, 96px on desktop, weight 700) — this is the page's anchor. Below it in muted grey 14px: `Your position in queue`. To the right of the number on desktop, a subtle animated downward chevron showing the position is decrementing.

3. **Estimated wait** — directly below the position card. Large secondary number `~4 min` (font 32px) with label `Estimated wait` (12px muted). A thin horizontal progress bar showing the rolling throughput rate, animated.

4. **"Users ahead of you" line** — `45,819 users ahead • 12,400 admitted in the last minute`. 14px, muted. This is the trust-building data point — proves the queue is moving.

5. **Status pill** — small pill below the wait estimate. Three states:
   - Green: `Holding your place` (default, animated dot pulsing)
   - Amber: `Railways system slow — your place is held` (shown when PRS lags)
   - Red: `Tatkal sold out for 12951 — see alternates` (terminal state)

6. **Selected train summary card** — collapsible. Shows train number, name, From → To, journey date, class. So the user remembers what they're queueing for.

7. **"What's happening?" expander** — accordion. Plain-language explanation: *"Tatkal opens for everyone at 10:00 AM. We've placed you in a queue so the system stays fair and stable. When it's your turn, you'll have 90 seconds to enter passenger details. Your place is held — please don't refresh."* Available in Hindi, English, and 8 regional languages via a language switcher in the top bar.

8. **Critical "Don't refresh" banner** — sticky at bottom of viewport on mobile. Bright but not alarming yellow background, dark text: `⚠ Refreshing or closing this tab will release your place.` Persistent.

9. **Pivot CTA (only when sold out)** — full-width primary button: `View other Tatkal trains NDLS → BCT`. Hidden until the sold-out state.

**States to design:**
- Default (queue moving normally)
- Slow (PRS lag — amber banner)
- Sold out for chosen train (red status, pivot CTA visible)
- Loading / connecting (skeleton state, before first position arrives)
- Reconnecting (websocket dropped, polling fallback active — small `Reconnecting…` chip near status pill)
- Position = 0 / handoff (full-screen "It's your turn — opening passenger details…" with 1-second transition)

**Annotations to include in the Figma file:**
- On the position number: *"Live counter, updates via websocket; HTTP poll fallback at 3s intervals"*
- On the wait estimate: *"Computed from rolling 30s throughput; updates every 5s"*
- On the bottom banner: *"Sticky on mobile; absolutely positioned on desktop above fold"*
- On the language switcher: *"Default = device language; persists in localStorage"*

**Comparison annotation:** include a small "Before" thumbnail in the corner of the Figma frame showing the current white screen / generic spinner, with an arrow to the new Queue screen. Caption: *"Before: zero feedback. After: position, wait estimate, status, and clear no-refresh contract."*

---

## Feature Spec 2: Reliable, URL-Encoded, Persistent Search Filters

### Problem Statement

The search results page filters fail in three ways simultaneously: they sometimes don't filter (a 3A-ticked list still contains trains without 3A availability), they reset on browser back-navigation, and they interact unpredictably under combinations (Part A, Problem 2, Steps 4, 6, 9). The cumulative effect is that users — especially first-time bookers, mobile users, and senior citizens facing 30–40 unfiltered trains — learn to distrust the filters entirely and revert to manual scrolling. The trust loss is the real damage.

### Current State (from Part A)

Part A documented that filters in IRCTC's left sidebar are stored in volatile client-side state with no URL parameter encoding and no `popstate` handling. The "Available only" filter is binary and ambiguous about WL/RAC inclusion. Class filters confuse "this train *has* this class" with "this train has *available seats* in this class." Step 9 — losing all filters when a user clicks Back from a train detail page — is the most damaging because it violates a baseline expectation of how the web works.

### Proposed Solution

Filters become a **first-class, URL-encoded, semantically clear** part of the search experience. Every filter selection updates the URL query string, so back-navigation, deep-linking, and bookmarking all work. Filter labels are rewritten to remove ambiguity: instead of "AC 3 Tier (3A)" we show "AC 3 Tier — *with seats available*" (toggleable); the "Available only" filter is replaced with a more granular **availability chip row** (`Available now` / `RAC` / `Waitlist ≤ 50` / `Waitlist > 50`) where each chip is independently toggleable. A persistent filter summary bar at the top of results lets users see *what's currently active* at a glance and clear individual filters with one tap.

### Proposed User Flow — Step by Step

1. User searches NDLS → BCT for next Wednesday. Lands on results page. URL is `/search?from=NDLS&to=BCT&date=2026-05-13`.
2. User taps the `AC 3 Tier — available` chip. URL updates to `…&class=3A&availability=avail`. Results re-render.
3. User taps the `Departs 6 PM – 12 AM` chip. URL updates to `…&depart=18-24`. Results re-render. A summary bar at top shows: `Filters: 3A • Available • 6 PM–12 AM` with an `×` next to each.
4. User clicks a candidate train → train details page loads.
5. User clicks browser Back. Filters are **fully restored** because they're in the URL.
6. User taps `×` on the `Available` chip in the summary bar. That filter clears; URL updates; other filters remain.
7. User shares the URL to a family member via WhatsApp. Family member opens the link and sees the same filtered results.

### Technical Implementation Plan

**System components affected:**
- Search results frontend (the SPA route that renders the train list)
- Search API (backend filtering logic for class availability vs. existence)
- Browser history management (`pushState` / `popstate`)

**New data requirements:**
- None at the database layer. Existing train inventory data already has the granularity needed (per-class availability is already returned by the search API).
- Add a `filter_session_id` for analytics so we can measure which filter combinations are used and which return zero results.

**API changes:**
- `GET /api/search` accepts new query parameters: `class[]`, `availability[]`, `depart_band[]`, `arrive_band[]`, `train_type[]`, `wl_threshold`. All are arrays so multi-select works.
- API returns `filter_metadata` alongside results: for each filter chip, whether toggling it would yield zero results (so the UI can grey out and warn before the user taps).

**Frontend changes:**
- Filter state moves from React component-local state to a URL-synced state hook (`useSearchParams` or equivalent). Every filter toggle calls `pushState`.
- New component: filter summary bar (sticky, shows active filters as removable chips).
- New component: filter chip group with disabled states (greyed out + tooltip when toggle would yield zero results).
- Rewritten filter labels to be unambiguous (label copy reviewed by content design).
- `popstate` handler to re-render results when user navigates back/forward.

**Third-party services:** None.

### Wireframe

See **Figma brief: Search Results with Persistent Filters** at the end of this spec. Saved export: `assets/wireframes/02-search-filters.svg`.

### Success Metrics

- **Filter abandonment rate** (sessions that apply a filter then clear all filters within 30s): drops from baseline to under 15%.
- **Back-navigation filter loss complaints** (currently the #1 search-related complaint): drops to near-zero within one release.
- **Sessions using 2+ filters simultaneously**: rises by 50%+ (because filters are now trustworthy).
- **Time-to-train-selection** for filter users: drops by 30%+ on mobile.
- **Shareable search URL usage** (sessions arriving from a deep-link with filter params): becomes a measurable, non-zero number — currently impossible.

### Edge Cases and Constraints

- **URL length limits**: extremely long filter combinations could push URL length past mobile browser limits (~2000 chars). Mitigate with short parameter names and a max of 6 active filters at once.
- **Filter combinations that yield zero results**: API returns `filter_metadata` so UI can grey out filters that would empty the list and show a tooltip: *"No trains match this combination. Try clearing time band first."*
- **Date / route changes from a filtered URL**: if a user changes From/To/Date but had filters in the URL, retain compatible filters (class) and silently drop incompatible ones (specific train types if they don't run on the new route), with a small toast: *"Some filters were cleared because they don't apply to this route."*
- **SEO and crawler concerns**: filtered URLs should be `noindex` to avoid duplicate-content issues, but shareable.
- **Backwards compatibility**: existing bookmarked search URLs (without filter params) must continue to work — defaults are sensible (no filters applied).
- **Graceful degradation**: if the URL-state hook fails (very old browser), fall back to current session-only filter state with a one-time banner explaining limitations.

### Figma Brief — Search Results with Persistent Filters

**Frame size:** `Mobile 390×844` and `Desktop 1440×900`.

**Layout (mobile):**

1. **Top bar** — back arrow, "NDLS → BCT • Wed, 13 May" (editable, taps open the search modal), filter icon with badge showing active-filter count.
2. **Filter summary bar** — sticky, just below top bar. Horizontally scrollable row of chips showing currently active filters: `3A ×` `Available ×` `6PM–12AM ×`. Tapping `×` clears that one filter. Last chip is `Clear all` (text link, not a chip).
3. **Filter chip groups** — collapsible section below the summary bar. Three groups: **Class** (1A / 2A / 3A / SL / 2S as toggleable chips), **Availability** (Available / RAC / WL ≤ 50 / WL > 50), **Time** (Early morning / Morning / Afternoon / Evening / Night as a 5-chip row). Greyed-out chips have a small tooltip icon; tapping shows *"No matches with current filters."*
4. **Results list** — train cards. Each card shows: train number + name (top), departure time and arrival time with duration, class-by-class availability strip at the bottom (`3A: Avail-12 • 2A: WL-8 • SL: Avail-45`). The relevant class (matching the filter) is highlighted.
5. **Empty state** — when filters yield zero results: large illustration, headline `No trains match these filters`, body text *"Try removing the time filter — there are 23 trains available without it."* with a one-tap suggestion button.

**Layout (desktop):**

- Two-column. Left sidebar (300px) holds the full filter UI, organised by group with section headers. Filters are *also* reflected in the URL and the summary bar at the top of the right column.
- Right column (1140px) shows the train list.
- Sticky behavior: filter sidebar sticks during scroll.

**States to design:**
- Default (no filters)
- 1 filter active
- 3+ filters active (summary bar shows all)
- Empty results (with suggestion)
- A filter chip in disabled-zero-result state (greyed + tooltip)

**Annotations:**
- On the URL bar of the screen mockup: show the actual URL changing as filters are toggled — *"URL is the source of truth"*.
- On the back arrow: *"Back navigation now restores filters because they live in the URL"*
- On the chip: *"Tap to toggle. Disabled state when toggle would yield zero results — prevents dead-ends."*

**Comparison annotation:** small Before/After thumbnails. Before: filters in sidebar, no summary bar, URL doesn't change. After: filters mirrored in URL + summary bar, back navigation works.

---

## Feature Spec 3: Berth Preference — Explicit Contract & Pre-Payment Confirmation

### Problem Statement

When users select a specific berth preference (Lower, Middle, etc.) on the passenger details page, the selection is silently lost or reduced to a non-binding hint by the time the booking reaches PRS. The user discovers the loss only after payment, when the confirmed allocation is wrong (Part A, Problem 3, Step 10). The hardest-hit segments are those with medical needs for lower berths — senior citizens, pregnant passengers, and parents with infants. Worse, IRCTC has never told users that "Lower Berth" is a *request*, not a *guarantee*, so users feel betrayed by what is partly a contract-clarity failure on top of an actual state-loss bug.

### Current State (from Part A)

Part A documented that the berth value is captured in the form state but is either not consistently serialized into the PRS request, or is sent as a non-binding preference with no UX disclosure (Step 10). The Review Booking screen does not surface berth preferences prominently, especially on mobile (Step 8). The deeper UX problem identified in Part A: IRCTC never communicates that "Lower" is a request, not a guarantee.

### Proposed Solution

Two changes, applied together:

1. **Fix the state-loss bug** — berth preference values are reliably persisted from the passenger form into the booking session and into the PRS request. Audited end-to-end with logging at every hop. This is the engineering fix.
2. **Make the contract explicit** — when a user selects "Lower," the form immediately shows a short inline disclosure: *"Lower berth requested. Final allocation depends on availability — confirmed berth will be shown before payment."* The Review screen, before payment, shows a prominent **Berth Allocation Block** for each passenger with the *actual* berth the system will request from PRS, plus a clear `Likelihood: High / Medium / Low` indicator based on real-time availability. If a passenger's requested berth cannot be honored at all (e.g. they asked for Lower but only Upper is available), the user sees this *before paying*, not after, and can choose to proceed, swap passengers around, or cancel.

This is partly a bug fix and partly a redesign of the trust contract.

### Proposed User Flow — Step by Step

1. User books a train with seat preference. Enters Passenger 1 (senior citizen, requests Lower), Passenger 2 (Middle).
2. As soon as "Lower" is selected for Passenger 1, an inline message appears under the field: *"Lower berth requested. Subject to availability — you'll see the confirmed allocation before payment."*
3. User completes the form, captcha, clicks "Continue."
4. Review screen loads. Each passenger row shows a **Berth Allocation Block**:
   - Passenger 1: *Requested: Lower • Likely allocation: Lower (Coach B4) • Likelihood: High*
   - Passenger 2: *Requested: Middle • Likely allocation: Side Upper • Likelihood: Lower berth not available, will be Side Upper*
5. The mismatch on Passenger 2 is highlighted in amber. User has three buttons: `Proceed anyway`, `Try different train`, `Modify passengers`.
6. User taps `Modify passengers`. Edits Passenger 2's preference to Upper, which is available. Returns to Review.
7. All allocations are now `High` likelihood. User pays.
8. Booking confirms. Allocated berths match what the Review screen showed.

### Technical Implementation Plan

**System components affected:**
- Passenger details form (frontend)
- Booking session state management (frontend + backend)
- PRS request serialization layer
- Review/confirm-booking screen
- Real-time inventory query (a new lightweight call to PRS *before* payment to check current berth availability)

**New data requirements:**
- Booking session must persist `berth_preference` per passenger across all transitions until PRS confirmation.
- New cached read-model: per-train, per-class current berth availability counts (Lower / Middle / Upper / SL / SU). This already exists in PRS but needs to be exposed via a low-latency endpoint.

**API changes:**
- New: `POST /api/booking/preview-allocation` — given the booking session (train, class, passengers, preferences), returns predicted berth allocation per passenger with confidence (High / Medium / Low). Called when the Review screen loads.
- Modified: existing booking-create endpoint must accept and forward the per-passenger `berth_preference` value reliably.
- New backend test suite asserting berth preference round-trips end-to-end.

**Frontend changes:**
- Inline disclosure component on the passenger form (shown when a specific berth is selected).
- New Berth Allocation Block component on the Review screen, with likelihood indicator.
- Mobile review screen redesign — current truncation of berth column is the silent-loss surface; redesign so berth allocation is a row-level block, not a column.

**Third-party services:** None.

### Wireframe

See **Figma brief: Passenger Form & Review Screen with Berth Allocation Block** at the end of this spec. Saved export: `assets/wireframes/03-berth-allocation.svg`.

### Success Metrics

- **Post-booking complaints about wrong berth allocation**: drop by 70%+ within 12 weeks (combination of fewer actual mismatches and clearer expectations when mismatches happen).
- **Cancellations within 1 hour of booking** (a proxy for "got the wrong berth and gave up"): drop by 30%+.
- **Senior-citizen / lower-berth-needed bookings completion rate**: rises (these users currently abandon mid-flow when they're not confident the berth will stick).
- **NPS on the Review screen** ("Was the berth information clear?"): target 85%+.

### Edge Cases and Constraints

- **Real-time inventory lag** — PRS availability can shift between Review and Pay (a few seconds). Mitigate with a soft-hold of the predicted allocation for 60 seconds during payment.
- **Mixed-class bookings** (rare but possible): the allocation block must work per-passenger when passengers are in different classes.
- **Group bookings (5+ passengers)**: showing 6+ allocation blocks on mobile is dense — collapse to a summary by default with an expand option.
- **PRS returns no berth data** (very old route, edge cases): show *"Berth allocation will be confirmed by Railways. Lower berth request noted."* — a graceful fallback to the current behavior, but at least labelled.
- **Dependency on PRS / CRIS**: real-time per-berth availability requires a low-latency endpoint that may not exist today. If it doesn't, this spec needs CRIS engineering coordination, which is non-trivial.
- **Graceful degradation**: if the preview-allocation API fails, fall back to showing the requested preference + a clear caveat: *"Allocation will be confirmed at booking. Subject to availability."*

### Figma Brief — Passenger Form & Review Screen with Berth Allocation Block

**Frame size:** `Mobile 390×844` (primary — most berth complaints originate on mobile per Part A) and `Desktop 1440×900`.

**Two screens to design:**

**Screen A — Passenger details form (modified)**

1. Each passenger card has the existing fields (name, age, gender) plus a **Berth Preference** dropdown.
2. **The new piece**: directly below the dropdown, an inline disclosure box (light blue background, 12px padding, 14px text): *"Lower berth requested. Subject to availability — you'll see the confirmed allocation before payment."* This box appears as soon as a non-default berth is chosen and disappears if the user reverts to "No Preference."
3. A small (i) icon next to the Berth Preference label opens a modal explaining how berth allocation works in plain language.

**Screen B — Review screen (redesigned)**

1. **Top section** — train summary (number, name, route, date, class).
2. **Berth Allocation Block** for each passenger — full-width card, prominent. Each card shows:
   - Passenger name + age (left)
   - Two-line block (centre): line 1 *Requested: Lower*; line 2 *Allocated: Lower (Coach B4)*
   - Likelihood pill (right): green `High`, amber `Medium`, red `Mismatch` (with the mismatch state showing a different allocated berth than requested)
3. **If any passenger shows a `Mismatch`**: an amber summary banner above the cards: *"1 passenger may not get their requested berth. Review below before paying."* Three CTAs: `Proceed anyway` (primary), `Try different train`, `Modify passengers`.
4. **Fare summary** below the allocation blocks (current Review screen content, unchanged).
5. **Pay button** at the bottom — disabled for 2 seconds after page load to ensure the user actually reads the allocation blocks.

**States to design:**
- All passengers `High` (happy path — green)
- One passenger `Mismatch` (amber banner + amber pill)
- All passengers `Mismatch` (red banner — strong recommendation to try different train)
- Inventory data unavailable (graceful fallback — `Subject to availability` pill, no likelihood)
- 6-passenger group (collapsed summary view)

**Annotations:**
- On the inline disclosure (Screen A): *"Sets the contract early — user knows allocation is a request, not a guarantee."*
- On the Berth Allocation Block (Screen B): *"This is the trust surface. Mismatches are surfaced before payment, not after."*
- On the Pay button: *"2-second delay after Review load — prevents accidental tap-through that ignores allocation block."*

**Comparison annotation:** Before: berth preference shown as a single truncated column on mobile, no allocation visible until SMS arrives post-payment. After: dedicated allocation block per passenger, mismatches highlighted, decision before payment.

---

## Feature Spec 4: Modern Bot Detection + Form-State Preservation on Validation Failure

### Problem Statement

IRCTC's captcha is at the edge of legibility for normal vision and effectively impossible for low vision (Part A, Problem 4). When users mistype it on the passenger details page, **the entire form — sometimes 25+ fields including 4 passengers' details — is wiped** (Step 7). This compounding failure punishes the most-affected segments — senior citizens, low-vision users, slow-network users — multiple times per booking. Login captcha alone produces ~30+ lakh solves per day across IRCTC; a non-trivial fraction fail and trigger total form resets.

### Current State (from Part A)

Part A documented two compounding failures: (a) the captcha image uses stretched, overlapping, confusable glyphs (`0/O`, `1/l/I`); and (b) a wrong captcha entry returns a fresh template page from the server with all 25 form fields blank. The form-state-on-validation-failure bug is the more severe of the two — even with a perfect captcha, occasional mistypes are inevitable, and total form wipe is the wrong response.

### Proposed Solution

Two independent changes, both of which independently improve the experience, but which compound powerfully when applied together:

1. **Replace the visual captcha with a modern bot-detection layer** — primarily invisible behavioural detection (Cloudflare Turnstile or hCaptcha invisible mode), with a fallback to a one-tap "I'm not a robot" challenge for borderline traffic, and only as a last resort a visible challenge — and even then, an audio-and-visual challenge with WCAG 2.2 AA compliance.

2. **Preserve form state on validation failure** — every form on IRCTC must round-trip the user's entered data on validation failure, not return a blank template. This is implemented as a client-side single-page form (state lives in the SPA, not the server-rendered template) so a captcha mismatch is an inline error at the captcha field, while every other field stays exactly as the user left it.

The combination means: for the vast majority of users, no captcha is ever shown; for the small fraction that get challenged, a single-tap challenge is enough; and even in the worst case where someone fails the challenge, **they don't lose 5 minutes of form entry.**

### Proposed User Flow — Step by Step

**Login (covered by change 1):**
1. User enters User ID and password, taps Sign In.
2. Background bot-detection runs — invisible. 95%+ of users see no challenge at all.
3. If the user looks borderline (new device, unusual IP, automation signals): a single Turnstile/hCaptcha tick-box appears.
4. If the borderline check fails: an audio + visual challenge with high-contrast, accessible glyphs is shown. Refresh option visible. Audio option visible. "I need help" option that routes to live support.

**Booking page captcha (covered by changes 1 + 2):**
5. User has filled 4 passengers' details. Taps Continue.
6. Background bot detection runs. If passed: form submits, user proceeds to payment. No challenge ever shown.
7. If borderline: one-tap challenge appears as a modal *over* the form (form state preserved beneath).
8. If user mistypes the (rare) visual challenge: an inline error appears at the challenge field. **All form fields below remain populated.** User retries the challenge only.

### Technical Implementation Plan

**System components affected:**
- Authentication endpoint (login)
- Booking endpoint (passenger details submission)
- Frontend forms (login form + passenger details form)
- Bot-detection edge layer (new)

**New data requirements:**
- Bot-detection score per request (logged for tuning).
- No PII added.

**API changes:**
- All form-submitting endpoints accept a `bot_token` from the bot-detection provider.
- All form-submitting endpoints return validation errors as a structured JSON response (`{ errors: { captcha: "Incorrect challenge response" }}`) rather than re-rendering an HTML template. This is the change that enables form-state preservation.

**Frontend changes:**
- Login form and passenger form converted to client-side controlled forms (state in SPA, not HTML form POST → re-render).
- Bot-detection SDK integrated at the page level.
- Inline error display at the challenge field (or at the failing field generally — this fix benefits all field validation, not just captcha).
- ARIA labels and audio fallback on any visible challenge (WCAG 2.2 AA compliance).

**Third-party services:**
- Cloudflare Turnstile (free, privacy-respecting, government-friendly) is the strong recommendation. Alternative: hCaptcha. Avoid Google reCAPTCHA on a government property due to data-localisation concerns.

### Wireframe

See **Figma brief: Login + Passenger Form with State Preservation** at the end of this spec. Saved export: `assets/wireframes/04-captcha-form-state.svg`.

### Success Metrics

- **Captcha-related abandonment** (sessions that reach the captcha step but don't submit): drops from current baseline by 80%+.
- **Average time to complete booking** (search to payment): drops by 90+ seconds for users on slow networks (no more form-redo).
- **Accessibility complaints from low-vision users**: drop by 90%+.
- **Bot detection efficacy**: false-bot-block rate < 0.1%; bot bookings caught (measurable via known scraper patterns) ≥ 95%.
- **Customer-care calls about "form keeps clearing"**: drop to near-zero.

### Edge Cases and Constraints

- **Government data-localisation rules**: bot-detection vendor must store data within India or be approved under MeitY guidelines. Cloudflare Turnstile is privacy-respecting (no cookies, no personal data) — but legal review is needed.
- **Users with assistive tech**: every visible challenge must have an audio alternative AND a "request help" path that connects to a human or to a phone-based booking flow.
- **Aggressive scrapers / agent-bot networks**: bot detection alone won't stop sophisticated bots. This spec is paired with rate limiting and agent-ID throttling (out of scope here but referenced).
- **Older browsers** (some users on legacy Android): if the bot-detection SDK can't load, fall back to the current visible captcha — but with the form-state preservation fix still applied, so even those users no longer lose form entries.
- **Backwards compatibility**: existing bookmarked deep links to login pages must continue to work; the page becomes a SPA route.
- **Graceful degradation**: if the bot-detection service is down, fall back to a visible challenge (clear, accessible) rather than blocking all bookings.

### Figma Brief — Login + Passenger Form with State Preservation

**Frame size:** `Mobile 390×844` and `Desktop 1440×900`.

**Two screens to design:**

**Screen A — Login (happy path + challenge path)**

1. **Default state**: User ID field, password field, Sign In button. No visible captcha. A small, muted line below the button: *"Protected by Turnstile."*
2. **Borderline state** (1 in 20 users): a single tick-box challenge appears between the password field and Sign In button: `[ ] I'm not a robot`. Below it: an audio icon to switch to audio challenge.
3. **Failed challenge state**: inline error at the tick-box: *"Challenge failed. Try again or use audio version."* Form fields above remain populated.

**Screen B — Passenger details form (validation failure state)**

1. **Default state**: passenger 1, passenger 2, ..., contact details, ID details. At the bottom: a small "Verify and continue" button. No visible captcha.
2. **Failed bot check state** (rare): a modal appears over the form: *"Quick check — please confirm you're a person."* with the Turnstile tick-box. Form fields are visibly preserved underneath (slightly dimmed).
3. **Failed challenge state**: inline error inside the modal. Form preserved beneath.
4. **Other validation failure states** (e.g. invalid PAN format): inline red border on the failing field, error text directly below the field. **All other fields remain populated.** This applies to every form on IRCTC, not just captcha.

**States to design:**
- Login default (no challenge)
- Login borderline (single tick-box)
- Login failed challenge (inline error, fields preserved)
- Login fully blocked (rare — directs to support phone number)
- Booking form default (no challenge)
- Booking form challenge modal (over preserved form)
- Booking form generic field validation failure (inline, preserved)

**Annotations:**
- On the "no challenge" default: *"Invisible bot detection — 95% of users see this state, never a challenge."*
- On the inline error: *"Critical: the form fields above this error remain populated. No more 25-field redo."*
- On the audio icon: *"WCAG 2.2 AA. Audio challenge available on every visible challenge."*

**Comparison annotation:** Before: full-page reload with empty fields after every captcha mistake. After: invisible by default, modal challenge if needed, inline errors that never wipe form state.

---

## Feature Spec 5: PNR Status — Plain-Language, Confirmation-Probability-Aware

### Problem Statement

PNR status is one of the most-checked pages on IRCTC — conservatively 36 lakh checks per day on web alone — yet it is buried in the footer, presented in railway jargon (`GNWL`, `PQWL`, `RLWL`), and silent on the single most important data point: **whether the chart has been prepared** (Part A, Problem 5, Steps 2, 7, 8). Waitlisted passengers — a huge segment — are forced to use third-party sites like ConfirmTKT and RailYatri to translate IRCTC's own data into actionable answers. IRCTC has effectively outsourced this user experience.

### Current State (from Part A)

Part A documented three compounding failures: PNR check is hidden in the footer (Step 2); status is shown in raw railway codes with no glossary (Step 7); chart-prepared status — the most actionable piece of information — is in small text (Step 8). The deeper miss: IRCTC has the historical waitlist-clearance data and could predict confirmation probability, but chooses to show raw codes instead.

### Proposed Solution

A redesigned PNR Status page that puts the user's actual question — *"will I get a confirmed seat?"* — at the centre. Three changes:

1. **Promote PNR status to a top-level entry point** — homepage hero card, top navigation, deep-linkable URL. Make it discoverable in two taps from any IRCTC surface.

2. **Translate every railway code into plain language** — `GNWL 12` becomes `Waitlist position 12 (General quota)` with a tooltip explaining the quota type. Hindi, English, and 8 regional languages.

3. **Add a confirmation probability indicator** — *"Likely to confirm: 70% — based on 90 days of historical data for this train, route, and class"* — with a clear visualisation (a confidence bar, not just a number) and a timeline showing when the chart will be prepared. The chart-prepared status becomes the *headline* of the page, not a footnote.

The probability prediction itself is the AI feature — see `AI-FEATURE.md` for the full spec on the model. This UI is the surface that consumes the model's output.

### Proposed User Flow — Step by Step

1. User opens irctc.co.in. Homepage shows a prominent **PNR Status** card alongside the Book Ticket card. URL: `/pnr`.
2. User taps PNR Status. Lands on `/pnr` — a sparse page with a 10-digit input and an invisible bot check (per Spec 4). No captcha visible.
3. User enters 10-digit PNR and submits.
4. Result page loads. URL becomes `/pnr/1234567890` (deep-linkable, shareable with family).
5. **Top of the page**: a single sentence in large text — *"Likely to confirm: 70%. Chart prepares in 4 hours."* with a horizontal probability bar (green at 70%).
6. **Below**: train summary, journey date, From → To, class.
7. **Per-passenger table**: each row shows passenger name (initial only for privacy), `Current status: WL 12`, plain-language translation `Waitlist position 12 (General Quota)`, and per-passenger confirmation probability. A small (i) icon explains how the prediction is calculated.
8. **Chart preparation timeline** — a small timeline showing `Booked → Now (WL 12) → Chart prep at 14:30 → Final status`. Currently-the-date marker shows where the user is on the timeline.
9. **Action buttons**: `Refresh status`, `Share status` (generates a WhatsApp-friendly summary), `Set notification` (alerts the user when status changes or chart prepares).
10. **If chart is already prepared**: the probability indicator is replaced with the final status — *"Confirmed: B4-45 Lower"* or *"Did not confirm — Waitlist 8 final."* No prediction; just the answer.

### Technical Implementation Plan

**System components affected:**
- Homepage (new prominent PNR Status entry card)
- New route: `/pnr` (input) and `/pnr/:pnr` (result)
- Existing PNR query backend
- New: confirmation-probability service (the AI feature, specified separately in `AI-FEATURE.md`)
- Notifications service (for "Set notification" action)

**New data requirements:**
- Historical waitlist-to-confirmation data: per train, per route, per class, per quota — at least 90 days, ideally 365 days. This data already exists in CRIS systems but needs to be exposed to the prediction service.
- PNR-to-notification subscription mapping for the alert feature.

**API changes:**
- New: `GET /api/pnr/:pnr` — returns the PNR status PLUS the confirmation probability (calls the prediction service internally) PLUS plain-language translations.
- New: `POST /api/pnr/:pnr/notify` — registers the user for status-change notifications.

**Frontend changes:**
- New homepage hero card (PNR Status) at parity with Book Ticket card.
- New PNR Status page (input + result).
- Probability bar component.
- Plain-language translation table for all railway codes — copy designed and reviewed by content design + railway domain experts.
- Multi-language support (Hindi, English, 8 regional).

**Third-party services:**
- The confirmation-probability model (see `AI-FEATURE.md`).
- Notification delivery (likely existing IRCTC SMS / app push infrastructure).

### Wireframe

See **Figma brief: PNR Status Page (Result View)** at the end of this spec. Saved export: `assets/wireframes/05-pnr-status.svg`.

### Success Metrics

- **PNR checks made via IRCTC vs third-party sites** (measurable via referrer / direct-traffic mix): IRCTC's share rises by 30 percentage points within 6 months.
- **Time-to-find-PNR-status from homepage**: drops from current ~25 seconds (with footer hunting) to under 10 seconds.
- **Repeat PNR checks per booking** (proxy for anxiety / unclear status): drops by 30%+ as users get better information per check.
- **Notification opt-in rate** for PNR status changes: target 40%+ of waitlisted users.
- **NPS on the PNR Status page**: target 80%+.

### Edge Cases and Constraints

- **Prediction model not yet trained for a route** (rare routes, new trains): show only the plain-language translation, with a clear note: *"Confirmation probability not available for this train."* No fake numbers.
- **Chart already prepared**: do not show probability; show final status. The model only predicts the *future*; once the chart is final, there's nothing to predict.
- **Privacy**: PNR is sensitive (links a person to a journey). Result page should not be indexed by search engines (`noindex`). Notifications must be tied to the booker's verified phone/email, not anonymous opt-in.
- **High-anxiety users may check repeatedly** — implement gentle rate limiting (e.g. one check per 60s per PNR per device) without blocking the user.
- **Dependency on CRIS for historical data**: if CRIS won't expose 90+ days of historical data, the AI feature is blocked. This must be negotiated alongside the model design.
- **Graceful degradation**: if the probability service is down, show the plain-language translation only — still a vast improvement over the current page, even without prediction.

### Figma Brief — PNR Status Page (Result View)

**Frame size:** `Mobile 390×844` and `Desktop 1440×900`.

**Layout (mobile, top to bottom):**

1. **Header** — back arrow, "PNR 1234567890" (the PNR itself), Share icon, Refresh icon.
2. **Headline block** — full-width card. Large text: *"Likely to confirm: 70%"*. Below it, in 16px: *"Chart prepares in 4 hours · 14:30 today"*. A green horizontal probability bar (full width, 8px tall) showing 70% fill. Tap on the (i) opens a modal explaining: *"Based on 90 days of historical data for this train, route, and class. 7 out of 10 similar bookings at WL 12 confirmed."*
3. **Train summary block** — train number + name, journey date, From → To, class, departure time.
4. **Passenger table** — one row per passenger (with PII partially masked):
   - Name initial + age (`R., 67y`)
   - Current status: `WL 12`
   - Plain language: `Waitlist position 12 (General Quota)`
   - Per-passenger probability bar (small, 4px)
5. **Chart preparation timeline** — horizontal timeline component:
   - `Booked` (grey, past) → `Currently WL 12` (current marker, blue) → `Chart prep 14:30` (grey, future) → `Final status` (grey, future)
6. **Action buttons** — three full-width-equal-split buttons: `Set notification` (primary), `Share`, `Refresh`. The `Set notification` opens a modal: *"Get an SMS when status changes or chart prepares."* with phone-number autofill from booking.
7. **"How predictions work" expander** — accordion at the bottom. Plain-language explanation of the model's basis (referenced to `AI-FEATURE.md`).

**Layout (desktop):**

- Two columns. Left (40%): headline block + train summary + timeline. Right (60%): passenger table + actions.

**States to design:**
- Confirmed booking (no probability needed; show final status)
- High probability (70%+ green)
- Medium probability (40–70% amber)
- Low probability (<40% red, with explicit text *"Likely will not confirm. Consider an alternate train."*)
- Chart already prepared, did not confirm (final state — no prediction, suggest alternates if available)
- Probability service down (graceful degradation — translation only, no probability bar)
- Invalid PNR (clear error with input field re-focus, no form wipe)

**Annotations:**
- On the headline block: *"Answers the user's actual question — 'will I get a seat?' — at the top of the page. Replaces buried jargon."*
- On the probability bar: *"Bar + percentage + sample-size explanation. Builds trust through transparency."*
- On the timeline: *"Chart prep is the headline event. Was a footnote — now it's surfaced."*
- On the share button: *"WhatsApp-friendly summary text — 'Bharti's PNR 1234567890 likely to confirm 70%. Chart in 4 hours.'"*

**Comparison annotation:** Before: footer-hidden entry, raw codes, chart-prep buried. After: top-nav entry, plain-language + probability, chart-prep is the headline.

---

## Feature Spec 6: Transaction-Aware Sessions + Payment Recovery Screen

### Problem Statement

When a session timer expires during the IRCTC ↔ payment-gateway ↔ bank handshake, the user lands on a generic "Session expired" login page after the bank has already debited their money — with no transaction reference, no booking status, and no recovery path (Part A, Problem 6, Steps 7–10). This is the highest-trust-damage problem of the six because it combines financial loss with information vacuum, and is most common during exactly the moments users need IRCTC most (Tatkal, slow networks, payment retry).

### Current State (from Part A)

Part A documented that IRCTC ties booking transactions to a session token that expires on a fixed timer, not tied to the in-flight transaction. Step 7 is the technical break (session expires while user is mid-payment); Step 8 is the UX break (generic login page, no recovery info). The user has lost the seat AND money is locked for 5–7 working days.

### Proposed Solution

Two changes, applied together:

1. **Transaction-aware sessions**: when a payment intent is created, the session token is bound to that intent and **does not expire while a payment transaction is in flight**, up to a maximum of 30 minutes. The session naturally ends when the payment resolves (success or definitive failure) — not on a fixed wall-clock timer.

2. **Payment recovery screen**: in the case where the session does still get killed (network failure, server-side bug, user-side issue) — instead of dumping the user on a login page, show a dedicated **Payment Recovery screen** that displays the transaction ID, the bank reference, the booking attempt details, and a clear status: *"Money debited / Ticket not yet confirmed / Resolution underway."* The screen offers concrete actions: retry without re-paying (using the transaction ID for idempotency), check refund status, or contact support with a pre-filled case ID.

The combination converts an opaque crisis into a manageable, traceable status update.

### Proposed User Flow — Step by Step

**Happy path (covered by change 1):**
1. User completes passenger form, taps Continue to Payment.
2. Payment intent is created on the IRCTC backend; session token is bound to this intent.
3. User selects UPI, completes payment on phone.
4. Even if the user takes 8 minutes to approve UPI, the session does not expire — it's bound to the in-flight transaction.
5. UPI confirms; redirect succeeds; ticket confirmation page loads.

**Recovery path (covered by change 2):**
6. (Same as 1–3) User completes payment on phone.
7. UPI confirms; bank debits; redirect to IRCTC fails (network drop, browser closed, server blip).
8. User reopens IRCTC. Lands on **Payment Recovery screen** at `/recover/:transaction_id` (URL is pre-bound to the transaction; works even from a fresh login).
9. Recovery screen shows:
   - *"Payment processed: ₹3,420 debited at 14:23. Transaction ID: TXN-2026-XYZ."*
   - *"Booking attempt: 12951 NDLS → BCT, 13 May, 3A, 4 passengers."*
   - *"Status: Confirming with Railways… (refresh in 30s)"* OR *"Status: Booked! PNR 1234567890."* OR *"Status: Booking failed. Refund initiated. Track refund."*
10. Action buttons: `Retry booking` (uses the same transaction for idempotency — no double charge), `Track refund`, `Contact support` (pre-fills case ID + transaction ID).

### Technical Implementation Plan

**System components affected:**
- Session management layer (the largest change)
- Payment gateway integration
- Booking transaction state machine
- New: `/recover/:transaction_id` route (publicly accessible by booker after re-auth)
- Refund tracking system

**New data requirements:**
- `payment_intent` table: transaction_id, user_id, booking_attempt_id, status, bank_reference, amount, created_at, last_status_update
- Session token has a new field: `bound_payment_intent_id`. When set, the session does not expire on the wall-clock timer.
- Idempotency keys for booking creation (transaction_id is the key).

**API changes:**
- New: `POST /api/payment/intent` — creates a payment intent and binds the session.
- Modified: session middleware — if session has a bound payment intent and the intent is `in_flight`, do not expire the session.
- New: `GET /api/payment/recover/:transaction_id` — returns full status of the transaction (debited, booked, refund-in-progress).
- New: `POST /api/booking/retry/:transaction_id` — idempotently retries booking with the same transaction; the bank is not re-charged.

**Frontend changes:**
- New Payment Recovery screen with status polling (every 5 seconds).
- Refund tracking widget.
- Pre-filled support form.

**Third-party services:**
- Payment gateways (Razorpay / BillDesk / etc.) must support idempotent capture and clear bank-reference exposure. Most modern Indian gateways do.

### Wireframe

See **Figma brief: Payment Recovery Screen** at the end of this spec. Saved export: `assets/wireframes/06-payment-recovery.svg`.

### Success Metrics

- **"Money debited but ticket not booked" complaints**: drop by 70%+ within 12 weeks.
- **Average refund inquiry resolution time**: drops from current several days to under 24 hours (because users now have a transaction ID to reference).
- **Successful recovery (retry without re-paying) rate**: target 60%+ of timeout cases recovered without manual intervention.
- **Customer-care call volume on payment-related issues**: drops by 40%+.
- **Trust NPS** (post-Tatkal-attempt survey question: *"If something went wrong with payment, would you trust IRCTC to fix it?"*): rises from very-low baseline to 70%+.
- **Counter-metric — payment timeout rate** (added post-peer-review): the absolute rate of `redirect-failed-after-bank-debit` events per 1,000 paid bookings. Tracked separately from the recovery metrics. **If this trends up, the recovery screen is masking a regression** and engineering must respond. Target: timeout rate trends flat or down quarter-over-quarter, regardless of how good the recovery screen looks. This is the metric that prevents the team from gaming its own dashboard.

### Edge Cases and Constraints

- **User cancels the UPI request on phone**: payment intent moves to `cancelled`; session expires gracefully; user sees a clean *"Payment cancelled. No money debited."* screen.
- **User completes payment but never returns to IRCTC**: backend reconciles via gateway webhook; ticket confirms server-side; user receives SMS + email even without ever returning to the website.
- **Multiple devices**: if the user reopens IRCTC on a different device, the recovery screen is reachable via login + transaction history.
- **Refund-pending state**: refund tracking must be honest. If RBI mandates 5–7 days for the bank, that's the timeline shown — but with a tracked status (`Initiated → Bank acknowledged → Credited`) so the user sees movement.
- **Idempotency edge cases**: if a transaction has already booked a ticket, `retry` is disabled and the screen surfaces the existing PNR.
- **Government / RBI compliance**: payment data handling must follow RBI guidelines on transaction logs and consumer disclosures. The recovery screen needs legal review.
- **Graceful degradation**: if the recovery service is down, fall back to a static screen with the transaction ID (recoverable from URL) and a phone number for support — the bare minimum to ensure the user is not left with nothing.

### Figma Brief — Payment Recovery Screen

**Frame size:** `Mobile 390×844` (the device most likely to lose connection mid-payment) and `Desktop 1440×900`.

**Layout (mobile, top to bottom):**

1. **Header** — IRCTC logo, "Payment Recovery" title. No back arrow (we want users to engage with this screen, not flee).
2. **Status block** — large card, full-width. Three states designed:
   - **Confirming** (yellow): icon spinner, headline *"Confirming your booking with Railways…"*, sub-text *"This usually takes 30 seconds. Don't close this page."*
   - **Booked** (green): check icon, headline *"Booked! PNR 1234567890"*, sub-text *"Your ticket has been confirmed."*
   - **Failed** (amber, not red — we want to convey 'recoverable, not catastrophic'): icon, headline *"Booking did not complete"*, sub-text *"Don't worry — your money is safe. Refund initiated."*
3. **Transaction details block** — always visible:
   - Amount: ₹3,420
   - Debited at: 14:23, 8 May 2026
   - Transaction ID: `TXN-2026-XYZ` (with copy button)
   - Bank reference: `RZP-987654321` (with copy button)
4. **Booking attempt block**:
   - Train: 12951 (Mumbai Rajdhani)
   - From → To: NDLS → BCT
   - Date: 13 May 2026
   - Class: 3A, 4 passengers
5. **Action buttons** — two full-width primary buttons depending on state:
   - Confirming: `Refresh status` (with auto-refresh every 5s)
   - Booked: `View ticket` (primary), `Done` (secondary)
   - Failed: `Retry booking` (primary, same transaction — clearly labelled *"No re-charge"*), `Track refund` (secondary)
6. **Support footer** — *"Something not right? Contact support — your case ID is pre-filled: TXN-2026-XYZ."* with a `Contact support` link.

**States to design:**
- Confirming (auto-polling)
- Booked (success — celebratory but not over-the-top)
- Failed — refund initiated
- Failed — refund completed (with credit confirmation)
- Failed — refund stuck > 24h (escalation path visible)
- User came back from a different device (login challenge first, then this screen)
- User's payment was actually never processed (clear "no money debited" state)

**Annotations:**
- On the URL: *"`/recover/:transaction_id` is deep-linkable. User can return to this screen from any device after re-auth."*
- On the transaction ID: *"Copy button — every support interaction starts with the user already having this ID."*
- On the `Retry booking` button: *"Idempotent — uses the existing transaction. Bank does not re-charge."*
- On the colour choice: *"Failed state is amber, not red. The crisis is recoverable; the colour should signal that."*

**Comparison annotation:** Before: dumped on login page with no information, money locked for a week. After: dedicated recovery screen with transaction ID, clear status, and a one-tap retry that doesn't double-charge.

---

## Cross-spec notes

- **Specs 4 and 6** ship together when possible — they both rely on a more modern session/state model, and Spec 4's "form state preservation" pattern applies to the recovery screen too.
- **Specs 1, 2, 4, and 6** reduce the chaos of the Tatkal window in different layers (queue UI, search clarity, no captcha resets, payment recovery). Their effects compound — measure them together where possible.
- **Spec 5** is the public-facing surface for the AI feature. The model itself is in `AI-FEATURE.md`.

---

## Peer Review Updates

After walking through the top two specs (Spec 6 — Payment Recovery and Spec 1 — Tatkal Queue) with peers, the following changes were made. Each represents a real challenge raised in review, not an editorial polish.

### Update 1 — Spec 1 (Tatkal Queue): added the "agent-bot abuse" edge case to the Edge Cases section

**Challenge raised:** *"Travel agents already use scripts to hit the Tatkal endpoint thousands of times per minute. A queue with a single token-per-user limit is trivially circumvented by farms of fake user IDs. Won't this just relocate the abuse from the booking endpoint to the user-creation endpoint?"*

**Why this was a real miss:** The original Edge Cases section mentioned travel-agent quotas (5 tokens per agent ID) but did not address the upstream attack — adversaries creating fresh user IDs en masse to dodge the per-user-id limit. The queue's fairness guarantee depends on user-id uniqueness, which is itself currently weakly enforced.

**Update applied:** Added a new edge case under Spec 1:

> **Account-farming and fake user IDs**: the per-user token limit is only as strong as IRCTC's user-id uniqueness. Adversaries can create thousands of fake accounts to bypass the cap. Mitigation: pair the queue with KYC-tier rate limiting (verified-Aadhaar accounts get full priority; unverified accounts get heavily throttled token issuance, especially during the Tatkal window). Surface this contract to legitimate users during sign-up: *"Verify your Aadhaar to get standard Tatkal access."* Out-of-scope for v1 of the queue itself, but must be roadmapped together — shipping the queue without account-farming defenses would be cosmetic.

This also affects MATRIX.md — the effort score for Spec 1 was already 5/5 (max), but the dependency on a parallel KYC-tiering effort makes the calendar even longer than originally implied. See MATRIX.md update note.

---

### Update 2 — Spec 6 (Payment Recovery): refined the Success Metrics to add a counter-metric

**Challenge raised:** *"You're optimising the recovery experience. Doesn't a beautiful recovery screen risk hiding the underlying failure rate? Someone reading the metrics post-launch could see 'recovery rate up' and conclude things are fine, while the absolute number of payment timeouts is unchanged or worse."*

**Why this was a real miss:** The original success metrics for Spec 6 measured outcomes *given* a payment timeout (recovery rate, refund inquiry resolution time, complaints). They did not measure whether the underlying timeout rate was rising or falling. A team that nails the recovery screen but ignores the timeout rate could ship a polished band-aid on a worsening wound.

**Update applied:** Added a counter-metric to Spec 6:

> **Counter-metric — payment timeout rate**: the absolute rate of `redirect-failed-after-bank-debit` events per 1,000 paid bookings. This must be tracked separately from the recovery metrics. **If this counter-metric trends up, the recovery screen is masking a regression**, and engineering must respond. Target: timeout rate trends flat or down quarter-over-quarter, regardless of how good the recovery screen looks.

This is the kind of metric that prevents a team from gaming its own dashboard.

---

### Update 3 — Spec 5 (PNR + AI): tightened the "low coverage" threshold and added a model-staleness alert

**Challenge raised** (this one came from a classmate who works with ML systems): *"Your coverage threshold of 20 historical bookings is way too low for a calibrated probability. You'd be showing users '47%' based on 20 data points — that's not a calibrated estimate, that's noise dressed up as math. And you have nothing about catching when the model's calibration drifts in production."*

**Why this was a real miss:** Two issues. First, 20 historical bookings is below the threshold where probability estimates are statistically meaningful for a binary outcome — the confidence interval at that sample size is enormous (a 50% point estimate from 20 samples has a 95% CI of roughly ±22%). Showing a single-point probability under those conditions is misleading even with a "limited data" caveat. Second, the AI feature spec mentioned weekly retraining but no production monitoring to catch when calibration drifts (which it will — railway demand patterns shift seasonally).

**Update applied:** Two changes to AI-FEATURE.md (cross-referenced here):

> 1. **Coverage threshold raised**: Medium coverage now means 50–149 similar historical bookings (up from 20–99). Below 50 → fallback to plain-language translation only, no probability shown. This is a tightening: more PNRs will fall back to "no probability shown," which is honest rather than spuriously confident.
> 2. **Production calibration monitoring**: a daily job computes Expected Calibration Error on the previous 7 days of predictions (where the chart has now resolved). If ECE crosses 7%, an automated alert fires and predictions are gated to a stricter coverage threshold or muted entirely until retraining catches up. This is the missing link between "we retrain weekly" and "we know when retraining is failing."

These changes propagate into MATRIX.md as well — the effort score for Spec 5 stays at 5/5, but the calibration-monitoring infrastructure adds about 1 week of engineering to the spec, narrowing the gap between Spec 5 and Spec 1 on calendar time.