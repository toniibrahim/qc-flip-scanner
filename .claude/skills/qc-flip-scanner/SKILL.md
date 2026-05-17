---
name: qc-flip-scanner
description: Scan the Quebec and Greater Montreal residential real estate market for live flip opportunities. Use whenever the routine is triggered to identify houses under CAD 600k in Laval, the North Shore, the South Shore, or Montreal outskirts where the asking price, realistic renovation cost, after-renovation resale value, and selling expenses leave strong net profit. Produces a graded deal table with confirmed-vs-estimated data disclosure. Always use this skill for any QC/Montreal flip, ARV, renovation cost, or deal-grading task — even when phrased loosely ("any good flips this week", "scan the South Shore", "what's available under 500k").
---

# Quebec / Greater Montreal Flip Scanner

This skill drives a scheduled cloud routine that screens the Greater Montreal residential market for flip opportunities and outputs a graded deal table. It runs autonomously: Toni is the operator, no human is in the loop during execution. Every output must therefore disclose what is **confirmed live data** vs **estimated** so a decision can be made afterwards without rerunning the scan.

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

Run these steps in order on every trigger. Stop early only if Step 3 returns zero live candidates.

### Step 1 — Sweep Listing Sources

Search the live listing portals below for active residential listings in the priority areas. Use first Firecrawl_MCP connector first and if it did not work, Use `web_search` and `web_fetch`. If the routine has Firecrawl, Playwright, or a similar MCP scraping connector, prefer it over raw `web_fetch` for JS-heavy pages like Centris.

Primary sources to query:

- Centris (`centris.ca`)
- REALTOR.ca
- RE/MAX Quebec, Royal LePage, Sutton Quebec, Via Capitale, Engel & Völkers
- DuProprio
- Individual broker / brokerage pages returned in search results

Build queries that combine: target city + price range + property type. Example: `"maison à vendre Laval 450000 600000 site:centris.ca"`.

### Step 2 — Filter Against Criteria

Drop any candidate that fails the hard filters in §3, §4, §5. Keep candidates that pass even if some data is missing — flag what is missing for §11 disclosure.

### Step 3 — Confirm Live Availability

For every surviving candidate, attempt to load the listing page directly. Mark each as one of:

- `LIVE` — listing page loads and shows active status
- `CONDITIONAL` / `PENDING` — shown explicitly on the page
- `SOLD` / `REMOVED` — drop the candidate, do not analyze
- `UNCONFIRMED` — page did not load, status not visible, or only cached snippet was available

If status is `UNCONFIRMED`, keep the candidate but flag it in the output as needing broker confirmation before any further work.

### Step 4 — Collect Listing Data

Pull from the live page (see §6 for full field list). Only record what is actually visible. Never invent values.

### Step 5 — Assess Condition & Renovation Scope

From listing photos, description, and visible mechanical/exterior cues, classify the renovation level as **Light**, **Medium**, or **Full Gut** (§7). Note any red flags (§10).

### Step 6 — Estimate ARV

Find renovated comparables in the same sector. Use the methodology in §8. Produce three resale scenarios: Conservative, Target, Optimistic.

**Critical rule:** in Quebec, sold prices are not always public. If broker-confirmed sold comparables are not in hand, the ARV is **based on public asking comparables, not confirmed sold comparables**. State this explicitly in the disclosure.

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

Pull only what is visible on the live listing page:

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
- Photo count and what the photos reveal
- Seller description
- Broker remarks (if shown)
- Listing URL and source platform

For each field, mark the **Source** (which platform), **Status** (Confirmed / Partial / Missing), and any **Notes**.

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

1. **Broker-provided Centris / MLS sold comparables** — strongest, when available.
2. **Public market comparables** — similar renovated houses currently listed, recent sold data that is publicly visible, comparable active or expired listings.
3. **Asking-price comparables only** — weakest reference. State the limitation.

### Comparable Match Criteria

Same city, same sector, similar property type, similar lot size, similar living area, similar bedrooms/bathrooms, similar basement, similar garage/driveway, similar renovation quality, similar school/neighborhood demand, recent date.

### Time Window

- Sold/listed within the last 3–6 months when possible.
- Up to 12 months if the local market is thin.

### Three Resale Scenarios

For every candidate, output:

- **Conservative ARV** — worst credible renovated resale.
- **Target ARV** — most likely renovated resale.
- **Optimistic ARV** — strong-market upside.

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

- Broker commission on resale
- Notary fees
- Welcome tax (Quebec land transfer tax / *taxe de bienvenue*)
- Financing cost
- Interest during holding period
- Insurance during holding period
- Utilities during holding period
- Municipal taxes during holding period
- School taxes during holding period
- Permit allowance
- Staging
- Marketing
- Cleaning
- Contingency
- Miscellaneous closing costs

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
- Use **Net Profit at Target ARV** in the Net Profit column.
- If a value cannot be confirmed, write `EST` next to it (e.g. `520,000 EST`) and explain in the disclosure block.
- If the candidate is `UNCONFIRMED` live, prefix the address with `⚠`.

### Section B — Per-Property Notes

For each row, a short paragraph covering:

- Why it survived the filters
- Renovation scope reasoning
- Which comparables were used and where they came from
- Key risks
- Open questions for the broker

---

## 12. Mandatory Data Quality Disclosure

Every run must end with a disclosure block that states:

- **Scan timestamp** and **trigger source** (scheduled / API / manual).
- **Sources actually reached** during this run (list which portals returned data, which were blocked or timed out).
- For each candidate:
  - Whether the listing was confirmed live (`LIVE`, `CONDITIONAL`, `UNCONFIRMED`).
  - Where the asking price came from.
  - Whether the ARV is supported by sold comparables or only by public asking comparables.
  - Whether renovation cost is based on photos only or on inspection documents.
  - Any municipal/legal item that needs broker or city confirmation.
- **Data quality level per candidate** using this scale:
  - **Level 1 — Confirmed**: directly visible from a live listing or official source.
  - **Level 2 — Strong Market Data**: broker-confirmed sold comps, assessment roll, official zoning, official flood map.
  - **Level 3 — Public Market Estimate**: active comparable listings, asking-price comps, public market examples.
  - **Level 4 — Assumption**: estimated from experience and market logic (pre-inspection reno cost, hidden repair allowance, probable demand).

A run with no `Level 1` or `Level 2` data for any candidate should be flagged at the top as **LOW-CONFIDENCE SCAN — BROKER CONFIRMATION REQUIRED BEFORE ANY ACTION**.

---

## 13. Decision Rules — What to Recommend

The Recommendation column in §11 must be one of:

- `Pursue — broker call this week` — A-grade, supported by comps.
- `Pursue with negotiation` — B-grade, numbers work only at a lower buy price; state the target buy.
- `Watch` — borderline, needs more data or a price drop.
- `Reject` — fails the final decision rule.

### Final Decision Rule

A property is worth pursuing only when **all** of the following are true:

1. It is still live and available.
2. The purchase price leaves enough room for renovation and profit.
3. Renovated resale comparables support the after-renovation value.
4. Renovation costs are realistic for Quebec contractor-market conditions.
5. Selling, financing, holding, and transaction costs are deducted.
6. The final net profit is strong enough for the risk.

If any of these fail, the recommendation is `Reject` regardless of how cheap the asking price looks.

---

## 14. Operating Rules for Autonomous Runs

Because this skill runs in a scheduled cloud routine without a human in the loop:

- **Never fabricate data.** Missing field → write `Missing` and flag in the disclosure. Never guess sold prices, lot sizes, year built, or comparable values.
- **Never claim a property is `LIVE` without loading its listing page during this run.** Search snippets are not sufficient.
- **Never treat asking-price comparables as confirmed resale value.** Always degrade the ARV confidence accordingly.
- **Always emit the disclosure block** even if it makes the run look weak. A weak honest run is more useful than a strong fabricated one.
- **If zero candidates survive**, emit the table with a single row stating `No qualifying candidates this scan window`, plus the full disclosure block listing which sources were swept.
- **If a source is blocked or rate-limited**, report it in the disclosure rather than silently skipping.
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

If any sources were blocked or rate-limited during the run, append ` — partial sweep` to the subject.

### Email Body Structure

The body is a single HTML email composed of the following blocks in this exact order:

**Block 1 — One-line headline**

A single sentence summarizing the scan, e.g.
*"Weekly QC flip sweep completed at 06:14 Kuwait time. Found 7 candidates, of which 1 grades A and 3 grade B."*

**Block 2 — Top picks (only if there are A or B grades)**

A short bulleted list of the A-grade and B-grade rows, each line:

```
{Grade} — {City sector} — {Property type} — Asking {Asking Price} — Net at Target {Net Profit} — {one-line recommendation}
```

Maximum 5 picks. If there are more, list the top 5 by Net Profit and note "(+N more in the full table)".

**Block 3 — Full deal table**

The complete table from §11 Section A, rendered as an HTML `<table>`. Right-align money columns. Use a light row stripe for readability. Do not collapse any columns.

**Block 4 — Per-property notes**

The §11 Section B per-property notes, one short paragraph per row, in the same order as the table.

**Block 5 — Disclosure block**

The full §12 disclosure, rendered as an HTML block with a thin left border so it visually separates from the analysis. Do not abbreviate. The disclosure is the audit trail and must be complete in every email.

**Block 6 — Footer**

A single line:

```
Generated by qc-flip-scanner routine · Trigger: {scheduled|API|manual} · Sources reached: {list} · Repo: {repo_url}
```

### Plain-text Fallback

Include a plain-text alternative part on the message. The table in plain text is rendered using pipe-and-dash markdown (the same format as §11). Email clients that prefer plain text will see a fully readable scan.

### Recipients

Recipients are configured in the **routine prompt**, not in this skill. The skill only formats the message — it never hardcodes an address.

### When to Send

Send the email on **every** completed run, including:

- Zero-candidate runs (still send — confirms the routine fired and the scan window was clean)
- `LOW-CONFIDENCE SCAN` runs (send with the warning flag in the subject)
- `partial sweep` runs where some sources were blocked

Do **not** send if the routine itself failed before producing the §11 + §12 output. In that case, leave the failure to the routine platform's own error reporting — a half-formed email would be worse than no email.

### Single Send Guarantee

Compose and send the email **once** per run. If the connector call fails, surface the error in the routine session log; do not retry from inside the skill (the routine platform will surface the failure).
