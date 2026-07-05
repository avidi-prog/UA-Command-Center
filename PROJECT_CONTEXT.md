# PROJECT CONTEXT — UA Command Center

> Handoff document for continuing development. Read this fully before touching `index.html`.

---

## 1. Project Purpose & Core Functionality

**What it is:** A single-file, zero-backend web dashboard that acts as an automated decision-support system for a mobile User Acquisition (UA) manager. The owner drops raw quarterly CSV exports into the browser and the app re-derives every judgment automatically — the explicit design goal is *"next quarter, same schema in → same quality of judgment out, no human intervention."*

**Who it's for:** A senior UA manager running ~$50–200K/month across Meta, Google, TikTok, and Apple Search Ads (ASA), promoting apps including kitUP, Language_App, AI_Video_Gen, IQTEST360.

**It does four things:**
1. **Audit** — 12 cleaning rules run on every campaign CSV; every rule reports transparently in an Audit Log (even when it catches nothing), with counts, sample rows, and the action taken.
2. **Analyze** — channel/campaign/adset performance tables and charts with statistical-confidence tiers and color-coded targets.
3. **Recommend** — a fully deterministic $50K/month budget allocation with visible reasoning per campaign (SCALE / MAINTAIN / REDUCE / CUT) and computed 30/60/90-day revisit tripwires.
4. **Creative intelligence (Part 2)** — joins creative-performance CSVs against a hardcoded AI-generated tag taxonomy (`creativeTagsDB`) and auto-generates tag-level insights (best hook, best format, attention leaks, winning recipe, kill candidates).

---

## 2. Tech Stack

Everything lives in **one `index.html`** — no build step, no backend, no npm.

| Layer | Choice | Loaded via |
|---|---|---|
| UI framework | React 18.3.1 (UMD, production) | unpkg CDN |
| JSX compilation | Babel Standalone 7.24.7, in-browser (`<script type="text/babel" data-presets="env,react">`) | unpkg CDN |
| Styling | Tailwind CSS (CDN runtime build), mimicking Tremor's card aesthetic | cdn.tailwindcss.com |
| Charts | Recharts 2.12.7 (UMD) | unpkg CDN |
| CSV parsing | PapaParse 5.4.1 | unpkg CDN |
| Data storage | None — all state in React memory; tag DB hardcoded as a JS constant | — |

⚠️ **Critical gotcha:** the Recharts UMD build silently requires a global `PropTypes`. The `prop-types@15.8.1` script tag **must stay before** the Recharts tag or `window.Recharts` is `undefined` and the app renders nothing (no console error — it fails during the UMD factory).

Console warnings about Tailwind CDN / Babel standalone being non-production are expected and accepted for this tool.

---

## 3. Architecture & File Layout

```
/Users/vidi/TradeBot/
├── index.html            ← THE ENTIRE APPLICATION (~1,700 lines)
├── PROJECT_CONTEXT.md    ← this document
└── .claude/launch.json   ← dev-preview config: python3 -m http.server 8931
```

Input data files (not in repo):
- `/Users/vidi/ua_campaign_data.csv` — Q1 2025 campaign data (5,157 lines). Schema: `date, channel, campaign, adset, attribution_window, spend_usd, impressions, clicks, installs, cpi_usd, d1_retention, d7_retention, d30_retention, revenue_d7_usd, revenue_d30_usd, revenue_currency`
- A creative-performance CSV is **expected but not yet delivered** (tested with synthetic data). Router expects `creative_id` + a `thumbstop*` column; tolerated columns: `spend_usd|spend`, `impressions`, `installs`, `revenue_d7_usd|revenue_d7|rev_d7_usd`, `hold*`.

`index.html` internal section map (all in one `<script type="text/babel">`):

| Section | Contents |
|---|---|
| 0 | `CONFIG` — every judgment threshold, documented. **All tuning happens here.** Also `creativeTagsDB` (12 creatives) |
| 1 | Formatters (`usd`, `pct`, `num`, `safeDiv`, `parseNum`) |
| 2 | `cleanData()` — the 12-rule audit pipeline (order matters, see §5) |
| 3 | `computeMultipliers()` — D30/D7 revenue multiplier from matured, retention-intact cohorts |
| 4 | `confidenceTier()` + `aggregateBy()` — one aggregator for channel/campaign/adset grain |
| 5 | `generateInsights()` — 8 campaign-level insight detectors |
| 6 | `buildBudgetPlan()` — classification + two-stage water-fill allocator + checkpoints |
| 7 | `dailyRoasSeries()` — 7-day trailing ROAS series for the trend chart |
| 7B | Creative pipeline: `buildCreativeModel()` (join/aggregate) + `generateCreativeInsights()` (8 detectors) |
| 8 | UI primitives: `Card`, `Badge`, color helpers, `useSort`, `MetricTable` |
| 9 | Campaign views: Overview / Performance / Audit / Insights / Budget |
| 9B | Creative views: `CreativeTable`, `CreativeLibraryView` |
| 10 | App shell: `DropZone`/`FileDrop`/`MissingData`, smart CSV router, tab state |

---

## 4. Current Status

### Completed (in order)
- **Part 1** — campaign dashboard: audit pipeline, trust tiers, D30 projection, insights, budget module, 30/60/90 checkpoints. Verified against the real Q1 CSV in a live browser.
- **Part 1 revisions (domain fixes from the UA manager):**
  1. *Tracking-outage rule*: spend > 0 with 0 installs → kept as "Wasted/Outage Spend" financial line, excluded from all rate metrics. (Q1 reality: 42 rows, all TikTok_VVO_Cold, Feb 10–16, $10,149.)
  2. *Retention monotonicity*: rows violating D1 ≥ D7 ≥ D30 are flagged `retCorrupt` — spend/installs kept, but excluded from retention averages, the projection multiplier, and projected D30 ROAS. **Note:** the user described this as "D7 > D1" but the actual violation in the data is **D30 > D7** (85 raw rows; 78 after outage rows are removed first). The implemented check covers all three inversions.
  3. *Demand-capture ceiling*: campaigns matching `CONFIG.DEMAND_CAPTURE_REGEX` (`/brand|retarget/i`) are funded FIRST but hard-capped at 1× historical monthly spend regardless of ROAS; all remaining budget water-fills prospecting campaigns by projected D30 ROAS.
- **Part 2, Task 1.1** — Creative Library: `creativeTagsDB` constant, header-sniffing CSV router, pure-ES6 inner join + per-creative aggregation, performance grid, 2 charts (ROAS by visual_format, thumbstop by hook_type), 8 automated tag-insight detectors, join report (invalid rows / untagged IDs / tagged-but-absent IDs).

### Last thing worked on
The Creative Library (Part 2, Task 1.1), verified with a **synthetic** creative CSV (the real one doesn't exist yet), followed by a full regression test confirming Part 1 outputs are unchanged.

### Regression expectations (Q1 campaign CSV — verify after ANY change)
| Check | Expected |
|---|---|
| Clean rows | 5,076 (from 5,156 raw) |
| Audit: outage | 42 rows, $10,149 |
| Audit: retention inversions | 78 rows |
| Audit: duplicates / refunds / rev regressions | 26 / 12 (≈$1,579) / 5 |
| Immature cohorts (projected) | 1,661 rows; portfolio multiplier ≈ 2.22× |
| Budget total | exactly $50,000/mo |
| Demand capture pinned | Meta_Retargeting $7,618 · Google_Search_Brand $4,897 · ASA_Brand $2,932 |
| Prospecting (desc) | Meta_LAL $7,303 · Spark $5,411 · AAA $4,744 · UAC_Install $4,636 · ASA_Disc $4,423 · UAC_Action $4,421 · ASA_Competitor $3,615 (REDUCE) |
| CUT | TikTok_VVO_Cold → $0 (vanity-CPI trap: $0.43 CPI, 18.5% proj. D30 ROAS after outage cleanup) |
| Plan KPIs | Expected D7 ROAS 62.3% · projected D30 ROAS 137.4% |

---

## 5. Critical Code Blocks & Behaviors

1. **`CONFIG` is the single source of judgment.** No campaign names, dates, or this-quarter numbers are hardcoded anywhere in logic. If a threshold needs changing, change it in CONFIG, never inline.

2. **`cleanData()` rule ORDER matters.** Dedupe → negative-spend refunds → outage (spend>0, installs=0) → funnel violations → currency → retention clamp → retention monotonicity flag → rev7>rev30 floor → maturity classification. E.g., outage exclusion happens before the monotonicity check, which is why 85 raw inversions report as 78.

3. **Two-class row exclusion semantics** — don't merge them:
   - *Excluded rows* (dupes, refunds, outage, funnel, non-USD): removed from `rows` entirely; money tracked in `meta` (`refundTotal`, `outageSpend`).
   - *`retCorrupt` rows*: **stay in `rows`** — they count toward spend/CPI/CTR/CVR/ROAS-D7, but `aggregateBy()` filters them out of `projRs` (retention averages, `projRoas30`, `projSpend`) and `computeMultipliers()` skips them.

4. **D30 projection:** cohorts younger than 30 days can't have D30 revenue. `projRoas30 = (matured rev30 + immature rev7 × multiplier) / projSpend`, multiplier per-campaign when matured D7 revenue ≥ $500, else portfolio (~2.22×). The Day-60 checkpoint exists to validate this multiplier against reality.

5. **Budget allocator (`buildBudgetPlan`)** — three interacting mechanisms:
   - Verdicts from `projRoas30` tiers (`CONFIG.ROAS30`), with LOW-confidence campaigns auto-CUT.
   - Two-stage allocation: `allocateAll()` funds demand-capture at caps first, then water-fills prospecting with the remainder (`waterFill` = iterative proportional-to-ROAS with per-campaign cap clipping).
   - Min-viable pruning loop: any surviving allocation < $2,000/mo is converted to CUT and `allocateAll()` reruns until stable. **A REDUCE campaign can survive pruning in one configuration and die in another — that's intended.**

6. **Smart CSV router (`onData` in `App`):** header-sniffs every upload. `creative_id` + `/thumbstop/i` → creative model; `date`+`campaign`+`/spend/i` → campaign model; else schema error. The two models are independent state (`campaignRaw` / `creativeRaw`) and coexist.

7. **Creative join (`buildCreativeModel`):** inner join on `creative_id` against `creativeTagsDB` via `Map`; rates are impression-weighted (fallback: spend-weighted); auto-detects percent vs fraction encoding (if max rate > 1.5, divides by 100); reports untagged IDs, tagged-but-absent IDs, and dropped invalid rows.

8. **Insight engines** (Sections 5 and 7B) are arrays of gated detectors: each fires only past a relative-gap threshold and always embeds the actual numbers + sample sizes. When adding detectors, follow this pattern — no static advice strings.

9. **Test hook:** `window.__loadCSVText(text)` parses a CSV string and routes it exactly like a file upload. Used for all automated browser QA.

10. **UI quirks:** `roasColor(v * 3)` is used to color D7 ROAS with the D30 scale (deliberate ×3 heuristic). Insight severities map to `sevStyles` keys: `critical | action | warning | opportunity | info`.

---

## 6. Next Steps & Roadmap

**Immediate:**
1. **Wire in the real creative-performance CSV** when the user provides it (Part 2 continuation). The synthetic test CSV had columns `date, creative_id, channel, spend_usd, impressions, clicks, installs, thumbstop_rate_3s, hold_rate_15s, revenue_d7_usd` — confirm the real export matches; the column-synonym resolver in `buildCreativeModel` is the place to extend if not.
2. **Await Part 2, Task 1.2+** from the user (the "Creative Library" was explicitly labeled Part 2 *Task 1.1*, so more creative-analytics tasks are coming — likely creative↔campaign linkage, fatigue curves over time, or brief-generation from winning tags).

**Known gaps / improvement candidates (not yet requested — confirm before building):**
- Small-sample honesty in Creative Library: several tag groups have n=1; consider bootstrap CIs or explicit n-gating like the campaign confidence tiers.
- Creative CSVs currently have no date-series view (fatigue detection over time would need it).
- No persistence: a page refresh loses uploaded data (localStorage or File System Access API would fix).
- Production hardening if this ever ships beyond internal use: precompile JSX, replace Tailwind CDN, pin/self-host the CDN bundles.
- The `attribution_window` comparability caveat is an insight only — a normalization model was deliberately NOT built.

**Testing workflow for the next model:**
```bash
# 1. Copy test CSVs next to index.html (they are gitignored/temporary — remove after)
cp /Users/vidi/ua_campaign_data.csv /Users/vidi/TradeBot/
# 2. Serve: python3 -m http.server 8931  (or the .claude/launch.json preview config)
# 3. In the browser console:
#    fetch('/ua_campaign_data.csv').then(r=>r.text()).then(window.__loadCSVText)
# 4. Check the regression table in §4 above.
```
