# IGCE Builder LH/T&M: Testing Record

# Part 1: For Federal Acquisition Users

## The bottom line

Independent testing in April 2026 (3 end-to-end runs, 42 binary assertions graded, Claude Opus 4.7) validates the IGCE Builder LH/T&M skill across three real-world federal acquisition scenarios: pure Labor Hour at Redstone Arsenal, T&M hybrid with materials at NAVWAR San Diego, and LH with multi-site travel at AFLCMC Wright-Patterson.

**Wave 1 aggregate: 39 of 42 assertions passed (92.9%). Round 1 patches shipped to address all finding categories.**

## Scenarios tested and how reliably they work

| Scenario | Score | Result |
|---|---|---|
| S1: Pure Labor Hour at Redstone Arsenal (Huntsville MSA, 4 LCATs, base + 4 options) | 14/14 | Reliable |
| S2: T&M hybrid with $250K/yr materials at NAVWAR San Diego (4 LCATs, base + 2 options) | 12/14 | Materials handling fee not surfaced (soft fail); FAR 16.601(c)(3) surveillance memo missing |
| S3: LH with multi-depot travel at AFLCMC Wright-Patterson (Dayton MSA, 4 LCATs, 3 destinations, base + 3 options) | 13/14 | FAR 31.205-46 travel cost principle not cited |

All three scenarios produced workbooks that delivered correct final numbers (once worker caught cell-reference drift mid-build in S2). The failures were gaps in narrative/methodology completeness and skill hardening gaps that allowed those gaps to slip through. All findings patched in Round 1.

## Manual-verification checklist

Scan every LH/T&M IGCE output for these before using in a contract file:

**1. Skill announced itself.** First line of the worker's response should acknowledge "IGCE Builder LH/T&M" was loaded. If the worker started building without naming the skill, the skill likely did not trigger and you are getting a generic xlsx build instead of a hardened IGCE.

**2. Per-FTE annualized cost is defensible.** Burdened hourly × productive hours × 1 FTE should land in $100K to $1M. Outside this range (especially above $1M) indicates formula cell-reference drift.

**3. Sheet 1 Grand Total equals Sheet 2 Mid-Scenario Grand Total.** Any divergence indicates cross-sheet reference drift.

**4. BLS aging factor is cell-referenced formula, not hardcoded.** Aging factor must be `=(1+B5)^(B10/12)` or equivalent; changing contract start date should cascade.

**5. FAR citations complete.** Must see 16.601 always, 16.601(b)(2) for T&M materials, 16.601(c)(2) for LCAT ceiling hours, 16.601(c)(3) for T&M surveillance, 31.205-46 when travel is in scope.

**6. Raw Data sheet shows rejected SOC alternatives** when a judgment-call re-pick happened (for example, IT PM switched from 15-1299 to 11-3021).

**7. Materials at cost (no burden) for T&M.** FAR 16.601(b)(2). Handling fee decision explicit in methodology, whether applied or not.

**8. Travel math uses FTR 301-11.101 75% first/last day M&IE rule** and day trips use single-partial-day M&IE (not full + two 75%).

## What the skill does not do

- **It does not produce FFP or CR estimates.** Use IGCE Builder FFP or IGCE Builder CR.
- **It does not produce OT Cost Analyses.** Use OT Cost Analysis skill.
- **It does not substitute for a contracting officer's price reasonableness determination.** IGCE is an estimate; the CO makes the determination per FAR 15.4.
- **It does not guarantee a specific burden multiplier.** Multiplier is a scenario input; user-provided values win over defaults.
- **It has not been tested on:** CONUS-to-OCONUS mixed performance, 24x7 shift coverage with the Step 0.5 math, Workflow A+ SOW/PWS decomposition from scratch, Workflow B rate validation only.

---

# Part 2: For Developers and Technical Reviewers

## Testing methodology

### Scenarios

Three scenarios designed to exercise distinct LH/T&M mechanics:

- **S1 (LH, baseline, no materials, no travel):** Redstone Arsenal software sustainment, 4 LCATs including a senior tier, GSA MAS IT Schedule 70 vehicle, base + 4 options. Tests burden multiplier math, BLS aging, CALC+ validation, LCAT ceiling hours table, FAR 16.601 citation, escalation.

- **S2 (T&M hybrid, materials at cost):** NAVWAR San Diego network security, 4 cybersecurity LCATs, $250K/yr materials, 60% on-site + 40% telework, base + 2 options. Tests FAR 16.601(b)(2) materials at cost, FAR 16.601(c)(3) surveillance memo, handling fee decision gate, cybersecurity SOC mapping, materials ceiling language.

- **S3 (LH with multi-depot travel):** AFLCMC Wright-Patterson logistics, 4 LCATs, quarterly site visits to Hill AFB/Tinker AFB/Robins AFB, OASIS+ commercial vehicle, base + 3 options. Tests Dayton MSA 19430 (renumbered from 19380), GSA Per Diem pulls for three destinations, FTR 301-11.101 75% M&IE rule, FY-not-yet-published fallback, FAR 31.205-46 travel cost principle.

Each scenario had a 14-point binary assertion matrix covering skill activation, data source correctness, burden multiplier defensibility, FAR citation completeness, workbook structural integrity, methodology completeness, and staffing handoff respect.

### Environment

- claude.ai web chat, fresh conversation per scenario
- Skills installed: `igce-builder-lh-tm` plus required L1s (`bls-oews-api`, `gsa-calc-ceilingrates`, `gsa-perdiem-rates`)
- Model: Claude Opus 4.7 on Max plan
- All three scenarios hit tool-use limits and required one "continue" per run

### Grading

Grader (Claude Code session separate from worker runs) read the worker's final response plus the produced xlsx. Workers not coached during runs. Assertions graded binary pass/fail. Partial credits allowed only with explicit notation.

## Wave 1 results

| Scenario | Score | Fails |
|---|---|---|
| S1 Redstone LH | 14/14 | — |
| S2 NAVWAR T&M | 12/14 | Materials handling fee decision not surfaced; FAR 16.601(c)(3) surveillance memo missing |
| S3 AFLCMC LH+travel | 13/14 | FAR 31.205-46 not cited for travel |
| **Total** | **39/42 (92.9%)** | 3 fails |

## Round 1 findings: 17 skill bugs surfaced

Across three workers' self-critiques plus direct grader observation, 17 distinct findings emerged. All shipped in Round 1.

### P0 (must-fix, skill produced wrong numbers or failed to load resources)

1. **Stale CALC+ URL in Step 4 (lines 294, 300).** Skill cited `https://calc.gsa.gov/api/v3/api/ceilingrates/` which returns HTTP 404. Correct URL lives in the CALC+ skill itself. S2 and S3 workers burned round-trips discovering the drift. Patched by removing hardcoded URL and referring to the CALC+ skill as authoritative.

2. **Assumption block cell-reference drift (Step 8).** Skill prose at line 465 did not explicitly map every downstream `$B$n` reference. S2 worker shipped formulas referencing `$B$4` as escalation (actually Burden High = 2.2) and `$B$6` as Base Year Months (actually Productive Hours = 1920), producing $105M-per-FTE base year numbers before catching on value inspection. Recalc did NOT flag this because formulas were syntactically valid. Patched with explicit DOWNSTREAM CELL REFERENCES block to memorize before writing Sheet 1.

3. **Post-recalc per-FTE sanity gate missing.** `recalc.py` returning zero formula errors is necessary but NOT sufficient; syntactically valid formulas can reference wrong cells and produce wildly wrong values. Patched with Step 8.5 requiring per-FTE cost check in `[$100K, $1M]` range, plus Sheet 1 == Sheet 2 cross-check and burden-multiplier cross-sheet check.

### P1 (ship this round, completeness and resilience)

4. **Sheet 1 Grand Total == Sheet 2 Mid-Scenario Grand Total** post-recalc assertion added to Step 8.5.

5. **Sheet 5 merged-cell collision pattern** breaks openpyxl ("'MergedCell' object attribute 'value' is read-only"). S2 worker hit this and had to refactor. Patched with explicit rule: do NOT merge section-header cells while also writing values to column B of the merged range. Use dedicated header rows.

6. **FAR 31.205-46 (travel costs) not required in methodology.** S3 cited FTR 301-11.101 for M&IE 75% but missed 31.205-46. Patched into required FAR citation set.

7. **FAR 16.601(c)(3) (T&M surveillance) not required in methodology.** S2 missed this. Patched into required FAR citation set with explicit required language.

8. **Materials handling fee decision not surfaced.** S2 applied pure at-cost silently without flagging the handling fee decision. Patched with explicit Materials Handling Fee Decision Gate in Step 5B; default to at-cost but require explicit mention; cite FAR 31.205-26 if fee applied.

9. **BLS 503 retry guidance missing from orchestration skill.** Workers figured it out each time. Patched upstream in the BLS OEWS skill (Round 5) with explicit retry pattern.

10. **FY per diem fallback missing from orchestration skill.** S3 worker hit empty FY27 rates. Patched upstream in Per Diem skill (Round 1) with explicit fallback rule.

11. **CALC+ endpoint sanity check missing.** Workers hit 404s chasing wrong URLs. Patched upstream in CALC+ skill (Round 3).

### P2 (opportunistic, quality improvements)

12. **IT PM decision rule.** S1 worker burned a round-trip on 15-1299 (-21% vs CALC+) before pulling 11-3021 (+4%). Patched with decision table: DoD/IC IT PM defaults to 11-3021; civilian dual-pull.

13. **Network Engineer SOC disambiguation.** 15-1241 (architect) vs 15-1244 (sysadmin). Patched with decision rule; default 15-1241 conservative.

14. **SOC-not-at-MSA fallback.** S1 hit 15-1256 not published at Huntsville. Patched upstream in BLS skill (Round 5) with parent-SOC-family rollup.

15. **Productive hours user-override reconciliation.** Skill defaults to 1,880 but users may provide 1,920 in handoff. Patched with explicit "user input wins" rule and back-solve protocol.

16. **Seniority inference for implicit tiers.** S2 worker had to improvise P75 for "Security Project Manager" without explicit seniority label. Patched: cleared/technical PM defaults to P75.

17. **Divergence-triggered SOC re-pick automation + Raw Data retention + Contract vehicle usage rule + Scenario block row formula fix (12→15 rows) + Pre-delivery sanity checklist.** Bundled as Step 8.6 and additions to Step 1 mapping and Information to Collect.

## Round 1 patches shipped

All 17 findings above shipped as Round 1 patches in April 2026 immediately following Wave 1 grading. Key additions:

- Rewrote Step 4 to reference CALC+ skill as authoritative endpoint source
- Added DOWNSTREAM CELL REFERENCES map to Step 8
- Added Step 8.5 post-recalc sanity gates (3 checks)
- Added Step 8.6 pre-delivery sanity checklist (14 items)
- Added Materials Handling Fee Decision Gate to Step 5B
- Expanded Sheet 5 Methodology section with full FAR citation set
- Added PM decision table to Step 1
- Added Network Engineer disambiguation to Step 1
- Added Contract Vehicle Usage Rule table tuning burden ranges by vehicle
- Added Productive Hours Reconciliation rule
- Added Seniority inference for implicit tiers
- Added Divergence-triggered SOC re-pick and Raw Data retention requirements
- Fixed Scenario block row formula (12→15 rows)

## Round 2 patches queued

None block current ship state. Queued items emerged from grader observation but are not reproducibility bugs:

1. **Workflow A+ SOW/PWS decomposition** not tested in Wave 1. Needs dedicated scenarios.
2. **Workflow B rate validation only** not tested. Needs dedicated scenarios.
3. **24x7 shift coverage (Step 0.5)** not tested. Needs dedicated scenario.
4. **Retest all three Wave 1 scenarios** with Round 1 patches applied to confirm regression-free fix.

## Wave 2: Post-MCP migration + Wave 5 FFP inheritance + v2 ai-boundaries gate (Claude Code Desktop, Opus 4.7)

### Context

Wave 2 is the first LH/T&M testing round since Wave 1. It consolidated three streams of work into a single ship:

1. **Inheritance of six universal patches derived from FFP Wave 5.** ai-boundaries positioning, pre-flight MCP dependency check, Workflow B data-only rewrite, Step 0 two-stage validation gate, DoD installation to GSA per diem crosswalk, multi-destination travel sheet parameterization, CLI recalc fallback.
2. **v2 ai-boundaries gate.** The original FFP Wave 5 ai-boundaries patch failed a live LH/T&M test. In S3 (described below), the skill drafted a full price reasonableness memo with 5 separate "rate is fair and reasonable" determinations, recommended negotiation positions toward CALC+ P75, and drafted Evaluation Notice language, all forbidden by the ai-boundaries patch. Root cause: the gate lived at Workflow B Step 6 "Stop," which is too far downstream; by that step the model was already in "helpful memo author" momentum. Fix: moved the gate to Step 0 with a token-scan + verbatim refusal template + Option A/B bifurcation (Option A = positioning data only; Option B = memo template fill with CO's verbatim rationale and determination).
3. **Two end-to-end scenarios validating workbook production end-to-end** against the post-inheritance skill.

### Scenarios

- **S1 FHWA Application Modernization** (DOT civilian IT, 14 FTE, no travel). SOW-driven build (Workflow A+). Exercises SOW decomposition, Step 0 Stage A/B gate, PM dual-pull decision, civilian-IT wrap preset, 5-year PoP with escalation, CALC+ rate validation on mixed seniority team.
- **S2 Cyber/IR Pentagon** (DoD cleared Secret, 4 analysts, 4 travel destinations: Fayetteville NC, Huntsville AL, NSA Bethesda, San Francisco CA). Structured input build (Workflow A-LH). Exercises DoD installation crosswalk, multi-destination Sheet 5 parameterization, day-trip M&IE (Bethesda), cleared burden preset, FY rollover, small cleared team PM SOC choice.
- **S3 Senior Data Scientist $225/hr DC rate validation only** (Workflow B). Exercises the ai-boundaries gate. Wave 5 equivalent on FFP triggered the original patch; re-run here against the post-Wave-5 inherited skill is what surfaced the gate-positioning failure described above.

### S1 findings (FHWA Application Modernization)

Workbook built cleanly, $12.5M mid 3-year total, all 3 sanity gates passed (per-FTE in [100K,1M], Sheet 1 total = Sheet 2 mid total, burden multiplier cross-sheet consistent).

PM dual-pull caught divergence: initial SOC 11-3021 landed +34.9% above CALC+ P50 title-match, rejected and re-picked to 13-1082 (Project Management Specialist) which landed -13.1% within the expected tier band. Raw Data sheet retained both SOC queries showing the decision trail.

Evaluator found 10 skill issues, all patched before ship:

1. **Cyber/IR PM SOC rule gap.** Civilian-IT PM rule was clear (dual-pull 11-3021 vs 13-1082). Cyber/IR PM rule was not. Added: cyber/IR PM defaults to 13-1082 (Project Management Specialist) with CALC+ dual-pull for validation.
2. **PM P75 too aggressive for small cleared teams.** Default P75 for PM role produced overpricing when the PM is effectively a lead rather than a layer above a team. Added: P50 default when team size is 6 or fewer, with note to shift to P75 if PM is explicitly a separate management layer.
3. **Step 9 `present_files` CLI mismatch.** CLI does not have `present_files`. Worker improvised a file-path report. Codified: Step 9 environment fork (claude.ai / CLI / macOS Numbers), CLI path = absolute file path in response.
4. **Step 8.5 Gate 1 column refs stale.** Gate 1 referenced Sheet 2 columns by letter (D, E, F) after a prior Sheet 2 layout revision shifted burdened-rate column from E to F. Rewrote as named references.
5. **NSA Bethesda crosswalk points to wrong locality.** Was Montgomery County; should be DC composite per GSA convention for NSA Bethesda staff. Fixed in crosswalk table.
6. **FY rollover guidance missing.** Contract PoP starting within 6 months of FY rollover should query both FY rates and note refresh on publication. Added.
7. **Stage A/B gate skip for structured inputs.** Workflow A with SOW/PWS Builder structured handoff does not need Stage A decomposition approval; only Workflow A+ raw SOW text does. Added skip rule.
8. **Secret vs TS/SCI burden split not explicit.** Cleared burden preset was a single row. Split into Secret (2.0-2.2) and TS/SCI (2.2-2.4) rows with note about SCIF overhead not in BLS/CALC+ data.
9. **CALC+ `keyword_search` returning full corpus for stats-only queries.** Redirect to `igce_benchmark` for percentile queries where record-level data is not needed.
10. **Tier-matched keyword rule missing.** Query each seniority tier with its own keyword string. Aggregate title-match pools produce false divergence flags when a Senior LCAT is compared against a pool containing Juniors.

### S2 findings (Cyber/IR Pentagon)

Workbook built cleanly, $9.7M mid 3-year total. PM divergence-triggered re-pick caught +53.7% overpricing on 11-3021; re-picked to 13-1082 which landed at +11.7% within the cleared-team premium band. Multi-destination travel sheet built with 4 blocks (Fayetteville, Huntsville, DC composite for NSA Bethesda, SF), day-trip M&IE fired correctly for NSA Bethesda (same-day return from Pentagon). Burden tuning 2.0 / 2.2 / 2.4 landed on the Cleared IDIQ row of the contract vehicle table.

Evaluator found 8 skill issues, all patched:

1. **Cyber/IR PM SOC rule.** Same as S1; both scenarios hit this gap independently. Confirmed the patch.
2. **PM P75 aggressive for small cleared teams.** Same as S1. Confirmed.
3. **Step 9 `present_files` CLI mismatch.** Same as S1. Confirmed.
4. **Step 8.5 Gate 1 column refs stale.** Same as S1. Confirmed.
5. **NSA Bethesda crosswalk.** Same as S1. Confirmed; S2 would have shipped with wrong lodging rate if the S1 patch had not been in place.
6. **FY rollover guidance.** Same as S1. Confirmed.
7. **Stage A/B gate skip.** Same as S1. Confirmed; S2 used structured input and the Stage A prompt read as unnecessary friction.
8. **Secret vs TS/SCI burden split.** S2 was Secret; the original single cleared preset row would have nudged burden too high. Confirmed patch.

S1 and S2 corroborated each other on 8 of 10 issues; 2 issues (CALC+ redirect, tier-matched keyword rule) were S1-only but ported across.

### S3 findings (rate validation) - ai-boundaries gate failure

S3 exposed the Wave 5 FFP ai-boundaries patch as insufficient in production. The skill, when asked "is $225/hr reasonable for a Senior Data Scientist in DC," produced:

1. A full Price Reasonableness Determination memo template populated with 5 separate "fair and reasonable" determinations.
2. A recommended negotiation position toward CALC+ P75 ("push back if the vendor can't articulate the clearance value").
3. Draft Evaluation Notice language.
4. An invented 15-25% TS/SCI clearance premium applied as if it were market data.

All four outputs are Tier-1 ai-boundaries violations per the repository's ai-boundaries.md. The Wave 5 patch's instruction "do NOT assert fair/reasonable" sat in Workflow B Step 6; by the time the model reached Step 6 it had already drafted most of the memo via Steps 1-5. The "Stop" instruction read as advisory.

**Fix: v2 ai-boundaries gate.**

- Moved to Step 0 as a verbatim refusal-template token scan. If the user prompt contains any of `reasonable / fair / defensible / recommend / negotiate / push back / counter / Evaluation Notice / PNM / determination`, the skill emits the verbatim refusal template as its first response and offers two options:
  - **Option A:** positioning data only. Skill pulls CALC+ + BLS data, places the proposed rate on the distribution, produces a positioning sheet with neutral labels ("Within CALC+ FFP premium range" / "Metro geographic premium; see Methodology for factor decomposition" / "CO review recommended for factors outside BLS/CALC+ data"). No evaluative verbs in any output.
  - **Option B:** memo template fill. Skill produces a memo template with `[CO to complete]` placeholders in the Determination, Conclusion, and Recommendation sections. Skill only fills those sections verbatim if the CO supplies the rationale and conclusion in the prompt.
- Evaluative-verb scrub across all output paths. "Defensible," "reasonable," "acceptable," "competitive," "outlier" removed from Methodology sheet prose, chat summary, validation sheet Status column.
- Out-of-data premiums (TS/SCI, OCONUS hazard, SCIF overhead, specialty labor market) named as gaps; skill flags rather than invents ranges.
- ai-boundaries citation block added at the top of the skill naming the rule explicitly with examples of what the skill does and does not do.

S3 re-run after the v2 gate patch: the skill emitted the refusal template at Step 0, received "Option A" from the test caller, and produced a clean positioning sheet with no evaluative claims. Gate held.

### Cross-skill audit: bloat removed

The skill had accumulated redundancies and verbose prose across the inheritance. Audit cut 925 to 832 lines (-93) while shipping all patches:

- **Burden Multiplier Guidance section removed.** Duplicated the Vehicle table in worse form.
- **Step 8.5 and Step 8.6 merged.** 8.5 was "post-recalc sanity gates," 8.6 was "pre-delivery sanity checklist." The 3 gates and the 14-item checklist overlapped on 8 items. Consolidated to a single Step 8.5 with the 3 gates and 6 unique checklist items.
- **Edge Cases reduced to traps list.** Pre-audit Edge Cases mixed genuine silent-wrong-answer traps with quality suggestions. Split: traps stay in Edge Cases, quality suggestions moved to a new "Optional enhancements" appendix.
- **Quick Start Examples cut from 12 to 4.** The 4 retained cover the distinct pricing-structure decision gates. The 8 trimmed were restatements against different agencies.

### Wave 2 aggregate

| Metric | Value |
|---|---|
| Rounds | 3 (S1, S2, S3) |
| Workbooks / documents produced | 2 workbooks + 1 positioning sheet (post-gate) |
| Tier-1 ai-boundaries violations identified | 1 (S3, pre-v2-gate) |
| Skill defects identified | 18 unique (10 in S1, 8 in S2, 8 overlap) |
| Skill defects fixed | all 18 + v2 gate |
| Line delta | 925 to 832 (-93) |
| All 3 sanity gates passed in S1 and S2 | yes |

### What has not been tested

- Wave 1 S1/S2/S3 scenario retest on the Wave-2-patched skill.
- 24x7 shift coverage (Step 0.5) still not exercised.
- Full Workflow B Option A positioning sheet production (covered briefly in S3 post-gate; no extended test).
- Option B memo template fill with CO-supplied rationale.
- Sonnet 4.6 parity.

Queued for Wave 3.

## Independent grading methodology

Wave 1 testing record produced under consistent methodology:

- Scenarios and assertion matrices committed in writing before any worker output was read
- Grader did not coach workers during runs
- Assertions graded strict on literal wording; soft fails noted explicitly
- Worker self-critiques incorporated as findings when corroborated by observation
- All findings come from direct observation of worker output, not inference from memory

---

*Testing record prepared April 2026 by James Jenrette / 1102tools. Independent grading methodology. MIT licensed. Source: github.com/1102tools/federal-contracting-skills.*
