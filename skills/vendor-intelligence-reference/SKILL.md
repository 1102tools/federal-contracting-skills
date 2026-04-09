---
name: vendor-intelligence-reference
description: >
  Reference file for the Vendor Intelligence skill. Do not trigger directly.
  Contains FAR 9.104-1 responsibility factor plain language guide, risk flag
  definitions with thresholds and rationale, business type code mappings,
  sample report format, and troubleshooting. Loaded on demand by the main
  vendor-intelligence skill.
---

# Vendor Intelligence Reference

Supplemental reference for the Vendor Intelligence skill. Load on demand when you need flag definitions, FAR 9 detail, business type lookups, or troubleshooting.

## Table of Contents

1. FAR 9.104-1 Responsibility Factors (Plain Language)
2. FAR Subpart 9.4 Debarment/Suspension Summary
3. Risk Flag Definitions
4. Business Type Code Reference
5. Sample Report Format
6. Troubleshooting

---

## 1. FAR 9.104-1 Responsibility Factors

The contracting officer must determine that a prospective contractor meets ALL seven general standards before making award. The burden is on the CO to find a vendor non-responsible; vendors are presumed responsible unless evidence indicates otherwise.

### (a) Adequate financial resources to perform, or the ability to obtain them

**What to look for:** Active SAM registration (demonstrates ongoing business operations). Award history showing sustained contract performance over multiple years. Total obligated dollars in proportion to the new requirement size. Entity age (very new entities may lack resources).

**What this skill checks:** SAM registration status, entity start date, total award volume over lookback period, largest prior single award.

**What this skill cannot check:** Bank statements, credit reports, lines of credit, D&B financial stress scores. The CO may request these directly from the vendor.

### (b) Able to comply with the proposed delivery or performance schedule

**What to look for:** Prior awards in the same NAICS or PSC code. Evidence of similar scope/scale work. Active contracts that might create capacity constraints.

**What this skill checks:** NAICS codes on prior awards (does vendor have experience in this work?), active contract count and dollar volume (capacity loading).

**What this skill cannot check:** Current staffing levels, subcontractor arrangements, facility capacity. Vendor proposal addresses this.

### (c) Satisfactory performance record

**What to look for:** CPARS ratings, terminations for default, cure notices, show cause notices.

**What this skill checks:** Award history showing completed contracts (inferential). No termination data is available via public API.

**What this skill cannot check:** CPARS ratings (login-gated, no public API). FAPIIS proceedings (requires FOUO access via SAM.gov integrityInformation section, which may be empty with a public key). The CO MUST check CPARS separately.

### (d) Satisfactory record of integrity and business ethics

**What to look for:** Exclusion records (debarment, suspension, ineligibility). FAPIIS proceedings. Criminal convictions. Civil judgments. Tax delinquencies.

**What this skill checks:** SAM.gov exclusion records (active and historical). SAM.gov integrityInformation section (if available). Officer overlap with other excluded entities. Shared address/CAGE patterns.

**What this skill cannot check:** Criminal background, civil litigation, tax compliance. The eCFR skill can pull FAR 52.209-5 (certification of no convictions) and FAR 52.209-11 (tax delinquency disclosure) for reference.

### (e) Necessary organization, experience, accounting, and operational controls

**What to look for:** Entity structure (corporation vs sole proprietor), years in operation, adequate accounting system (DCAA-approved for cost-type contracts).

**What this skill checks:** Entity structure from SAM, entity age, business types. Cannot verify accounting system adequacy.

**What this skill cannot check:** Accounting system audits (DCAA), management structure, internal controls. For cost-reimbursement contracts, an adequate accounting system is a prerequisite per FAR 16.301-3.

### (f) Necessary production, construction, and technical equipment and facilities

**What to look for:** Physical capacity to perform. For services, this may mean office space, IT infrastructure, cleared facilities.

**What this skill checks:** Physical address from SAM (confirms a real location exists). Cannot verify equipment or facilities.

**What this skill cannot check:** Facility inspections, equipment inventories, security clearance facility certifications. Pre-award survey (SF 1403) addresses this when warranted.

### (g) Otherwise qualified and eligible to receive award under applicable laws and regulations

**What to look for:** SAM registration (required per FAR 4.1102 for most awards). Not on excluded parties list. Meets any special eligibility requirements (e.g., SB certification for set-aside awards, organizational conflict of interest clearance).

**What this skill checks:** SAM registration active and not expired. No active exclusions. Business type certifications on record.

**What this skill cannot check:** Organizational conflicts of interest, special statutory eligibility, agency-specific qualification requirements.

---

## 2. FAR Subpart 9.4 Debarment/Suspension Summary

**Debarment (9.406):** An exclusion from government contracting for a reasonable, specified period. Grounds include: conviction of fraud, embezzlement, tax evasion, bribery; violation of antitrust statutes; commission of any offense indicating lack of business integrity; any other cause so serious that it affects the present responsibility of the contractor.

**Suspension (9.407):** A temporary exclusion pending investigation or legal proceedings. Based on adequate evidence of the grounds for debarment or indictment. Duration: generally 12 months unless legal proceedings are initiated, then duration of proceedings.

**Key rules for COs:**
- FAR 9.405: COs shall not solicit offers from, award contracts to, or consent to subcontracts with debarred, suspended, or proposed-for-debarment contractors UNLESS the agency head makes a compelling reason determination.
- FAR 9.405-1: COs shall not add new work to existing contracts with excluded contractors.
- The exclusion applies to the contractor and its affiliates unless the debarring official limits the scope.

**SAM.gov exclusion record fields that matter:**
- `recordStatus`: Active = currently excluded. Inactive/Terminated = historical.
- `exclusionType`: Debarment, Suspension, Ineligible (Proceedings Pending), Ineligible (Proceedings Completed), Voluntary Exclusion.
- `exclusionProgram`: Reciprocal (government-wide), Procurement, NonProcurement.

---

## 3. Risk Flag Definitions

### CRITICAL

**ACTIVE_EXCLUSION**
- Source: SAM.gov Exclusions v4
- Trigger: Any exclusion record with `recordStatus = "Active"`
- Meaning: Vendor is currently debarred, suspended, or otherwise excluded. Award is generally prohibited per FAR 9.405 unless agency head makes a compelling reason determination.
- Design note: Pre-award gate check; the most critical flag in the system
- Action: Stop. Do not award without agency head determination.

### HIGH

**NEW_ENTITY**
- Source: SAM.gov entity `entityStartDate`
- Trigger: Entity incorporated/formed within last 180 days
- Threshold: `today - entityStartDate <= 180 days`
- Meaning: Very new entity. May lack financial resources, performance history, and organizational infrastructure. Common in shell company schemes. Not disqualifying on its own.
- Design note: 180-day window chosen because COs evaluate vendors well before award date. Wider than a post-award audit would use.
- Action: Request financial capability documentation. Verify physical presence.

**NO_FEDERAL_HISTORY**
- Source: USASpending + SAM.gov Contract Awards
- Trigger: Zero prior federal contract awards in lookback period (default 5 years)
- Meaning: First-time federal contractor. Higher risk of performance issues, accounting system inadequacy, compliance gaps. Not disqualifying.
- Design note: No dollar or competition filter applied. Any zero-history vendor triggers regardless of award size or competition type.
- Action: Consider pre-award survey (FAR 9.106). Request additional capability evidence.

**AWARD_GROWTH_SPIKE**
- Source: SAM.gov Contract Awards modification history
- Trigger: Any active contract where current obligation exceeds 300% of initial obligation
- Threshold: `(current_obligation - initial_obligation) / initial_obligation >= 3.0`
- Meaning: Contract grew far beyond original scope/price. May indicate poor estimating, scope creep, or deliberate low-bidding. Raises questions about vendor's cost management.
- Design note: 300% threshold balances sensitivity with false positives. Catches runaway growth without flagging normal option exercises.
- Action: Review modification history on flagged contracts. Consider in FAR 9.104-1(c) performance assessment.

### MEDIUM

**EXPIRING_REGISTRATION**
- Source: SAM.gov entity `registrationExpirationDate`
- Trigger: Registration expires within 60 days
- Meaning: Vendor may become ineligible before award. FAR 4.1102 requires active SAM registration at time of award.
- Design note: Pre-award variant checks expiring registrations rather than late registrations
- Action: Notify vendor to renew. Do not award after expiration.

**NAICS_MISMATCH**
- Source: SAM.gov assertions.goodsAndServices.naicsList vs user-provided NAICS
- Trigger: Solicitation NAICS code not found in vendor's registered NAICS list
- Meaning: Vendor does not self-identify as performing this type of work. May indicate capability gap or simply incomplete SAM profile. For SB set-asides, the NAICS code determines the size standard.
- Design note: Critical for SB set-asides where NAICS determines size standard
- Action: Verify vendor capability through other means. For SB set-asides, the NAICS must match.

**EXCLUSION_HISTORY**
- Source: SAM.gov Exclusions v4, records with `recordStatus != "Active"`
- Trigger: Any historical (terminated/inactive) exclusion on record
- Meaning: Vendor was previously excluded but the exclusion has ended. May have addressed underlying issues. Pattern of exclusion/reinstatement warrants closer look.
- Action: Review exclusion details. Consider in FAR 9.104-1(d) integrity assessment.

**SHARED_ADDRESS**
- Source: SAM.gov entity search by address
- Trigger: 3 or more SAM-registered entities at the same physical address
- Meaning: Multiple entities at one address may indicate shell company networks, pass-through arrangements, or affiliated entities used for bid manipulation. Could also be a legitimate shared office building or incubator.
- Design note: Well-known indicator in procurement oversight for shell company detection
- Action: Review the other entities at the address. Check for common officers (OFFICER_OVERLAP). Could be innocuous (office building) or concerning (same principals, multiple entities bidding on same work).

**SHARED_CAGE**
- Source: SAM.gov entity search by CAGE code
- Trigger: Multiple UEIs sharing the same CAGE code
- Meaning: CAGE codes are supposed to be unique to a single entity. Sharing may indicate entity restructuring, acquisition, or data quality issues. Could also indicate affiliated entities.
- Design note: CAGE codes should map to a single entity; sharing indicates restructuring or affiliated entities
- Action: Verify entity identity. May be a legitimate parent/subsidiary relationship.

**CONCENTRATION_RISK**
- Source: USASpending spending by agency
- Trigger: 80%+ of vendor's total federal contract dollars from a single awarding agency
- Meaning: Vendor heavily dependent on one customer. Loss of that customer would threaten financial viability (FAR 9.104-1(a)). May also indicate preferential treatment or wired procurements.
- Design note: Measures single-customer dependency as a financial viability indicator
- Action: Consider financial viability. Not disqualifying but relevant to risk assessment.

### LOW

**BUSINESS_TYPE_GAP**
- Source: SAM.gov business types + USASpending total awards
- Trigger: Vendor claims small business status but has $100M+ in total federal awards over lookback period
- Meaning: Revenue level seems inconsistent with small business status. May have graduated, may be approaching size standard limits, or may have recently been acquired. Self-certification of SB status can be protested.
- Action: Verify SB status via SBA DSBS (manual check, no API). For 8(a)/HUBZone/SDVOSB, verify certification is current.

---

## 4. Business Type Code Reference

### SAM.gov businessTypeList codes

| Code | Description | Category |
|------|-------------|----------|
| 2X | For Profit Organization | Structure |
| A8 | Non-Profit Organization | Structure |
| LJ | Limited Liability Company | Structure |
| XS | Subchapter S Corporation | Structure |
| 23 | Minority-Owned Business | Socioeconomic |
| 27 | Self-Certified Small Disadvantaged Business | Socioeconomic |
| 8W | Women-Owned Small Business | Socioeconomic |
| A2 | Women-Owned Business | Socioeconomic |
| A5 | Veteran-Owned Business | Socioeconomic |
| QF | Service-Disabled Veteran-Owned Small Business | Socioeconomic |
| OY | Black American Owned | Socioeconomic |
| PI | Hispanic American Owned | Socioeconomic |
| MF | Manufacturer of Goods | Capability |

### SAM.gov sbaBusinessTypeList codes

| Code | Description | Certification |
|------|-------------|---------------|
| XX | 8(a) Certified | SBA certified |
| JT | Joint Venture | SBA registered |
| HZ | HUBZone Certified | SBA certified |

### Set-aside relevance mapping

| Set-aside | Required SAM codes | Verify via |
|-----------|-------------------|------------|
| Total SB (SBA) | Any SB indicator | SAM registration |
| 8(a) (8A/8AN) | XX in sbaBusinessTypeList | SBA DSBS (manual) |
| SDVOSB (SDVOSBC/SDVOSBS) | QF in businessTypeList | VA CVE or SBA cert |
| WOSB (WOSB/WOSBSS) | 8W in businessTypeList | SBA WOSB cert |
| HUBZone (HZC/HZS) | HZ in sbaBusinessTypeList | SBA DSBS (manual) |

---

## 5. Sample Report Format

```
VENDOR INTELLIGENCE REPORT
Generated: [date]
Analyst: [name, if provided]

VENDOR: [Legal Business Name]
UEI: [UEI]  |  CAGE: [CAGE]  |  Status: [Active/Expired]

--- RISK FLAGS ---
[CRITICAL] ACTIVE_EXCLUSION: Active debarment by [agency], effective [date]
[HIGH] NEW_ENTITY: Incorporated [X] days ago ([date])
[MEDIUM] NAICS_MISMATCH: Solicitation NAICS 541512 not in registered list
-- or --
No risk flags identified.

--- ENTITY PROFILE ---
Registration: Active, expires [date]
Address: [full address]
Structure: [entity type]
Business Types: [list]
Primary NAICS: [code] - [name]
All NAICS: [list]
Entity Start Date: [date] ([X] years in operation)

--- EXCLUSION STATUS ---
Active exclusions: None
Historical exclusions: [count] (all terminated)

--- AWARD HISTORY (FY[start]-FY[current]) ---
Total awards: [N] contracts, $[X] obligated
By fiscal year: [table]
Top agencies: [list]
Contract types: [FFP: N/$X, T&M: N/$X, CR: N/$X]
Set-aside awards: [N] of [total] ([%])
Largest award: [PIID], [agency], $[amount], [date]
NAICS on awards: [list]

--- FAR 9.104-1 RESPONSIBILITY MAPPING ---
[table from Step 4]

--- LIMITATIONS ---
- CPARS past performance ratings: requires manual lookup at cpars.gov
- Financial statements: not available via public API
- Facility/equipment verification: requires vendor submission or pre-award survey
- FAPIIS proceedings: may require FOUO-level SAM.gov access
- Security clearance status: not available via public API

--- DISCLAIMER ---
This report is assembled from public data sources for CO decision support.
It does not constitute a responsibility determination. The contracting
officer retains sole authority for responsibility decisions per FAR 9.103.
```

---

## 6. Troubleshooting

| Issue | Cause | Resolution |
|-------|-------|------------|
| SAM entity lookup returns 0 results | UEI wrong, vendor not registered, or `samRegistered=Yes` excludes unregistered | Try name search. Try `samRegistered=No` to find assigned-but-unregistered UEIs. |
| Exclusion check returns 0 but vendor is excluded | Exclusion may be under a different UEI or entity name (pre-UEI era used DUNS) | Search by entity name via `sam_exclusions({"entityName": name})` |
| Award history empty but vendor claims federal experience | USASpending lag (up to 30 days). SAM Contract Awards lag (up to 10 days). Subcontract work not in FPDS. | Check both SAM awards and USASpending. Ask vendor for contract references. |
| NAICS_MISMATCH fires but vendor clearly does this work | Vendor may not have updated their SAM NAICS list. Common with small businesses. | Not disqualifying. Verify capability through other means. |
| SHARED_ADDRESS fires in a major city | Office buildings, co-working spaces, and registered agent addresses are shared legitimately | Check if other entities at the address have common officers or similar names. |
| SHARED_CAGE fires | CAGE codes can persist through entity restructuring. Parent/subsidiary may share CAGE. | Verify entity identity. Check DLA CAGE program if needed. |
| BUSINESS_TYPE_GAP fires on a legitimate SB | Revenue thresholds vary by NAICS. Some NAICS allow $40M+ and still qualify as SB. | Check SBA size standards for the specific NAICS. $100M flag is a rough heuristic. |
| Entity start date is very old (1800s) | SAM data quality issue. Some legacy registrations have incorrect dates. | Ignore flag if entity is clearly established. Check incorporation records if concerned. |
| CONCENTRATION_RISK fires on a small business | Small vendors often start with one agency customer. Normal growth pattern. | Less concerning for small businesses. More relevant for large established contractors. |
| Multiple vendors to check | Skill runs one vendor at a time. Each run uses 5-10 SAM API calls. | Space runs 1-2 minutes apart to respect rate limits. Federal system accounts get 10K calls/day. |
