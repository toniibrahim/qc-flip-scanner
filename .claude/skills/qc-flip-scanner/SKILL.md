---
name: qc-flip-scanner
description: Scan the Quebec and Greater Montreal residential real estate market for live flip opportunities. Use whenever the routine is triggered to identify houses under CAD 600k in Laval, the North Shore, the South Shore, or Montreal outskirts where the asking price, realistic renovation cost, after-renovation resale value, and selling expenses leave strong net profit. Produces a graded deal table with confirmed-vs-estimated data disclosure. Always use this skill for any QC/Montreal flip, ARV, renovation cost, or deal-grading task — even when phrased loosely ("any good flips this week", "scan the South Shore", "what's available under 500k").
---

# Quebec / Greater Montreal Flip Scanner

This skill drives a scheduled cloud routine that screens the Greater Montreal residential market for flip opportunities and outputs a graded deal table. It runs autonomously: Toni is the operator, no human is in the loop during execution. Every output must therefore disclose what is **confirmed live data** vs **estimated** so a decision can be made afterwards without rerunning the scan.

---

## 0. Tool Discipline — READ FIRST, APPLIES THROUGHOUT

This section is a hard constraint on every other section. It overrides any wording elsewhere in this skill that might suggest a different tool.

### Allowed tools for fetching web data

| Purpose | Tool to use |
|---|---|
| Find listing URLs by area / price / type | `Firecrawl_MCP:firecrawl_search` |
| Discover all listing URLs on a portal section | `Firecrawl_MCP:firecrawl_map` |
| Load a specific listing page and read its contents | `Firecrawl_MCP:firecrawl_scrape` |
| Pull structured fields (price, beds, baths, etc.) from a listing | `Firecrawl_MCP:firecrawl_extract` with a JSON schema |
| Crawl a whole brokerage section | `Firecrawl_MCP:firecrawl_crawl` |
| Read the routine's own GitHub repo (broker feeds, prior scans) | the GitHub connector tools |
| Compose / send the daily email | the Gmail connector tools |

### Forbidden tools

- **`web_search` — NEVER call this tool in this skill.** Quebec real estate portals do not render usefully through search snippets, and falling back to search snippets is the failure mode this skill exists to prevent.
- **`web_fetch` — NEVER call this tool in this skill.** Centris, REALTOR.ca, DuProprio, and the major brokerages return HTTP 403 to raw fetchers. Attempting them wastes the run and risks contaminating the output with snippet text.
- **No other built-in browsing tool.** If a tool isn't in the Allowed table above, do not call it for listing or comparable data.

### What to do if Firecrawl fails

If a `firecrawl_*` call returns an error, a rate-limit response, or an empty result that looks like a block:

1. **Do not** retry with `web_search` or `web_fetch`. Those will not work and the output will be polluted.
2. Retry the same Firecrawl call once with a slightly different query (broader area, fewer filters).
3. If that also fails, mark the source as `BLOCKED` for this run and continue to the next source.
4. If **every** Firecrawl call in Step 1 fails, abort the run per §14 — emit a single-row table stating `NO LIVE DATA AVAILABLE — Firecrawl connector failed across all sources`, plus the full disclosure block. Do not fabricate.

### Verification (Claude must do this internally before Step 1)

Before starting Step 1, confirm that `firecrawl_search` and `firecrawl_scrape` are available as tools in this session. If they are not present, abort with a single-row table stating `NO LIVE-DATA CONNECTOR ATTACHED — Firecrawl MCP not configured on this routine`. Do not attempt to substitute `web_search` or `web_fetch`.

---

## 1. Mission

Identify properties where:

1. The listing is **live and available** at scan time.
2. The asking price leaves room for realistic renovation cost.
3. Renovated comparable properties support the after-renovation resale value.
4. Net profit after selling, financing, holding, and transaction costs meets the target.

A cheap property is **not** automatically a good deal. A property is worth pursuing only when renovated resale value leaves enough profit after realistic renovation and all selling-related costs.

---

## 2. Execution Workflow

Run these steps in order on every trigger. Stop early only if Step 3 returns zero live candidates. All web data flows through Firecrawl per §0 — there are no exceptions.

### Step 1 — Discover Candidate Listings via Firecrawl

For each priority area (§3), call `Firecrawl_MCP:firecrawl_search` with a natural-language query describing the target. Do **not** build a Google-style search-engine query — Firecrawl interprets natural language and dispatches its own scrapers.

**Good Firecrawl queries (use these patterns):**

- `"single-family houses for sale in Laval Quebec between 400000 and 600000 CAD"`
- `"bungalows for sale Brossard Quebec under 550000"`
- `"detached houses Saint-Eustache Quebec 450000 to 600000"`
- `"maisons à vendre Longueuil entre 400000 et 600000"`

**Sources Firecrawl will reach for you** (do not call these directly):

- Centris (centris.ca)
- REALTOR.ca
- RE/MAX Québec, Royal LePage, Sutton Québec, Via Capitale, Engel & Völkers
- DuProprio
- Individual brokerage and broker pages

For each area, run one `firecrawl_search` call. Collect the candidate listing URLs and short summaries it returns. Then for each candidate URL that looks promising, call `Firecrawl_MCP:firecrawl_scrape` to load the full page contents.

**Alternative for known portal sections:** if there's a stable Centris or REALTOR.ca search URL for a sector (e.g. a saved-search link), use `Firecrawl_MCP:firecrawl_map` to enumerate listing URLs from it, then `firecrawl_scrape` each.

**Broker feed check (Level 2 source, highest priority):** before running Firecrawl, check whether a broker MLS export exists in the repo at `/broker-feeds/`. If a file dated within the last 7 days is present, parse it as the primary candidate source. Firecrawl then supplements rather than replaces.

### Step 2 — Filter Against Criteria

Drop any candidate that fails the hard filters in §3, §4, §5. Keep candidates that pass even if some data is missing — flag what is missing for §12 disclosure.

### Step 3 — Confirm Live Availability via Firecrawl

For every surviving candidate, the listing page must have been loaded via `firecrawl_scrape` during this run. Mark each as one of:

- `LIVE` — `firecrawl_scrape` returned the listing page successfully with active-listing markers
- `CONDITIONAL` / `PENDING` — shown explicitly on the scraped page
- `SOLD` / `REMOVED` — shown on the scraped page or scrape returned a 404 / "no longer available"; drop the candidate
- `UNCONFIRMED` — `firecrawl_scrape` errored or returned suspicious content; flag for broker check, do not proceed to full analysis

A candidate may be marked `LIVE` **only if `firecrawl_scrape` was called on its URL during this run**. Search summaries from Step 1 are not sufficient evidence of live status.

### Step 4 — Collect Listing Data via Firecrawl Output

Pull from the `firecrawl_scrape` result for each `LIVE` candidate (see §6 for the full field list). For listings where structured extraction would be more reliable than parsing markdown, call `Firecrawl_MCP:firecrawl_extract` with an explicit JSON schema covering: asking_price, address, property_type, bedrooms, bathrooms, living_area_sqft, lot_size, year_built, basement, garage, taxes, municipal_assessment, photo_count, description, broker_remarks, listing_url.

Only record what `firecrawl_scrape` / `firecrawl_extract` actually returned. Never invent values. Missing field → record `Missing`, not a guess.

### Step 5 — Assess Condition & Renovation Scope

From the scraped photos, description, and visible mechanical/exterior cues in the Firecrawl output, classify renovation level as **Light**, **Medium**, or **Full Gut** (§7). Note any red flags (§10).

If photo URLs were returned but image content is not directly readable, note this limitation in the disclosure — vision-based condition assessment is not available in this routine.

### Step 6 — Estimate ARV via Firecrawl Comparable Search

Find renovated comparables in the same sector. Use `Firecrawl_MCP:firecrawl_search` again with comparable-focused queries:

- `"recently sold renovated houses Laval Quebec 2025 2026"`
- `"renovated bungalows for sale Brossard Quebec"`
- `"comparable homes 3 bedroom 2 bathroom Saint-Eustache renovated"`

Then `firecrawl_scrape` the most relevant 3–5 comparable URLs to extract pricing and feature details. Apply the comparable-match criteria in §8 to score each.

Produce three resale scenarios: Conservative, Target, Optimistic.

**Critical rule:** in Quebec, sold prices are not always public on portals. If broker-confirmed sold comparables are not in `/broker-feeds/`, the ARV is **based on public asking comparables, not confirmed sold comparables**. State this explicitly in the disclosure.

### Step 7 — Compute Profit

Apply the formulas in §9. Use realistic Quebec transaction and holding costs. Use a contractor margin of 15–20% inside the renovation cost (not 30–35%).

### Step 8 — Grade Each Deal

Apply the rubric in §10 to assign A / B / C / Reject.

### Step 9 — Emit Output

Write the table defined in §11 plus the disclosure block in §12. The disclosure block is mandatory on every run.

---

## 3. Target Locations

### Priority Areas

- **Laval** — all sectors
- **North Shore** — Blainville, Terrebonne, Sainte-Thérèse, Saint-Eustache, Boisbriand. Mirabel only if the numbers are strong.
- **South Shore** — Longueuil, Saint-Hubert, Brossard, Chambly, Boucherville. Saint-Bruno only if the price allows profit.
- **Montreal outskirts** — only when the numbers work.

### Avoid or Downgrade

- Weak-demand sectors
- Remote areas with slow resale
- Sectors with no strong renovated comparables
- Sectors where after-renovation resale is uncertain

---

## 4. Property Type

### Preferred

- Detached houses
- Bungalows
- Cottages
- Split-level houses
- Low-priced semi-detached houses
- Duplexes **only** if there is clear upside

### Reject

- Condos
- Properties with high condo fees
- Mixed-use unless analyzed separately
- Limited resale demand
- Cheap houses with no clear renovation upside

---

## 5. Asking Price Range

- **Main target:** under CAD 600,000, preferably CAD 400k–550k.
- **Above CAD 600k:** include only if the renovated ARV is clearly strong, renovation cost is controlled, comparable renovated sales support the ARV, and net profit remains attractive.

The asking price is the seller's requested price, not market value. A low asking price does not save a deal whose ARV does not support enough profit.

---

## 6. Listing Data to Collect

Pull only what `firecrawl_scrape` or `firecrawl_extract` actually returns from the live listing page:

- Asking price
- Address
- Property type
- Bedrooms
- Bathrooms
- Living area (if shown)
- Lot size
- Year built
- Basement information
- Garage / driveway
- Taxes (if shown)
- Municipal assessment (if shown)
- Photo count and what the photos reveal (text descriptions / alt attributes from the scrape)
- Seller description
- Broker remarks (if shown)
- Listing URL and source platform

For each field, mark the **Source** (which Firecrawl call returned it), **Status** (Confirmed / Partial / Missing), and any **Notes**.

---

## 7. Renovation Strategy & Cost Assumptions

### Preferred Renovation Scope

Full interior renovation when purchase price is low enough; kitchen replacement; bathroom renovation; flooring replacement; painting; basement finishing; adding a bedroom if legal and practical; adding a bathroom if layout allows; opening or improving layout; electrical upgrade if required; HVAC / heat pump / furnace replacement if required; roofing only if the purchase price already reflects it.

### Avoid

Renovation cost too high vs ARV; structural issues with unclear cost; foundation movement; major water infiltration; serious mold; major hidden risks; poor unfixable layout; very small house with no resale upside; no basement potential when price is already high.

### Renovation Cost Levels

| Level | Typical Scope |
|---|---|
| **Light** | Paint, minor flooring, minor repairs, cosmetic. Usually not preferred unless deeply discounted. |
| **Medium** | Kitchen refresh/replacement, bathroom renovation, flooring, paint, minor electrical/plumbing, basement improvement. Often the target zone. |
| **Full Gut** | New kitchen, new baths, flooring, full paint, electrical updates, plumbing modifications, HVAC, basement finishing, layout improvement. Roof/windows only if required and supported by ARV. |

**Margin:** include a contractor margin of **15% to 20%** inside the renovation cost figure. Do **not** apply a 30–35% margin unless a separate ASC-style markup strategy is explicitly being analyzed.

### Basement Potential

Strong positive: acceptable basement height, legal-bedroom-capable windows, room for bedroom/bathroom/family-room additions, existing plumbing rough-in, dry basement.

Weak / negative: no basement, crawlspace only, low ceiling, no legal-bedroom windows, water infiltration signs, foundation cracks, excavation or underpinning required.

---

## 8. ARV & Comparable Methodology

### Source Hierarchy

1. **Broker-provided Centris / MLS sold comparables** — strongest, when available. Check `/broker-feeds/` in the repo first.
2. **Firecrawl-scraped public comparables** — recently sold (if public) or actively listed renovated houses in the same sector. Found via `firecrawl_search` and loaded via `firecrawl_scrape`.
3. **Asking-price comparables only** — weakest reference. State the limitation explicitly in §12.

### Comparable Match Criteria

Same city, same sector, similar property type, similar lot size, similar living area, similar bedrooms/bathrooms, similar basement, similar garage/driveway, similar renovation quality, similar school/neighborhood demand, recent date.

### Time Window

- Sold/listed within the last 3–6 months when possible.
- Up to 12 months if the local market is thin.

### Three Resale Scenarios

For every candidate, output Conservative, Target, and Optimistic ARV.

---

## 9. Profit & Cost Formulas

### Gross Profit

```
Gross Profit = Resale Price − Purchase Price − Renovation Cost
```

### Net Profit

```
Net Profit = Gross Profit − Selling / Holding / Financing / Transaction Costs
```

### Costs to Deduct After Gross Profit

Broker commission on resale; notary fees; welcome tax (taxe de bienvenue); financing cost; interest during holding period; insurance during holding period; utilities during holding period; municipal taxes during holding period; school taxes during holding period; permit allowance; staging; marketing; cleaning; contingency; miscellaneous closing costs.

### Profit Targets

| Tier | Net Profit (before tax) |
|---|---|
| Minimum acceptable | 15–25% margin |
| Strong deal | CAD 80,000 – 100,000 |
| Excellent deal | CAD 120,000+ |
| Reject | Below realistic Minimum after all costs |

---

## 10. Deal Grading & Red Flags

### Grade Rubric

| Grade | Definition |
|---|---|
| **A** | Clear profit, strong renovated comps, controlled reno cost, good location, manageable risk, good resale demand. |
| **B** | Decent numbers, some risks, negotiable, inspection-dependent, acceptable but not exceptional. |
| **C** | Low or uncertain profit, high reno cost, weak ARV support, only interesting at a much lower purchase price. |
| **Reject** | Insufficient upside, too risky, overpriced, bad location, reno cost kills profit, ARV not supported. |

### Red Flags — Downgrade or Reject

Foundation issues; water infiltration; mold; structural movement; major roof failure; obsolete electrical needing major upgrade; oil tank issues; asbestos risk; UFFI risk; pyrite risk; flood zone risk; heritage restrictions; illegal basement bedroom; low ceiling basement; poor resale demand; small layout with no expansion potential; seller disclosure problems; no renovated comparable sales available.

---

## 11. Required Output Format

Every routine run must emit **exactly** the following two sections, in this order.

### Section A — Deal Table

| Address | Source | Status | Asking Price | Target Buy Price | Reno Level | Est. Reno Cost | ARV Conservative | ARV Target | ARV Optimistic | Gross Profit | Net Profit | Grade | Main Risk | Recommendation |
|---|---|---|---:|---:|---|---:|---:|---:|---:|---:|---:|---|---|---|

Rules:
- One row per candidate that survived Step 2.
- All money values in CAD.
- The **Source** column records which Firecrawl call produced the data (e.g. `firecrawl_search → firecrawl_scrape(centris.ca/...)`).
- Use **Net Profit at Target ARV** in the Net Profit column.
- If a value cannot be confirmed, write `EST` next to it and explain in the disclosure block.
- If the candidate is `UNCONFIRMED` live, prefix the address with `⚠`.

### Section B — Per-Property Notes

For each row, a short paragraph covering: why it survived filters; renovation scope reasoning; which comparables were used (with the Firecrawl URLs); key risks; open questions for the broker.

---

## 12. Mandatory Data Quality Disclosure

Every run must end with a disclosure block that states:

- **Scan timestamp** and **trigger source** (scheduled / API / manual).
- **Tools called this run**: list each `firecrawl_*` tool call count and any errors / rate-limits encountered.
- **Sources actually reached** during this run: which portals Firecrawl returned data for, which were blocked.
- For each candidate:
  - Whether the listing was confirmed live (`LIVE`, `CONDITIONAL`, `UNCONFIRMED`) and which `firecrawl_scrape` URL backs the claim.
  - Where the asking price came from.
  - Whether the ARV is supported by broker-feed sold comparables or only by Firecrawl-scraped asking comparables.
  - Whether renovation cost is based on photos only or on inspection documents.
  - Any municipal/legal item that needs broker or city confirmation.
- **Data quality level per candidate** using this scale:
  - **Level 1 — Confirmed**: directly returned by `firecrawl_scrape` from a live listing or official source.
  - **Level 2 — Strong Market Data**: broker-confirmed sold comps from `/broker-feeds/`, assessment roll, official zoning, official flood map.
  - **Level 3 — Public Market Estimate**: active Firecrawl-scraped comparable listings, asking-price comps.
  - **Level 4 — Assumption**: estimated from experience and market logic (pre-inspection reno cost, hidden repair allowance, probable demand).

A run with no `Level 1` or `Level 2` data for any candidate should be flagged at the top as **LOW-CONFIDENCE SCAN — BROKER CONFIRMATION REQUIRED BEFORE ANY ACTION**.

A run where Firecrawl errored on all attempted sources should be flagged as **NO LIVE DATA AVAILABLE** per §0.

---

## 13. Decision Rules — What to Recommend

The Recommendation column in §11 must be one of:

- `Pursue — broker call this week` — A-grade, supported by comps.
- `Pursue with negotiation` — B-grade, numbers work only at a lower buy price; state the target buy.
- `Watch` — borderline, needs more data or a price drop.
- `Reject` — fails the final decision rule.

### Final Decision Rule

A property is worth pursuing only when **all** of the following are true:

1. It is still live and available (`firecrawl_scrape` confirmed during this run).
2. The purchase price leaves enough room for renovation and profit.
3. Renovated resale comparables support the after-renovation value.
4. Renovation costs are realistic for Quebec contractor-market conditions.
5. Selling, financing, holding, and transaction costs are deducted.
6. The final net profit is strong enough for the risk.

If any of these fail, the recommendation is `Reject` regardless of how cheap the asking price looks.

---

## 14. Operating Rules for Autonomous Runs

Because this skill runs in a scheduled cloud routine without a human in the loop:

- **Tool discipline is non-negotiable.** Only the tools in §0 Allowed table. `web_search` and `web_fetch` are forbidden in this skill — period.
- **Never fabricate data.** Missing field → write `Missing` and flag in the disclosure. Never guess sold prices, lot sizes, year built, or comparable values.
- **Never claim a property is `LIVE` without a successful `firecrawl_scrape` call on its URL during this run.** Search summaries are not sufficient.
- **Never treat asking-price comparables as confirmed resale value.** Always degrade the ARV confidence accordingly.
- **Always emit the disclosure block** even if it makes the run look weak. A weak honest run is more useful than a strong fabricated one.
- **If zero candidates survive Step 2**, emit the table with a single row stating `No qualifying candidates this scan window`, plus the full disclosure block listing which Firecrawl queries were attempted.
- **If Firecrawl fails on all sources**, abort per §0 with `NO LIVE DATA AVAILABLE`. Do not fall back to `web_search`. A clean failure is better than fabricated confidence.
- **Prefer fewer, well-supported A/B candidates** over a long list of C-grade noise.

---

## 15. Email Delivery

When the routine is configured with a Gmail (or other email) connector and instructed to deliver results by email, package the output as follows.

### Subject Line

Format:

```
[QC Flip Scan] {YYYY-MM-DD} — {N} candidates, {A_count}A / {B_count}B / {C_count}C
```

Examples:

- `[QC Flip Scan] 2026-05-18 — 7 candidates, 1A / 3B / 3C`
- `[QC Flip Scan] 2026-05-18 — 0 candidates, no qualifying listings`
- `[QC Flip Scan] 2026-05-18 — LOW-CONFIDENCE SCAN — broker confirmation required`
- `[QC Flip Scan] 2026-05-18 — NO LIVE DATA — Firecrawl failed across all sources`

If any sources were blocked or rate-limited during the run, append ` — partial sweep` to the subject.

### Email Body Structure

The body is a single HTML email composed of the following blocks in this exact order:

**Block 1 — One-line headline** — a single sentence summarizing the scan and the trigger time.

**Block 2 — Top picks** (only if there are A or B grades) — a short bulleted list of the A-grade and B-grade rows: `{Grade} — {City sector} — {Property type} — Asking {Asking Price} — Net at Target {Net Profit} — {one-line recommendation}`. Max 5; if more, list top 5 by Net Profit and note "(+N more in the full table)".

**Block 3 — Full deal table** — the §11 Section A table as an HTML `<table>`. Right-align money columns. Light row stripe. Do not collapse any columns.

**Block 4 — Per-property notes** — §11 Section B, one short paragraph per row, in table order.

**Block 5 — Disclosure block** — the full §12 disclosure as an HTML block with a thin left border. Do not abbreviate.

**Block 6 — Footer** — a single line: `Generated by qc-flip-scanner routine · Trigger: {scheduled|API|manual} · Firecrawl calls: {count} · Repo: {repo_url}`.

### Plain-text Fallback

Include a plain-text alternative on the message. Plain table uses pipe-and-dash markdown.

### Recipients

Configured in the routine prompt, not in this skill. The skill formats; it never hardcodes an address.

### When to Send

Send the email on **every** completed run, including zero-candidate, low-confidence, and partial-sweep runs. Do **not** send if the routine itself errored before producing §11 + §12 (let the platform surface the failure).

### Single Send Guarantee

Compose and send the email **once** per run. If the Gmail connector call fails, surface the error in the routine session log; do not retry from inside the skill.
