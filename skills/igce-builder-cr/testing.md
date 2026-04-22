# IGCE Builder CR: Testing Record

# Part 1: For Federal Acquisition Users

## The bottom line

This testing record documents inherited patches from FFP Wave 5 and LH/T&M Wave 2 testing. **No CR-specific scenarios have been run directly against this skill yet.** CR regression testing is queued for a future wave.

The CR skill orchestrates the same three MCP servers (bls-oews, gsa-calc, gsa-perdiem) as FFP and LH/T&M, produces structurally similar workbooks (cost pool buildup with fee analysis instead of wrap rate or burden multiplier), and inherits the same patch set validated on those siblings.

## Inherited patches (Wave 1, not directly tested on CR)

All universal patches derived from FFP Wave 5 and LH/T&M Wave 2 testing apply to CR:

- **ai-boundaries positioning (v2 gate):** Workflow B Step 0 token-scan + verbatim refusal template. Skill does not originate "fair and reasonable" determinations, price reasonableness memos, or negotiation recommendations. Two options at gate: Option A (positioning data only) or Option B (memo template fill with CO's verbatim rationale + determination).
- **Pre-flight MCP dependency check:** validates bls-oews, gsa-calc, gsa-perdiem tools and API keys before any workflow runs.
- **Step 0 two-stage validation gate:** Stage A (decomposition) + Stage B (build parameters including fee type: CPFF/CPAF/CPIF) as separate AskUserQuestion calls. Skip for Workflow A with structured inputs.
- **DoD installation to GSA per diem crosswalk:** 15-row table mapping military installations to GSA civilian localities. NSA Bethesda routes to DC composite (not Montgomery County).
- **Multi-destination travel sheet:** Sheet 5 parameterized for M destinations, Sheet 1 Travel SUMs across blocks.
- **CLI recalc fallback:** Python expected-total check when LibreOffice recalc.py unavailable.
- **Step 9 environment fork:** delivery path varies by environment (claude.ai / Claude Code CLI / macOS Desktop with Numbers).
- **CALC+ query optimizations:** keyword_search to igce_benchmark for stats-only; tier-matched keywords (not aggregate pool) to avoid false divergence flags.
- **FY rollover guidance:** if contract PoP start within 6 months of next FY, query both and document refresh on publication.
- **Raw Data sheet granularity:** summary tables with query parameters, not raw JSON dumps.

## What has NOT been tested on CR

- SOW/PWS decomposition into CR-specific labor mix
- Fee structure selection workflow (CPFF vs CPAF vs CPIF)
- CPAF award fee pool modeling with assumed-earned percentage
- CPIF share ratio mechanics and 3x3 matrix (cost scenarios x fee outcomes)
- Statutory fee cap enforcement (15% R&D, 10% non-R&D practical ceiling)
- BAA-driven builds under FAR 35.016
- Cost pool rate scenario analysis (fringe/overhead/G&A at low/mid/high)
- 24x7 coverage with CR fee math
- Multi-location CR with fee calculated on each location's cost

## Regression plan (queued)

A Wave 2 CR testing round should cover at minimum:
- One CPFF BAA-driven build with R&D scope
- One CPAF build with 10-person team and award fee pool
- One CPIF build with share ratio and 3x3 matrix
- Rate validation (Workflow B) with ai-boundaries gate exercised

---

*Testing record prepared April 2026 by James Jenrette / 1102tools. Inherited-only status documented honestly. MIT licensed. Source: github.com/1102tools/federal-contracting-skills.*
