---
name: grants-program-description-builder
description: >
  Build the Program Description section of a Notice of Funding Opportunity
  (NOFO) for federal grants and cooperative agreements under 2 CFR 200.
  Produces a government-written .docx for the agency's NOFO template and
  a chat-only handoff table for the Grants Budget Builder. Trigger for:
  grant program description, NOFO program description, Notice of Funding
  Opportunity narrative, cooperative agreement scope, program purpose,
  allowable activities, expected outcomes, 2 CFR 200.202, 2 CFR 200.203,
  grant program scope, HHS/CDC/NIH/HRSA/SAMHSA NOFO, write a program
  description for a grant. Also trigger when user needs a program
  description before a grant budget or has authorizing legislation to
  develop. Never contains budget figures or cost categories. Do NOT use
  for grant budgets (use grants-budget-builder), full NOFO assembly,
  applicant-side narratives, contract SOW/PWS (use sow-pws-builder),
  or OT project descriptions (use ot-project-description-builder).
---

# Grants Program Description Builder

## Overview

This skill walks a program officer (or analyst acting on their behalf) through structured program scope decisions and assembles the answers into the Program Description section of a NOFO. The core insight: the program office doesn't need to know per-award dollar amounts up front — they need to make program scope decisions (who, what, where, why, how measured), and the budget envelope falls out as a downstream input to the Grants Budget Builder.

**Outputs (TWO SEPARATE ARTIFACTS — never combined):**

1. **A .docx Program Description** with standard NOFO sub-section structure — the sub-section the program officer drops into the agency's NOFO template. This document contains NO budget figures, NO per-award funding estimates, NO cost category allocations, and NO total program funding data. It defines what the program is, what recipients will do, who is served, what success looks like, and what the federal role is.

2. **A program parameters handoff table** — an internal government workpaper presented in **chat output only** at the end of the skill run. This is the data handoff to the Grants Budget Builder skill. It is NEVER embedded in the Program Description document. It is NEVER saved as a companion file. It exists solely as a markdown table in the conversation so the user can review it and the downstream Grants Budget Builder can consume it.

**No external APIs required.** This is a decision tree + document generation skill.

**Regulatory basis:** 2 CFR 200.202 (requirement to provide public notice of federal financial assistance programs), 2 CFR 200.203 (Notices of Funding Opportunity — required sections), 45 CFR 75 subpart C (HHS-specific NOFO requirements), and the program's specific authorizing statute (Public Health Service Act, Social Security Act, agency organic statute, or other legislation).

**Scope note:** A NOFO is a single document that contains what the FAR contracting world splits across multiple RFP sections (Section C scope, Section L instructions, Section M evaluation, clauses). This skill covers the **Program Description** portion only — the direct scope-parallel to a SOW/PWS. The user drops the output into their agency's NOFO template alongside the other required NOFO sections (Eligibility, Application and Submission Information, Evaluation Criteria, Award Administration, Other Information), which this skill does not generate.

## Workflow Selection

### Workflow A: Full Build (Default)
User needs a Program Description from scratch or from a rough program concept. Execute Program Authority Intake, then all three phases.
Triggers: "write a program description for a new grant," "build the program section of our NOFO," "I need a program description for an HHS cooperative agreement."

### Workflow B: Convert from Existing Document
User has authorizing legislation, a policy memo, a strategic plan excerpt, or a prior NOFO and needs it developed into a new Program Description. Start with Phase 0 (document intake), then Phases 1–3.
Triggers: "convert this policy memo into a program description," "develop the authorizing statute into a NOFO program section," "we have last year's NOFO and need to update it for this cycle."

### Workflow C: Scope Reduction
User has an existing Program Description or budget output that exceeds the program's funding envelope. Walk through the program scope to identify what to cut, defer, or narrow, then produce a revised document.
Triggers: "our appropriation dropped, what program activities do we cut," "reduce program scope to fit the funding envelope," "need to narrow eligibility or activities to stay within budget."

## Program Authority Intake

Before diving into the scope decision tree, collect four framing decisions that shape everything that follows. Ask these up front, in a single pass, before Block 1.

### Intake Question 1: Instrument Type

| Type | Federal Role | Use When |
|------|-------------|----------|
| Grant | Funds transferred with minimal federal involvement; recipient executes | Standard financial assistance where federal role is oversight only |
| Cooperative Agreement | Substantial federal involvement expected during performance | Federal staff collaborate, co-lead activities, provide technical assistance, or jointly decide direction |

Per 31 USC 6304 (Grant) vs 31 USC 6305 (Cooperative Agreement), the distinguishing factor is "substantial federal involvement." If federal staff will be decision-makers, co-investigators, or technical co-leads during performance, it's a cooperative agreement. If federal role is limited to review of reports and monitoring, it's a grant. If the user is unsure, default to Grant — cooperative agreements require specific federal role definition in Section 9 of the output document.

### Intake Question 2: Program Category

| Category | Description |
|----------|-------------|
| Research | Knowledge generation, scientific investigation, evaluation studies |
| Training | Workforce development, curriculum, credentialing, continuing education |
| Services | Direct service delivery to individuals or populations |
| Infrastructure | System-building, capacity-strengthening, workforce capacity, facility support |
| Planning | Needs assessments, strategic planning, readiness-building for future programs |
| Demonstration | Pilot implementation with evaluation, replicability focus |

Program category shapes the expected activities, outcome measures, and performance reporting structure. If the user says "hybrid," identify the dominant category — most programs have a primary orientation even when they include mixed activities.

### Intake Question 3: Statutory Authority

Collect the specific authorizing statute and section. Common HHS authorities:

| Statute | Common Agencies |
|---------|----------------|
| Public Health Service Act (42 USC 241 et seq.) | CDC, HRSA, SAMHSA, NIH, HHS components |
| Social Security Act Title X, XVI, XIX, XX, XXI | CMS, ACL, ACF, SSA |
| Older Americans Act (42 USC 3001 et seq.) | ACL |
| Child Abuse Prevention and Treatment Act | ACF |
| Violence Against Women Act | ACF (some), DOJ (most) |
| Agency organic statute or appropriations language | Varies |

Capture the specific section(s) that authorize THIS program. Example: "Section 330 of the Public Health Service Act (42 USC 254b) — Community Health Centers." The authority citation goes in Section 13 of the output document and drives the legal boundaries of allowable activities.

### Intake Question 4: Program Duration

| Duration | Description |
|----------|-------------|
| Single-year | One budget period, one performance period |
| Multi-year continuation | Multi-year project period with annual budget periods and non-competing continuations |
| Multi-year segmented | Multi-year project period with distinct phases or segments (e.g., planning year then implementation years) |

This affects how the Program Description frames expected activities over time and how the handoff table flags budget periods for the Grants Budget Builder.

### Why these four questions come first

The same programmatic intent produces different Program Description structures depending on whether it's a grant or cooperative agreement, which category it fits, which statute authorizes it, and how it phases over time. A single-year training grant under PHSA 330A reads differently from a five-year infrastructure cooperative agreement under SSA Title XIX. Collecting these upfront means Phase 1 can frame scope questions appropriately and Phase 2 can reference the correct statutory boundaries without a retrofit.

## Phase 0: Document Intake (Workflow B Only)

When the user provides an existing document (authorizing legislation, policy memo, strategic plan section, prior NOFO):

1. **Read and extract.** Parse for: programmatic purpose, authorizing authority, allowable activities (if specified), priority populations, performance expectations, program history, evidence base.

2. **Gap identification.** Flag what the document provides vs. what a Program Description needs:
   - Typically present in legislation: authority, broad purpose, allowable activities, any statutory conditions
   - Typically present in policy memos: programmatic intent, priority populations, desired outcomes, policy context
   - Typically present in prior NOFOs: full structure, performance measures, evidence base citations
   - Typically missing: specific performance measures, sustainability expectations, federal involvement (for new coop agreements), current evidence base updates

3. **Decision bridge.** Present the gaps as questions Phase 1 will answer. Frame it as: "This document tells us the program's authorizing intent. Phase 1 asks the questions that turn policy intent into an executable program description."

4. **Carry forward.** Pre-populate Phase 1 answers with anything the document already decided (authority, high-level purpose, statutory allowable activities). Don't re-ask settled questions.

**Prior-NOFO conversion note:** If working from a prior NOFO, flag any changes since the prior cycle: new Administration priorities, revised strategic plans, updated evidence base, new statutory amendments, changed appropriations language. The new Program Description should reflect current policy context, not just copy forward.

## Phase 1: Scope Decision Tree

Ask questions in this sequence. Each decision narrows program scope and implies handoff parameters for the Grants Budget Builder. Present as structured choices where possible. Collect in a single pass where possible.

### Block 1: Programmatic Purpose

1. **What is the policy or programmatic goal?** One to two sentences. What does this program exist to accomplish?
2. **What HHS strategic priority or Administration priority does this align with?** Cite specific strategic plan objective, Executive Order, or Secretarial priority if applicable.
3. **What is the status quo being corrected?** Describe the gap, need, or problem that motivates this funding.
4. **What would success at the program level look like?** Not a recipient outcome — a program-level outcome (e.g., "Improved access to maternal health services in 50 rural communities over 5 years").

### Block 2: Expected Activities and Allowable Uses

5. **Core allowable activities.** Bulleted list of activities recipients will carry out. Keep this specific but not prescriptive — recipients propose approaches, the program defines the activity space.
6. **Unallowable activities or carve-outs.** Explicit list of what funds may NOT be used for. Common examples: construction (often restricted absent specific authority), lobbying, research not germane to program, services outside the target population, supplanting existing state/local funding.
7. **Technical assistance requirements.** Will recipients receive or provide TA? Is participation in national TA calls/networks mandatory?
8. **Collaboration and partnership expectations.** Must recipients partner with specific entity types (e.g., state agencies, academic institutions, community-based organizations)?

### Block 3: Target Populations and Geographic Scope

9. **Priority populations.** Which populations are served or targeted? Be specific: age, income, health condition, geographic community, demographic group.
10. **Geographic targeting.** National | Regional (specify) | State-level | Tribal | Territories | Rural | Urban underserved | Specific geographic criteria (e.g., counties with poverty rate > X%).
11. **Population reach expectations.** Minimum number served per recipient? Minimum geographic coverage? Floor or ceiling on scope?
12. **Health equity considerations.** Is this a program with explicit health equity framing? Underserved community emphasis? Medically underserved population focus per HRSA criteria?

### Block 4: Expected Outcomes and Measurement

13. **Expected outcomes.** Outputs (activities completed, people served) vs. outcomes (changes in knowledge, behavior, condition) vs. impact (population-level change). Specify at each level.
14. **Required performance measures.** Which measures MUST be reported? Which are standard HHS or agency measures vs. program-specific?
15. **Data collection expectations.** What data must recipients collect? Any required data systems (e.g., UDS for community health centers, TEDS for substance use services)?
16. **Evaluation requirements.** Is a local evaluation required of each recipient? Is a cross-site evaluation planned? Is there an evaluation contractor outside the recipient pool?

### Block 5: Federal Involvement (Cooperative Agreements Only — Skip if Grant)

17. **Nature of federal involvement.** Collaboration, technical assistance, oversight, joint activity, co-investigation. Be specific.
18. **Project Officer role.** What decisions does the Project Officer make jointly with the recipient? What technical guidance do they provide?
19. **Required federal concurrences.** Which recipient decisions require federal concurrence before execution? (e.g., key personnel changes, major scope shifts, subaward approvals above threshold)
20. **Disengagement conditions.** Under what circumstances does the federal role reduce or end? (e.g., after Year 1 planning, upon achievement of implementation milestones)

### Block 6: Evidence Base, Policy Alignment, and Sustainability

21. **Evidence base.** What research, prior program evaluation, or pilot results support this program design? Cite 2-5 key sources if available.
22. **Alignment with authorizing intent.** How does the program design fulfill the authorizing statute's purpose?
23. **Alignment with current policy.** Which Executive Orders, Secretarial priorities, or strategic plan objectives does this program advance?
24. **Sustainability expectations.** What should recipients do during the project period to build sustainability beyond federal funding? (e.g., diversify funding sources, institutionalize program components, transition to state/local or third-party payer support)

### Decision-to-Program-Description Derivation Rules

After collecting decisions, derive the Program Description structure. These are heuristics:

| Program Category | Emphasis In Output Document |
|---|---|
| Research | Evidence generation objectives, dissemination requirements, scientific rigor standards, IRB expectations |
| Training | Workforce outcomes, curriculum components, credentialing alignment, participant competency measures |
| Services | Beneficiary outcomes, service delivery standards, access metrics, cultural competency |
| Infrastructure | System-level outputs, capacity indicators, workforce capacity, sustainability plans |
| Planning | Deliverable-oriented (strategic plan, needs assessment, readiness framework), stakeholder engagement |
| Demonstration | Evaluation design, replicability documentation, lessons-learned dissemination, scalability criteria |

**Present the derived program description scaffold to the user for validation before proceeding to Phase 2.** Format:

```
Program Category:         [from Intake Q2]
Core Purpose:             [from Block 1 Q1]
Strategic Alignment:      [from Block 1 Q2]
Allowable Activities:     [bulleted list from Block 2 Q5]
Unallowable Activities:   [bulleted list from Block 2 Q6]
Priority Populations:     [from Block 3 Q9]
Geographic Scope:         [from Block 3 Q10]
Primary Outcomes:         [from Block 4 Q13]
Performance Measures:     [from Block 4 Q14]
Federal Role:             [from Block 5 or "Grant - oversight only"]
Evidence Base:            [from Block 6 Q21]
```

**User validation gate.** The user confirms, adjusts, or overrides any line. If they override, document the rationale — it matters for the program parameters handoff and the sustainability narrative.

## Phase 2: Document Assembly

Generate the Program Description using the docx skill. Read `/mnt/skills/public/docx/SKILL.md` before generating output.

### Program Description Section Structure

This is NOT a FAR Section C or Section L/M format. Program Descriptions follow 2 CFR 200.203 sub-section requirements and agency-specific NOFO templates:

**Section 1: Purpose**
- One paragraph stating the program's policy or programmatic goal
- Link to the authorizing statute's intent
- Link to the Administration or Secretarial priority this advances

**Section 2: Background**
- The gap, need, or problem the program addresses
- Current state of federal, state, and local response
- Why federal financial assistance is the appropriate vehicle (vs. a contract, or no federal action)
- Brief summary of prior program history if applicable

**Section 3: Program Description**
- Narrative overview integrating the purpose, target populations, and intended approach
- This is the "plain English" heart of the document — the section most applicants read first and most carefully

**Section 4: Expected Activities**
- Numbered list of core allowable activities recipients will carry out
- Each activity described at the program-definition level (not prescriptive of method)
- Example: "Activity 1: Deliver evidence-based maternal health services to pregnant and postpartum individuals in priority rural counties." NOT: "Activity 1: Hire one nurse practitioner and provide 40 prenatal visits per patient."

**Section 5: Unallowable Activities**
- Explicit list of activity types not allowed under this program
- Reference statutory prohibitions where applicable
- Common entries: construction (unless specifically authorized), lobbying (per 2 CFR 200.450), supplanting state/local funding (per program statute if applicable), services outside target population

**Section 6: Priority Populations and Geographic Scope**
- Priority populations with specific definitions
- Geographic targeting criteria
- Population reach or coverage expectations
- Health equity emphasis where applicable

**Section 7: Expected Outcomes**
- Program-level outcomes (what the program as a whole will accomplish)
- Recipient-level outcomes (what each recipient is expected to achieve)
- Outputs, outcomes, impacts — distinguished

**Section 8: Performance Measurement and Reporting**
- Required performance measures (output, outcome, impact)
- Data collection expectations
- Required data systems (if any)
- Reporting cadence (quarterly, annual, final)
- Evaluation requirements (local, cross-site, federal)

**Section 9: Federal Involvement** (Cooperative Agreements Only)
- Include this section ONLY when Intake Q1 = Cooperative Agreement
- Omit entirely when the instrument is a grant
- Content: nature of federal involvement, Project Officer role, required concurrences, disengagement conditions
- Reference 31 USC 6305 for the substantial federal involvement distinction

**Section 10: Evidence Base and Policy Alignment**
- Key evidence sources supporting the program design (2-5 citations typical)
- Alignment with HHS strategic plan, agency strategic plan, or Secretarial priority
- Alignment with relevant Executive Orders or Administration initiatives

**Section 11: Sustainability**
- Expectations for recipient sustainability planning during the project period
- Required sustainability activities (if any)
- Note that federal funding is not perpetual; recipients are expected to build sustainability plans

**Section 12: Program Duration**
- Total project period
- Budget period structure (typically annual)
- Phased or segmented design if applicable (e.g., planning year followed by implementation years)

**Section 13: Authority**
- Full statutory citation (e.g., "Section 330 of the Public Health Service Act, 42 USC 254b")
- Any relevant implementing regulations
- Appropriations authority if from a specific appropriations line

**Section 14: Definitions** (if needed)
- Program-specific terms requiring definition for applicants to understand scope
- Only include if terms are used in the Program Description that aren't self-evident

### Language Rules

**Program Description language:** Grant-specific, outcome-oriented, recipient-focused.

**Use:**
- "Recipients will..." (expected outcomes and activities)
- "The program will support..." (allowable activities framing)
- "Funds may be used for..." / "Funds may not be used for..."
- "Priority populations include..." / "Geographic scope is..."
- "Expected outcomes include..."

**Avoid:**
- Contracting language: "contractor," "CLIN," "task order," "deliverable order," "RFP," "SOW/PWS," "acceptable quality level," "QASP"
- FAR clause references: this is not a FAR-based instrument
- "Offeror" (that's a contract term; use "applicant" for pre-award, "recipient" for post-award)
- Prescriptive activity language that tells recipients HOW to perform (recipients propose approaches; the program defines the scope)
- Vague success criteria: "satisfactory performance," "best efforts," "as determined by the program" (outcome language must be measurable and specific)

**Every expected activity and outcome must be:** specific, measurable or observable, achievable within the program duration, aligned with authorizing intent, and time-bound. If any element fails these, flag it during assembly.

## Phase 3: Validation and Handoff

### Document Review Checklist

Before presenting the final document, verify:

- [ ] Authority citation matches Intake Q3 and is specific to program section
- [ ] Instrument type (grant vs. cooperative agreement) is consistent throughout
- [ ] Section 9 (Federal Involvement) appears only if cooperative agreement; omitted if grant
- [ ] Every allowable activity traces to authorizing statute intent
- [ ] Unallowable activities list is explicit and cites authority for each restriction
- [ ] Priority populations and geographic scope are specific and measurable
- [ ] Expected outcomes distinguish outputs, outcomes, and impact
- [ ] Performance measures are concrete and data-collectable
- [ ] Evidence base cites specific sources (not "research shows...")
- [ ] Policy alignment references specific strategic plan objective or priority
- [ ] No budget figures, per-award dollar amounts, or funding profile data anywhere in document
- [ ] No contracting terminology (contractor, CLIN, SOW, RFP, QASP)
- [ ] All user overrides from Phase 1 validation are reflected

### Program Parameters Handoff Table (CHAT OUTPUT ONLY — NEVER IN THE DOCUMENT)

**After the Program Description .docx is saved to disk, present the program parameters handoff table as chat output only.** It is NOT a section of the Program Description. It is NOT saved as a .docx, .xlsx, .csv, or any other file. It exists solely as a markdown table in the conversation so (a) the user can review the program parameters for budget readiness, and (b) the Grants Budget Builder skill can consume it as input when the user says "build the grant budget."

Present it like this, verbatim:

```
=== PROGRAM PARAMETERS HANDOFF TABLE — FOR GRANTS BUDGET BUILDER ===
Internal government workpaper. NOT part of the Program Description NOFO sub-section.
Do not paste this into the NOFO.

| Parameter                       | Value                              | Notes                                    |
|--------------------------------|------------------------------------|------------------------------------------|
| Instrument Type                | Grant / Cooperative Agreement      | [from Intake Q1]                         |
| Program Category               | Research / Training / Services / etc. | [from Intake Q2]                      |
| Authorizing Statute            | [citation]                         | [from Intake Q3]                         |
| Project Period                 | [X years]                          | [from Intake Q4 / Block 1]               |
| Budget Period                  | Annual / Other                     | [typically annual]                       |
| Expected Annual Federal Investment | $[range or TBD]                | [total program funding available annually] |
| Expected Number of Awards      | [range]                            | [e.g., "15-25 awards per year"]          |
| Expected Award Size            | Floor: $[X], Ceiling: $[Y]         | [per-award range]                        |
| Allowable Cost Categories       | [list]                             | [personnel, travel, supplies, subawards, etc.] |
| Unallowable Cost Categories     | [list]                             | [construction, lobbying, supplanting, etc.] |
| Statutory Indirect Cost Cap    | None / [%]                         | [some training grants capped at 8% F&A]  |
| De Minimis Rate Permitted      | Yes / No                           | [per 2 CFR 200.414(f); default Yes]      |
| Match / Cost Share Requirement | None / [%] / [specific]            | [statutory vs. voluntary committed]      |
| Required Data Systems          | [list or None]                     | [UDS, TEDS, program-specific]            |
| Performance Measurement Frequency | Quarterly / Annual / Final       | [from Block 4]                           |
| Federal Involvement (if coop)  | [description]                      | [from Block 5, coop ag only]             |
```

Include rows for every program parameter derived in Phase 1. After the table, tell the user in plain chat prose: *"This program parameters table is ready for handoff to the Grants Budget Builder. Say 'build the grant budget' and the Grants Budget Builder will estimate per-award costs using these parameters, orchestrating BLS OEWS and GSA Per Diem APIs to produce a detailed SF-424A budget workbook. That's the sister skill that completes the grants pair — the Program Description is the what, the Grants Budget Builder is the how much."*

**DO NOT, under any circumstances:**

- Write the program parameters table into the Program Description document body, any section, or any appendix.
- Save the program parameters table as a separate .docx, .xlsx, .csv, .md, or any other file format.
- Include budget figures, per-award dollar amounts, funding profile data, or cost category allocations anywhere in the Program Description document.
- Label any part of the Program Description as "Program Parameters," "Budget Parameters," "Grants Budget Handoff," or any similar phrasing.
- Include the phrases "Grants Budget Builder," "say 'build the grant budget'," "sister skill," "the how much," or any skill-chain plumbing messaging anywhere in the Program Description document.
- Emit the table as anything other than a markdown code block or markdown table in the chat conversation.

**Why this rule exists:** The Program Description is a public-facing document (once assembled into the NOFO and posted to Grants.gov). Applicants read it to decide whether to apply and how to design their proposals. Internal program parameters (budget ranges, number of awards, cost category allocations) are partially disclosed in the NOFO's Federal Award Information section — which is a SEPARATE NOFO sub-section assembled by the user, not this skill. Leaking internal parameters into the Program Description text compromises the government's design of competition and can create protest exposure if applicants anchor on numbers not yet finalized. The handoff table stays in chat so it informs the Grants Budget Builder without contaminating the public-facing Program Description.

## Edge Cases

**User unsure whether it's a grant or cooperative agreement:** Walk through the federal involvement question. If federal staff will be decision-makers, co-investigators, or technical co-leads during performance, it's a cooperative agreement. If federal role is limited to report review and monitoring, it's a grant. If still unsure, default to grant — cooperative agreements require active Section 9 content that a grant doesn't need.

**Authorizing statute is broad (e.g., "general research authority"):** Ask for the specific program-creating authority within the broad statute. If none exists and the program is being stood up under general authority, flag that the Program Description needs to be more conservative in defining allowable activities, and the authority section will reference the broad grant of authority rather than a specific program section.

**Program predates current Administration priorities:** The Program Description should reflect the authorizing statute's intent (stable) and current policy alignment (updated each cycle). If user is concerned about alignment, note in the document that alignment with current priorities was considered and cite the specific priority supported.

**Cooperative agreement with minimal federal involvement:** If the described federal role amounts to "reviewing reports and doing site visits," that's grant-level involvement, not cooperative agreement involvement. Push back: "The federal role you've described is standard grant oversight. Cooperative agreements require substantive technical or programmatic collaboration. Should this be a grant instead?"

**Statutory indirect cost caps:** Some authorities cap F&A (e.g., HRSA training programs at 8% of total direct costs, some CDC categorical grants). Capture in the handoff table for the Grants Budget Builder and state in Section 13 Authority. If no statutory cap, default to "recipient's NICRA or de minimis rate per 2 CFR 200.414" in the Program Description and flag in the handoff table.

**Multi-agency program:** If the program is jointly funded or administered with another HHS agency or a non-HHS agency, define the lead agency, the secondary agency role, and which agency's NOFO template governs. The Program Description is single-lead; coordination with the secondary agency is noted in Section 9 (if cooperative agreement) or Section 2 Background (if grant).

**Formula / block grant context:** This skill is designed for competitive discretionary grants and cooperative agreements, not for formula or block grants. If the user describes a formula grant (e.g., Title V Maternal and Child Health Block Grant allocations to states), flag that formula grants don't typically require a Program Description in the NOFO sense — they follow formula allocation rules, not competitive review. The skill can still produce narrative content, but it won't match the expected structure of a competitive NOFO.

**Budget-constrained scope reduction (Workflow C):** When reducing program scope to fit a reduced envelope, walk backwards: which allowable activities are most essential to authorizing intent? Which priority populations are core vs. expansive? Can geographic scope narrow? Can expected outcomes be scaled down without violating the authorizing statute? Present trade-offs: "Narrowing the priority population to pregnant individuals (dropping pediatric) reduces scope by [estimate] but requires re-framing Section 6. Dropping the demonstration component reduces evaluation requirements but sacrifices replicability evidence." Let the user choose reductions, then regenerate affected sections.

**Applicant-side narrative confusion:** If the user starts describing what applicants should propose (their research plan, their specific approach, their staffing), stop and redirect: "This skill is government-side. The Program Description defines what the program IS — the applicable universe. Applicants propose their specific approaches within that universe in response to the NOFO. The applicant's project narrative is a different document, written by the applicant, not by the federal program officer. Should we continue with the program-side scope or did you want help with an applicant-side document?"

## What This Skill Does NOT Cover

- **Grant budgets** — use `grants-budget-builder` (the sister skill in the grants family)
- **Full NOFO assembly** — the user bolts this Program Description into their agency's NOFO template; other NOFO sections are not generated here
- **Eligibility section of NOFO** — separate NOFO sub-section, not this skill
- **Application and Submission Information section** — separate NOFO sub-section, not this skill
- **Evaluation / Review Criteria section** — separate NOFO sub-section, not this skill
- **Award Administration Information section** — separate NOFO sub-section, not this skill
- **Applicant-side grant narrative drafting** — applicant writes the project narrative in response to the NOFO; this skill is government side only
- **FOA/NOFO compliance review for applicants** — this skill does not review or score applications
- **Contract SOW/PWS** — use `sow-pws-builder`
- **OT project descriptions** — use `ot-project-description-builder`
- **Formula or block grant allocation narratives** — this skill is designed for competitive discretionary programs
- **Post-award program administration documents** — continuation applications, carryover requests, program reviews are separate workflows
- **Cooperative agreement Notice of Award (NoA) terms** — the NoA is issued post-award by the Grants Management Officer, not this skill


---

*MIT © James Jenrette / 1102tools. Source: github.com/1102tools/federal-contracting-skills*
