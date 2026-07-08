# BI Handoff — Retention RevOps Scorecard

**Live prototype:** https://viome.github.io/retention-revops-dashboard/
**Source:** `index.html` in this repo (single self-contained file — HTML/CSS/vanilla JS, no build step)
**Status:** working prototype with a **static data snapshot** (as of Jun 30, 2026). The ask for BI: productionize the **data layer**. The UI, layout, and filters are settled — please don't redesign; swap the data underneath.

---

## 1. What this is

A leadership-facing scorecard: 6 top metrics (Total Revenue is the "core" card) with a period selector, plus the retention-roadmap initiatives mapped to the metric each one drives. Today every number is hardcoded into two JS objects (`PERIODS`, `INITIATIVES` — see §4). The goal state: those objects are generated from governed data on a schedule, and the metrics currently marked INTERIM/PENDING become real time-series.

---

## 2. Metric-by-metric spec

| # | Metric | Definition (as implemented) | Current source & query | Status | "Done" looks like |
|---|--------|------------------------------|------------------------|--------|-------------------|
| 1 | **Total Revenue** | `total_sales`, all sales channels, selected period | ShopifyQL: `FROM sales SHOW total_sales SINCE <start> UNTIL <end>` | LIVE (manual pull) | Automated per-period feed; reconcile to the WBR scorecard so the numbers match finance |
| 2 | **Retention Revenue %** | Subscription revenue ÷ total revenue = (`Loop Subscriptions` + `Bold Subscriptions` channels) ÷ `total_sales` | ShopifyQL: `FROM sales SHOW total_sales GROUP BY sales_channel` | LIVE (manual pull) | **See open decision §3.1** — the official definition may be broader (all returning-customer revenue). Shopify's API cannot split revenue by new-vs-returning on this store (headless checkout), so the broader calc must come from the warehouse |
| 3 | **AOV** | `average_order_value`, all channels, selected period | ShopifyQL: `FROM sales SHOW average_order_value` | LIVE (manual pull) | Automated per-period feed |
| 4 | **LTV** | All-time blended: `total_sales` (since 2017) ÷ unique customers = $195.2M ÷ 413,020 ≈ **$473** | ShopifyQL: `FROM sales SHOW total_sales, customers SINCE 2017-01-01` | LIVE (all-time constant) | Ideally a cohort-based LTV from the warehouse (24-month spend per the finance convention) instead of the blended all-time number |
| 5 | **Retest Rate** | Customers with results released for **>1 kit** ÷ customers with results for **≥1 kit** = 58,351 ÷ 343,703 = **17.0%** (all-time) | Klaviyo segments on profile property `totalRecsReleasedKits` (segments `SamCAY` / `QYB3DG`) | INTERIM | Time-windowed + eligibility-adjusted: numerator = retested in period; denominator = customers whose *first* results are ≥4 months old. Needs results-release **dates** (warehouse events or the Klaviyo date property being populated). The 17% understates true retest propensity because the denominator includes recent first-timers |
| 6 | **Attach Rate** | Released results → supplement subscription conversion = **29.1%** | Tableau, Q1 2026 (one-time read) | INTERIM | Recurring per-period feed from Tableau/warehouse with a documented definition (post-R&R attach window) |

**Period selector:** This Month · Last Month · Last Week · Q2 · YTD. Each period needs metrics 1–3 (metric 4 is constant; 5–6 as available).

---

## 3. Build list (prioritized)

1. **Lock the retention-revenue definition** — the single most important item; the 60% goal depends on it. Options on the table: (a) subscription-channel share (what the dashboard shows, ~44–49%), (b) all returning-customer revenue ÷ total (the WBR "ret revenue" figure — warehouse-only). Decision owner: **TBD (finance/BI)**. Whichever wins, expose it as a per-period feed.
2. **Per-period metric feed** — generate the `PERIODS` object (§4) from governed queries on a schedule (daily is plenty). This alone converts the dashboard from snapshot to live.
3. **Time-windowed retest rate** — see row 5 above. Blocked on results-release dates being queryable.
4. **Attach rate feed** — see row 6.
5. **Δ-since-launch methodology** — the initiatives table shows observed metric movement since each initiative launched. Today only one is measurable (While You Wait: retention 43.0% → 47.4% around its Jun 8 launch — co-movement, not attribution). To make this column trustworthy: pre/post reads per initiative at minimum, holdouts where feasible. Needs a per-initiative launch-date + metric-baseline log.
6. **(Optional) cohort LTV** — replace the blended $473 with 24-month cohort LTV.

---

## 4. Data contract

The UI reads exactly two structures. Replacing the hardcoded values with generated JSON (same shape) is the entire integration.

```js
// One entry per period button. t = % change vs prior equivalent period (null → renders "—").
const PERIODS = {
  month: { label:'This Month', sub:'Jun 1–30, 2026',
    total:     { v:'$2.52M', t:-4.8 },          // formatted value + trend %
    retention: { pct:48.8,  t:'+4.9pp' },        // share of total + trend in pp
    aov:       { v:'$163',  t:-3.2 } },
  // lastmonth / week / q2 / ytd — same shape
};

// One entry per roadmap initiative.
{ name:'While You Wait experience launch',
  status:'Launched',            // Planned | In Progress | Launched
  owner:'Lifecycle',            // Lifecycle | CX | Product | Ecom | Community | Ops
  quarter:'Q2 2026',            // launch quarter, 'TBD' if unset
  drives:'retention',           // total | retention | aov | ltv | retest | attach
  exp:'+2–4 pp retention',      // expected impact (directional estimate)
  delta:'+4.4 pp',              // observed change since launch; null → renders "—"
  note:'Launched Jun 8; …' }
```

Initiative metadata (name/status/owner/quarter) mirrors the airfocus **"Retention Plan"** workspace: status mapping Done→Launched, Now/Next→In Progress, Later→Planned; owner = the airfocus Team field. Quarters marked TBD are unset in airfocus.

---

## 5. Caveats & housekeeping

- **Expected-impact figures are modeled, directional estimates** — not commitments, not measured. Keep them labeled as such.
- **Δ since launch is co-movement**, not causal attribution, until item §3.5 lands.
- **This public GitHub Pages URL is temporary.** It exposes internal financials and should move to an internal (SSO/IP-allowlisted) host — at which point the productionized version can also drop the "static snapshot" constraint entirely and query the warehouse directly.
- The prototype intentionally shows honest gaps (TBD quarters, "not isolatable" deltas) rather than filled-in guesses. Please preserve that discipline.

**Questions:** Marissa Tullis (marissa.tullis@viome.com)
