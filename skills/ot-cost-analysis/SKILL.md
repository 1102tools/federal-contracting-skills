---
name: ot-cost-analysis
description: >
  Build should-cost estimates and price reasonableness analyses for Other
  Transaction (OT) agreements under 10 USC 4021/4022. Produces milestone-
  based cost analysis workbooks with cost-sharing breakdowns, per-milestone
  funding profiles, and price reasonableness memos. Orchestrates BLS OEWS,
  GSA CALC+, and GSA Per Diem skills for labor benchmarking. Trigger for:
  OT cost analysis, OTA cost estimate, OT price reasonableness, prototype
  cost estimate, OT should-cost, OT milestone pricing, OT cost-sharing
  analysis, OT funding profile, prototype IGCE, 10 USC 4021 cost analysis,
  milestone payment schedule, build an OT cost analysis. Also trigger when
  user has an OT project description and needs cost analysis or a performer
  proposed price needing reasonableness assessment. Do NOT use for FAR IGCEs
  (use IGCE Builder FFP, LH/T&M, or CR), OT project descriptions (use OT
  Project Description Builder), or grant budgets (use Grant Budget Builder).
---

# OT Cost Analysis

## Overview

This skill produces should-cost estimates and price reasonableness analyses for OT agreements. Unlike FAR-based IGCEs that structure costs around contract type pricing mechanisms (wrap rates, burden multipliers, cost pools), OT cost analysis structures the estimate around milestones and cost-sharing arrangements. The performer proposes a price per milestone; the government's job is to determine whether that price is reasonable and how much of it the government pays vs. the performer.

**Why this is one skill, not three:** The existing IGCE builders (FFP, LH/T&M, CR) are split because each has a structurally different calculation engine -- FFP uses layered wrap rates, LH/T&M uses a single burden multiplier, CR uses cost pools plus three fee subtypes. OTs don't have that divergence. The pricing mechanism is consistently milestone-based: estimate should-cost per milestone, apply cost-sharing ratio, compare to proposed price. Fixed-price milestones vs. cost-type milestones change one column, not the engine. Cost-sharing is a ratio, not a multi-subtype fork.

**Required L1 skills (must be installed):**
1. **BLS OEWS API** -- market wage data for labor benchmarking
2. **GSA CALC+ Ceiling Rates API** -- market rate validation
3. **GSA Per Diem Rates API** -- federal travel rates

**Required API keys (must be in user memory):**
- BLS API key (v2) for BLS OEWS
- api.data.gov key for GSA Per Diem
- CALC+ requires no key

If a key is missing, prompt the user to register: BLS at https://data.bls.gov/registrationEngine/, api.data.gov at https://api.data.gov/signup/

**Statutory basis:** 10 USC 4021 (prototype project authority), 10 USC 4022 (OT authority, cost-sharing requirements, follow-on production). NOT FAR 15.404 -- OTs are outside the FAR. Price reasonableness for OTs is based on the agreements officer's judgment informed by market data, analogous pricing, and parametric analysis, not certified cost or pricing data.

## Workflow Selection

### Workflow A: Full OT Cost Analysis (Default)
User has a milestone table (from OT Project Description Builder or user-provided) and needs a complete should-cost estimate with cost-sharing and price reasonableness analysis. Execute Steps 1 through 9.
Triggers: "build the OT cost analysis," "price this OT," "estimate costs for this prototype," "milestone cost estimate."

### Workflow A+: From Concept (No Milestone Table)
User has a prototype concept but no structured milestone table. Execute Step 0 (milestone decomposition) first, validate, then Steps 1-9.
Triggers: "estimate costs for this prototype concept," "how much should this OT cost," or when the user provides a block of prototype description text rather than a milestone table.

### Workflow B: Price Reasonableness Check
User has a performer's proposed price and wants to assess reasonableness against a should-cost estimate.
Triggers: "is this OT price reasonable," "validate this proposed price," "check this prototype proposal," "price reasonableness for OT."

**Workflow B steps:**
1. Collect the performer's proposed price (total and/or per-milestone).
2. Run Steps 1-7 to build the government's independent should-cost estimate.
3. Compare proposed vs. should-cost per milestone and in total.
4. Position each milestone: below should-cost (aggressive), within 10% (competitive), 10-25% above (premium), above 25% (requires justification).
5. Produce the full workbook with emphasis on the price reasonableness memo (Sheet 6).

## Information to Collect

Ask for everything in a single pass. Provide defaults where noted.

### Required Inputs

| Input | Description | Example |
|-------|-------------|---------|
| Milestone table | Milestone IDs, descriptions, durations, deliverables, TRL | From OT Project Description Builder or user |
| Performer proposed price | Total and/or per-milestone | $2.4M total, or $300K/M1, $500K/M2, etc. |
| Performer type | NDC, traditional, small business, consortium | NDC startup |
| Performance location | City/state or metro area | San Diego, CA |
| Period of performance | Total and per-milestone/phase | 24 months total |

### Optional Inputs (Defaults Applied If Not Provided)

| Input | Default | Notes |
|-------|---------|-------|
| Cost-sharing ratio | 1/3 performer for traditional; 0% for NDC/SB | Per 10 USC 4022(d) path |
| Cost-sharing type | Cash | Cash or in-kind |
| Burden multiplier | 2.0x | For labor benchmarking; market context, not binding |
| Escalation rate | 2.5%/yr | Applied across milestone periods |
| Consortium management fee | 5% | Only if consortium-brokered |
| Travel destinations | None | City/state per destination |
| Travel frequency | None | Trips/year per destination |
| Travel duration | None | Nights per trip |
| Materials/hardware BOM | None | Categories and estimated costs per milestone |
| Labor categories | Derived from milestone scope | SOC codes for benchmarking |
| Staffing per milestone | Derived from scope and duration | FTEs or level of effort |
| FY for per diem | Current federal FY | Oct-Sep cycle |
| Milestone payment type | Fixed-price | Fixed-price, cost-type, or mixed |
| Analogous OT pricing | None | Prior OT awards for similar prototypes |

### Cost-Sharing Guidance

Cost-sharing requirements depend on the 10 USC 4022(d) eligibility path:

| Performer Type | Cost-Share Required? | Typical Arrangement |
|---------------|---------------------|---------------------|
| NDC (significant participation) | No | Government funds 100% |
| Small business (significant participation) | No | Government funds 100% |
| Traditional + NDC team | No (NDC participation satisfies) | Government funds 100% |
| Traditional, sole (no NDC/SB) | Yes, per 4022(d)(1)(C) | 1/3 performer typical |
| Traditional (competition commitment) | No, per 4022(d)(1)(D) | Government funds 100% but must compete follow-on |

"Significant participation" is not defined by statute. It is an agreements officer determination. Common thresholds: 33%+ of work, meaningful technical contribution (not just pass-through subcontracting).

If the user doesn't know the cost-sharing arrangement, ask the performer type and derive the requirement from the table above. Default to 1/3 performer cost share for traditional contractors without NDC/SB participation.

## Constants Reference

| Constant | Value | Source |
|----------|-------|--------|
| Standard work year | 2,080 hours | 40 hrs x 52 weeks; converts annual wages to hourly |
| Default productive hours | 1,880 hours/year | 2,080 minus holidays and avg leave |
| Default burden multiplier | 2.0x | Mid-range; for benchmarking only |
| Low scenario burden | 1.8x | Lower bound |
| High scenario burden | 2.2x | Upper bound |
| Default escalation rate | 2.5% annually | Standard federal assumption |
| BLS wage cap (annual) | $239,200 | May 2024 OEWS reporting ceiling |
| BLS wage cap (hourly) | $115.00 | May 2024 OEWS reporting ceiling |
| OEWS data year | 2024 | May 2024 estimates |
| GSA mileage rate | $0.70/mile | CY2025 GSA POV rate |
| First/last day M&IE | 75% of full day | FTR 301-11.101 |
| Default cost share (traditional) | 33% performer | Common OT practice per 4022(d)(1)(C) |
| Default consortium fee | 5% | Typical consortium management fee |

## Orchestration Sequence

### Step 0: Milestone Decomposition (Workflow A+ Only)

Converts an unstructured prototype concept into a milestone table suitable for cost analysis.

**Process:**
1. **Sufficiency check.** Scan for: prototype objective, technology area, TRL indicators, schedule, deliverables, performer information. Flag anything missing. Hard stop if prototype objective is absent.

2. **TRL mapping.** Identify current TRL and target TRL from the description. Map to phases using the TRL progression table from OT Project Description Builder.

3. **Milestone derivation.** Apply the same heuristics as OT Project Description Builder Phase 1:
   - TRL 3-4: PDR, detailed design
   - TRL 4-5: build, component test, integration
   - TRL 5-6: system test, relevant-environment demo
   - TRL 6-7: operational demo, production readiness

4. **Present milestone table** for user validation:
```
Milestone | Phase | Description                  | Est. Duration | TRL In/Out | Payment Type
M1        | 1     | Preliminary Design Review    | 3 months      | 3/4        | Fixed
M2        | 2     | Prototype Build Complete     | 4 months      | 4/5        | Fixed
M3        | 3     | System Demonstration         | 3 months      | 5/6        | Fixed
```

5. **User validation gate.** Confirm before proceeding to Step 1.

### Step 1: Validate Milestone Structure

Before costing, verify the milestone table is internally consistent:

- TRL progression has no gaps (each milestone's exit TRL equals the next milestone's entry TRL or is within the same phase)
- Total duration of milestones sums to within 10% of stated total PoP
- Each milestone has at least one deliverable
- Each milestone has a payment type (fixed or cost-type)
- No single milestone exceeds 40% of total estimated value (too much risk concentration)

Flag any issues for user resolution before proceeding.

### Step 2: Labor Benchmarking

**Use the BLS OEWS API skill.** OT labor benchmarking serves a different purpose than FAR IGCE labor pricing. In a FAR IGCE, BLS wages are the basis for the government's cost estimate. In an OT cost analysis, BLS wages provide market context for evaluating whether the performer's proposed staffing costs are reasonable. The government is not setting rates; it is checking whether proposed rates are within market range.

**Step 2a: Map labor categories to SOC codes.** Use the same SOC mapping table as the IGCE builders (see IGCE Builder LH/T&M Step 1 for the common mapping table). If the milestone table includes labor categories, map those. If not, infer from milestone scope:

| Milestone Type | Typical Labor |
|---------------|---------------|
| Design phase | Systems engineer, software architect, mechanical engineer |
| Build/fabrication | Software developer, hardware engineer, technician |
| Test | Test engineer, QA analyst, data analyst |
| Integration | Systems integrator, middleware engineer, DevOps |
| Project oversight | Program manager, principal investigator |

**Step 2b: Pull BLS wages.** For each labor category, query BLS OEWS at the performance location. Use metro-level prefix (OEUM) when available. If the MSA returns no data (small metro suppression), fall back to state-level (OEUS), then national (OEUN). Document the fallback in the methodology: "BLS OEWS data for [MSA] was suppressed for [SOC codes]. [State/National] median used as benchmark." Use median (50th percentile) as the default benchmarking basis.

**Step 2c: Age wages.** Same aging methodology as IGCE builders:
```
months_gap = months between BLS data vintage (May 2024) and agreement start
aging_factor = (1 + escalation_rate) ^ (months_gap / 12)
aged_wage = annual_median * aging_factor
```

**Step 2d: Apply burden multiplier.** Convert to burdened rates for benchmarking:
```
hourly_base = aged_wage / 2080
burdened_rate = hourly_base * burden_multiplier
```

**Important caveat for OTs:** Document in the methodology that BLS wages and burden multipliers are market benchmarks, not the pricing basis. OT performers -- especially NDCs -- may have cost structures that differ significantly from traditional defense contractors. A startup may have low overhead but high equity-based compensation. A university lab may have high fringe but no profit margin. The burden multiplier is a sanity-check tool, not an audit standard. OTs do not require DCAA-auditable indirect rates.

### Step 3: Cross-Reference Against CALC+

**Use the GSA CALC+ Ceiling Rates API skill.** Same methodology as IGCE builders:

```python
aggs = response_json["aggregations"]
median   = aggs["histogram_percentiles"]["values"]["50.0"]
p25      = aggs["histogram_percentiles"]["values"]["25.0"]
p75      = aggs["histogram_percentiles"]["values"]["75.0"]
```

**WARNING:** Do NOT read from `wage_percentiles` (empty when page_size=0). Always use `histogram_percentiles`.

Compare BLS burdened rate to CALC+ median. Flag divergence >25%. Note in methodology: CALC+ reflects GSA MAS ceiling rates for traditional contractors; OT performers (especially NDCs) are not bound by GSA schedule pricing.

### Step 4: Materials and Hardware Estimation

For prototype OTs, materials and hardware are often the dominant cost element (40-70% for hardware prototypes). This is a first-class estimation step, not an afterthought.

**If user provides a BOM or materials list:**
- Create rows per materials category per milestone
- Apply escalation across milestones if multi-year
- Materials are at cost (no burden applied)

**If user provides milestone scope but no materials specifics:**
Estimate materials using prototype-type heuristics:

| Prototype Type | Materials as % of Total | Common Categories |
|---------------|------------------------|-------------------|
| Software only | 10-20% | Cloud hosting, licenses, test infrastructure |
| Software + hardware integration | 20-40% | Components, test equipment, cloud, licenses |
| Hardware prototype (single unit) | 40-60% | Raw materials, components, fabrication, test equipment |
| Hardware prototype (multiple units) | 50-70% | All above, scaled by unit count |
| Process/workflow prototype | 5-15% | Software tools, test environments |

Present the estimated materials breakdown for user validation. Mark as "estimated from prototype-type heuristic; refine with performer input" in methodology.

**If user says materials exist but has no estimates:** Include placeholder rows with "TBD" per milestone. Note that cost analysis is incomplete without materials estimates for hardware prototypes.

### Step 5: Travel Estimation (If Applicable)

**Use the GSA Per Diem Rates API skill.** Same methodology as IGCE builders.

OT travel is typically lighter than traditional contract travel but may include:
- Technical interchange meetings (performer site to government site)
- Test events (travel to government test ranges or facilities)
- Demonstrations (travel to operational environments)
- Program reviews

Allocate travel costs to specific milestones rather than spreading evenly across the PoP. Test milestones typically have higher travel costs than design milestones.

### Step 6: Should-Cost Buildup per Milestone

Build the government's independent should-cost estimate for each milestone:

```
milestone_labor = sum(burdened_rate * hours_per_category) for each labor category in that milestone
milestone_materials = sum(materials_costs) allocated to that milestone
milestone_travel = travel_costs allocated to that milestone
milestone_odcs = other direct costs allocated to that milestone

milestone_should_cost = milestone_labor + milestone_materials + milestone_travel + milestone_odcs
```

**Labor hours per milestone:** Estimate based on milestone scope, duration, and staffing:
```
milestone_labor_hours = productive_hours_per_year * (milestone_duration_months / 12) * FTEs_for_milestone
```

If the user provides per-milestone staffing, use it directly. If not, derive from scope heuristics:

| Milestone Type | Typical Team Size | Notes |
|---------------|------------------|-------|
| Design review | 2-4 engineers + 1 PM | Shorter duration, intensive effort |
| Build/fabrication | 3-8 engineers + 1-2 techs + 1 PM | Depends on complexity |
| Component test | 2-4 test engineers + 1-2 subject matter experts | May overlap with build |
| System integration | 3-6 engineers + 1 PM | Cross-discipline team |
| System demonstration | 2-4 engineers + 1 PM + travel team | Often shorter but high-tempo |

Present the per-milestone should-cost breakdown for user review before applying cost-sharing.

### Step 7: Cost-Sharing Calculation

Apply cost-sharing based on the performer type and arrangement:

**No cost share (NDC/SB participation or competition commitment path):**
```
government_obligation = milestone_should_cost
performer_share = $0
```

**Cost share required (traditional, sole, no NDC/SB):**
```
performer_share = milestone_should_cost * cost_share_ratio    # default 0.33
government_obligation = milestone_should_cost * (1 - cost_share_ratio)
```

**In-kind cost share:**
```
# In-kind means performer contributes resources not charged to the agreement
# The agreement value reflects only the government's obligation
# Document the in-kind contribution description but don't subtract from should-cost
government_obligation = milestone_should_cost * (1 - cost_share_ratio)
in_kind_value = milestone_should_cost * cost_share_ratio
# Note: in-kind requires the performer to track and report their contribution
```

**Consortium-brokered OT:**
```
consortium_fee = milestone_should_cost * consortium_fee_rate    # default 0.05
government_obligation = (milestone_should_cost * (1 - cost_share_ratio)) + consortium_fee
```

**Cumulative funding profile:** Sum government obligations across milestones in chronological order. This is the government's funding requirement, showing when money needs to be available.

### Step 8: Scenario Analysis

Three scenarios varying labor burden and materials estimates:

```
low_should_cost  = labor_at_low_burden + materials_low + travel + odcs
mid_should_cost  = labor_at_mid_burden + materials_mid + travel + odcs
high_should_cost = labor_at_high_burden + materials_high + travel + odcs
```

**Scenario multipliers:**

| Component | Low | Mid | High |
|-----------|-----|-----|------|
| Labor burden | 1.8x | 2.0x | 2.2x |
| Materials | Estimate * 0.85 | Estimate * 1.0 | Estimate * 1.20 |
| Travel | Same across scenarios | Same | Same |

Materials variance is wider for hardware prototypes (fabrication uncertainty) and narrower for software (cloud costs are more predictable). Adjust the multiplier if the user specifies prototype type.

Apply cost-sharing to each scenario:
```
low_govt_obligation  = low_should_cost * (1 - cost_share_ratio)
mid_govt_obligation  = mid_should_cost * (1 - cost_share_ratio)
high_govt_obligation = high_should_cost * (1 - cost_share_ratio)
```

Add consortium fees if applicable.

### Step 9: Produce the OT Cost Analysis Workbook

Generate a multi-sheet .xlsx workbook using openpyxl. Run the recalc script (`python /mnt/skills/public/xlsx/scripts/recalc.py <file>`) before presenting if available; otherwise use openpyxl's calculation capabilities.

**CRITICAL: ALL calculated cells MUST use Excel formula strings, not Python-computed values.** Write `ws.cell(row=r, column=c, value='=H11-E11')`, NOT `ws.cell(row=r, column=c, value=121675)`. The entire point of the assumption block is that changing a blue-font input (burden multiplier, cost-share ratio, escalation rate) recalculates every dependent cell in the workbook. A workbook with hard-coded values is useless as an analytical tool because the user cannot adjust assumptions. This applies to: all cost subtotals (SUM formulas), all cost-sharing calculations (reference $B$7), all variance calculations (reference Proposed - Should-Cost), all scenario calculations (reference $B$2/$B$3/$B$4), all cumulative funding profile cells (running SUM), and all labor cost cells (rate * hours). The only cells that should contain literal numbers are: blue-font assumption inputs (rows 2-8), text labels, milestone durations, FTE counts, BLS wage data, CALC+ data, and per diem rates.

**Workbook structure (7 sheets):**

**Sheet 1: Cost Analysis Summary.** Milestones as rows. Columns: Phase, Duration, Should-Cost (Mid), Government Share, Performer Share, Proposed Price, Variance ($), Variance (%).

**Assumption cell layout (Sheet 1, rows 1-9):**
```
A1: "OT Cost Analysis Assumptions"       (bold, merged A1:B1)
A2: "Burden Multiplier (Low)"            B2: 1.8    (blue font)
A3: "Burden Multiplier (Mid)"            B3: 2.0    (blue font)
A4: "Burden Multiplier (High)"           B4: 2.2    (blue font)
A5: "Escalation Rate"                    B5: 0.025  (blue font, percentage)
A6: "Productive Hours/Year"              B6: 1880   (blue font)
A7: "Cost-Share Ratio (Performer)"       B7: 0.00   (blue font, percentage; 0 if no share)
A8: "Consortium Fee Rate"                B8: 0.00   (blue font, percentage; 0 if direct)
A9: (blank row separator)
A10: header row for milestone data
```

All formulas reference these cells. Changing any assumption recalculates the entire workbook.

**Summary table layout (starting row 10):**
```
Columns: A=Milestone ID | B=Phase | C=Description | D=Duration (mo) | E=Should-Cost (Mid) | F=Govt Share | G=Performer Share | H=Proposed Price | I=Variance ($) | J=Variance (%)

Variance formula: =H[row]-E[row]
Variance %: =IF(E[row]>0,(H[row]-E[row])/E[row],0)

Totals row at bottom with SUM formulas for E through I.
```

**Sheet 2: Milestone Detail.** One block per milestone showing the cost element breakdown:

```
"Milestone [ID]: [Description]"                               (bold header)
A="Duration"                  B=[months]                      (blue font)
A="TRL Entry/Exit"            B=[X/Y]

A="LABOR"                                                     (bold subheader)
  [Labor category]            [Hours]   [Rate]   [Cost]       (formula rows)
  Labor Subtotal                                  [SUM]        (formula)

A="MATERIALS"                                                  (bold subheader)
  [Materials category]                            [Cost]       (blue font or formula)
  Materials Subtotal                              [SUM]        (formula)

A="TRAVEL"                                                     (bold subheader)
  [Destination]                                   [Cost]       (formula ref to Sheet 5 if applicable)
  Travel Subtotal                                 [SUM]        (formula)

A="ODCs"                                                       (bold subheader)
  [ODC category]                                  [Cost]       (blue font)
  ODC Subtotal                                    [SUM]        (formula)

A="MILESTONE SHOULD-COST"     B==labor+materials+travel+odcs   (formula, bold)
A="Government Share"          B==should_cost*(1-$B$7)          (formula)
A="Performer Share"           B==should_cost*$B$7              (formula)
A="Consortium Fee"            B==should_cost*$B$8              (formula; $0 if direct)
A="Total Govt Obligation"     B==govt_share+consortium_fee     (formula, bold)
```

For cost-type milestones, add a "Ceiling" column next to Should-Cost. For fixed-price milestones, the should-cost is the benchmark against the fixed payment.

**Sheet 3: Scenario Analysis.** Three columns (low/mid/high) for total should-cost. Apply cost-sharing to each scenario. Summary: "Government obligation range: $X (low) to $Y (high), Mid estimate: $Z."

Per-milestone scenario rows if milestone count is small enough (10 or fewer). Otherwise, summary level only.

**Sheet 4: Labor Benchmarking.** BLS burdened rate (mid), CALC+ 25th/50th/75th percentiles, min/max range, divergence percentage. Include caveat row: "OT labor benchmarks are market context for price reasonableness, not binding pricing constraints. NDC and non-traditional performers may have different cost structures."

**Sheet 5: Cost-Sharing Detail.** Per-milestone breakdown:
```
Columns: Milestone ID | Should-Cost | Govt Share ($) | Govt Share (%) | Performer Share ($) | Performer Share (%) | Consortium Fee | Total Govt Obligation

Cumulative row: running total of government obligations across milestones (funding profile)
```

Cash vs. in-kind distinction if applicable.

**Sheet 6: Methodology / Price Reasonableness Memo.**

This is the narrative document for the agreement file. Structure:

1. **Authority and Basis.** "This cost analysis supports a price reasonableness determination for an Other Transaction agreement under 10 USC 4021. OT agreements are outside the Federal Acquisition Regulation; price reasonableness is determined by the Agreements Officer's judgment based on market data, analogous pricing, and independent cost analysis. This analysis does not constitute a FAR 15.404 cost or price analysis."

2. **Methodology.** Describe the should-cost buildup approach: labor benchmarking (BLS OEWS + CALC+ market data), materials estimation, travel costs, milestone structure.

3. **Labor Benchmarking Basis.** BLS data vintage, CALC+ query date, performance location, burden multiplier rationale. Note that BLS/CALC+ data reflects traditional contractor market rates; NDC performers may price differently.

4. **Materials Basis.** Source of materials estimates (BOM, performer input, heuristic), escalation applied, major cost drivers.

5. **Cost-Sharing Analysis.** 10 USC 4022(d) eligibility path, cost-sharing ratio, cash vs. in-kind, per-milestone distribution.

6. **Price Reasonableness Determination.** Compare proposed price to should-cost estimate:
   - Per-milestone variance analysis
   - Position within scenario range (below low = aggressive, within low-mid = competitive, within mid-high = premium, above high = requires additional justification)
   - Overall assessment: "The proposed price of $X falls [within/above/below] the government's should-cost estimate range of $Y (low) to $Z (high), with a mid-point of $W. The proposed price represents a [X%] variance from the mid-point estimate."

7. **Data Sources.** BLS series IDs, CALC+ keywords, per diem FY, any analogous pricing sources.

8. **Exclusions and Limitations.** What is not included, caveats about non-traditional cost structures, areas where the estimate could be refined with additional performer data.

**Sheet 7: Raw Data.** All API query parameters and responses: BLS series IDs, CALC+ keywords and record counts, per diem query details.

**Formatting standards (same as IGCE builders):**
- Blue font (RGB 0,0,255) for all user-adjustable inputs and assumptions
- Black font for all formula cells
- Currency: $#,##0 with negatives in parentheses
- Percentage: 0.0%
- Bold headers with light gray fill
- Freeze panes below assumptions block on Sheet 1 (below row 9)
- Auto-size column widths

Never output as .md or HTML unless explicitly requested.

## Edge Cases

**No performer proposed price yet (pre-solicitation):** Produce should-cost estimate only. The workbook becomes a government internal estimate for budget planning. Set Proposed Price column to "TBD." Variance columns MUST use conditional formulas, not static "N/A" text: `=IF(J[row]="TBD","",J[row]-E[row])` for dollar variance, `=IF(J[row]="TBD","",IF(E[row]>0,(J[row]-E[row])/E[row],0))` for percentage. This way, when the user later enters a proposed price in column J, the variance auto-calculates without editing formulas. Note in methodology that this is a pre-solicitation estimate and instruct the user to enter the proposed price in the Proposed column when available.

**Performer provides total price but not per-milestone breakdown:** Allocate the proposed total across milestones proportionally to the should-cost estimate (each milestone's share of proposed = proposed_total * (milestone_should_cost / total_should_cost)). Document this in the methodology: "The performer's proposed total of $X was allocated across milestones in proportion to the government's should-cost estimate. Per-milestone proposed pricing was not provided. This allocation is an analytical assumption; actual milestone payments may differ." Flag this as a limitation in Sheet 6 Section 8 (Exclusions). If the proportional allocation produces per-milestone variances that are uniformly identical (all milestones show the same % variance), note that this is an artifact of the allocation method, not evidence that every milestone is equally over/under-priced.

**Performer is an NDC with no historical pricing:** BLS/CALC+ benchmarks are the primary tool. Note in methodology that NDC cost structures may differ from market data. If the performer provides a cost breakdown with their proposal, use it alongside market benchmarks.

**Materials dominate the estimate (hardware prototype):** If materials exceed 60% of should-cost, flag that the labor benchmarking has limited influence on the total. Price reasonableness for hardware prototypes often relies on BOM validation and vendor quotes more than labor rate analysis.

**Cost-type milestones:** Add a ceiling column to the milestone detail. The should-cost is the estimated cost; the ceiling is typically should-cost plus a contingency margin (10-20%). Note in methodology that cost-type milestones require the performer to track and report actual costs.

**Consortium-brokered OT:** Add consortium management fee (default 5%) to government obligation. The fee is on top of the performer's cost, not deducted from it. Some consortiums charge a flat fee instead of a percentage; handle as a fixed cost per milestone or as a one-time upfront fee.

**Multiple performers (competitive prototype):** If the government is funding 2-3 performers to compete on the same prototype, the cost analysis may need to be run per performer. Each gets its own workbook. Note the total program cost = sum of all performer obligations.

**Analogous pricing available:** If the user provides prior OT awards for similar prototypes (from USASpending or internal records), add an analogous pricing comparison in Sheet 6. This strengthens the price reasonableness narrative.

**BLS wage at reporting cap:** Same handling as IGCE builders -- use $239,200/$115.00 as lower bound, flag in methodology.

**No CALC+ results:** Same handling as IGCE builders -- try broader keywords, note unavailable, mark "No CALC+ data."

## What This Skill Does NOT Cover

Include as placeholder rows or methodology notes:
- **OT project description** -- use OT Project Description Builder
- **Traditional FAR IGCEs** -- use IGCE Builder (FFP, LH/T&M, CR)
- **Agreement terms and conditions** -- legal review required
- **NDC eligibility determination** -- agreements officer determination per 10 USC 3014
- **DCAA audit or indirect rate validation** -- not applicable to OTs with NDC performers
- **Production follow-on cost estimates** -- if production follow-on under 4022(f) uses a FAR-based contract, use the appropriate IGCE Builder
- **Subcontractor cost analysis** -- requires separate estimate or sub-tier cost data
- **OCONUS travel** -- per diem covers CONUS only; State Dept rates for OCONUS
- **Grant budgets** -- use Grant Budget Builder

## Quick Start Examples

**From milestone table:** "Build the OT cost analysis" (after running OT Project Description Builder)
Claude will: consume the milestone handoff table, collect proposed price, benchmark labor, estimate materials, apply cost-sharing, produce 7-sheet workbook.

**From concept:** "How much should a TRL 4-6 cyber prototype cost in DC, 18 months, small team?"
Claude will: Workflow A+. Decompose into milestones, benchmark labor (cybersecurity SOC codes), estimate materials (software-dominant), produce should-cost estimate. No proposed price comparison unless provided.

**Price reasonableness:** "Performer proposes $3.2M for a 5-milestone prototype. Reasonable?"
Claude will: Workflow B. Collect milestone details, build should-cost, compare per-milestone and total, produce price reasonableness memo.

**Cost-sharing analysis:** "Traditional prime, no NDC or SB, prototype OT in Huntsville. What does cost-sharing look like?"
Claude will: apply 1/3 performer cost share per 4022(d)(1)(C), show government vs. performer obligations per milestone, cumulative funding profile.

**Consortium OT:** "DIU prototype, NDC performer, 4 milestones. Price this."
Claude will: no cost share (NDC path), add 5% consortium management fee, show government obligation including fee per milestone.

**Hardware prototype:** "Estimate should-cost for a 3-unit sensor prototype, TRL 4-6, 24 months"
Claude will: flag materials-heavy estimate, derive BOM categories from prototype type, benchmark labor, produce workbook with materials as dominant cost element.


---

*MIT James Jenrette / 1102tools. Source: github.com/1102tools/federal-contracting-skills*
