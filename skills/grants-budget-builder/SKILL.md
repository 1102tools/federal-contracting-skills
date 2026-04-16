---
name: grants-budget-builder
description: >
  Build federal grant budget estimates aligned to 2 CFR 200 (Uniform
  Guidance) and SF-424A format by orchestrating BLS OEWS and GSA Per
  Diem skills. Supports R01-style research grants, cooperative
  agreements, and program grants. Calculates personnel costs with
  percent-effort and institutional base salary, fringe by appointment
  type, travel via Per Diem, and indirect costs via NICRA or de
  minimis rate. Trigger for: grant budget, SF-424A, budget
  justification, grant cost estimate, 2 CFR 200 budget, cooperative
  agreement budget, R01 budget, NIH budget, grant personnel costs,
  fringe benefits, indirect cost rate, NICRA, F&A rate, MTDC, grant
  travel estimate, budget narrative, subaward budget, or modular
  budget. Also trigger for grant-related pricing or effort-based
  costing. Does NOT cover contract IGCEs (use IGCE Builder FFP,
  LH/T&M, or CR). Does NOT cover NOFO program descriptions (use
  grants-program-description-builder). Requires BLS OEWS API and
  GSA Per Diem Rates API skills.
---

# Grants Budget Builder

## Overview

This skill builds federal grant budget estimates aligned to the Uniform Guidance (2 CFR 200) and the SF-424A budget format. It orchestrates two underlying data skills and Claude's analytical capabilities to produce a complete, defensible grant budget with justification narrative.

**This is NOT an IGCE.** Grants and cooperative agreements are funded under a fundamentally different framework than contracts. Contracts are priced under FAR Part 15 using cost/price analysis. Grants are budgeted under 2 CFR 200 using object class categories with institutional rate structures. The cost categories, regulatory citations, allowability rules, and output formats are entirely different.

**When to use this skill vs. IGCE Builder Core:**
- Funding instrument is a **grant or cooperative agreement** (financial assistance) → Use this skill
- Funding instrument is a **contract** (acquisition, any FAR contract type) → Use IGCE Builder Core
- Unsure → Ask the user. If the funding comes through Grants.gov or uses an SF-424 application, it is a grant. If it comes through SAM.gov or uses an SF-1449/SF-33, it is a contract.

**Required skills (must be installed):**
1. **BLS OEWS API**: market wage data for personnel cost benchmarking
2. **GSA Per Diem Rates API**: federal lodging and M&IE rates for travel estimates

**Optional but not required:**
- CALC+ is NOT used. GSA schedule pricing is irrelevant for grant recipients.
- Per Diem skill uses the same api.data.gov key as the IGCE Builder.

**Required API keys (must be in user memory):**
- BLS API key (v2) for the BLS OEWS skill
- api.data.gov key for the GSA Per Diem skill

**What this skill produces:** A multi-sheet Excel workbook containing a detailed budget by SF-424A object class categories, a budget justification narrative suitable for the grant application, multi-year cost projections with escalation, and an indirect cost calculation using the applicant's NICRA or de minimis rate.

**Regulatory basis:** 2 CFR 200 Subpart E (Cost Principles), specifically:
- 2 CFR 200.413: Direct costs
- 2 CFR 200.414: Indirect (F&A) costs
- 2 CFR 200.431: Compensation (personal services)
- 2 CFR 200.432: Conferences
- 2 CFR 200.439: Equipment and other capital expenditures
- 2 CFR 200.453: Materials and supplies
- 2 CFR 200.456: Participant support costs
- 2 CFR 200.474: Travel costs
- 2 CFR 200.475: Trustees (travel for advisory councils)

## Workflow Selection

This skill supports three workflows.

### Workflow A: Full Grant Budget Build (Default)
The user needs a complete grant budget from scratch. Execute Steps 1 through 7 in order.

Trigger phrases: "build a grant budget," "SF-424A budget," "grant cost estimate," "budget for my R01," "how much should this grant cost."

### Workflow B: Budget from Narrative/Description
The user provides a research plan, project narrative, or scope description and needs a budget built from it. Execute Step 0 (Scope Decomposition) first, get user validation, then proceed through Steps 1-7.

Trigger phrases: "build a budget from this proposal," "here's my research plan, what should the budget be," "budget this project description."

Detection: If the user provides a block of text describing research or program activities rather than structured budget inputs, default to Workflow B.

### Workflow C: Budget Review / Benchmarking
The user has an existing budget (from an applicant, a subawardee, or a prior year) and wants to check whether the costs are reasonable. Compare personnel rates against BLS, travel against Per Diem, and indirect costs against typical NICRA ranges.

Trigger phrases: "is this budget reasonable," "review this grant budget," "benchmark these costs," "check this subaward budget."

**Workflow C steps:**
1. Collect the existing budget line items.
2. For personnel: query BLS for comparable occupations at the performance location. Flag any salary that exceeds the 75th percentile or the NIH salary cap (if applicable).
3. For travel: query Per Diem for stated destinations. Flag any lodging/M&IE that exceeds published rates.
4. For indirect costs: compare stated F&A rate against typical ranges by institution type. Flag if no NICRA is cited.
5. Produce a Budget Review summary with flags, benchmarks, and recommendations.

## Information to Collect from the User

Collect all inputs in a single pass. Grant budgets require more institutional context than contract IGCEs because the cost structure is driven by the applicant's rate agreements.

### Required Inputs

| Input | Description | Example |
|-------|-------------|---------|
| Personnel | Names or roles, percent effort, and institutional base salary (IBS) or salary range | PI at 25% effort, $180,000 IBS; Postdoc at 100%, $61,008 |
| Performance location | Institution city/state | Bethesda, MD |
| Project period | Start date, number of budget years | 5 years starting 7/1/2026 |
| Indirect cost rate | Applicant's negotiated F&A rate and base (MTDC, TDC, or S&W) | 55% MTDC |

### Optional Inputs (Use Defaults If Not Provided)

| Input | Default | Notes |
|-------|---------|-------|
| Fringe benefit rate | See fringe table below | Varies by appointment type; ask if not provided |
| Escalation rate | 3% annually | Higher than contract default; reflects academic salary increases |
| Travel destinations | None | Conference travel and/or site visits |
| Travel frequency | None | Trips per year |
| Equipment (>$5,000/unit) | None | 2 CFR 200.313; excluded from MTDC base |
| Supplies | None | Consumables, lab materials, small equipment under $5,000 |
| Contractual / subawards | None | First $25,000 of each subaward is in MTDC base; remainder excluded |
| Participant support | None | Stipends, travel, subsistence for participants; excluded from F&A per 2 CFR 200.456 |
| Other direct costs | None | Publication costs, computing, animal care, human subjects |
| Tuition remission | None | For graduate research assistants; may be direct or F&A depending on institution |
| NIH salary cap | Current cap | If NIH-funded, salary charged is capped. Check current cap. |
| Cost sharing | None | If required by funding announcement; 2 CFR 200.306 |
| Consortium / subaward F&A | 8% de minimis or actual | Subawardees without NICRA use 10% de minimis per 2 CFR 200.414(f) |

### Fringe Benefit Rate Guidance

Grant fringe rates are institution-specific, negotiated with the cognizant federal agency, and vary by appointment type. If the user does not provide rates, use these benchmarks and note they are estimates pending the institution's actual rates:

| Appointment Type | Typical Range | Benchmark Default | Components |
|---|---|---|---|
| Faculty / senior staff | 25-35% | 30% | FICA (7.65%), health insurance, retirement, life/disability |
| Postdoctoral | 20-30% | 25% | FICA, health, may have limited retirement |
| Graduate research assistant | 8-15% | 12% | FICA, student health, tuition remission often separate |
| Temporary / hourly | 10-18% | 15% | FICA, workers' comp, limited benefits |
| Summer salary (academic year appt) | Same rate as base | Same rate | Fringe applies to summer salary at same institutional rate |

**Always ask for the institution's actual fringe rates.** These benchmarks are for estimation only. The actual rates will be in the institution's NICRA or rate agreement letter.

### Indirect Cost (F&A) Rate Guidance

| Institution Type | Typical F&A Range | Applied To | Notes |
|---|---|---|---|
| Major research university | 50-65% | MTDC | Negotiated with DHHS (cognizant agency for most universities) |
| Smaller university / college | 35-50% | MTDC | Lower facilities component |
| Nonprofit research institute | 20-40% | MTDC or TDC | Varies widely |
| Hospital / medical center | 55-75% | MTDC | High facilities costs |
| State / local government | 10-30% | MTDC or S&W | Often use cost allocation plans instead of NICRA |
| Organization without NICRA | 10% de minimis | MTDC | 2 CFR 200.414(f); available to any org that has never had a NICRA |

**MTDC (Modified Total Direct Costs):** Total direct costs minus equipment, capital expenditures, patient care charges, rental of off-site facilities, tuition remission, scholarships/fellowships, participant support costs, and the portion of subawards exceeding $25,000 per 2 CFR 200.1.

## Constants Reference

| Constant | Value | Source / Basis |
|----------|-------|---------------|
| De minimis indirect rate | 10% of MTDC | 2 CFR 200.414(f) |
| Subaward MTDC inclusion cap | First $25,000 per subaward | 2 CFR 200.1 (MTDC definition) |
| Equipment threshold | $5,000 per unit | 2 CFR 200.1 (Equipment definition); some institutions set lower |
| NIH salary cap (FY2025) | $221,900 | NOT Act; verify current year via web search before using |
| Standard work year | 12 calendar months or 9 academic months + 3 summer | Effort is expressed as percent of appointment period |
| GSA mileage rate | $0.70/mile | CY2025; for local travel estimates |
| First/last day M&IE | 75% of full day | FTR 301-11.101; same as contract travel |
| Default escalation | 3% annually | Academic salary increase benchmark; higher than contract 2.5% |

## Orchestration Sequence

### Step 0: Scope Decomposition (Workflow B Only)

When the user provides a research plan or project narrative instead of structured budget inputs, decompose it into budget-relevant elements.

**Process:**

1. **Identify key personnel.** Extract named roles (PI, Co-PI, Co-Investigator, Postdoc, Graduate RA, Research Technician, Project Coordinator, etc.) and any stated effort levels or FTE counts.

2. **Identify activities requiring travel.** Look for conference attendance, site visits, field work, collaborator meetings, advisory board meetings, data collection trips.

3. **Identify equipment needs.** Extract any mentioned instruments, computing hardware, or specialized equipment over $5,000.

4. **Identify supplies and materials.** Lab consumables, reagents, animal costs, computing charges, software licenses, participant incentives.

5. **Identify subawards or consortium arrangements.** Named collaborating institutions, subcontracted components.

6. **Identify other direct costs.** Publication fees, IRB/IACUC fees, subject recruitment, data management, open access fees.

7. **Present decomposition for validation:**

```
SCOPE DECOMPOSITION (Draft, pending your review)

Source: [narrative title or "user-provided description"]

PERSONNEL:
  PI ([name/role]) - [X]% effort - Salary: [if stated, or "TBD - please provide IBS"]
  Postdoc - 100% effort - Salary: [if stated, or "$61,008 NIH NRSA rate"]
  Graduate RA - 50% effort - Stipend: [if stated, or "TBD"]

TRAVEL:
  [X] conferences/year, domestic
  [X] site visits to [location]

EQUIPMENT:
  [item] - estimated $[X]

SUPPLIES:
  [category] - estimated $[X]/year

SUBAWARDS:
  [Institution] - [scope] - estimated $[X]

OTHER:
  [items identified]

MISSING INFORMATION:
  - Institutional base salary for [roles]
  - F&A rate and fringe rates (institution-specific)
  - [other gaps]
```

8. **Validation gate.** Confirm personnel, effort, and major cost categories before proceeding. The most critical missing inputs for grants are institutional salary and rate data.

### Step 1: Calculate Personnel Costs

Personnel is almost always the largest grant budget category (typically 40-60% of total direct costs).

**Effort-based calculation:**
```
salary_charged = institutional_base_salary * percent_effort
```

If the PI has an IBS of $180,000 and commits 25% effort:
```
salary_charged = $180,000 * 0.25 = $45,000
```

**NIH salary cap check:**
If the grant is NIH-funded (or any agency that applies an executive salary cap), the salary charged to the grant cannot exceed the cap rate at the stated effort level:
```
max_salary_charged = min(IBS, salary_cap) * percent_effort
```

Example: IBS = $250,000, cap = $221,900, effort = 25%:
```
max_salary_charged = $221,900 * 0.25 = $55,475  (not $250,000 * 0.25 = $62,500)
```
The institution may pay the difference from other funds, but the grant only covers up to the cap.

**BLS benchmarking (optional but recommended):**
Use the BLS OEWS skill to pull market wages for comparable occupations at the performance location. This is NOT to set the salary (grants use institutional salaries, not market rates), but to flag any salary that is unusually high or low relative to the market. Common mappings:

| Grant Role | SOC Code | BLS Title |
|---|---|---|
| Principal Investigator (faculty) | 19-1042 / 19-2099 | Medical Scientists / Physical Scientists |
| Postdoctoral Researcher | 19-4099 | Life/Physical Science Technicians (closest proxy) |
| Research Technician | 19-4021 | Biological Technicians |
| Biostatistician | 15-2041 | Statisticians |
| Project Coordinator | 13-1082 | Project Management Specialists |
| Data Analyst | 15-2051 | Data Scientists |
| Administrative Support | 43-6014 | Secretaries and Admin Assistants |

**Apply escalation across budget years:**
```
year_N_salary = base_salary_charged * (1 + escalation_rate) ^ (N - 1)
```
Default 3% for academic institutions. Note: some agencies (NIH) cap escalation or require justification for increases above a threshold.

### Step 2: Calculate Fringe Benefits

```
fringe_cost = salary_charged * fringe_rate
```

Apply the appropriate rate by appointment type. If the institution provides a single blended rate, use that. If they provide rates by category, apply each to the corresponding personnel line.

Fringe is a direct cost in grant budgets (unlike contracts where it is folded into burden). It gets its own line in the SF-424A.

**Fringe is included in the MTDC base** for indirect cost calculation.

### Step 3: Calculate Travel Costs

**Use the GSA Per Diem Rates API skill.** Same calculation as the IGCE Builder:

```
lodging_per_trip = nightly_rate * nights
travel_days = nights + 1
mie_per_trip = (mie_rate * full_days) + (mie_rate * 0.75 * 2)  # first/last day at 75%
trip_total = lodging_per_trip + mie_per_trip
annual_travel = trip_total * trips_per_year * travelers
```

**Grant-specific travel notes:**
- Conference travel: typically 1-2 domestic conferences per year for PI/Co-I. Estimate $1,500-$2,500 per domestic conference trip (registration not included in per diem; list separately under Other).
- Foreign travel: requires specific justification in most federal grants. Use State Department per diem rates (not GSA). Note this limitation if the user mentions international travel.
- Airfare: same as IGCE Builder. Use GSA City Pair fares if origin/destination known, otherwise estimate or use TBD placeholder.
- Local travel: GSA mileage rate ($0.70/mile CY2025) for local site visits.

**Travel is included in the MTDC base.**

### Step 4: Estimate Non-Personnel Direct Costs

Collect from the user or estimate based on project scope. Present each category with the 2 CFR 200 reference:

**Equipment (2 CFR 200.439):**
- Items with per-unit cost >= $5,000 and useful life > 1 year
- Usually budgeted in Year 1 only unless multi-phase procurement
- **EXCLUDED from MTDC base** for F&A calculation
- Must be specifically justified in budget narrative

**Supplies (2 CFR 200.453):**
- Consumables and items under the equipment threshold
- Lab supplies, reagents, animal purchase/per diem, computing supplies
- Included in MTDC base
- Estimate by category with annual costs

**Contractual / Subawards (2 CFR 200.318):**
- First $25,000 of each subaward included in MTDC base
- Amount above $25,000 excluded from MTDC base
- Each subaward should have its own mini-budget if amount is significant
- Consortium F&A: subawardee applies their own indirect rate to their costs

**Participant Support Costs (2 CFR 200.456):**
- Stipends, travel, subsistence, and other expenses paid to participants (not employees)
- **EXCLUDED from MTDC base**
- Cannot be used for other categories without agency approval
- Common in training grants, workshops, conferences

**Other Direct Costs:**
- Publication costs (page charges, open access fees)
- Consultant costs (daily rate, typically $500-$750/day for grant consultants)
- Computing / cloud costs
- Animal care per diem (institutional rates)
- Human subjects costs (IRB fees, participant compensation, recruitment)
- Tuition remission for graduate RAs (if direct cost at the institution)
- Conference registration fees

### Step 5: Calculate Indirect Costs (F&A)

This is where grants diverge most from contracts. The indirect cost calculation depends on the institution's NICRA and the applicable base.

**MTDC base calculation:**
```
total_direct_costs = personnel + fringe + travel + equipment + supplies + contractual + participant_support + other

MTDC_exclusions = equipment + participant_support + tuition_remission + subaward_amounts_over_25k + rental_of_offsite_facilities + patient_care_charges

MTDC_base = total_direct_costs - MTDC_exclusions

indirect_costs = MTDC_base * F&A_rate
```

**If the institution uses TDC (Total Direct Costs) base:**
```
indirect_costs = total_direct_costs * F&A_rate
```
(TDC rates are lower because the base is larger)

**If the institution uses S&W (Salaries and Wages) base:**
```
indirect_costs = (personnel + fringe) * F&A_rate
```
(S&W rates are higher because the base is smaller; less common)

**De minimis rate (2 CFR 200.414(f)):**
Organizations that have never had a NICRA may use 10% of MTDC. This is a safe default for new or small organizations. Once elected, it can be used indefinitely or until the organization negotiates a NICRA.

**Always ask for the institution's actual F&A rate and base type.** If the user cannot provide it, use the benchmarks from the F&A Rate Guidance table and flag that these are estimates.

### Step 6: Assemble Total Budget and Apply Multi-Year Escalation

**Budget summary structure (SF-424A object class categories):**
```
A. Personnel                    $XXX,XXX
B. Fringe Benefits              $XX,XXX
C. Travel                       $XX,XXX
D. Equipment                    $XX,XXX
E. Supplies                     $XX,XXX
F. Contractual                  $XX,XXX
G. Construction                 $0          (rare in research grants)
H. Other                        $XX,XXX
I. Total Direct Costs (A-H)     $XXX,XXX
J. Indirect Costs               $XX,XXX
K. Total Costs (I + J)          $XXX,XXX
L. Participant Support          $XX,XXX     (shown separately; excluded from F&A)
M. Total Federal Request        $XXX,XXX
```

**Multi-year projection:**
- Personnel and fringe: escalate at stated rate (default 3%)
- Travel: escalate at 2.5% (federal per diem adjustment rate)
- Equipment: typically Year 1 only; do not escalate unless phased procurement
- Supplies: escalate at 2-3%
- Subawards: escalate unless fixed subaward amount
- Indirect costs: recalculated each year on that year's MTDC base
- Participant support: escalate only if tied to stipend rates that increase

### Step 7: Produce the Grant Budget Workbook

Generate a multi-sheet .xlsx workbook using openpyxl. Follow the xlsx skill's best practices: use Excel formulas for all totals, subtotals, and escalation so the workbook is fully dynamic.

**Workbook structure:**

- **Sheet 1: Budget Summary (SF-424A Format)**
  One column per budget year, rows for each object class category (A through M). Totals via SUM formulas. This is the primary deliverable matching the SF-424A Section B format.

  Assumption cell layout (rows 1-7):
  ```
  A1: "Grant Budget Assumptions"          (bold, merged A1:B1)
  A2: "F&A Rate"                          B2: 0.55     (blue font, percentage)
  A3: "F&A Base Type"                     B3: "MTDC"   (blue font)
  A4: "Personnel Escalation"              B4: 0.03     (blue font, percentage)
  A5: "Non-Personnel Escalation"          B5: 0.025    (blue font, percentage)
  A6: "NIH Salary Cap"                    B6: 221900   (blue font; set to 0 if no cap applies)
  A7: (blank row separator)
  A8: header row
  ```

- **Sheet 2: Personnel Detail**
  One row per person/role. Columns: Name/Role, Title, Appointment Type, IBS, Percent Effort, Salary Charged, Fringe Rate, Fringe Amount, Total Compensation, BLS Benchmark (if pulled), Variance.

  For NIH-funded grants, include a "Cap-Adjusted Salary" column showing the capped amount.

  All salary formulas reference percent effort and IBS cells so the user can adjust. Escalation formulas reference the assumption block.

- **Sheet 3: Travel Detail**
  Same layout as IGCE Builder Travel Detail sheet. Per diem components by destination with formula-driven calculations. Add a conference travel section with estimated registration fees.

- **Sheet 4: Non-Personnel Detail**
  Itemized breakdown of Equipment, Supplies, Contractual/Subawards, Participant Support, and Other Direct Costs. Each item has: Description, Quantity, Unit Cost, Annual Cost, Justification column (brief text).

  For subawards: show total subaward amount, amount included in MTDC ($25K cap), and amount excluded.

- **Sheet 5: Indirect Cost Calculation**
  Shows the MTDC base calculation step by step:
  ```
  Row 1: "Indirect Cost Calculation"           (bold header)
  Row 2: A="Total Direct Costs"                B=[formula summing all direct]
  Row 3: A="Less: Equipment"                   B=[equipment total, negative]
  Row 4: A="Less: Participant Support"          B=[participant total, negative]
  Row 5: A="Less: Tuition Remission"            B=[tuition total, negative]
  Row 6: A="Less: Subaward Amounts >$25K"       B=[excess subaward, negative]
  Row 7: A="MTDC Base"                          B=[formula: B2+B3+B4+B5+B6]
  Row 8: A="F&A Rate"                           B=[reference to assumptions]
  Row 9: A="Indirect Costs"                     B==B7*B8                       (formula, bold)
  ```

  Repeat per budget year in adjacent columns.

- **Sheet 6: Budget Justification Narrative**
  Text-based narrative suitable for copy/paste into the grant application. Structured by SF-424A category:

  **A. Personnel:** For each person, state name/role, percent effort, base salary (note if capped), and role on the project. Example: "Dr. [Name], Principal Investigator, will devote 25% effort to this project. Institutional base salary is $180,000; salary charged to the grant is $45,000 (25% of IBS)."

  **B. Fringe Benefits:** State the institutional fringe rate(s) by appointment type and the source (NICRA, institutional rate agreement). Example: "Fringe benefits are calculated at the institution's federally negotiated rate of 30% for faculty and 25% for postdoctoral researchers."

  **C. Travel:** Justify each trip category with purpose, destination, duration, and per diem basis. Reference GSA rates.

  **D. Equipment:** Justify each item with research need, cost basis (vendor quote, catalog price), and why the item is essential.

  **E. Supplies:** Justify by category with estimated quantities and unit costs.

  **F. Contractual:** Justify each subaward with scope, PI, institution, and estimated cost. Note consortium F&A rate.

  **G-H. Other:** Justify each line item.

  **Indirect Costs:** State the F&A rate, base type, and NICRA reference. Example: "Indirect costs are calculated at the institution's federally negotiated rate of 55% of Modified Total Direct Costs, per NICRA dated [date] negotiated with DHHS."

  Use merged cells with word wrap for readability.

- **Sheet 7: Raw Data**
  All API query parameters and response values for audit trail. BLS series IDs, Per Diem queries, salary cap verification source.

**Formatting standards:**
- Blue font (RGB 0,0,255) for all user-adjustable inputs: salaries, effort percentages, fringe rates, F&A rate, escalation rates, salary cap
- Black font for all formula cells
- Currency format: $#,##0
- Percentage format: 0.0%
- Bold headers with light gray fill
- Freeze panes below header row on each sheet
- Column widths auto-sized

After building the workbook, run the recalc script (`python /mnt/skills/public/xlsx/scripts/recalc.py <file>`) to populate calculated values and verify zero formula errors before presenting.

Never output as .md or HTML unless explicitly requested.

## Handling Edge Cases

### NIH Modular Budget
NIH R01 and similar mechanisms use a modular budget format for requests up to $250,000 per year in direct costs. Modular budgets request funds in $25,000 modules rather than itemized line items. If the user mentions "modular budget":
- Calculate the detailed budget internally to determine the correct number of modules
- Round up to the nearest $25,000 module
- Output a simplified modular budget page alongside the detailed backup
- Still produce the detailed budget as supporting documentation

### Multiple PIs (MPI)
For multi-PI grants, each PI may have different institutional salary and fringe rates. Track each PI's institution separately. If PIs are at different institutions, the non-lead institution is typically a subaward with its own F&A rate.

### Subaward Budgets
Each significant subaward should have its own mini-budget. If the user provides subawardee details, create a separate budget block per subaward showing their direct costs and their F&A rate applied to their MTDC base. Roll up the total into the prime budget's Contractual line.

### Cost Sharing / Matching
If the funding announcement requires cost sharing (2 CFR 200.306):
- Add a "Cost Share" column next to the "Federal" column on Sheet 1
- Show which costs are federally funded vs. cost-shared
- Cost-shared effort still counts toward the MTDC base for F&A purposes
- Note 2 CFR 200.306 requirements in the narrative: cost share must be verifiable, from non-federal sources, necessary and reasonable, allowable, and not used as match for other federal awards

### Foreign Subawards / OCONUS Travel
- Foreign subawards: F&A treatment varies; some agencies cap foreign F&A at 8%
- OCONUS travel: use State Department per diem rates, not GSA. Note this limitation and suggest the user look up rates at aoprals.state.gov

### No NICRA Available
If the applicant does not have a NICRA:
- They may use the 10% de minimis rate per 2 CFR 200.414(f)
- They may negotiate a rate with the cognizant agency
- First-time applicants often use de minimis; note this and flag for follow-up

### Graduate Student Tuition Remission
Treatment varies by institution:
- Some institutions charge tuition as a direct cost (appears in "Other" category)
- Some include tuition in the fringe rate
- If direct, it is typically excluded from the MTDC base
- Ask the user how their institution handles tuition

## What This Skill Does NOT Cover

- **Contract IGCEs:** Use IGCE Builder Core for any FAR-based acquisition (T&M, LH, FFP, CR).
- **OCONUS per diem:** State Department rates apply; this skill covers CONUS GSA rates only.
- **Agency-specific budget forms:** Some agencies have custom budget templates beyond SF-424A (e.g., NSF Budget Justification has specific formatting). This skill produces the standard SF-424A format.
- **Facilities & Administrative rate negotiation:** This skill uses provided rates; it does not calculate what an institution's F&A rate should be.
- **Cost sharing calculations beyond basic tracking:** Complex cost share verification and documentation is outside scope.
- **Budget revisions / carryover requests:** This skill builds initial budgets, not post-award modifications.

## Quick Start Examples

**Standard research grant:** "Build a 5-year grant budget for an R01 with a PI at 25% effort ($180K IBS), a postdoc at 100% ($61K), and a grad student at 50%. F&A rate is 55% MTDC. Two domestic conferences per year."
Claude will: calculate personnel with effort and fringe by appointment type, estimate travel via Per Diem, apply 55% F&A to MTDC base (excluding equipment and participant support), produce 7-sheet xlsx with SF-424A summary, personnel detail, travel detail, indirect cost calculation, and budget justification narrative.

**From a research plan (Workflow B):** "Here's my specific aims page, build me a budget" [user pastes text]
Claude will: run Step 0 to extract personnel, travel, equipment, and supply needs from the research plan. Present decomposition for validation. Ask for institutional rates (salary, fringe, F&A). Then build the full budget.

**Budget review (Workflow C):** "A subawardee submitted this budget, is it reasonable?" [user provides budget]
Claude will: pull BLS data for personnel benchmarking at the subawardee's location. Compare travel against Per Diem rates. Check F&A rate against typical ranges for the institution type. Flag any line items that appear above market or improperly categorized.

**Cooperative agreement:** "Budget for a cooperative agreement with a community health org, they don't have a NICRA"
Claude will: use 10% de minimis rate per 2 CFR 200.414(f). Note the de minimis election in the budget narrative. Flag that the organization should consider negotiating a NICRA if they anticipate significant federal funding.

**NIH with salary cap:** "PI salary is $280,000 but this is NIH-funded"
Claude will: verify current NIH salary cap via web search, apply the cap to the PI's effort-based salary calculation, note the difference in the personnel detail sheet, and explain the cap in the budget justification narrative.


---

*MIT © James Jenrette / 1102tools. Source: github.com/1102tools/federal-contracting-skills*