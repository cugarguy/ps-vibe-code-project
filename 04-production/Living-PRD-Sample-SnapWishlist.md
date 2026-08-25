# Living PRD: Sample · "SnapWishlist" (Airbnb)

> **What this is:** a fully worked example of the eight-block Living PRD from Module 4, written for a fictional Airbnb feature called **SnapWishlist**. Use it as a reference when extracting your own PRD from your prototype. A *Living* PRD is **extracted, not imagined**: every block below describes what the prototype actually does, then layers the product reasoning on top.

**Product:** Airbnb · SnapWishlist
**Stage:** Prototype → Production transition (Module 4)
**Owner:** [your name]
**Last updated:** [date]
**Live prototype:** [Lovable share link]

---

## 01 · Product Overview
SnapWishlist lets a guest snap or upload a photo of a place they love, a friend's living room, a hotel lobby, a Pinterest screenshot, and instantly surfaces real Airbnb listings that match the *vibe* (style, palette, layout). It turns ambient inspiration into a bookable shortlist.

- **One-liner:** "Save the feeling, find the stay."
- **Surface:** new tab in the Airbnb mobile app + a web entry point from the Wishlists screen.
- **Primary action:** upload image → get a ranked set of visually-similar, available listings.

## 02 · Problem & Hypothesis
- **Problem:** Guests browse for inspiration constantly (social, travel blogs) but lose the thread between *"I love this look"* and *"here's a place I can actually book."* Search is text- and filter-first, not vibe-first.
- **Hypothesis:** *We're testing whether* image-based discovery increases wishlist-to-booking conversion for inspiration-driven travellers.
- **Risk type:** Value (will anyone use vibe-first discovery?) with a Usability follow-on (does the match feel "right"?).
- **Kill switch:** If fewer than 15% of SnapWishlist sessions produce a saved listing, the premise is wrong, pivot to a curated-collection model instead.

## 03 · User Flow & Screen Map
1. **Entry**: "SnapWishlist" tab → camera / upload sheet.
2. **Capture**: take photo or pick from library; optional caption ("cozy cabin, warm light").
3. **Processing**: visual-match spinner with 3 example tiles fading in.
4. **Results**: ranked grid of available listings with a "match %" chip and a *why it matched* line ("similar palette + open-plan layout").
5. **Refine**: chips to nudge results (Warmer · Brighter · More minimal · Near me).
6. **Save / Book**: add to a Wishlist or open the listing PDP.

*Screen map:* `Tab → Capture → Processing → Results → (Refine ↺) → PDP / Wishlist`

## 04 · Success Metrics
| Metric | Definition | Target |
|---|---|---|
| Activation | % of sessions that upload at least one image | ≥ 40% |
| Match save rate | % of result sets producing a saved listing | ≥ 15% (kill-switch floor) |
| Wishlist→booking | 30-day conversion vs. text-search baseline | +5pp |
| Refine usage | % of sessions using a refine chip | ≥ 30% (signals match quality gap) |
| Latency | image upload → results | < 3s p90 |

## 05 · Technical Reality
*(Extracted from the prototype, what the AI actually built.)*
- **Frontend:** single-page React flow; image upload via a mocked client-side handler.
- **Matching:** currently a **mocked** similarity score (random within a seeded range) returning from a static listings JSON. **No real embedding model yet.**
- **Data:** 24 hard-coded sample listings with image, palette tags, layout tags, price, availability flag.
- **State:** wishlist saves held in component state only, **not persisted** (no backend).
- **Gap to production:** needs a real image-embedding service + vector search, a listings API, and a persistence layer (see Block 08).

## 06 · Assumptions & Risks
- **Assumption:** guests can articulate a "vibe" via a single image. *Risk:* ambiguous images → low-confidence matches → trust erosion.
- **Assumption:** enough visually-distinct, *available* inventory exists in the guest's date/location window. *Risk:* thin inventory returns weak matches.
- **Risk (privacy):** user-uploaded photos may contain people/locations, needs an upload policy and on-device pre-checks.
- **Risk (cost):** embedding + vector search per upload has real per-call cost; cache aggressively.
- **Comprehension debt:** the match logic is currently a black box (mocked), this PRD is the first step in paying that debt down.

## 07 · In vs. Out of Scope
**In scope (v1):**
- Image upload + mocked-then-real visual match, ranked results, match %, refine chips, save to Wishlist.

**Out of scope (v1):**
- Multi-image / mood-board input.
- Booking inside the SnapWishlist flow (link out to existing PDP).
- Host-side tagging tools.
- Personalised ranking from past bookings (fast-follow).

## 08 · Engineering Recommendation
- **Front Door:** connect the repo to GitHub now; this PRD lives at `04-production/PRD.md`.
- **Cleanup:** refactor the mocked match handler into a named `matchService` with a clear interface so the real embedding call is a drop-in swap.
- **Memory:** move listings + wishlist saves to Supabase (PostgreSQL + Auth); replace component state.
- **Next spike:** validate one real image-embedding + vector-search path end-to-end behind a feature flag before widening inventory.
- **Handoff:** see `04-production/handoff-note.md` for the engineer-ready summary.

---
*Module 4 · Vibe Coding Certification, the prototype is the source of truth; the spec is the output.*
