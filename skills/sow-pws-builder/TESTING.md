# SOW/PWS Builder: Testing Record

## Executive Summary

This skill was tested end-to-end against eight federal acquisition scenarios across two Claude models on claude.ai web chat, the same environment the skill's actual end users run in. A separate Claude instance graded each run independently against a 14-point assertion matrix covering FAR compliance, document structure, voice consistency, and handoff format. The tests surfaced one confirmed bug and several behavioral inconsistencies. All findings were addressed in skill version with nine patches before publication.

| Metric | Value |
|---|---|
| Test runs | 8 (6 canonical scenarios plus 2 lazy-prompt stress tests) |
| Claude models tested | Claude Opus 4.7 and Claude Sonnet 4.6 |
| Workflows exercised | All three (Full Build, SOO Conversion, Scope Reduction) |
| Scenario assertions graded | 84 |
| Scenario assertions passed | 83 |
| Confirmed bugs found | 1 (fixed in patch) |
| Skill improvements shipped | 9 |

## What Was Tested

Three canonical scenarios covering every workflow in the skill, plus a lazy-prompt stress test of the AskUserQuestion feature.

### Scenario 1: Non-commercial T&M SOW

Workflow A (Full Build). Test Range Instrumentation and Data Collection Support for the Air Force Research Laboratory. Classified weapons testing context under FAR Part 15 and FAR 16.601. Exercises the T&M Labor Category Ceiling Hours exception under FAR 16.601(c)(2).

### Scenario 2: SOO-to-PWS Conversion

Workflow B. Treasury Digital Customer Experience Modernization. Commercial Firm-Fixed-Price service under FAR Part 12 converted from an existing Statement of Objectives. Exercises SOO parsing, gap identification, carry-forward of decided items, and performance-based outcome framing.

### Scenario 3: Scope Reduction

Workflow C. Cybersecurity Incident Response and Digital Forensics PWS reduced from 62 million per year to 40 million per year budget ceiling. Exercises trade-off menu generation, user validation gate, section regeneration, updated staffing handoff, and Section 14 cut documentation.

### AskUserQuestion Test

One-sentence lazy prompt ("Need a PWS for help desk services for my agency. Can you help me write one?") to test whether the skill drives structured multi-choice intake when a non-expert user provides minimal context. This is the flagship experience for the skill's target audience of federal requirement writers who may not have contracting expertise.

## How It Was Tested

Each worker session ran end-to-end in a fresh claude.ai web chat. A separate Claude Code Opus 4.7 instance (running at 1M context window, max effort mode, Claude Max 20x subscription) graded each output. The grader had no access to the worker's drafting conversation, ensuring "observed behavior" was distinct from "claimed behavior."

Each run was graded against 14 binary pass or fail assertions:

**General assertions (8):** Acquisition Strategy Intake collected; Phase 1 decision tree executed; .docx produced; no FTE counts or SOC codes or staffing tables in document body (FAR 37.102(d)); no staffing or IGCE appendix; staffing handoff chat-only not saved as file; section numbering sequential; Key Personnel names roles with qualifications not counts.

**Scenario-specific assertions (6 per scenario):** SOW task language versus PWS outcome language; Section 3 organized by task or objective; T&M Section 5 Labor Category Ceiling Hours table per FAR 16.601(c)(2); Labor Category table columns clean; FAR Part 12 or 15 clause structure applied correctly; scenario-specific workflow behavior (SOO gap ID for Scenario 2; trade-off menu for Scenario 3).

Each assertion received a binary pass or fail result plus a one-line note on anything suspicious.

## Results by Scenario

### Opus 4.7 worker results

| Scenario | Workflow | Assertions | Result |
|---|---|---|---|
| Non-commercial T&M SOW | A | 14 of 14 pass | PASS |
| SOO-to-PWS Conversion | B | 14 of 14 pass | PASS |
| Scope Reduction | C | 14 of 14 pass | PASS |

### Sonnet 4.6 worker results

| Scenario | Workflow | Assertions | Result |
|---|---|---|---|
| Non-commercial T&M SOW | A | 14 of 14 pass | PASS |
| SOO-to-PWS Conversion | B | 14 of 14 pass | PASS |
| Scope Reduction | C | 13 of 14 pass | FAIL (G4) |

### AskUserQuestion feature test

| Model | Behavior observed |
|---|---|
| Opus 4.7 | Defaulted to prose questions on first pass. Reached for the structured multi-choice tool only when explicitly prompted by the user. |
| Sonnet 4.6 | Auto-triggered the structured multi-choice tool on the first message. Batched 12 questions across 4 groups of 3. Included "Not sure" and "Something else" escape hatches. Produced a 422-paragraph contract-file-ready PWS after the user selected option one for every question. |

### The one confirmed bug

**Sonnet Scenario 3 G4 failure.** The Workflow C scope-reduction documentation in Section 14 contained specific staffing counts ("3-4 FTE examiners," "2-3 platform engineers," "8 Tier 2 analysts") as prior-state descriptors. Document bodies remain subject to FAR 37.102(d) even when describing historical or prior state. Opus passed the same scenario using capability-framed language ("night-shift onsite headcount eliminated," "no dedicated playbook engineer") without specific counts.

**Root cause:** The skill's FAR 37.102(d) enforcement block was focused on staffing tables and the Phase 3 handoff. It did not explicitly address Section 14 scope-reduction narrative. Workers interpreted "describe what was cut" as license to quote the prior staffing plan.

**Fix:** Added explicit guidance in the Section 14 block requiring capability-and-coverage language, not staffing counts, for both current-state and historical-state descriptions. Included compliant and non-compliant examples. Prescribed the five-field documentation structure (Prior Scope, Revised Scope, Estimated Annual Savings, Rationale, Residual Risk).

## Cross-Scenario Patterns

Several behavioral inconsistencies emerged across multiple runs that did not fail the assertion matrix but indicated room for skill improvement.

| Pattern | Observed in | Skill response |
|---|---|---|
| Staffing handoff table referenced but not emitted | 3 of 7 runs across both models | Moved unconditional emission rule to top of Phase 3 with emphatic language |
| Phase 1 decision tree compressed silently when user said "use defaults" | Both models, multiple runs | Added Phase 2 Invocation Gate requiring visible Phase 1 summary before docx generation |
| SOW documents used "QASP Summary" label (PWS terminology) for Section 12 | Both models, Scenario 1 | Added SOW versus PWS label differentiation in Section 12 block |
| Redundant framing questions re-asked despite explicit user answers | Both models, AskUserQuestion test | Added anti-redundancy rule requiring prompt review before asking |
| No Table of Contents in multi-section documents | All runs | Added TOC instruction for documents exceeding 8 sections |

## Changes Made Based on Testing

Nine skill patches were applied before publication of this version. Each addresses a specific observed behavior or failure mode.

| Fix | Section affected | Trigger |
|---|---|---|
| Unconditional handoff rule moved to top of Phase 3 with emphatic language | Phase 3 opening | 3 of 7 runs skipped handoff emission |
| SOW "Inspection and Acceptance" versus PWS "QASP" label differentiation | Phase 2 Section 12 block | Scenario 1 runs both models |
| Table of Contents instruction for documents exceeding 8 sections | Phase 2 opening | All runs produced 300-700 paragraph docs without TOC |
| SOO-implied objectives rule | Phase 0 | Scenario 2 both models added unstated objectives legitimately |
| Phase 2 Invocation Gate requiring Phase 1 Decision Summary before docx | Phase 2 opening | Cross-model Phase 1 compression plus network-blip resilience |
| Section 14 assumption format template (4-column) | Phase 2 Section 14 block | Workers independently invented divergent structures |
| Workflow C Section 14 compliance rule (no FTE counts in cut descriptions) | Phase 2 Section 14 block | The only G4 failure of the wave |
| AskUserQuestion tool usage instruction | Phase 1 opening | Opus defaulted to prose without explicit instruction |
| Anti-redundancy rule | Phase 1 opening | Both models re-asked explicit answers |

Skill version before patches: 361 lines. Version after patches: 380 lines. The ceiling for a single skill file is 1,000 lines.

## What Was Not Tested

The tests above cover the three core workflows and the most common contract-type and commercial-determination combinations. Several edge cases and contract types were not exercised and remain untested in this wave:

- Cost-Reimbursement contract variants (CPFF, CPAF, CPIF) were not exercised end-to-end.
- Labor-Hour-only contracts (T&M without materials) were not separately tested; T&M coverage is assumed to generalize.
- Hybrid contract type structures were not tested.
- Special Access Program (SAP) and Special Access Required (SAR) requirements were not exercised. DoD classified work was tested at the Secret and Top Secret collateral level only.
- The Workflow C scope reduction test used a bulleted summary of the prior PWS rather than an actual prior .docx. True document-to-document section preservation was not verified.
- Multi-year ramping Acceptable Quality Level structures (where targets escalate across option years) were exercised informally but not systematically.
- Hardware-intensive Test and Evaluation domains were tested once (Scenario 1) but not across multiple runs.

Users working in these contexts should expect to validate outputs more carefully and may encounter edge cases that the test wave did not surface.

## Independent Grading

Every assertion was graded by a separate Claude instance reading only the worker's final output and the chat transcript. The grader had no access to the worker's internal reasoning and did not see the worker's self-assessment until after grading was complete. This separation is the load-bearing credibility claim of this testing program: it distinguishes "the worker claimed to do X" from "the output demonstrates X."

---

**Testing Methodology**

Evaluators: James Jenrette (1102tools) and Claude Code Opus 4.7 (1M context window, max effort mode, Claude Max 20x subscription).

Worker models tested: Claude Opus 4.7 and Claude Sonnet 4.6 on claude.ai web chat, the same environment the skill's end users run in.

Test runs: 8. Assertions graded: 84. Passes: 83. Confirmed failures: 1 (fixed in patch).

Date: April 2026.

Skill: sow-pws-builder, post-patch version. Source: github.com/1102tools/federal-contracting-skills. License: MIT.
