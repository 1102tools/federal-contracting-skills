---
name: sow-pws-builder
description: >
  Build Statements of Work (SOW) or Performance Work Statements (PWS) for
  federal contracts through a structured scope decision workflow that produces
  a contract-file-ready .docx and a separate chat-only staffing handoff table
  for the IGCE Builder. Use this skill whenever the user asks to write a SOW,
  PWS, statement of work, performance work statement, convert a SOO to a
  SOW/PWS, develop requirements, define contract scope, build a work statement,
  or draft requirement language. Also trigger when the user says they need a
  SOW/PWS before building an IGCE, when they have a SOO and need to develop it
  into executable requirements, or when they need to reduce scope to fit a
  budget. The SOW/PWS document NEVER contains
  staffing tables, FTE counts, labor category counts, or IGCE-related content;
  that data lives in a separate chat-output handoff for the IGCE Builder
  skills (FFP, LH/T&M, CR). Do NOT use for IGCEs themselves (use IGCE Builder
  skills). Do NOT use for market research reports (use Market Research Builder).
---

# SOW/PWS Builder

## Overview

This skill walks a program office (or analyst acting on their behalf) through structured scope decisions and assembles the answers into a properly formatted SOW or PWS. The core insight: the program office doesn't need to know FTE counts or cost estimates up front — they need to make scope decisions, and staffing falls out as a derived output.

**Outputs (TWO SEPARATE ARTIFACTS — never combined):**

1. **A .docx SOW or PWS** with standard federal sections — the contract file deliverable. This document contains NO staffing table, NO FTE counts, NO labor category counts, NO SOC codes, NO hours-per-year estimates, and NO IGCE-related content. FAR 37.102(d) requires the requirement to be described in terms of results, not hours or number of people performing the work. The one narrow exception is T&M/LH contracts, which get a Labor Category Ceiling Hours table in Section 5 per FAR 16.601(c)(2) — see Phase 2 Section 5 for details.

2. **A staffing handoff table** — an internal government workpaper presented in **chat output only** at the end of the skill run. This is the data handoff to the IGCE Builder skills (FFP, LH/T&M, CR). It is NEVER embedded in the SOW/PWS document. It is NEVER saved as a companion .docx or any other file. It exists solely as a markdown/code-block table in the conversation so the user can review it and the downstream IGCE Builder can consume it.

**No external APIs required.** This is a decision tree + document generation skill.

**Regulatory basis:** FAR 37.102(d) (describe work in terms of results, not hours or number of people), FAR 37.602 (performance-based acquisition), FAR 46.401 (quality assurance), FAR 7.105 (written acquisition plans — requirement description), FAR 16.601(c)(2) (T&M/LH ceiling price mechanism).

## Workflow Selection

### Workflow A: Full Build (Default)
User needs a SOW/PWS from scratch or from a rough concept. Execute all three phases.
Triggers: "write a SOW," "build a PWS," "I need a work statement for..."

### Workflow B: SOO Conversion
User has an existing SOO and needs it developed into a SOW/PWS. Start with Phase 0 (SOO intake), then Phases 1–3.
Triggers: "convert this SOO to a SOW," "develop this SOO into a PWS," "we have a SOO and need a work statement."

### Workflow C: Scope Reduction
User has an existing SOW/PWS or IGCE output that exceeds budget. Walk through scope decision tree to identify what to cut, then produce a revised document.
Triggers: "this is too expensive, what can we cut," "reduce scope to fit budget," "need to descope."

## Acquisition Strategy Intake

Before diving into the scope decision tree, collect three framing decisions that shape everything that follows. Ask these up front, in a single pass, before Block 1. These are the acquisition strategy context — they determine document type, pricing structure, and FAR Part applicability.

### Intake Question 1: SOW or PWS?

| | SOW | PWS |
|---|---|---|
| Prescribes | Tasks and methods (how) | Outcomes and standards (what) |
| Contractor flexibility | Low — government directs approach | High — contractor proposes approach |
| Best for | Well-understood, repeatable work | Complex or innovative requirements |
| QASP focus | Task completion | Performance metrics |
| FAR preference | — | FAR 37.602 prefers PBA/PWS |

If the user is unsure, default to PWS — it's the FAR-preferred approach for services and gives the contractor room to propose efficient solutions. The decision tree is identical either way; only the output language changes.

### Intake Question 2: Contract Type?

**FFP | T&M | LH | CR | Hybrid (specify)**

This is asked up front (not in Block 5) because contract type frames the rest of the scope decisions:

- **FFP** pairs naturally with PWS (outcomes + performance standards). Contractor owns labor mix risk. No labor info in the document body.
- **T&M** and **LH** require Labor Category Ceiling Hours in Section 5 per FAR 16.601(c)(2). The SOW/PWS must tell offerors what LCATs to propose rates against and the ceiling hours per period.
- **CR** (cost-reimbursement) pairs with either SOW or PWS. No labor info in the document body. Requires approved accounting system findings before award per FAR 16.301-3.
- **Hybrid** — identify which CLINs are which type; apply the rules per CLIN.

If the user says "unsure," recommend FFP for well-defined services, T&M for requirements where level of effort is unknown, and CR for R&D or high-uncertainty work. FAR Part 16 analysis is the contracting officer's call — this skill accepts the user's decision; it does not advise on contract type selection.

### Intake Question 3: Commercial or Non-Commercial?

**Commercial service (FAR Part 12) | Non-commercial (FAR Part 15) | Unsure — flag for Contracting Officer**

- **Commercial service** = routinely sold to the general public in the course of normal business operations, per FAR 2.101 commercial product/service definition. Help desk services, facility maintenance, COTS software support, training, staff augmentation for standard skills are typically commercial.
- **Non-commercial** = government-unique work such as classified, research, weapon systems development, or work performed using specialized government processes not available in the commercial marketplace.
- When **commercial**, FAR Part 12 governs. The SOW/PWS Section 5 CLIN structure should reference FAR 52.212-4 terms; full Section I/L buildout is not required. Contract type is typically FFP per FAR 12.207. Performance-based description is preferred per FAR 12.102(g).
- When **non-commercial**, FAR Part 15 governs. Full Section I/L clause buildout applies. All contract types available.
- If the user is unsure, note "Commercial item determination pending — flag for Contracting Officer" in Section 14 Constraints and Assumptions and proceed with Phase 1.

### Why these three questions come first

The issue a requirement writer faces: the same scope decisions produce different document language, different clause references, and different FAR Part applicability depending on these three framing choices. Collecting them up front means Phase 1 can frame its questions accordingly (e.g., for a commercial FFP PWS, skip Phase 1 questions about Government-unique processes; for a T&M SOW, ask more detail about labor categories and ceilings). It also means the generated document in Phase 2 can reference the correct FAR sections and clauses without a retrofit.

## Phase 0: SOO Intake (Workflow B Only)

When the user provides an existing SOO:

1. **Read and extract.** Parse the SOO for: background, objectives, constraints, performance location, period of performance, known systems, volume data, key personnel, security requirements, and any appendix data.

2. **Gap identification.** Flag what the SOO provides vs. what a SOW/PWS needs:
   - ✅ Typically present in SOO: background, high-level objectives, constraints, PoP, location
   - ❌ Typically missing from SOO: task decomposition, staffing, deliverables with acceptance criteria, CLIN structure, QASP metrics, reporting cadence, transition details

3. **Decision bridge.** Present the gaps as the questions Phase 1 will answer. Frame it as: "The SOO tells us what the agency wants to achieve. Phase 1 asks the questions that turn objectives into executable requirements."

4. **Carry forward.** Pre-populate Phase 1 answers with anything the SOO already decided (location, PoP, known constraints). Don't re-ask settled questions.

## Phase 1: Scope Decision Tree

Ask questions in this sequence. Each decision narrows scope and implies staffing. Present as structured choices, not open-ended questions. Collect in a single pass where possible.

### Block 1: Mission and Service Model

1. **What is the core service?** (Provide options based on context, or ask open-ended if starting from scratch)
2. **Service delivery model:** Government FTEs with contractor augmentation | Fully contracted service | Hybrid (specify which functions are FTE vs. contractor)
3. **Coverage model:** Business hours (M-F 8-5) | Extended hours (specify) | 24/7/365 | Seasonal/surge
4. **Geographic scope:** Single site | Multi-site (how many?) | Virtual/remote | Hybrid

### Block 2: Technical Scope

5. **Build vs. Buy:** Custom development | COTS/SaaS configuration | Hybrid (custom integrations on COTS platform)
6. **Systems in scope:** List all systems the contractor will build, configure, integrate with, or maintain. For each: new build vs. existing system.
7. **Integration complexity:** Standalone | Integrates with 1-3 systems | Integrates with 4+ systems | Enterprise-wide integration
8. **Data migration:** No legacy data | Migrate from 1-2 sources | Migrate from 3+ sources | Complex multi-system consolidation
9. **AI/automation:** None | Basic automation (IVR, rules-based routing) | AI-assisted (NLP, ML classification) | Advanced AI (chatbots, predictive analytics)

### Block 3: Scale and Volume

10. **Transaction/contact volume:** Provide numbers if known, or characterize as low/medium/high
11. **User population:** Internal users (how many?) | External/public-facing | Both
12. **Concurrent user requirements:** If applicable
13. **Growth expectations:** Stable | Moderate growth (10-25%/yr) | High growth (>25%/yr) | Unknown

### Block 4: Organizational Scope

14. **How many organizational units?** (offices, centers, divisions served)
15. **Phasing:** All at once | Phased rollout (how many phases?) | Pilot then expand
16. **Stakeholder complexity:** Single program office | Multiple offices, single agency | Cross-agency

### Block 5: Contract Structure

Contract type is already captured in the Acquisition Strategy Intake above; do not re-ask it here.

17. **Period of performance:** Base year + option years (how many?)
18. **Base year scope:** Full performance from day 1 | Ramp-up/transition-in period (how long?) | Design/development only, production in options
19. **CLIN structure preference:** By period (CLINs = base year, OY1, OY2...) | By function (CLINs = development, operations, maintenance, training) | By deliverable | Unsure (skill will recommend)
20. **Transition requirements:** Transition-in from incumbent? | Transition-out plan required? | Both

### Block 6: Quality and Oversight

21. **Acceptable quality level (AQL):** For key performance metrics — e.g., 95% first-call resolution, 99.95% uptime, <2hr mean time to resolve
22. **Reporting cadence:** Weekly | Monthly | Quarterly | As-needed
23. **Key personnel:** Which roles must be designated as key? (PM always; others?)

### Decision-to-Staffing Derivation Rules

After collecting decisions, derive the staffing implications. These are heuristics, not formulas:

| Decision | Staffing Implication |
|---|---|
| 24/7 coverage | Minimum 3x the single-shift headcount for covered roles |
| Custom development | 1 architect + 2-4 developers per major system/module |
| COTS configuration | 1 architect + 1-2 configurators per platform |
| AI/NLP components | +1-2 data scientists or ML engineers |
| 4+ system integrations | +1-2 integration/middleware engineers |
| Data migration from 3+ sources | +1 data engineer + 1 DBA (may be time-limited to base year) |
| Multi-site or 3+ org units | +1 change management/training specialist per 2-3 units |
| Contact center: volume ÷ 250 days ÷ contacts/agent/day | = required agent headcount (adjust for FTE vs. contractor split) |
| Agile development | 1 scrum master or PM per 2 dev teams (team = 5-7 people) |
| FISMA/security requirements | +1 information security analyst |
| O&M phase | Typically 40-60% of development-phase staffing |
| Transition-in | +0.5-1 FTE for knowledge transfer (time-limited) |

**Present the derived staffing table to the user for validation before proceeding to Phase 2.** Format:

```
Task Area               | Labor Category      | Est. FTEs | Derivation Basis        | Phase
Development             | Solution Architect  | 1         | Enterprise CRM platform | Base-OY1
Development             | Software Developer  | 4         | 2 Agile teams × 2 devs  | Base-OY1
Operations              | Tier 1 Agent        | 10        | 127K contacts ÷ vol/day | Base-OY4
O&M                     | Systems Admin       | 2         | 50% of dev staffing     | OY2-OY4
```

**User validation gate.** The user confirms, adjusts, or overrides any line. If they override, document the override reason — it matters for the IGCE methodology narrative.

## Phase 2: Document Assembly

Generate the SOW or PWS using the docx skill. Read `/mnt/skills/public/docx/SKILL.md` before generating output.

### SOW/PWS Section Structure

**Section 1: Introduction**
- 1.1 Purpose
- 1.2 Background (from SOO or user input)
- 1.3 Scope Summary (one paragraph synthesizing all Phase 1 decisions)
- 1.4 Applicable Documents and Standards

**Section 2: Definitions and Acronyms**

**Section 3: Requirements** (this is the core — structure depends on SOW vs. PWS)

*For SOW:* Section 3 is organized by task area. Each task has:
- Task number and title
- Task description (what the contractor shall do)
- Subtasks if applicable
- Deliverables produced by this task
- Government-furnished resources for this task

*For PWS:* Section 3 is organized by performance objective. Each objective has:
- Objective number and title
- Required outcome (what the contractor shall achieve)
- Performance standard (measurable threshold)
- Acceptable Quality Level
- Method of assessment (inspection, demonstration, analysis, test)
- Incentive/disincentive if applicable

**Section 4: Deliverables**
- Deliverables table: ID | Title | Description | Format | Frequency | Due Date/Trigger | Acceptance Criteria
- Standard deliverables to always include: Monthly Status Report, Transition-In Plan (if applicable), Transition-Out Plan, System Documentation, Training Materials

**Section 5: CLIN Structure**
- Map deliverables and tasks/objectives to CLINs
- Note which CLINs are priced (FFP) vs. estimated (T&M/LH) vs. cost-reimbursable
- Include CLIN descriptions suitable for the schedule
- **T&M/LH exception (FAR 16.601(c)(2)):** If the contract type selected in the Acquisition Strategy Intake is T&M or LH, include a **Labor Category Ceiling Hours** table in Section 5. This table tells offerors which labor categories the government expects them to propose rates against and the ceiling hours per LCAT per period. It is a pricing risk-sharing mechanism, NOT an FTE estimate or a staffing prescription. Table columns: `Labor Category | Base Year Ceiling Hours | OY1 | OY2 | OY3 | OY4 | Total Ceiling`. Do NOT include SOC codes, FTE counts, "derivation basis," or anything resembling the Phase 3 staffing handoff table — those remain in chat output only. For FFP and CR contracts, omit this table entirely; no labor information goes in Section 5 or anywhere else in the document body.

**Section 6: Period of Performance**

**Section 7: Place of Performance**

**Section 8: Government-Furnished Property/Information**

**Section 9: Security Requirements**

**Section 10: Key Personnel**
- Roles, minimum qualifications, certification requirements
- Only roles where government needs approval of specific individuals
- Include the FAR 52.237-2 substitution language: substitution of Key Personnel requires prior written approval of the Contracting Officer
- **DO NOT include FTE counts, labor category counts, SOC codes, hours-per-year, staffing tables, or any "how many" information in Section 10 or anywhere else in the SOW/PWS body.** Key Personnel names ROLES (Program Manager, Technical Lead, Information Security Officer) with minimum qualifications — it does not quantify the workforce. FAR 37.102(d) requires the requirement to be described in terms of results, not hours or number of people performing the work. The Phase 3 staffing handoff table is for the IGCE Builder only, lives in chat output, and never appears in this document — not as a section, not as an appendix, not as a table at the bottom.

**Section 11: Reporting and Oversight**
- Reporting schedule and content requirements
- Meeting cadence (kickoff, weekly status, monthly program review, quarterly executive)
- Points of contact

**Section 12: Quality Assurance Surveillance Plan (QASP) Summary**
- Performance metrics table: Metric | Standard | AQL | Method | Frequency | Incentive
- Full QASP may be a separate document; include summary here

**Section 13: Transition**
- 13.1 Transition-In (if applicable): knowledge transfer, incumbent cooperation, parallel operations
- 13.2 Transition-Out: data return, documentation, contractor cooperation, timeline

**Section 14: Constraints and Assumptions**

**Appendices** (as applicable):
- A: Current Environment Description
- B: Volume Data and Historical Metrics
- C: System Interface Specifications
- D: Acronym List

**The appendix set above is the only authorized appendix set for the SOW/PWS.** Include only those appendices that are applicable to the requirement; omit those that are not. **DO NOT** create any appendix with the following titles or similar phrasing, under any circumstances:

- "Appendix A: Implied Staffing Table"
- "Appendix A: Staffing Implications Table"
- "Appendix A: Staffing Table"
- "Appendix A: Labor Category Staffing"
- "Appendix A: IGCE Handoff"
- "Appendix A: FTE Allocation"

**DO NOT** replace the default appendix set (A: Current Environment, B: Volume Data, C: System Interface Specifications, D: Acronym List) with a staffing-related appendix. **DO NOT** append a staffing table after Appendix D as a supplementary section. The staffing handoff is a SEPARATE chat-output artifact (see Phase 3) — it does not belong anywhere in this document. If the program office wants to see the staffing estimate before running the IGCE Builder, they can read it from the chat output; it does not need to be in the contract file deliverable.

### Language Rules

**SOW language:** "The contractor shall [verb]..." — prescriptive, task-oriented.
**PWS language:** "The contractor shall achieve/maintain/ensure..." — outcome-oriented, measurable.

**Avoid:**
- "Support" as a standalone verb (too vague — support how?)
- "As needed" or "as required" without defining the trigger
- "Best practices" without specifying which standard
- "Coordinate with" without specifying the deliverable or decision that results
- Requirements that cannot be measured or verified

**Every requirement must be:** specific, measurable, achievable, relevant, and time-bound. If a requirement fails any of these, flag it for the user during assembly.

## Phase 3: Validation and Handoff

### Document Review Checklist

Before presenting the final document, verify:

- [ ] Every task/objective maps to at least one deliverable
- [ ] Every deliverable maps to at least one CLIN
- [ ] Every CLIN has a pricing basis (FFP, T&M, etc.)
- [ ] Key personnel roles match the staffing table
- [ ] Security requirements are consistent throughout
- [ ] Period of performance matches CLIN structure
- [ ] QASP metrics are measurable and have defined AQLs
- [ ] Transition-in/out timelines are realistic
- [ ] No orphaned requirements (stated but never deliverable-mapped)
- [ ] No scope gaps (Phase 1 decisions not reflected in requirements)

### Staffing Handoff Table (CHAT OUTPUT ONLY — NEVER IN THE DOCUMENT)

**After the SOW/PWS .docx is saved to disk, present the staffing handoff table as chat output only.** It is NOT a section of the SOW/PWS. It is NOT saved as a .docx, .xlsx, .csv, or any other file. It exists solely as a markdown table in the conversation so (a) the user can review the derived staffing for scope validation, and (b) the IGCE Builder skill can consume it as input when the user says "build the IGCE."

Present it like this, verbatim:

```
=== STAFFING HANDOFF TABLE — FOR IGCE BUILDER ===
Internal government workpaper. NOT part of the SOW/PWS contract deliverable.
Do not paste this into the contract file.

| Labor Category          | SOC Code | FTEs | Phase        | Hrs/Yr | Notes |
|-------------------------|----------|------|--------------|--------|-------|
| Program Manager         | 13-1082  | 1    | Base-OY4     | 1,880  | Key personnel |
| Software Developer      | 15-1252  | 4    | Base-OY1     | 1,880  | Reduces to 1 in OY2+ |
| Tier 1 Agent            | 43-4051  | 10   | Base-OY4     | 1,880  | Derivation: 127K contacts ÷ volume/day |
| Systems Admin           | 15-1244  | 2    | OY2-OY4      | 1,880  | O&M phase only |
```

Include rows for every labor category derived in Phase 1. Notes should capture user overrides and derivation basis. Columns are fixed: Labor Category, SOC Code, FTEs, Phase, Hours/Yr, Notes.

After the table, tell the user in plain chat prose: *"This staffing table is ready for handoff to the IGCE Builder. Say 'build the IGCE' and I'll hand it off to the FFP, LH/T&M, or CR skill based on the contract type selected in the Acquisition Strategy Intake. The IGCE Builder will produce the formal Independent Government Cost Estimate as a separate .docx document — that's where the staffing, rates, wraps, and costs are formally documented for the contract file."*

**DO NOT, under any circumstances:**

- Write the staffing table into the SOW/PWS document body, any section, or any appendix.
- Save the staffing table as a separate .docx, .xlsx, .csv, .md, or any other file format.
- Label any part of the SOW/PWS as "Staffing Implications Table," "Implied Staffing Table," "Staffing Table," "Labor Category Staffing," "IGCE Handoff," or any similar phrasing.
- Include the phrases "ready for the IGCE Builder," "say 'build the IGCE'," "FFP skill," "IGCE development," or any skill-chain plumbing messaging anywhere in the SOW/PWS document.
- Emit the table as anything other than a markdown code block or markdown table in the chat conversation.

**Why this rule exists:** Prior versions of this skill allowed the staffing table to be embedded as Appendix A of the SOW/PWS with a "not prescriptive" disclaimer. This violated FAR 37.102(d) because offerors anchor on FTE counts when they appear in a requirement document, regardless of disclaimers. It also leaked skill-internal messaging ("say 'build the IGCE'") into contract file deliverables. The staffing data is still needed — it just belongs in chat output between the two skills, not in the document submitted to the contracting officer.

**The one exception** is the T&M/LH Labor Category Ceiling Hours table in Section 5 per FAR 16.601(c)(2). That is a pricing mechanism (ceiling hours, no SOC codes, no FTE counts, no derivation basis) and is not the same thing as this staffing handoff table.

## Edge Cases

**User doesn't know the answer to a decision:** Provide a recommended default with rationale. Mark it as an assumption in the document. Example: "If you're unsure about coverage model, business hours (M-F 8-5) is the default for non-emergency federal contact centers. I'll mark this as an assumption you can revisit."

**SOO is too vague to decompose:** If the SOO is under 300 words or provides fewer than 3 actionable scope details, tell the user: "This SOO doesn't have enough specificity to convert directly. I'll use it as background context, but Phase 1 will need to collect most decisions fresh."

**Scope exceeds single contract:** If Phase 1 decisions imply >50 FTEs, >$75M, or span >3 distinct technical domains, flag that this may need to be broken into multiple contracts. Suggest logical break points (e.g., development vs. operations, platform vs. contact center staffing).

**User wants to skip Phase 1:** Don't let them. The decision tree is the value. If they say "just write the SOW from this SOO," explain: "The SOO describes objectives, but a SOW requires decisions the SOO intentionally leaves open — like staffing model, build vs. buy, phasing, and CLIN structure. These take about 10 minutes to walk through, and they're what make the SOW defensible."

**Budget-constrained scope reduction (Workflow C):** When reducing scope to fit budget, work backwards: identify the highest-cost labor categories from the IGCE, map them to tasks/objectives, and present trade-offs: "Removing 24/7 coverage saves ~$X by eliminating night-shift agents, but means no live response outside business hours. Replacing custom AI with COTS chatbot saves ~$Y in developer FTEs but limits NLP accuracy." Let the user choose which scope reductions are acceptable, then regenerate affected SOW/PWS sections.

## What This Skill Does NOT Cover

- **Pricing or cost estimates** — use IGCE Builder skills (FFP, LH/T&M, CR)
- **Market research** — use Market Research Builder
- **Source selection criteria** — separate document, though CLIN structure informs it
- **J&A or sole source justification** — separate document
- **Full QASP** — this skill includes a QASP summary table; a detailed QASP with surveillance schedules is a separate deliverable
- **Clause selection (Section I/L)** — requires contracting officer determination per FAR Parts 12, 15, 16, 52
- **Contract type determination** — this skill accepts the user's contract type decision; it does not advise on which type to select (that's a FAR 16 analysis)


---

*MIT © James Jenrette / 1102tools. Source: github.com/1102tools/federal-contracting-skills*