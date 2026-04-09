---
name: vendor-intelligence
description: >
  Pre-award vendor due diligence combining SAM.gov, USASpending, and eCFR into
  a single responsibility determination workflow. Produces entity profiles,
  exclusion checks, award history, FAR 9.104-1 factor mapping, and risk flags.
  Trigger on: vendor intelligence, vendor due diligence, responsibility
  determination, pre-award check, vendor risk, vendor background, vendor
  profile, is this vendor responsible, can I award to this vendor, check this
  contractor, vet this vendor, vendor research, entity check, vendor check.
  Requires sam-gov-api, usaspending-api, and ecfr-api skills installed.
---

# Vendor Intelligence Skill

**Version 1.0** | April 2026

| Version | Date | Change |
|---------|------|--------|
| 1.0 | Apr 2026 | Initial release: full pre-award due diligence with 11 risk flags, FAR 9.104-1 mapping, award history |

## Overview

This skill orchestrates three API skills to produce pre-award vendor intelligence reports. It contains no API code of its own. It tells Claude what to pull from SAM.gov, USASpending, and eCFR, in what order, and how to synthesize findings into a structured due diligence report with responsibility factor evidence and risk flags.

**Required skills (must be installed):**
1. **SAM.gov API (v1.1+)**: entity registration, exclusions, contract awards
2. **USASpending API (v1.5+)**: federal award spending data
3. **eCFR API (v1.2+)**: FAR/DFARS clause text

**Required API keys:** SAM.gov API key (same key used by SAM.gov skill). USASpending and eCFR require no keys.

**What this skill produces:** A structured vendor intelligence report covering entity profile, exclusion status, contract award history, FAR 9.104-1 responsibility factor mapping, and risk flags. The report is for pre-award file documentation and CO decision support.

**Regulatory basis:** FAR 9.103 requires the contracting officer to make a determination of responsibility before awarding any contract. FAR 9.104-1 lists seven general standards. This skill systematically gathers evidence for each standard from public data sources and flags risks that warrant further investigation.

## Information to Collect

Ask for all inputs in a single pass.

### Required

| Input | Description | Example |
|-------|-------------|---------|
| Vendor identifier | UEI, CAGE code, or legal business name | QVZMH5JLF274 or "Leidos, Inc." |

### Optional

| Input | Default | Notes |
|-------|---------|-------|
| Solicitation NAICS | None | If provided, enables NAICS_MISMATCH risk flag |
| Lookback years | 5 | Fiscal years of award history to pull |
| Include FAR text | No | Whether to include full FAR 9.104-1 text in output |

### Input Resolution

If the user provides a name instead of UEI:
1. Call `sam_entity({"legalBusinessName": name, "registrationStatus": "A"})` 
2. If multiple results, present the top 5 with UEI, CAGE, and address
3. Ask the user to confirm which entity
4. If single result, proceed with that UEI

If the user provides a CAGE code:
1. Call `sam_entity({"cageCode": cage})` to resolve to UEI
2. Proceed with the returned UEI

## Workflow

Execute these steps in order. Each step uses helpers from the required skills. Between API calls, pause 0.5s to respect rate limits. All three skills share the same SAM.gov API key budget.

### Step 1: Entity Profile (SAM.gov)

```python
result = sam_entity({
    "ueiSAM": uei,
    "includeSections": "entityRegistration,coreData,assertions",
    "samRegistered": "Yes"
})
```

Extract and present:
- **Legal business name**, DBA name (if different)
- **UEI** and **CAGE code**
- **Registration status** (Active/Expired/Inactive) and **expiration date**
- **Activation date** (most recent renewal activation, NOT first-ever registration)
- **Entity start date** from `coreData.entityInformation.entityStartDate` (date of incorporation/formation)
- **Physical address** (street, city, state, ZIP, country)
- **Congressional district**
- **Entity structure** (corporate, LLC, sole proprietor, etc.)
- **Business types** from `businessTypeList` -- map codes to names:
  - QF = Service-Disabled Veteran-Owned SB
  - 8W = Women-Owned SB
  - A2 = Women-Owned Business
  - 23 = Minority-Owned Business
  - 27 = Self-Certified SDB
  - 2X = For Profit Organization
  - A8 = Non-Profit Organization
  - LJ = Limited Liability Company
  - XS = Subchapter S Corporation
- **SBA business types** from `sbaBusinessTypeList` (8(a), HUBZone, etc.). Note: empty SBA types return as `[{"sbaBusinessTypeCode": null, ...}]`, not an empty list. Treat null-valued entries as "none."
- **Primary NAICS** and full **NAICS list** from assertions.goodsAndServices
- **PSC codes** if available in assertions (same null-object pattern as SBA types)

### Step 2: Exclusion Check (SAM.gov)

```python
result = sam_exclusions({"ueiSAM": uei})
```

Present:
- **Total exclusion records** found
- For each record (fields are NESTED, not top-level):
  - Exclusion type: `exclusionDetails.exclusionType` (Debarment, Suspension, Ineligible, etc.)
  - Exclusion program: `exclusionDetails.exclusionProgram` (Reciprocal, Procurement, NonProcurement)
  - Excluding agency: `exclusionDetails.excludingAgencyName`
  - Record status: `exclusionActions.listOfActions[0].recordStatus` (Active vs Inactive/Terminated). **NOT a top-level field.**
  - Activation date: `exclusionActions.listOfActions[0].activateDate`
  - Termination date: `exclusionActions.listOfActions[0].terminationDate`
- **Cross-references**: `exclusionOtherInformation.crossReferencesList[].name` -- names of other excluded entities/individuals linked to this vendor (reveals affiliated excluded parties)
- **Summary statement**: "No active exclusions on record" or "ACTIVE EXCLUSION FOUND -- see details"

Also search exclusions by entity name as a cross-check: `sam_exclusions({"q": legal_business_name})`. Some exclusions are filed under different UEIs or pre-UEI DUNS numbers. The name search may return records the UEI search misses.

### Step 3: Contract Award History (SAM.gov Awards + USASpending)

Pull from both sources. SAM.gov Contract Awards has raw FPDS records with full field fidelity. USASpending has aggregated spending data with better search.

**SAM.gov Contract Awards:**
```python
result = sam_awards({
    "awardeeUniqueEntityId": uei,
    "limit": "100",
    "offset": "0"
})
```
Page through all results if totalRecords > 100. Note: empty results use the `awardResponse` wrapper with string-typed values, not `awardSummary`. See SAM.gov skill Rule 26.

**Name-based fallback:** If SAM awards returns 0 results for the UEI, search by vendor name:
```python
result = sam_awards({
    "awardeeLegalBusinessName": legal_business_name,
    "limit": "100",
    "offset": "0"
})
```
Large contractors (Leidos, Booz Allen, etc.) maintain dozens of UEIs. A single UEI may show zero awards while the entity family has billions. The name search captures awards across all UEIs. Note this in the report if the fallback was used.

**Important: name search does fuzzy/partial matching.** "Music Mountain Tech" matches "Mountain Music Group" because all words appear somewhere in the corpus. After name-based fallback, filter results by checking `awardeeData.awardeeUEIInformation.uniqueEntityId` matches the target UEI, or `awardeeData.awardeeHeader.legalBusinessName` exactly matches the target name. Discard non-matching records. For small vendors with common-word names, this filtering is essential to avoid massive false positive sets.

**USASpending (for spending aggregation):**

The USASpending skill's `search_awards()` helper only supports `keywords` filter. For vendor intelligence, build the POST request directly using the Common Filter Object documented in the USASpending skill:

```python
# Build filter dict per USASpending skill's Common Filter Object section
filters = {
    "recipient_search_text": [legal_business_name],  # NOT UEI -- unreliable for UEI matching
    "time_period": [{"start_date": start, "end_date": today}],
    "award_type_codes": ["A", "B", "C", "D"]  # contracts only
}

# Award search: build raw POST (search_awards() helper only supports keywords filter)
# POST /api/v2/search/spending_by_award/
payload = {
    "filters": filters,
    "fields": ["Award ID", "Recipient Name", "Award Amount", "Awarding Agency",
               "Start Date", "Contract Award Type", "Description", "NAICS"],
    "limit": 50, "page": 1,
    "sort": "Award Amount", "order": "desc",
    "subawards": False
}

# Award count: use get_award_counts(filters) helper from USASpending skill
# Agency breakdown: use spending_by_category("awarding_agency", filters) helper
```

**Multi-UEI awareness:** If the vendor is part of a larger corporate family (common for entities with $1B+ in awards), awards may span multiple UEIs. USASpending name search captures this naturally. SAM awards by name also captures cross-UEI results. Note in the report when results include awards under multiple UEIs.

Synthesize into:
- **Total awards** (count) and **total dollars obligated** over lookback period
- **Awards by fiscal year** (count and dollars per FY)
- **Top 5 awarding agencies** by dollar volume
- **Contract types used**: FFP count/dollars, T&M count/dollars, CR count/dollars, other
- **Set-aside awards**: count awarded under SB set-asides vs full-and-open
- **Largest single award**: PIID, agency, dollars, date signed, description
- **NAICS codes awarded under**: list of NAICS on actual awards (for NAICS_MISMATCH check)
- **Active contracts with significant growth**: any PIID where current obligation > 300% of initial mod 0 obligation (for AWARD_GROWTH_SPIKE). Use SAM awards with `piidAggregation=Y` to compare base vs current totals.

### Step 4: FAR 9.104-1 Responsibility Factor Mapping

For each of the seven general standards in FAR 9.104-1, map what evidence we found. If `include_far_text` is True, fetch the text:

```python
# eCFR skill uses get_content() + extract_section_text(), NOT ecfr_content()
xml = get_content(title_number=48, section="9.104-1")
text = extract_section_text(xml)
```

Present as a table:

| Factor | Standard | Evidence Found | Gaps |
|--------|----------|----------------|------|
| (a) | Adequate financial resources | Registration active, $X in awards over Y years | No financial statements available |
| (b) | Able to comply with delivery/performance | N prior awards in same NAICS | No CPARS data |
| (c) | Satisfactory performance record | N awards, no terminations visible | CPARS requires manual check |
| (d) | Satisfactory integrity and ethics | No active exclusions, no integrity flags | FAPIIS requires manual check |
| (e) | Organization, experience, accounting, controls | Entity age: X years, structure: Y | Cannot verify accounting system |
| (f) | Necessary equipment and facilities | Cannot verify via public API | Requires vendor submission |
| (g) | Otherwise qualified and eligible | SAM registered, not excluded | Verify any special eligibility |

### Step 5: Risk Flags

Run each flag check against the data already collected. Do not make additional API calls unless noted.

**CRITICAL:**

| Flag | Check | Trigger |
|------|-------|---------|
| ACTIVE_EXCLUSION | Step 2 results | Any exclusion with recordStatus = "Active" |

**HIGH:**

| Flag | Check | Trigger |
|------|-------|---------|
| NEW_ENTITY | Step 1 entity start date | Incorporated within last 180 days |
| NO_FEDERAL_HISTORY | Step 3 results | Zero prior federal contract awards in lookback period |
| AWARD_GROWTH_SPIKE | Step 3 results | Any active contract with 300%+ obligation growth |

**MEDIUM:**

| Flag | Check | Trigger |
|------|-------|---------|
| EXPIRING_REGISTRATION | Step 1 expiration date | SAM registration expires within 60 days |
| NAICS_MISMATCH | Step 1 NAICS list vs user input | Solicitation NAICS not in vendor's registered NAICS list |
| EXCLUSION_HISTORY | Step 2 results | Past exclusion exists (terminated/inactive) |
| SHARED_ADDRESS | Step 1 address | Search SAM entities by physical address; flag if 3+ entities share it |
| SHARED_CAGE | Step 1 CAGE | Search SAM entities by CAGE code; flag if multiple UEIs share it |
| CONCENTRATION_RISK | Step 3 results | 80%+ of federal dollars from a single awarding agency |

**LOW:**

| Flag | Check | Trigger |
|------|-------|---------|
| BUSINESS_TYPE_GAP | Step 1 + Step 3 | Claims SB set-aside status but has $100M+ in total awards |

For SHARED_ADDRESS and SHARED_CAGE, make one additional SAM.gov call each:
```python
# Shared address check -- use q= free-text search with street number + street name
# Do NOT use physicalAddressCity+State (returns entire city, 10 per page, impractical)
sam_entity({"q": "street_number street_name city state", "registrationStatus": "A"})
# CAVEAT: q= matches BOTH physical AND mailing addresses. Large corporate
# HQs (e.g., 1750 Presidents St Reston) may return 100+ subsidiaries that
# use HQ as mailing address but operate elsewhere. Check that the PHYSICAL
# address matches before counting. If most results are corporate affiliates
# of the same parent, this is expected and not a red flag.
# Flag if 3+ distinct UNRELATED UEIs appear at the same physical address

# Shared CAGE check
sam_entity({"cageCode": cage})
# Flag if multiple UEIs returned
```

### Step 6: Assemble Report

Present the report in this order:
1. **Header**: Vendor name, UEI, date of report, analyst (if provided)
2. **Entity Profile** (Step 1)
3. **Exclusion Status** (Step 2)
4. **Risk Flags Summary** (Step 5) -- show flags that triggered with severity and one-line explanation. If no flags triggered, state "No risk flags identified."
5. **Award History** (Step 3)
6. **FAR 9.104-1 Responsibility Mapping** (Step 4)
7. **Limitations**: List what this report cannot verify (CPARS, financial statements, facilities, security clearances)
8. **Disclaimer**: "This report is assembled from public data sources for CO decision support. It does not constitute a responsibility determination. The contracting officer retains sole authority for responsibility decisions per FAR 9.103."

## Risk Flag Reference

See the `vendor-intelligence-reference` skill for detailed flag definitions, thresholds, rationale, and recommended actions.

## Example Prompts

- "Run vendor intelligence on UEI QVZMH5JLF274 for a 541512 requirement"
- "Check if Booz Allen Hamilton is a responsible contractor for an IT services award"
- "Do pre-award due diligence on CAGE code 12DZ1"
- "I need a vendor background check for my responsibility determination on [vendor name]"
- "Vet this company before I make award: [vendor name], UEI [code]"
- "Is there anything concerning about [vendor name] before I award this contract?"

## Rate Limit Awareness

This workflow makes 5-10 SAM.gov API calls per vendor (entity, exclusions, awards, address check, CAGE check). The SAM.gov daily budget is shared across all SAM.gov skill usage. A single vendor intel run uses roughly the same budget as 5-10 individual SAM.gov lookups. Plan accordingly if running multiple vendors in sequence.
