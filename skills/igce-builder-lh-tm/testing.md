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

## Independent grading methodology

Wave 1 testing record produced under consistent methodology:

- Scenarios and assertion matrices committed in writing before any worker output was read
- Grader did not coach workers during runs
- Assertions graded strict on literal wording; soft fails noted explicitly
- Worker self-critiques incorporated as findings when corroborated by observation
- All findings come from direct observation of worker output, not inference from memory

---

*Testing record prepared April 2026 by James Jenrette / 1102tools. Independent grading methodology. MIT licensed. Source: github.com/1102tools/federal-contracting-skills.*
