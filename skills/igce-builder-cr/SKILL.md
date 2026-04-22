---
name: igce-builder-cr
description: >
  Build IGCEs for Cost-Reimbursement (CPFF/CPAF/CPIF) federal contracts
  using layered cost pool buildup with fee structure analysis. Orchestrates
  BLS OEWS, GSA CALC+, and GSA Per Diem skills. Supports SOW/PWS
  decomposition into labor categories and rate validation against CALC+
  market data. Trigger for: cost reimbursement IGCE, CPFF estimate,
  CPAF estimate, CPIF estimate, cost-plus estimate, BAA cost estimate,
  CR IGCE, cost reimbursement cost estimate, fixed fee estimate,
  award fee estimate, incentive fee estimate, fee structure analysis.
  Also trigger for cost pool buildup, fee rate analysis, share ratio,
  or CR scenario analysis. Do NOT use for FFP contracts or wrap rate
  buildup (use IGCE Builder FFP). Do NOT use for Labor Hour or T&M
  (use IGCE Builder LH/T&M). Do NOT use for grant budgets under
  2 CFR 200 (use Grant Budget Builder). Requires BLS OEWS API, GSA
  CALC+ Ceiling Rates API, and GSA Per Diem Rates API skills.
---

# IGCE Builder: Cost-Reimbursement (CPFF / CPAF / CPIF)

## Overview

This skill produces Independent Government Cost Estimates for cost-reimbursement contracts. CR contracts reimburse the contractor for allowable costs incurred plus a fee. The cost buildup is structurally similar to FFP (layered cost pools: fringe, overhead, G&A), but instead of profit-as-markup, CR contracts use a negotiated fee that varies by subtype. The IGCE estimates what those allowable costs should be and what fee structure is appropriate.

CR contracts are common outcomes from BAAs (FAR 35.016), R&D contracts, and complex requirements where the government assumes cost risk but controls it through auditable cost pools and negotiated fee structures.

**Required MCP servers:**
1. **bls-oews** -- market wage data by occupation and geography. Key tools: `get_wage_data`, `igce_wage_benchmark`, `list_common_metros`, `list_common_soc_codes`.
2. **gsa-calc** -- awarded GSA MAS ceiling hourly rates. Key tools: `suggest_contains`, `exact_search`, `keyword_search`, `igce_benchmark`, `price_reasonableness_check`.
3. **gsa-perdiem** -- federal CONUS travel lodging and M&IE. Key tools: `lookup_city_perdiem`, `estimate_travel_cost`, `get_mie_breakdown`.

**Regulatory basis:** FAR 15.402 (cost/pricing data). FAR 15.404-1(a) (cost analysis). FAR 15.404-4 (profit/fee analysis). FAR 16.301 through 16.307 (cost-reimbursement contracts). 10 USC 3322(a) (statutory fee caps).

## Operating Principle (ai-boundaries)

This skill **assembles data** and **formats documents** from reasoning the contracting officer supplies. It does NOT originate evaluative conclusions. Specifically:

- The skill pulls BLS wages, CALC+ ceiling rates, and Per Diem rates, and formats them into a workbook.
- The skill does NOT determine whether a rate is "fair and reasonable" under FAR 15.404-1. That determination is the CO's.
- The skill does NOT assert premiums (TS/SCI, OCONUS, SCIF, specialty labor) that are outside BLS/CALC+/Per Diem data. If a premium is needed and the data does not support it, the skill names the gap and hands the decision back to the CO.
- The skill does NOT draft a price reasonableness memo, a responsibility determination, or any FAR-citing signature document unless the CO has already supplied the rationale and conclusion, in which case the skill formats the CO's text into the template.
- Narrative prose (chat summaries, Methodology sheet, Rate Validation status) avoids evaluative verbs: "defensible," "reasonable," "acceptable," "competitive," "outlier." Replace with neutral positioning: "at P77 of CALC+ pool (n=X)," "within BLS P90 cost-plus-fee equivalent," "above P50 by Y%, document stacked factors in Methodology."

If you find yourself writing a conclusion about whether a number is right or wrong, stop. Present the data and let the CO conclude.

## Pre-flight: MCP dependency check

**Runs before Workflow Selection. Required on every skill trigger.**

This skill needs three MCP servers: `bls-oews`, `gsa-calc`, `gsa-perdiem`. Do not proceed to any workflow until both checks below pass.

**Check 1: MCP presence.** Verify all three are available in the current session by looking for one known tool from each:

- `bls-oews` (check for `mcp__bls-oews__detect_latest_year`)
- `gsa-calc` (check for `mcp__gsa-calc__suggest_contains`)
- `gsa-perdiem` (check for `mcp__gsa-perdiem__get_mie_breakdown`)

If any are missing, respond with:

> This skill requires the `bls-oews`, `gsa-calc`, and `gsa-perdiem` MCP servers. Missing: [list]. Install and configure them in your MCP client before using this skill.

**Check 2: API key presence.** Two of the three need API keys. Verify by lightweight ping:

- `mcp__bls-oews__detect_latest_year` (needs a BLS API key)
- `mcp__gsa-perdiem__get_mie_breakdown` (needs an api.data.gov key)
- `gsa-calc` needs no key, skip

If either ping returns an auth error or missing-key error, respond with:

> [bls-oews | gsa-perdiem] is installed but its API key is not set. This MCP needs a free API key (BLS for `bls-oews`, api.data.gov for `gsa-perdiem`). Register the key with the provider and add it to the MCP's configuration, then restart your MCP client.

Only proceed to Workflow Selection after both checks pass. Do not try to work around missing MCPs by calling APIs directly; the skill relies on MCP-guaranteed behaviors (MSA renumbering lookups, JSON path normalization, first/last day M&IE math).

## Workflow Selection

### Workflow A: Full CR IGCE Build (Default)
User needs a complete cost-reimbursement estimate. Execute Steps 1 through 9.
Triggers: "cost reimbursement IGCE," "CPFF estimate," "CPAF estimate," "CPIF estimate," "cost-plus estimate," "BAA cost estimate."

### Workflow A+: SOW/PWS-Driven CR Build
User provides a requirement document instead of structured labor inputs. Execute Step 0 first, validate, then Steps 1-9.
Triggers: "build a CR IGCE from this SOW," "price this BAA requirement," or when user provides requirement text and specifies cost-reimbursement.

Detection: If the user mentions a BAA and does not specify contract type, suggest CR as the most likely fit and confirm before proceeding.

### Workflow B: CR Rate Positioning (Data Only, No Determination)

User has proposed rates and wants to see where they sit against market data. The skill returns the data and the CO decides reasonableness. **The skill does not produce a "fair and reasonable" determination, a signed memo, or advisory text telling the CO how to negotiate.**

Triggers: "is this CR rate reasonable," "validate these cost pool rates," "check this cost proposal."

**Step 0 / GATE (MANDATORY FIRST — runs before any other Workflow B step).**

Before any analysis, scan the user's prompt for these tokens (case-insensitive): "memo", "determination", "fair and reasonable", "price reasonableness", "reasonableness memo", "draft the memo", "for the file", "contract file", "document this", "memorandum".

If ANY of those tokens appear, the ENTIRE first response must be the refusal template below, emitted verbatim. No rate analysis. No CALC+ pull. No BLS pull. No "let me start with the analysis" preamble. No offer to continue with the memo if the user provides more info in the same response. Emit the template. Stop. Wait for the user's explicit choice.

**Refusal template (emit verbatim):**

> I can pull positioning data that shows where each proposed rate sits against CALC+ ceiling rates and BLS market wages. I cannot draft a price reasonableness memo, write a "fair and reasonable" determination, or recommend negotiation positions. Those are Contracting Officer decisions under FAR 15.404-1, not skill outputs.
>
> Tell me which you want:
>
> **Option A — Positioning data only.** I produce a table: per-LCAT proposed rate, CALC+ P25/P50/P75/P90 with sample size, BLS metro burdened equivalent. No verdict. No recommendation. You draw the conclusion.
>
> **Option B — Memo template fill.** You provide your rationale (what supports or doesn't support each rate) and your determination (fair and reasonable / not fair and reasonable / declining to determine). I drop your text verbatim into the memo template, add the benchmark tables underneath, mark it DRAFT. I will not originate determinations, recommend negotiation positions, or add hedging language.
>
> Which option?

**Proceed to Steps 1-5 only after:**
- User explicitly selects Option A, OR
- User provides Option B inputs (rationale text + determination text)

**Hard prohibitions at ALL times (Option A or Option B):**
- Do NOT write "the rate is fair and reasonable" or "not fair and reasonable" unless quoting the user's Option B text verbatim.
- Do NOT label rates "competitive," "aggressive," "outlier requiring justification," "premium warrants clarification," or any equivalent evaluative phrase. Use positional language only ("at P77," "above P50 by X%," "below P25").
- Do NOT recommend negotiation positions, evaluation notices, or counter-offer dollar figures.
- Do NOT write "Summary of findings" or "Determination" sections that draw conclusions the user has not supplied.

**Workflow B steps (run only after Step 0 gate clears):**

1. Collect the vendor's proposed labor categories, fully burdened rates (or cost pool breakdown), and any scope context (metro, fee type, experience tier).

2. For each LCAT, call `mcp__gsa-calc__price_reasonableness_check(labor_category, proposed_rate, experience_min, education_level)`. The MCP returns count, min/max, median, IQR bounds, z-score, and percentile position. If sample size is below ~25 records, label the pool "directional only, not statistical validation."

3. For senior LCATs, also run the dual-pool flow (title-match + experience-match) and present both medians side-by-side.

4. If metro context matters, pull BLS OEWS for that metro via `mcp__bls-oews__get_wage_data` and present the P50/P75/P90 with the cost pool buildup at default fringe / overhead / G&A rates. This shows how the rate sits against local market labor independent of the CALC+ nationwide pool.

5. **Present a neutral positioning summary** using non-evaluative language:
   - "Rate sits at CALC+ P77 (n=7, thin-corpus directional only)."
   - "Above P50 by X%; stacked factors: [metro, seniority tier, aging, cost pool assumption]."
   - "Below P25; pool composition or LCAT alignment may warrant CO review."
   - "CR cost + fee often runs below CALC+ median because CALC+ reflects MAS ceiling (profit embedded) while CR separates cost and fee. CO sets the relevant comparison band."

6. **Stop.** Do not write "the rate is fair and reasonable." Do not recommend negotiation positions. Do not suggest "push back only if..." text. Do not label rates "competitive," "aggressive," "outlier requiring justification."

7. **Memo output (Option B path only).** If the user selected Option B at the Step 0 gate and supplied rationale + determination text, fill the memo template: benchmark tables from Steps 2-4, the user's rationale text verbatim in the findings section, the user's determination text verbatim in the Determination section. Do NOT paraphrase, do NOT add hedging, do NOT originate any conclusion text. Mark DRAFT. Use `[Contracting Officer Name]` and `[Agency]` placeholders. If Option A was selected, skip this step entirely.

## Information to Collect

Ask for everything in a single pass. Provide defaults where noted.

### Required Inputs

| Input | Description | Example |
|-------|-------------|---------|
| Labor categories | Job titles or SOC codes | Research Scientist, Data Analyst, PM |
| Performance location | City/state or metro area | Bethesda, MD |
| Staffing | Headcount per labor category | 3 researchers, 1 analyst, 1 PM |
| Hours per year | Productive hours per person (default: 1,880) | 1,880 |
| Period of performance | Base year + option years | Base + 2 OYs |
| Fee type | CPFF, CPAF, or CPIF | CPFF |
| Contract start date | For wage aging | 2026-10-01 |

### Optional Inputs (Defaults Applied If Not Provided)

| Input | Default | Notes |
|-------|---------|-------|
| Fringe rate | 32% | FICA + health + retirement + PTO + workers' comp |
| Overhead rate | 80% | Applied to labor + fringe |
| G&A rate | 12% | Applied to subtotal |
| Fee percentage (CPFF) | 8% | Fixed fee as % of estimated cost |
| Base fee (CPAF) | 3% | Minimum fee regardless of performance |
| Award fee pool (CPAF) | 7% | Max additional fee based on evaluation |
| Target fee (CPIF) | 8% | Fee at target cost |
| Share ratio (CPIF) | 80/20 gov/contractor | Applied to over/underruns |
| Min fee (CPIF) | 3% | Floor on fee |
| Max fee (CPIF) | 12% | Ceiling on fee |
| Escalation rate | 2.5%/yr | Applied to labor and travel |
| Shift coverage | Single shift | Specify 24x7 if needed; Step 0.5 computes FTE |
| Travel destinations | None | City/state per destination |
| Travel frequency | None | Trips/year per destination |
| Travel duration | None | Nights per trip (0 = day trip) |
| Number of travelers | All staff | Travelers per trip |
| Travel months | Max monthly rate | Specific months if known |
| FY for per diem | Current federal FY | Compute at build time (Oct-Sep cycle) |
| Duty station / origin | Performance location | For City Pair airfare lookup |
| NAICS code | None | Include in output if provided |
| PSC code | None | Include in output if provided |
| Partial start | Full year (12 months) | Specify months if base year is partial |

### Cost Pool Rate Guidance

Provide this when the user is unsure:

| Component | Low | Mid | High | Notes |
|-----------|-----|-----|------|-------|
| Fringe | 25% | 32% | 40% | Higher for generous benefits, union shops |
| Overhead | 60% | 80% | 120% | Higher for SCIF/cleared, large firms, R&D labs |
| G&A | 8% | 12% | 18% | Higher for large corporate structures |

### Fee Structure Reference

| Type | FAR Ref | Mechanism | Default | When Used |
|------|---------|-----------|---------|-----------|
| CPFF | 16.306 | Fixed fee set at award, unchanged by actual costs | 8% | Most common CR. R&D, studies, analysis. |
| CPAF | 16.305 | Base fee (0-3%) plus award pool (5-10%) earned on performance | 3% base + 7% pool | Performance-driven with periodic evaluation |
| CPIF | 16.304 | Target fee adjusted by share ratio; bounded by min/max | 8% target, 80/20 share | Complex work with cost uncertainty but measurable efficiency |

**Statutory fee caps:** R&D contracts: 15% of estimated cost (10 USC 3322(a)). Non-R&D: no statutory cap but 10% is the practical ceiling per agency policy.

## Constants Reference

| Constant | Value | Source |
|----------|-------|--------|
| Standard work year | 2,080 hours | 40 hrs x 52 weeks; converts annual wages to hourly |
| Default productive hours | 1,880 hours/year | 2,080 minus holidays and avg leave |
| Annual coverage hours (24x7) | 8,760 hours | 24 x 365; divide by 2,080 × availability for FTE |
| BLS wage cap (annual) | $239,200 | May 2024 OEWS reporting ceiling |
| BLS wage cap (hourly) | $115.00 | May 2024 OEWS reporting ceiling |
| OEWS data year | 2024 | May 2024 estimates |
| GSA mileage rate | $0.70/mile | CY2025 GSA POV rate |
| First/last day M&IE | 75% of full day | FTR 301-11.101 |
| City Pair fare source | GSA City Pair Program | cpsearch.fas.gsa.gov; use YCA fare |

## Orchestration Sequence

### Step 0: Requirements Decomposition (Workflow A+ Only)

Converts an unstructured SOW/PWS into structured pricing inputs.

**Process:**
1. **Sufficiency check.** Scan for six priceable elements: labor categories, staffing levels, performance location, period of performance, deliverables, and travel. Flag anything missing. Hard stop if performance location is absent. If 3+ elements missing and document under 500 words, ask user whether to proceed with assumptions or get clarification.

2. **Task decomposition.** Parse into discrete task areas with description, skill discipline, complexity, and recurring vs. finite classification.

3. **Domain triage.** Identify agency domain (DoD / IC / DOE / civilian IT / research / medical) BEFORE SOC mapping. Domain signals which SOC block applies: DOE → 17-2xxx physical engineering; IC/DoD cyber → 15-1212; civilian IT → 15-125x software/systems; research → 19-1xxx / 15-2xxx; medical → 29-xxxx.

4. **Labor category mapping.** Map tasks to SOC codes using Step 1 heuristics with domain triage result. When a task spans disciplines, map to multiple categories.

5. **Staffing estimation.** Estimate FTEs per category based on scope indicators. If 24x7 coverage is required, invoke Step 0.5. Present as ranges when ambiguous.

6. **Present decomposition table** for user validation.

7. **User validation gate (CRITICAL) - two stages, not one.** Stage A/B gate applies to Workflow A+ (SOW-driven builds). Skip for Workflow A-LH/A-TM (FFP, CR) when the user provides structured inputs (LCATs, FTE, location, PoP) — those don't need decomposition validation, go straight to build parameters if anything is missing. Do not conflate "confirm the decomposition" with "pick build parameters." Run them separately:

   **Stage A - Decomposition validation.** After presenting the decomposition table, ask the user to confirm or amend it. Use `AskUserQuestion` with options like "Decomposition looks right, proceed" / "Modify LCAT X" / "Add LCAT Y" / "Adjust FTE estimates." Response MUST END after this question. Wait for explicit confirmation before continuing.

   **Stage B - Build parameters.** Only after the decomposition is confirmed, ask the remaining parameter questions in a separate `AskUserQuestion` call: fee type ("Cost-reimbursement contracts require a fee structure. Based on [rationale], I recommend CPFF. Should I proceed with CPFF, or do you need CPAF or CPIF?"), cost pool rates, metro confirmation, contract start, shift coverage density if 24x7. Response MUST END after this question.

   DO NOT self-approve either stage. DO NOT skip Stage A to go straight to parameters. Proceeding to Step 1+ before Stage B is also answered is a skill violation. The user must affirmatively validate BOTH the decomposition and the parameters before build work begins.

### Step 0.5: Shift Coverage Staffing (If 24x7 or Multi-Shift)

If the requirement specifies 24x7 coverage, around-the-clock SOC, NOSC, help desk, or continuous monitoring, headcount must be grossed up from productive hours to coverage hours.

**Single-seat 24x7 (one analyst always on duty):**
```
annual_coverage_hours = 24 * 365 = 8,760
productive_hours_per_fte = 2,080
availability_factor = 0.50   # leave, training, turnover, overlap
single_seat_fte = 8,760 / (2,080 * availability_factor) = ~8.4 FTE
```
Simplification: use 4.2 FTE for single-seat 24x7 as the common industry convention (accounts for 50% availability + leave + overlap).

**Double-seat 24x7 (two analysts always on duty):** 8.4 FTE.

**12x5 coverage (business hours, weekdays only):** 60 hrs/wk × 52 = 3,120 annual hrs. 3,120 / 1,880 = 1.66 FTE single-seat, ~2 FTE with overlap.

**16x7 coverage (extended hours, every day):** 16 × 365 = 5,840 annual hrs. 5,840 / 1,880 × availability = ~3.1 FTE single-seat.

Document the FTE math in Sheet 6 Methodology. Do NOT quietly use 3 FTE for 24x7 coverage: that understaffs by 28%.

### Step 1: Map Labor Categories to SOC Codes

Map user job titles to SOC codes. Apply domain triage from Step 0 first.

**IT and Professional Services (most common):**

| Common Title | SOC Code | BLS Title |
|-------------|----------|-----------|
| Program Manager (general ops) | 11-1021 | General and Operations Managers |
| Program Manager (IT) | 11-3021 | Computer and Information Systems Managers |
| Program Manager (engineering) | 11-9041 | Architectural and Engineering Managers |
| Project Manager | 13-1082 | Project Management Specialists |
| Management Analyst | 13-1111 | Management Analysts |
| Systems Engineer / Analyst (IT) | 15-1211 | Computer Systems Analysts |
| Software Developer | 15-1252 | Software Developers |
| Cybersecurity / InfoSec | 15-1212 | Information Security Analysts |
| Network Architect | 15-1241 | Computer Network Architects |
| DBA | 15-1242 | Database Administrators |
| Sysadmin | 15-1244 | Network and Computer Systems Administrators |
| QA Tester | 15-1253 | Software QA Analysts and Testers |
| Help Desk | 15-1232 | Computer User Support Specialists |
| Data Scientist | 15-2051 | Data Scientists |
| Technical Writer | 27-3042 | Technical Writers |

**Physical / DOE / Defense Engineering (use these for hardware, labs, weapons systems, DOE M&O, physical infrastructure):**

| Common Title | SOC Code | BLS Title |
|-------------|----------|-----------|
| Aerospace Engineer | 17-2011 | Aerospace Engineers |
| Biomedical Engineer | 17-2031 | Biomedical Engineers |
| Chemical Engineer | 17-2041 | Chemical Engineers |
| Civil Engineer | 17-2051 | Civil Engineers |
| Electrical Engineer | 17-2071 | Electrical Engineers |
| Electronics Engineer | 17-2072 | Electronics Engineers, Except Computer |
| Environmental Engineer | 17-2081 | Environmental Engineers |
| Industrial Engineer | 17-2112 | Industrial Engineers |
| Mechanical Engineer | 17-2141 | Mechanical Engineers |
| Nuclear Engineer | 17-2161 | Nuclear Engineers |
| Petroleum Engineer | 17-2171 | Petroleum Engineers |
| Engineers, All Other (catch-all) | 17-2199 | Engineers, All Other |

**Research / Science (BAAs, R&D contracts):**

| Common Title | SOC Code | BLS Title |
|-------------|----------|-----------|
| Research Scientist (life sci) | 19-1099 | Life Scientists, All Other |
| Biochemist / Biophysicist | 19-1021 | Biochemists and Biophysicists |
| Microbiologist | 19-1022 | Microbiologists |
| Epidemiologist | 19-1041 | Epidemiologists |
| Physicist | 19-2012 | Physicists |
| Chemist | 19-2031 | Chemists |
| Statistician | 15-2041 | Statisticians |
| Mathematician | 15-2021 | Mathematicians |

When mapping is ambiguous, query multiple SOC codes and present the range. PM mapping is context-dependent: do NOT default to 11-3021 for non-IT programs.

### Step 2: Pull BLS Wage Data

**Use the BLS OEWS API skill.** For each labor category, query datatypes 04 (annual mean), 11-15 (10th through 90th percentiles) at the performance location.

**BLS series ID component breakdown (25 chars total):**
```
prefix(4) + area(7) + industry(6) + SOC(6) + datatype(2) = 25
OEU  M  +  0047900 +  000000  +  151212 +  13  = OEUM004790000000015121213
```
- Prefix OEUM = metro; OEUS = state; OEUN = national
- Area must be 7 chars (pad with leading zeros)
- SOC must be exactly 6 chars (no trailing zeros: 151212 not 15121200)
- Industry 000000 for cross-industry

Use metro-level prefix (OEUM) when available. Fall back to state (OEUS), then national (OEUN). Present the full wage distribution.

**Seniority modeling via percentiles:** When an LCAT is explicitly Junior / Mid / Senior, map to wage percentiles rather than pulling three separate SOCs:
- Junior → P25 (datatype 12)
- Mid → P50 median (datatype 13)
- Senior → P75 (datatype 14)

Pull all 5 percentiles (P10/P25/P50/P75/P90) for any multi-level LCAT. Cite which percentile was used per LCAT in methodology.

**Silent-wrong-answer traps:**
- **MSA renumbering (2024 OMB Bulletin 23-01).** If a metro query returns NO_DATA across EVERY SOC (not just one), the metro was renumbered, not suppressed. Cleveland moved from 17460 to 17410. Dayton may also have moved. Verify against the current BLS MSA list: `https://www.bls.gov/oes/current/oessrci.htm`. Do NOT fall back to state assuming occupation suppression until you've checked the code.
- **Wrong trailing zeros.** 151212 is the 6-char SOC. 15121200 is a Standard SOC 8-digit format that will fail the 25-char series ID assertion AFTER several queries have already constructed.

If BLS returns "-" with footnote code 5, the wage exceeds the $239,200 cap. Use the cap as a lower bound and flag in the narrative.

### Step 2B: Age BLS Wages to Contract Start Date

BLS OEWS data has a ~2-year lag (May 2024 estimates released April 2025). If the contract Period of Performance starts after the data reference period, the base wages must be aged forward to avoid understating costs.

```
months_gap = months between BLS data vintage (May 2024) and contract PoP start date
aging_factor = (1 + escalation_rate) ^ (months_gap / 12)
aged_annual_wage = annual_median * aging_factor
```

Example: if the contract start is 29 months after the BLS data vintage, at 2.5% escalation the aging_factor = 1.025^(29/12) = ~1.061. A $100,000 BLS median becomes $106,100 before cost pool buildup.

**Aging factor must be a cell-referenced formula in the workbook, NOT hardcoded.** Use the assumption block rows to hold BLS_vintage, contract_start, months_gap, and aging_factor. If the user changes the contract start assumption, the whole sheet must recompute correctly. See Step 8 assumption block layout.

Use the aged wage as the basis for all subsequent calculations. Document the aging adjustment in the Methodology sheet: "BLS OEWS wages (data vintage: [BLS_vintage]) aged forward [X] months to [contract start] at [escalation rate]%/yr to account for data lag."

If the user does not provide a contract start date, ask for one. If unknown, default to 6 months from today and note the assumption.

The escalation applied in Step 7 across option years starts AFTER this aging adjustment. Step 2B ages the base wage to the contract start; Step 7 escalates from that adjusted base across the period of performance. These are not double-counted.

### Step 3: Cost Pool Buildup

Build the estimated cost layer by layer for each labor category:

```
1. Direct Labor Rate    = aged_annual_wage / 2080
2. Fringe               = Direct Labor * fringe_rate
3. Labor + Fringe       = Direct Labor + Fringe
4. Overhead             = Labor_Fringe * overhead_rate
5. Subtotal             = Labor_Fringe + Overhead
6. G&A                  = Subtotal * ga_rate
7. Total Estimated Cost = Subtotal + G&A
8. Fee                  = see fee calculation by type below
9. Total Estimated Price = Total_Estimated_Cost + Fee
```

**CPFF fee:**
```
fixed_fee = total_estimated_cost * cpff_fee_rate
total_price = total_estimated_cost + fixed_fee
```
Fee is fixed at award. Does not change with actual costs.

**CPAF fee:**
```
base_fee = total_estimated_cost * cpaf_base_rate
award_fee_pool = total_estimated_cost * cpaf_pool_rate
estimated_fee = base_fee + (award_fee_pool * 0.85)   # assume 85% earned
total_price = total_estimated_cost + estimated_fee
```
For IGCE purposes, assume 85% of award fee pool earned (common convention). Note in methodology.

**CPIF fee:**
```
target_cost = total_estimated_cost
target_fee = target_cost * cpif_target_rate

# At target cost
fee_at_target = target_fee

# 10% overrun scenario
overrun_cost = target_cost * 1.10
fee_at_overrun = target_fee - (overrun_cost - target_cost) * contractor_share_over
fee_at_overrun = max(fee_at_overrun, target_cost * cpif_min_fee)

# 10% underrun scenario
underrun_cost = target_cost * 0.90
fee_at_underrun = target_fee + (target_cost - underrun_cost) * contractor_share_under
fee_at_underrun = min(fee_at_underrun, target_cost * cpif_max_fee)
```

**Three-scenario approach:** Vary each cost pool component:

| Component | Low | Mid | High |
|-----------|-----|-----|------|
| Fringe | 25% | 32% | 40% |
| Overhead | 60% | 80% | 120% |
| G&A | 8% | 12% | 18% |

Fee is calculated on each scenario's total cost. For CPIF, this produces a 3x3 matrix: 3 cost scenarios x 3 fee outcomes (underrun/target/overrun).

### Step 4: Cross-Reference Against CALC+

**Use the GSA CALC+ Ceiling Rates API skill.**

**CRITICAL: Use the correct query signature or you will get silent wrong answers.**

**Endpoint:** `https://calc.gsa.gov/api/v3/api/ceilingrates/`
**Parameter:** `keyword=` (NOT `q=`; `q=` returns the full 265K-record corpus silently)
**Fetch aggregations:** `page_size=0`. When you only need pool stats (count, min/max, percentiles) for validation, call `mcp__gsa-calc__igce_benchmark` instead of `keyword_search` — igce_benchmark returns trimmed stats without the 50KB+ labor_category/current_price aggregation buckets that blow up response size.

Match the vendor's tier in the keyword, not the aggregate title. For 'Mid Software Developer' query `Software Developer II` not `Software Developer` — the aggregate pool mixes interns through Senior levels and can falsely flag rates as +70% divergent when the tier-matched pool is +11%.

Example:
```
GET https://calc.gsa.gov/api/v3/api/ceilingrates/?keyword=Research+Scientist&page_size=0
```

**CRITICAL JSON paths:**
```python
aggs = response_json["aggregations"]
count    = aggs["wage_stats"]["count"]
min_rate = aggs["wage_stats"]["min"]
max_rate = aggs["wage_stats"]["max"]
avg_rate = aggs["wage_stats"]["avg"]
median   = aggs["histogram_percentiles"]["values"]["50.0"]   # CORRECT median
p25      = aggs["histogram_percentiles"]["values"]["25.0"]
p75      = aggs["histogram_percentiles"]["values"]["75.0"]
```

**WARNING:** Do NOT read `wage_stats` or `histogram_percentiles` from the top level. They live under `aggregations.*`. Do NOT read from `wage_percentiles` (empty when page_size=0). Always use `histogram_percentiles`.

**Dual-pool analysis for senior LCATs:** When title-match alone returns N<10, add a second query with experience-anchored keyword:
- Pool A (title-match): `keyword=Senior+Research+Scientist`
- Pool B (experience-match): `keyword=Research+Scientist+10+years`

Report both counts and medians. Use Pool A primary if N≥10; otherwise blend or cite Pool B as sanity layer.

**Rate validation band for CR (burdened cost + fee vs CALC+ median):**

| Divergence | Interpretation | Action |
|------------|---------------|--------|
| 0 to ±10% | Expected range | Accept without explanation |
| ±10 to ±25% | Cite fee structure or cost pool variance | Document in narrative |
| > ±25% | Needs explicit justification | Flag in Status column |

CR burdened rates often diverge from CALC+ more than LH/TM because CALC+ reflects MAS ceiling rates (which include contractor profit), while CR has separate cost + fee. A CR cost + fee slightly below CALC+ median is normal. Far above CALC+ median suggests inflated cost pools.

### Step 5: Pull Per Diem Rates (If Travel Required)

**Use the GSA Per Diem Rates API skill.** Query monthly lodging and M&IE for each destination.

**DoD installation → GSA per diem city crosswalk.** GSA keys per diem by civilian locality, not by military installation. Looking up "Fort Meade" or "Pentagon" directly returns empty. Translate the installation to its GSA locality BEFORE calling the MCP:

| DoD installation | GSA locality | Metro |
|---|---|---|
| Fort Meade, MD | Annapolis / Anne Arundel County, MD | Baltimore |
| Fort Belvoir, VA | Fairfax / Alexandria, VA | DC |
| Pentagon, VA | Arlington, VA (use DC rate) | DC |
| Joint Base Andrews, MD | District of Columbia | DC |
| NSA Bethesda / Walter Reed, MD | District of Columbia (composite) | DC |
| Fort Liberty (Bragg), NC | Fayetteville, NC | Fayetteville |
| Peterson SFB / Schriever SFB / Fort Carson, CO | Colorado Springs, CO | Colorado Springs |
| Wright-Patterson AFB, OH | Dayton, OH | Dayton |
| Eglin AFB, FL | Fort Walton Beach, FL | Pensacola-Crestview |
| JB San Antonio / Randolph / Lackland, TX | San Antonio, TX | San Antonio |
| Hanscom AFB, MA | Bedford / Boston, MA | Boston |
| Redstone Arsenal, AL | Huntsville, AL | Huntsville |
| Offutt AFB, NE | Omaha / Bellevue, NE | Omaha |
| Cape Canaveral / Patrick SFB, FL | Cocoa Beach / Cape Canaveral, FL | Palm Bay-Melbourne |
| JB Lewis-McChord, WA | Tacoma / Pierce County, WA | Seattle-Tacoma |

If the installation is not on this list, query `mcp__gsa-perdiem__lookup_city_perdiem` with the nearest civilian city and cross-check the result's `county` field.

**City Pair airfare (optional):** When origin and destination known, look up YCA fares at cpsearch.fas.gsa.gov. Skip if origin unknown, OCONUS, local travel, or user provides own airfare.

**Per-trip cost by trip length:**

**Standard multi-night trip (2+ nights):**
```
lodging_per_trip = nightly_rate * nights
travel_days = nights + 1
full_day_mie = mie_rate * max(0, travel_days - 2)
partial_day_mie = mie_rate * 0.75 * 2
mie_per_trip = full_day_mie + partial_day_mie
trip_total = lodging_per_trip + mie_per_trip
```

**1-night trip:**
```
lodging_per_trip = nightly_rate * 1
mie_per_trip = mie_rate * 0.75 * 2   # both days partial
trip_total = lodging_per_trip + mie_per_trip
```

**0-night day trip (same-day return):**
```
lodging_per_trip = 0                 # no overnight stay
mie_per_trip = mie_rate * 0.75       # single partial day only
trip_total = mie_per_trip
```

```
annual_travel = trip_total * trips_per_year * travelers
```

If the contract PoP start is within 6 months of the next federal fiscal year, query both FYs; if the target FY is not yet published (FY{N+1} publishes mid-August {N}), use current FY as conservative baseline and note in methodology: 'refresh on FY{N+1} publication.'

**No travel case:** If user confirms zero travel, do NOT build Sheet 5 with placeholder zeros that break SUM formulas. Use a minimal sheet with text "Travel Not Applicable" and no cell references. Sheet 1 Travel row = 0 literal.

### Step 6: Handle Multi-Location Weighting

**Option A (default blend):** Use highest median across locations per category. Use when user does not specify per-location headcount.

**Option B (weighted):** `weighted_wage = (wage_A * pct_A) + (wage_B * pct_B)`. Use when user provides split percentages.

**Option C (separate lines, DEFAULT when headcount per location is explicit):** Dedicated staff per location get separate rows. Do NOT prompt for Option A/B/C when user gave headcount like "3 FTE at Bethesda, 2 FTE at San Diego": go straight to Option C.

### Step 7: Calculate Estimated Costs by Period and Apply Escalation

**Per-period calculation:**
```
period_labor_cost = sum(burdened_rate * productive_hours * headcount) per category
period_travel = travel costs from Step 5
period_total_cost = period_labor_cost + period_travel
period_fee = period_total_cost * fee_rate (varies by CR subtype)
period_total_price = period_total_cost + period_fee
```

**Partial-year proration:**
```
prorated_hours = productive_hours * (months_in_period / 12)
prorated_travel = annual_travel * (months_in_period / 12)
```

**Escalation across option years:** `year_N_cost = base_year_cost * (1 + escalation_rate) ^ N`

Escalation applies to labor and travel. Fee rate stays constant as a percentage; dollar fee grows with escalated cost.

**Three-scenario math:** Vary cost pool components (fringe/overhead/G&A) at low/mid/high. Fee calculated on each scenario's total cost.

For CPIF, each cost scenario shows three fee outcomes (underrun/target/overrun), producing a 3x3 matrix:
```
              | Low Cost  | Mid Cost  | High Cost
Underrun Fee  | $X        | $X        | $X
Target Fee    | $X        | $X        | $X
Overrun Fee   | $X        | $X        | $X
```

Travel is identical across all scenarios.

### Step 8: Produce the CR IGCE Workbook

Generate a multi-sheet .xlsx workbook using openpyxl. Use Excel formulas for all calculations. Run recalc script (`python /mnt/skills/public/xlsx/scripts/recalc.py <file>`) before presenting.

**Environment-specific recalc handling:**
- **claude.ai web chat:** rerun `python /mnt/skills/public/xlsx/scripts/recalc.py <file>`.
- **Claude Code CLI (no LibreOffice):** the recalc script is unavailable. Instead, compute the expected grand total in Python against the raw inputs and cost pool buildup:

```python
expected = sum(
    total_price_per_fte * fte * scenario_months / 12
    for lcat in lcats
    for total_price_per_fte in [lcat.total_estimated_price_annual_per_fte]
)
```

Verify the computed total lands within 1% of your Python-side expected total. If yes, dimensional correctness is confirmed; Excel/Numbers will recalculate cell formulas on open. If no, the aging-factor cross-reference or cost-pool block-indexing shift is likely the cause.

- **macOS Claude Desktop with Numbers installed:** Numbers auto-recalculates on file open; no script needed.

**Workbook structure (7 sheets, or 6 if no travel):**

**Sheet 1: IGCE Summary.** Labor categories as rows, periods as columns. Shows Total Estimated Cost, Fee (labeled by type: "Fixed Fee," "Estimated Award Fee," or "Target Fee"), and Total Estimated Price. Travel rows below labor. Placeholder rows for Airfare, Ground Transportation, ODCs as numeric 0 (NOT text "TBD") to prevent #VALUE! errors in SUM formulas. Grand total with SUM formulas.

**Assumption cell layout (Sheet 1, rows 1-13):**
```
A1: "IGCE Assumptions (Cost-Reimbursement)"   (bold, merged A1:B1)
A2: "Fringe Rate"                              B2: 0.32     (blue, pct)
A3: "Overhead Rate"                            B3: 0.80     (blue, pct)
A4: "G&A Rate"                                 B4: 0.12     (blue, pct)
A5: "Fee Type"                                 B5: "CPFF"   (blue)
A6: "Fee Rate"                                 B6: 0.08     (blue, pct)
A7: "Escalation Rate"                          B7: 0.025    (blue, pct)
A8: "Productive Hours/Year"                    B8: 1880     (blue)
A9: "Base Year Months"                         B9: 12       (blue; <12 for partial)
A10: "BLS Vintage"                             B10: "May 2024"  (blue)
A11: "Contract Start"                          B11: 2026-10-01  (blue, date)
A12: "Months Gap"                              B12: =DATEDIF(B10,B11,"m")  (formula)
A13: "Aging Factor"                            B13: =(1+B7)^(B12/12)        (formula)
A14: (blank row separator)
A15: header row for data table
```

For CPAF: replace B6 with "Base Fee Rate" and add separate rows for Award Fee Pool Rate (0.07) and Assumed Earned % (0.85).
For CPIF: replace B6 with "Target Fee Rate" and add rows for Share Ratio (Over), Share Ratio (Under), Min Fee, Max Fee.

**Sheet 2: Cost Buildup.** One block per labor category showing BLS base through Total Price. Blocks are 20 rows each (18 content + 2 separator).

**Block layout formula:** `row(N) = 1 + (N-1) * 20` where N is the LCAT index.
- BLS Base Wage at offset +1
- Direct Labor Rate at offset +2
- Total Estimated Cost at offset +12
- Total Estimated Price at offset +17
- Implied Multiplier at offset +18 (Total Price / Direct Labor)

```
Row 2: A="BLS Base Wage (Annual)"     B=[aged annual median using =BLS*$B$13]
Row 3: A="Direct Labor Rate (Hourly)" B==B2/2080               (formula)
Row 5: A="Fringe Rate"                B==$B$2                  (formula, refs assumption)
Row 6: A="Fringe Amount"              B==B3*B5                  (formula)
Row 7: A="Labor + Fringe"             B==B3+B6                  (formula)
Row 8: A="Overhead Rate"              B==$B$3                  (formula, refs assumption)
Row 9: A="Overhead Amount"            B==B7*B8                  (formula)
Row 10: A="Subtotal"                  B==B7+B9                  (formula)
Row 11: A="G&A Rate"                  B==$B$4                  (formula, refs assumption)
Row 12: A="G&A Amount"                B==B10*B11                (formula)
Row 13: A="Total Estimated Cost"      B==B10+B12                (formula)

Fee Analysis:
Row 15: A="Fee Type"                  B==$B$5                  (formula)
Row 16: A="Fee Rate"                  B==$B$6                  (formula, refs assumption)
Row 17: A="Fee Amount"                B==B13*B16                (formula)
Row 18: A="Total Estimated Price"     B==B13+B17                (formula, bold)
Row 19: A="Implied Multiplier"        B==B18/B3                 (formula, 0.00"x")
```

For CPAF: rows 15-19 expand to base fee, award pool, assumed earned %, calculated fee, total price.
For CPIF: rows 15-21 expand to target fee, overrun/underrun scenarios with share ratio, min/max bounds.

**Annotation text gotcha:** Annotation cells (column C or D methodology notes) cannot START with `= + - @` or Excel tries to parse as a formula. Prefix with apostrophe (`'=2,080 hours/year`) or lead with a non-operator character (`"Note: 2,080 hours/year"`). Applies anywhere a cell value starts with those four characters.

**Sheet 3: Scenario Analysis.** Three cost columns (low/mid/high) with fee calculated on each. Display component rates at top. For CPIF: expand to 3x3 matrix (cost scenarios x fee outcomes). Summary row with range.

**Sheet 4: Rate Validation.** BLS burdened cost (mid) + fee vs. CALC+ distribution.
```
Row 1: "Rate Validation (CR)"
Row 3: Headers: Category | BLS Cost+Fee (mid) | CALC+ 25th | CALC+ 50th | CALC+ 75th | CALC+ Count | Divergence vs Median | Status
Row 4+: one row per category
  Divergence = (Cost_Fee - CALC_50th) / CALC_50th
  Status =
    IF(ABS(Divergence) <= 0.10, "Expected range",
    IF(ABS(Divergence) <= 0.25, "Cite fee structure or cost pool variance",
    "Needs justification"))
```

Dual-pool columns when title-match N<10: add "Pool A (Title)" and "Pool B (Experience)" median columns, cite N for each.

**Sheet 5: Travel Detail.** Formula-driven per destination (skip this sheet if no travel, use text "Travel Not Applicable" only).

**Multi-destination parameterization.** For M destinations, block N starts at row `1 + (N-1) * 17` with a 16-row content layout and 1-row separator. In-block row indices shift by `(N-1) * 17`. Do NOT hard-code one city: if the user provides "2 trips/yr to Huntsville + 4 trips/yr to San Diego," build two blocks with distinct labeled headers ("Travel Cost Detail: Huntsville, AL" and "Travel Cost Detail: San Diego, CA") and have Sheet 1 Travel rows SUM across destinations:

```
Annual travel (Sheet 1) = SUM across destinations of annual_travel_cost cell
                        = SUM('Travel Detail'!$B$14, 'Travel Detail'!$B$31, ...)
```

Single-destination single-block layout (block 1):

```
Row 3: A="Fiscal Year"           B=<current federal FY>          (blue)
Row 4: A="Nightly Lodging Rate"  B=[max monthly]                 (blue)
Row 5: A="M&IE Daily Rate"       B=[rate]                        (blue)
Row 6: A="First/Last Day M&IE"   B==B5*0.75                      (formula)
Row 7: A="Nights per Trip"       B=[nights, 0 for day trip]       (blue)
Row 8: A="Travel Days"           B==IF(B7=0,1,B7+1)              (formula)
Row 9: A="Lodging per Trip"      B==B4*B7                        (formula, 0 when nights=0)
Row 10: A="M&IE per Trip"        B==IF(B7=0,B6,B5*MAX(0,B8-2)+B6*2)  (formula)
Row 11: A="Trip Total"           B==B9+B10                       (formula)
Row 12: A="Trips per Year"       B=[trips]                       (blue)
Row 13: A="Travelers"            B=[count]                       (blue)
Row 14: A="Annual Travel Cost"   B==B11*B12*B13                  (formula, bold)
```

**Sheet 6: Methodology.** CR-specific narrative. Include: cost pool buildup with each pool explained, shift coverage FTE math if 24x7, fee type selection rationale and FAR reference, fee-specific notes (CPFF: fee fixed regardless of cost outcome; CPAF: assumed earned % and evaluation basis; CPIF: target cost/fee, share ratios, min/max), statutory fee caps (10 USC 3322(a) for R&D), FAR 16.301-16.307 references, data sources with dates, BLS vintage + aging adjustment, escalation basis, travel methodology (including 0-night day trips if applicable), exclusions, NAICS/PSC if provided.

**Sheet 7: Raw Data.** All API query parameters and responses. Record summary tables (count, percentiles, series IDs, query parameters) — NOT raw JSON dumps. A reviewer should reproduce the query from the parameters, not wade through 50KB of aggregation buckets.

**Formatting standards:**
- Blue font (RGB 0,0,255) for all user-adjustable inputs
- Black font for formula cells
- Currency: `$#,##0` with negatives in parentheses
- Percentage: `0.0%`
- Bold headers with light gray fill
- Freeze panes below assumption block (below row 14)
- Auto-size column widths
- Multiplier display: `0.00"x"`

When base year is partial, prorate labor and travel using $B$9. Full option years ignore $B$9.

Never output as .md or HTML unless explicitly requested.

### Step 9: Present the File

**Environment-specific delivery:**
- **claude.ai web chat:** copy to `/mnt/user-data/outputs/<name>.xlsx` and call `present_files([...])`.
- **Claude Code CLI:** write to `$PWD` or user-supplied path. Print the absolute path. On macOS also run `open <path>`; on Linux `xdg-open <path>`; on Windows `start "" <path>`. Do NOT try `/mnt/user-data/outputs/` — does not exist outside claude.ai.
- **macOS Claude Desktop with Numbers:** write path, run `open <path>`. Numbers auto-recalculates on open.

Do NOT skip delivery. A workbook in the sandbox that isn't surfaced looks like a silent failure.

## Edge Cases

**BAA without contract type specified:** Suggest CPFF as default (most common for R&D BAAs under FAR 35.016). Confirm with user.

**Fee exceeds statutory cap:** If calculated fee exceeds 15% for R&D or 10% practical ceiling, flag and reduce to cap. Note in methodology.

**CPIF share ratio edge cases:** If fee calculation hits min or max bound, the share ratio no longer applies in that range. Contractor's fee is capped.

**Silent-wrong-answer traps:**
- `q=` parameter on CALC+ returns the full 265K-record corpus silently. Always use `keyword=` or `search=`.
- `wage_stats` read from top level returns None. Always read from `aggregations.wage_stats`.
- MSA code renumbering (Cleveland 17460 → 17410) returns NO_DATA silently. Verify code if all datatypes return empty.
- BLS SOC with trailing zeros (151212 vs 15121200) fails the 25-char assertion AFTER you've already constructed several queries. Use exactly 6-char SOC.
- Annotation text starting with `= + - @` triggers Excel formula parse. Escape with apostrophe or lead with text.
- ODC / placeholder cells set to text "TBD" break SUM formulas. Use numeric 0 literal.

## What This Skill Does NOT Cover

Include as placeholder rows or methodology notes:
- **Airfare:** Use City Pair YCA fares when origin/destination known; otherwise numeric 0 placeholder
- **Ground transportation:** Rental cars, mileage ($0.70/mile), taxi, rideshare
- **ODCs:** Equipment, licenses, materials (user must provide; placeholders as numeric 0)
- **Subcontractor costs:** Requires separate estimate or vendor input
- **DCAA audit rates:** Actual indirect rates from contractor disclosure statements
- **OCONUS travel:** Per diem covers CONUS only; State Dept rates for OCONUS
- **FFP contracts:** Use IGCE Builder FFP
- **T&M/LH contracts:** Use IGCE Builder LH/T&M
- **Grant budgets:** Use Grant Budget Builder

## Quick Start Examples

**CPFF:** "CPFF IGCE for an R&D contract, 3 researchers in Bethesda, base plus 2 OYs" → map SOC, pull Bethesda BLS with P25/P50/P75, build cost pools with cell-referenced aging, calculate 8% fixed fee, validate against CALC+, 7-sheet xlsx.

**BAA:** "We're issuing a BAA for AI research, need a cost estimate" → confirm CR (CPFF default for BAAs), ask for labor details, run full workflow. Note FAR 35.016 in methodology.

**CPIF with 24x7 coverage:** "CPIF IGCE with 80/20 share ratio for continuous systems monitoring in Cleveland, base plus 2 OYs" → Step 0.5 → 4.2 FTE single-seat; pull Cleveland 0017410; build cost pools; produce 3x3 matrix (low/mid/high cost × underrun/target/overrun fee) bounded by min/max fee.

**Rate validation:** "Contractor proposes $195/hr cost + fee on a CPFF contract. Reasonable?" → Workflow B. Pull CALC+, run BLS cost pool buildup, position within ±10 / ±25 / outside bands (no determination).


---

*MIT © James Jenrette / 1102tools. Source: github.com/1102tools/federal-contracting-skills*
