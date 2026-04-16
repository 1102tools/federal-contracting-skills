---
name: ot-project-description-builder
description: >
  Build milestone-based project descriptions for Other Transaction (OT)
  agreements under 10 USC 4021/4022. Produces the agreement attachment
  that replaces a SOW/PWS in OT prototype and production follow-on
  contexts. Structures work around TRL progression phases (design, build,
  test, demonstrate) with milestone payment schedules tied to technical
  deliverables. Trigger for: OT project description, OTA project
  description, other transaction agreement, prototype agreement, OT scope,
  10 USC 4021, 10 USC 4022, prototype OT, production follow-on OT,
  TRL-based project description, milestone-based scope, write an OT
  project description. Also trigger when user needs a project description
  before an OT cost analysis or has a BAA white paper/SOO to develop.
  Never contains cost estimates, funding profiles, or cost-sharing
  calculations. Do NOT use for SOW/PWS (use sow-pws-builder), OT cost
  analysis (use OT Cost Analysis), or FAR Part 15 requirements.
---

# OT Project Description Builder

## Overview

This skill walks an agreements officer (AO) or program office through structured prototype scope decisions and assembles the answers into a milestone-based project description for an Other Transaction agreement. The core insight: OTs structure work around technology maturation phases and technical milestones, not task/subtask CLINs or performance objectives. The program office defines what must be demonstrated at each gate, and the performer proposes how to get there.

**Outputs (TWO SEPARATE ARTIFACTS -- never combined):**

1. **A .docx project description** with milestone-based structure -- the agreement attachment. This document contains NO cost estimates, NO should-cost figures, NO funding profiles, NO cost-sharing calculations, and NO government budget data. It defines what the prototype must achieve, by when, and what constitutes success at each milestone.

2. **A milestone handoff table** -- an internal government workpaper presented in **chat output only** at the end of the skill run. This is the data handoff to the OT Cost Analysis skill. It is NEVER embedded in the project description document. It is NEVER saved as a companion file. It exists solely as a markdown table in the conversation so the user can review it and the downstream OT Cost Analysis skill can consume it.

**No external APIs required.** This is a decision tree + document generation skill.

**Statutory basis:** 10 USC 4021 (authority to carry out prototype projects), 10 USC 4022 (other transaction authority and follow-on production), 10 USC 4003 (definition of prototype project). NOT FAR-based. OTs operate outside the FAR; the project description replaces the SOW/PWS as the primary scope document attached to the agreement.

**Related policy:** DoD OT Guide (USD(R&E) / USD(A&S)), individual service OT policies (Army AFC, Navy NSWC, Air Force AFRL/AFWERX guidance). These are policy references, not binding regulations like the FAR.

## Workflow Selection

### Workflow A: Full Build (Default)
User needs a project description from a prototype concept or objective. Execute Acquisition Context Intake, then all three phases.
Triggers: "write an OT project description," "build a prototype agreement," "I need a project description for an OT," "scope a prototype."

### Workflow B: Convert from Existing Document
User has an existing SOW, SOO, BAA white paper, or proposal abstract and needs it developed into an OT project description. Start with Phase 0 (document intake), then Phases 1-3.
Triggers: "convert this SOW to an OT project description," "develop this white paper into a prototype scope," "we have a BAA response and need a project description."

### Workflow C: Scope Reduction
User has an existing OT project description or cost analysis output that exceeds budget or schedule. Walk through the milestone structure to identify what to cut or defer, then produce a revised document.
Triggers: "this prototype is too expensive," "we need to reduce scope," "can we drop a phase," "defer TRL 6 to a follow-on."

## Acquisition Context Intake

Before diving into the scope decision tree, collect four framing decisions that shape everything that follows. Ask these up front, in a single pass, before Block 1.

### Intake Question 1: OT Type

| Type | Statutory Authority | Use When |
|------|-------------------|----------|
| Prototype | 10 USC 4021 | R&D, technology maturation, proof of concept through system demo |
| Production Follow-On | 10 USC 4022(f) | Transitioning a successful prototype to limited or full-rate production |
| Research | 10 USC 4021 | Basic or applied research, typically pre-TRL 4 |

If the user is unsure, default to Prototype -- it covers the widest range of OT work and is the most common. Production follow-on requires a completed prototype OT as a predecessor. Research OTs are less common and typically issued by service labs (NRL, ARL, AFRL).

### Intake Question 2: Performer Type

| Type | Cost-Sharing Implication | 10 USC 4022(d) Path |
|------|------------------------|---------------------|
| Nontraditional Defense Contractor (NDC) | NDC participation satisfies 4022(d)(1)(A) | No cost share required if NDC has significant participation |
| Traditional + NDC Team | NDC sub or team member satisfies 4022(d)(1)(A) | Same as above |
| Small Business | Small business significant participation satisfies 4022(d)(1)(B) | No cost share required |
| Consortium Member | Depends on consortium structure and awardee | Varies by consortium agreement |
| Traditional (sole, no NDC/SB participation) | Must have cost-sharing arrangement per 4022(d)(1)(C) | Cost share required (typically 1/3 performer) |
| Traditional (with follow-on competition commitment) | Competitive follow-on satisfies 4022(d)(1)(D) | No cost share required but must compete production |

An NDC is defined at 10 USC 3014: an entity that has not had a contract or subcontract of $500K+ in the prior year, or is a nonprofit performing DoD-relevant research, or any other entity the contracting officer determines as nontraditional. The performer type determines whether cost-sharing is required and shapes the project description's cost-sharing section.

### Intake Question 3: TRL Entry and Exit

| TRL | Description | Typical OT Phase |
|-----|-------------|-----------------|
| 1 | Basic principles observed | Research OT |
| 2 | Technology concept formulated | Research OT |
| 3 | Proof of concept | Early Prototype |
| 4 | Component validation in lab | Prototype (design/build) |
| 5 | Component validation in relevant environment | Prototype (build/test) |
| 6 | System demo in relevant environment | Prototype (test/demonstrate) |
| 7 | System prototype demo in operational environment | Late Prototype / Pre-production |
| 8 | System complete and qualified | Production Follow-On |
| 9 | System proven in operational environment | Production |

Collect: TRL at project start (entry) and TRL target at project end (exit). This determines the number of phases and the nature of milestones. Most prototype OTs span TRL 3-6 or TRL 4-7. Research OTs are typically TRL 1-3 or TRL 2-4.

If the user gives a range wider than 4 TRL levels (e.g., TRL 2 to 8), flag that this may exceed typical prototype OT scope and suggest breaking into two agreements or phasing with go/no-go gates.

### Intake Question 4: Consortium or Direct

| Approach | Description | Agreement Structure |
|----------|------------|---------------------|
| Direct OT | Government awards directly to a single performer | Bilateral agreement between government and performer |
| Consortium-brokered | Award through a consortium organization | Agreement may route through consortium (DIU, AFWERX, NavalX, NSTXL, SOSSEC, MTEC, etc.) |

If consortium: identify which consortium. Consortium OTs often follow consortium-specific templates and add a management fee (typically 3-5%). The project description content is the same; the agreement wrapper differs.

### Why these four questions come first

OT project descriptions must be shaped by the statutory authority, the cost-sharing path, the technology maturation scope, and the agreement structure. A TRL 3-6 prototype with an NDC performer reads differently from a TRL 6-7 production-readiness effort with a traditional prime requiring cost-sharing. Collecting these up front means Phase 1 can frame scope questions appropriately.

## Phase 0: Document Intake (Workflow B Only)

When the user provides an existing document (SOW, SOO, BAA white paper, proposal abstract):

1. **Read and extract.** Parse for: technical objective, current state of technology, systems involved, TRL indicators, proposed approach, deliverables, schedule, performer qualifications, prior work.

2. **Gap identification.** Flag what the document provides vs. what an OT project description needs:
   - Typically present: technical objective, background, proposed approach
   - Typically missing: TRL entry/exit mapping, milestone definitions, go/no-go criteria, data rights assertions, cost-sharing structure

3. **Decision bridge.** Present gaps as questions Phase 1 will answer. Frame it as: "This document tells us what the performer wants to build. Phase 1 asks the questions that turn a concept into executable prototype milestones with decision gates."

4. **Carry forward.** Pre-populate Phase 1 answers with anything the document already decided. Don't re-ask settled questions.

**SOW conversion note:** If converting from a traditional SOW, flag that OTs do not use task/subtask CLIN structure, FAR clause references, or performance-based language (those are FAR constructs). The conversion maps task areas to TRL phases and deliverables to milestone completion criteria.

## Phase 1: Scope Decision Tree

Ask questions in this sequence. Each decision narrows scope and implies milestones. Present as structured choices, not open-ended questions. Collect in a single pass where possible.

### Block 1: Prototype Objective and Technical Baseline

1. **What is being prototyped?** Hardware system | Software application/platform | Process or workflow | Algorithm or model | Hybrid (hardware + software) | Integration of existing systems
2. **What is the current state?** Concept only (no existing implementation) | Lab prototype exists (needs maturation) | Commercial product exists (needs military adaptation) | Prior government prototype (needs iteration)
3. **What does success look like?** Define in terms of: "Demonstrate [capability X] at [performance level Y] in [environment Z]." This becomes the top-level prototype objective.
4. **Data rights posture:** Government Purpose Rights (GPR) | Limited Rights | Unlimited Rights | Negotiated split by deliverable category | Performer retains all background IP, government gets GPR on foreground

Data rights in OTs are negotiated per agreement, not prescribed by FAR Part 27. DFARS 252.227-7013 (technical data) and 252.227-7014 (software) provide a framework but can be tailored. The data rights posture should be decided early because it affects performer willingness and cost.

### Block 2: TRL Progression and Phase Structure

5. **Number of phases:** Typically 2-4 for a prototype OT. Each phase spans 1-2 TRL levels.
6. **For each phase, define:**
   - Entry TRL and exit TRL
   - Primary activity: Design | Build/Fabricate | Integrate | Test | Demonstrate
   - Duration estimate (months)
   - Key technical risk to retire in this phase
7. **Go/no-go criteria between phases:** What must be demonstrated to proceed? Examples:
   - "Component X achieves [metric] in lab environment" (TRL 4 gate)
   - "Integrated system passes [test] in simulated operational conditions" (TRL 5 gate)
   - "System demonstration meets [threshold] observed by government test team" (TRL 6 gate)
8. **Off-ramp conditions:** Under what circumstances would the government terminate before completing all phases? (Technical failure, budget reduction, requirement change, superior alternative identified)

### Block 3: Technical Scope

9. **Systems and technologies:** List all systems, subsystems, and enabling technologies the prototype will develop, integrate with, or demonstrate against.
10. **Integration requirements:** Standalone prototype | Must integrate with [existing government system] | Must demonstrate interoperability with [allied/partner system]
11. **Test environment:** Laboratory | Simulated operational (specify fidelity) | Operational (specify location/unit) | Multiple environments across phases
12. **Data and analytics:** Does the prototype produce data that requires analytics, ML model training, or algorithm validation? If yes, describe the data pipeline.

### Block 4: Scale and Deliverables

13. **Prototype units (if hardware):** Number of units to build. Single prototype | 2-5 units (for testing/evaluation) | Pre-production quantity (10+, likely production follow-on)
14. **Test events and demonstrations:** Number and type of formal test/demo events. Who observes? (Government test team, operational users, senior leadership, allied partners)
15. **Data deliverables:** Technical data packages | Design documents | Test reports | Source code | Training data sets | Algorithm documentation | User manuals
16. **Software deliverables (if applicable):** Source code | Compiled binaries | Container images | API documentation | Deployment scripts

### Block 5: Agreement Structure

17. **Period of performance by phase:** Duration per phase in months. Total PoP typically 12-36 months for prototype OTs.
18. **Milestones per phase:** Typically 2-4 milestones per phase. More milestones = more government oversight but also more payment events.
19. **Milestone payment type:** Fixed-price milestones (most common -- fixed payment upon completion) | Cost-type milestones (ceiling per milestone, reimburse actuals) | Mixed (some fixed, some cost-type)
20. **Cost-sharing arrangement (if applicable per Intake Q2):**
    - Performer cash contribution (what percentage?)
    - Performer in-kind contribution (lab time, equipment, personnel not charged to the agreement)
    - No cost share (NDC/SB participation path or competition commitment path)
21. **Production follow-on option (10 USC 4022(f)):** Yes -- include provisions for production transition | No -- prototype only | TBD -- evaluate after prototype results

### Block 6: Oversight and Reporting

22. **Technical review cadence:** Monthly | Quarterly | At milestone completion only | Per-phase technical interchange meetings (TIMs)
23. **Milestone acceptance process:** Government program office review | Independent test team evaluation | Joint government-performer review board | Combination
24. **Government test participation:** Government observers only | Government co-testers | Government provides test environment/range | Performer-led testing with government data access
25. **Key personnel:** Which performer roles require government approval? (Principal Investigator/Technical Lead always; others as needed)

### Decision-to-Milestone Derivation Rules

After collecting decisions, derive the milestone structure. These are heuristics based on typical OT prototype structures:

| Decision | Milestone Implication |
|----------|----------------------|
| TRL 3 to 4 phase | 1-2 milestones: preliminary design review (PDR), detailed design complete |
| TRL 4 to 5 phase | 2-3 milestones: prototype fabrication/build, component test, subsystem integration |
| TRL 5 to 6 phase | 2-3 milestones: system integration test, relevant-environment demo, test report delivery |
| TRL 6 to 7 phase | 1-2 milestones: operational-environment demo, production readiness review |
| Software prototype | Milestones per iteration: working demo, user acceptance test, deployment to test environment |
| Hardware prototype | Milestones per build cycle: design freeze, first article, qualification test |
| Each formal test event | 1 milestone for test execution + report delivery |
| Each government demonstration | 1 milestone for demo completion + government acceptance |
| Data deliverable package | 1 milestone per major data deliverable (TDP, source code drop, algorithm package) |

**Present the derived milestone table to the user for validation before proceeding to Phase 2.** Format:

```
Phase | Milestone | Description                     | TRL In/Out | Est. Duration | Key Deliverable
1     | M1        | Preliminary Design Review       | 3/3        | 3 months      | Design document
1     | M2        | Detailed Design Complete        | 3/4        | 3 months      | Final design, BOM
2     | M3        | Prototype Build Complete         | 4/4        | 4 months      | First article
2     | M4        | Component Test Complete          | 4/5        | 2 months      | Test report
3     | M5        | System Demonstration             | 5/6        | 3 months      | Demo report, video
```

**User validation gate.** The user confirms, adjusts, or overrides any milestone. If they override, document the rationale.

## Phase 2: Document Assembly

Generate the project description using the docx skill. Read `/mnt/skills/public/docx/SKILL.md` before generating output.

### Project Description Section Structure

This is NOT a FAR Section L/M format. OT project descriptions follow the structure below:

**Section 1: Agreement Overview**
- 1.1 Purpose: one paragraph stating the prototype objective
- 1.2 Authority: cite 10 USC 4021 for prototype, 10 USC 4022(f) for production follow-on
- 1.3 Parties: Government organization and performer (or "to be determined" if pre-award)
- 1.4 Agreement Type: Prototype OT | Production Follow-On OT | Research OT

**Section 2: Technical Background and Current State**
- Problem or capability gap the prototype addresses
- Current state of the technology (TRL at entry, existing implementations, prior efforts)
- Why an OT is the appropriate vehicle (speed, innovation, NDC access, flexibility)

**Section 3: Prototype Objectives**
- Numbered objectives, each with:
  - Objective statement: "Demonstrate [capability] at [performance level] in [environment]"
  - Success criteria: measurable, testable, unambiguous
  - Relationship to phase and TRL exit
- These are the top-level "what" -- the performer proposes the "how"

**Section 4: Technical Approach by Phase**
- For each phase:
  - Phase number and title (e.g., "Phase 1: Design and Preliminary Prototyping")
  - Entry TRL and exit TRL
  - Primary activities and focus areas
  - Key technical risks to retire
  - Go/no-go criteria for proceeding to next phase
  - Duration

**Section 5: Milestone Schedule**
- Milestone table: ID | Phase | Description | Deliverables Due | Completion Criteria | Payment Type (Fixed/Cost)
- Milestones are payment events tied to technical achievement, not calendar dates
- Each milestone must have unambiguous completion criteria that the government can verify

**Section 6: Deliverables**
- Deliverables table: ID | Title | Format | Due Trigger (milestone ID, e.g. "M3") | Acceptance Criteria
- Column header must read "Due Trigger" (not "Due At" or "Due Date") to make clear that deliverables are triggered by milestone completion, not calendar dates
- Include both technical deliverables (prototypes, test reports, design docs) and programmatic deliverables (status reports, risk registers)

**Section 7: Data Rights**
- Data rights assertions by deliverable category
- Background IP: what the performer brings (performer retains rights)
- Foreground IP: what the agreement produces (negotiated rights)
- Reference DFARS 252.227-7013 and -7014 as framework, noting OT flexibility to tailor
- License rights by category: Unlimited | Government Purpose | Limited | Specifically Negotiated
- Source code escrow provisions if applicable

**Section 8: Period of Performance**
- Total PoP and PoP per phase
- Phase start dates may be contingent on go/no-go decisions
- Note that go/no-go gates may extend or compress subsequent phases

**Section 9: Government Responsibilities**
- Government-furnished equipment, facilities, or test environments
- Government technical points of contact
- Access to government systems, data, or operational environments for testing
- Government test team participation

**Section 10: Key Personnel**
- Roles requiring government approval (Principal Investigator/Technical Lead minimum)
- Minimum qualifications per role
- Substitution requires prior written approval from the Agreements Officer

**Section 11: Reporting and Oversight**
- Technical review schedule (per phase or at milestones)
- Status reporting requirements (format, frequency)
- Milestone completion review and acceptance process
- Risk and issue reporting

**Section 12: Cost-Sharing Arrangement (if applicable)**
- Include this section ONLY when the performer type requires cost sharing per 10 USC 4022(d)(1)(C) (traditional contractor, no NDC/SB significant participation)
- Omit entirely when no cost share applies (NDC, small business, or competition commitment path)
- Content: statutory basis (10 USC 4022(d)(1)(C)), cost-sharing ratio (e.g., 1/3 performer), contribution type (cash vs. in-kind), how the share applies to fixed-price milestones (payment reflects government share only) vs. cost-type milestones (performer identifies share in cost reporting, government reimburses its share up to ceiling), and note that the arrangement satisfies 4022(d) for follow-on production eligibility
- Do NOT include dollar amounts, funding profiles, or government obligation figures -- those belong in the OT Cost Analysis, not the project description

**Section 13: Production Follow-On Provisions (if applicable)**
- 10 USC 4022(f) authority for follow-on production without further competition
- Conditions that must be met: successful prototype completion, production-ready TRL, continued need
- Transition planning requirements
- If no follow-on: state "This agreement covers prototype development only. Any production requirement will be addressed through a separate acquisition action."

**Section 14: Constraints and Assumptions**
- Technical constraints (size/weight/power, interface standards, operating environments)
- Schedule constraints (event-driven milestones, dependency on external programs)
- Assumptions about government-furnished resources, access, or participation
- Any user overrides from Phase 1 validation with rationale

### Language Rules

**OT project descriptions use direct, technical language:**
- "The performer shall demonstrate..." (milestone completion)
- "The prototype shall achieve..." (performance requirement)
- "The government will provide..." (government responsibilities)
- "Phase [N] concludes upon..." (go/no-go gate)

**Avoid:**
- FAR-specific terms: "shall perform in accordance with," "deliverable order," "contracting officer" (use "agreements officer" instead)
- CLIN-based language: "CLIN 0001," "task order," "option year"
- Performance-based acquisition terms: "acceptable quality level," "QASP," "performance standard" (these are FAR 46 constructs)
- Vague success criteria: "satisfactory performance," "as determined by the government," "best efforts"

**Every milestone completion criterion must be:** specific (what is demonstrated), measurable (what metric or threshold), testable (how the government verifies), and binary (pass/fail, not subjective).

## Phase 3: Validation and Handoff

### Document Review Checklist

Before presenting the final document, verify:

- [ ] Every phase has at least one milestone
- [ ] Every milestone has completion criteria and at least one deliverable
- [ ] TRL progression is traceable from entry to exit with no gaps
- [ ] Go/no-go criteria are defined for each phase transition
- [ ] Data rights assertions cover all deliverable categories
- [ ] Cost-sharing path is consistent with performer type (Intake Q2)
- [ ] Production follow-on provisions match user intent (Intake Q5-21)
- [ ] No cost estimates, should-cost figures, or funding data in the document
- [ ] No FAR clause references (this is an OT, not a FAR contract)
- [ ] Milestone count and structure match the validated table from Phase 1
- [ ] All user overrides from Phase 1 are documented in Section 13

### Milestone Handoff Table (CHAT OUTPUT ONLY -- NEVER IN THE DOCUMENT)

**After the .docx is saved to disk, present the milestone handoff table as chat output only.** It is NOT a section of the project description. It is NOT saved as a separate file. It exists solely as a markdown table in the conversation so (a) the user can review the milestone structure for cost analysis readiness, and (b) the OT Cost Analysis skill can consume it as input.

Present it like this, verbatim:

```
=== MILESTONE HANDOFF TABLE -- FOR OT COST ANALYSIS ===
Internal government workpaper. NOT part of the project description.
Do not paste this into the agreement document.

| Milestone ID | Phase | Description                  | Est. Duration | Deliverables             | TRL In/Out | Payment Type |
|-------------|-------|------------------------------|---------------|--------------------------|-----------|--------------|
| M1          | 1     | Preliminary Design Review    | 3 months      | Design document, TDP     | 3/4       | Fixed        |
| M2          | 1     | Detailed Design Complete     | 3 months      | Final design, BOM        | 4/4       | Fixed        |
| M3          | 2     | Prototype Build Complete     | 4 months      | First article            | 4/5       | Fixed        |
| M4          | 2     | Component Test Complete      | 2 months      | Test report              | 5/5       | Fixed        |
| M5          | 3     | System Demonstration         | 3 months      | Demo report, video       | 5/6       | Fixed        |

Performer Type: [from Intake Q2]
Cost-Sharing Path: [from Intake Q2 and Block 5-Q20]
Total PoP: [X months]
Milestone Payment Type: [Fixed / Cost-type / Mixed]
```

Include rows for every milestone derived in Phase 1. After the table, tell the user in plain chat prose: *"This milestone table is ready for handoff to the OT Cost Analysis. Say 'build the OT cost analysis' and I'll estimate should-costs per milestone, apply cost sharing, produce the funding profile, and generate the price reasonableness memo citing 10 USC 4021."*

**DO NOT, under any circumstances:**

- Write the milestone handoff table into the project description body, any section, or any appendix.
- Save the milestone handoff table as a separate file.
- Include cost estimates, should-cost figures, or government budget data anywhere in the project description.
- Include the phrases "OT Cost Analysis skill," "say 'build the OT cost analysis'," or any skill-chain plumbing messaging in the project description document.
- Emit the table as anything other than a markdown code block or table in the chat conversation.

**Why this rule exists:** The project description defines technical scope and milestones. The cost analysis is a separate government workpaper that evaluates price reasonableness. Mixing them compromises the government's negotiating position and leaks internal cost assumptions to the performer. Same separation principle as the SOW/PWS Builder's staffing handoff.

## Edge Cases

**User doesn't know TRL entry/exit:** Walk through the TRL descriptions and ask what currently exists. Map their description to a TRL level. Example: "You have a working algorithm in a lab environment with test data? That sounds like TRL 4. A relevant-environment demo would be TRL 6. So this is a TRL 4-6 prototype."

**TRL range too wide:** If the requested range spans more than 4 TRL levels (e.g., TRL 2-7), suggest breaking into two agreements: a research/early prototype (TRL 2-4) and a late prototype (TRL 5-7), with a go/no-go between them.

**User wants to skip Phase 1:** Don't let them. The milestone structure is the value. If they say "just write the project description from this white paper," explain: "A white paper describes a concept, but an OT project description requires decisions the paper leaves open -- like milestone definitions, go/no-go criteria, data rights, and cost-sharing path. These take about 15 minutes to walk through, and they're what make the milestones defensible and the agreement executable."

**Existing SOW is heavily FAR-flavored:** When converting a traditional SOW, strip FAR references (Section I/L, FAR clauses, QASP, CLINs), map task areas to TRL phases, and restructure deliverables as milestone completion criteria. Flag the conversion for the user: "I've restructured the task-based scope into TRL phases and mapped deliverables to milestone gates. Review the phase structure to confirm the technology maturation path makes sense."

**Production follow-on scope creep:** If the user starts describing production quantities or sustainment, flag that this may no longer be a prototype. Production follow-on under 4022(f) requires a completed prototype; sustainment is typically a separate FAR-based contract.

**Budget-constrained scope reduction (Workflow C):** Work backwards from the cost analysis or budget ceiling. Options: reduce the number of phases (stop at TRL 5 instead of 6), reduce prototype units, defer certain test events, narrow the integration scope. Present trade-offs: "Stopping at TRL 5 saves [estimated effort] but means you'll need a separate effort to demonstrate in a relevant environment. Reducing from 3 prototype units to 1 saves fabrication cost but limits your test matrix."

**Consortium-specific templates:** If the user identifies a specific consortium (e.g., DIU, AFWERX), note that the project description content follows this skill's structure but the agreement wrapper may use consortium-specific formats. The milestone table and technical content are the same.

## What This Skill Does NOT Cover

- **Cost analysis or should-cost estimates** -- use OT Cost Analysis
- **Traditional SOW/PWS** -- use SOW/PWS Builder
- **IGCE for FAR contracts** -- use IGCE Builder skills (FFP, LH/T&M, CR)
- **Market research** -- use Market Research Builder (though OT market research has different emphasis)
- **NDC eligibility determination** -- agreements officer determination per 10 USC 3014
- **OT competition strategy** -- this skill accepts the user's competition approach
- **Agreement terms and conditions** -- legal review required; this skill covers technical scope only
- **Intellectual property negotiations** -- this skill documents the data rights posture; actual IP terms require legal counsel
- **10 USC 4022(f) determination** -- the decision to exercise follow-on production authority is a separate action


---

*MIT James Jenrette / 1102tools. Source: github.com/1102tools/federal-contracting-skills*
