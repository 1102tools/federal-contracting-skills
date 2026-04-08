---
name: sam-gov-api
description: >
  Query SAM.gov REST APIs for entity registration, exclusion/debarment records, contract opportunities, and contract awards (FPDS replacement). Entity Management v3: UEI/CAGE lookups, registration status, business types, NAICS/PSC, reps and certs. Exclusions v4: debarment/suspension checks. Opportunities v2: solicitations, sources sought, award notices. Contract Awards v1: award search, PIID lookup, modification history, vendor awards. Trigger on: SAM.gov, UEI, CAGE code, entity registration, debarment, suspension, excluded parties, exclusion check, responsibility determination, contract opportunities, solicitations, sources sought, contract awards, FPDS, PIID lookup, award search, award history, modification history, vendor award profile, vendor due diligence, entity validation, pre-award checks.
---

# SAM.gov API Skill (v1.1)

## Changelog
- v1.1: Added Contract Awards API (v1), the FPDS.gov replacement. New `sam_awards()` helper, rules 24-30, endpoint map, quick reference examples, and composite workflows. Updated USASpending relationship guidance.
- v1.0: Initial release covering Entity Management (v3), Exclusions (v4), Get Opportunities (v2), PSC lookup

## Overview

The SAM.gov APIs (https://api.sam.gov) provide access to entity registration data, exclusion records, contract opportunities, and contract award data across the federal government. SAM.gov consolidated the former CCR, EPLS, FBO, CFDA, and FPDS systems; each legacy system became a separate API domain sharing one authentication key. The Contract Awards API (v1) replaced FPDS.gov, which was decommissioned February 24, 2026.

Base URL: `https://api.sam.gov`
Docs: https://open.gsa.gov/api/

**Relationship to USASpending skill:** USASpending aggregates and normalizes award data for spending analysis, trend reporting, obligation tracking, and vendor spending totals (no key, no rate limits). SAM.gov Contract Awards provides raw FPDS transaction records with full field fidelity for individual award lookups, PIID-specific research, modification tracking, and contractor-level award history. Use USASpending for "how much did agency X spend on NAICS Y" and SAM.gov Contract Awards for "show me every modification on PIID W52P1J-20-C-0001." SAM.gov also covers pre-award intelligence that USASpending cannot: entity registration status, exclusion/debarment records, and active solicitations.

**For entity section schemas, exclusion response fields, contract awards response schema, opportunity parameter tables, notice type codes, set-aside codes, composite workflows, and troubleshooting:** `view` the REFERENCE.md file in this skill directory.

## Authentication and Rate Limits

- API key required for all endpoints
- Pass as query parameter: `?api_key=KEY`
- Key format: `SAM-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`

| Account Type | Daily Limit |
|---|---|
| Non-federal, no SAM role | 10/day |
| Non-federal with SAM role | 1,000/day |
| Federal personal key | 1,000/day |
| Federal system account | 10,000/day |

**Key conservation:** Prefer targeted lookups (single UEI, specific PIID) over broad scans. A typical vendor responsibility check uses 2 calls (entity + exclusion). An opportunity search uses 1 call. Plan queries to stay well within daily limits.

## Critical Rules

### 1. Date Format (MOST COMMON MISTAKE)
Entity Management and Exclusions use `MM/DD/YYYY`. Ranges use brackets: `[01/01/2025,09/30/2025]`. Get Opportunities also uses `MM/DD/YYYY` for `postedFrom`/`postedTo`. Do NOT use ISO 8601 dates.

### 2. Opportunities Require Date Parameters
`postedFrom` and `postedTo` are mandatory on every Opportunities search. The API returns an error without them.

### 3. Entity Management Returns 10 Records Per Page (HARD CAP)
Default and MAXIMUM `size=10`. Requesting `size=25` or higher returns HTTP 400 ("Size Cannot Exceed 10 Records"). For bulk needs, use extract mode (`format=csv` or `format=json`) which returns up to 1,000,000 records asynchronously. Contrast with Opportunities, which accepts `limit=1000` or higher in a single call.

### 4. Exclusions API Uses `size`, NOT `limit`
Passing `limit` returns HTTP 400 (INVALID_SEARCH_PARAMETER). Use `page` and `size` for pagination.

### 5. Opportunity Description Is a URL, Not Inline Text
The `description` field returns a URL like `https://api.sam.gov/prod/opportunities/v1/noticedesc?noticeid=ID`. Fetch that URL separately with `&api_key=KEY` appended to get the JSON response containing the HTML description. Do NOT expect inline description text in search results.

### 6. Boolean Operators in Entity/Exclusion Params
`&` = AND (default between params), `~` = OR within a param, `!` = NOT. Multi-value example: `ueiSAM=[UEI1~UEI2]` returns either entity. Up to 100 values in a single multi-value param.

### 7. Forbidden Characters and Special Character Name Gotcha
These characters cannot appear in parameter values: `& | { } ^ \`. Entity names containing `&` or `()` cannot be searched directly with `legalBusinessName`. The API returns 0 results for `JOHNSON & JOHNSON` or `GENERAL DYNAMICS (GD)`. Workaround: search by UEI or CAGE for exact lookup, or search just the main name portion (e.g., `legalBusinessName=GENERAL DYNAMICS`) and scan results.

### 8. includeSections Controls Entity Response Size
Without `includeSections`, you get `entityRegistration` and `coreData` only. `repsAndCerts` and `integrityInformation` must be explicitly requested; they are NOT included in `includeSections=All`.

### 9. Registered vs. ID-Assigned Entities
`samRegistered=Yes` (default) returns only registered entities. `samRegistered=No` returns ID-assigned-only entities (have a UEI but never completed registration). Most 1102 lookups want registered entities.

### 10. PSC Lookup Uses a Different Base Path
PSC endpoint lives at `/prod/locationservices/v1/api/publicpscdetails`, not under `/entity-information/`. Same API key works.

### 11. Do NOT Set Accept: application/json Header
The Exclusions API returns HTTP 406 (Not Acceptable) if you send `Accept: application/json`. Do not set an Accept header on any SAM.gov request; let it default. All endpoints return JSON by default without the header.

### 12. deptname and subtier Filters Are SILENTLY IGNORED on Opportunities (CRITICAL)
The `deptname` and `subtier` parameters are accepted without error but DO NOT FILTER results. Passing `deptname=XYZGARBAGE` returns the same count as no filter. To filter by agency, you must post-filter in Python by checking `fullParentPathName` in the results. Working filters: `title`, `solnum`, `noticeid`, `ptype`, `ncode`, `ccode`, `typeOfSetAside`, `state`, `zip`, `rdlfrom`/`rdlto`.

### 13. Description URL Requires API Key
The `description` field in opportunity results contains a URL, not inline text. That URL returns 404 unless you append `&api_key=KEY`. Always append the API key when fetching descriptions.

### 14. Entity Name Search Returns Partial Matches Without Relevance Ranking
`legalBusinessName=LEIDOS` returns 107 results including JVs, subsidiaries, and realty LLCs, with no relevance sorting. The parent company may not be the first result. When the user wants a specific entity, guide them to use UEI or CAGE for exact lookup. When using name search, retrieve multiple pages and filter client-side if needed.

### 15. Opportunity Date Range Max Is 364 Days
`postedFrom` to `postedTo` must span less than 1 year (364 days max). 365 days returns HTTP 400: "Date range must be null year(s) apart." For solicitation number lookups, use a recent 6-month window, not a multi-year range. If the notice could be older, make multiple calls with sequential date windows.

### 16. NAICS and PSC Codes Are in assertions, NOT coreData
NAICS/PSC data lives at `assertions.goodsAndServices.naicsList` and `assertions.goodsAndServices.pscList`. You MUST include `assertions` in `includeSections` to get them. `includeSections=entityRegistration,coreData` returns zero NAICS/PSC data. The field for small business status is `sbaSmallBusiness` (not `isSmallBusiness`). Primary NAICS is a separate field: `assertions.goodsAndServices.primaryNaics` (a string, not a boolean on each item).

### 17. Always Include entityRegistration in includeSections
If you request only `coreData` or `pointsOfContact` without `entityRegistration`, the response contains no entity name, UEI, or registration status. You cannot identify which entity the data belongs to. Always include `entityRegistration` alongside any other section.

### 18. POC Email and Phone Require FOUO System Account
With a public API key, `pointsOfContact` returns name, title, and mailing address only. Email, phone, and fax fields are omitted. These require a Federal System Account with FOUO read permissions.

### 19. Multi-ptype Requires Separate Calls or Manual URL
The `sam_opportunities()` helper passes `ptype` as a single value. To search multiple notice types at once (e.g., solicitations AND sources sought), make separate calls for each ptype and merge results, or build the URL manually with repeated params: `&ptype=o&ptype=r`.

### 20. Country Codes Must Be 3-Character (ISO Alpha-3)
Entity and Exclusion APIs require 3-char country codes: USA, CAN, CHN, GBR, etc. Two-character codes (US, CA, CN) return 0 results. This applies to `physicalAddressCountryCode`, `countryOfIncorporationCode`, and Exclusion `country` parameters.

### 21. exclusionURL Contains a Literal Placeholder
The `exclusionURL` field in entity records contains `REPLACE_WITH_API_KEY` as literal text, not your actual key. You must string-replace this placeholder before fetching: `excl_url.replace("REPLACE_WITH_API_KEY", API_KEY)`.

### 22. Opportunity ccode Requires Exact PSC Match
`ccode=R425` returns results but `ccode=R4` and `ccode=R` return 0. The opportunity PSC filter requires the exact 4-character code, not a prefix. To search a PSC family, look up child codes via the PSC API first, then query each code individually.

### 23. Entity q Parameter Uses AND Logic
Multiple words in the `q` free text parameter are ANDed together. `q=cybersecurity cloud` returns only entities matching BOTH words (49 results), not either word (cybersecurity alone = 259, cloud alone = 1,038).

### 24. Contract Awards Uses limit/offset, NOT page/size
Contract Awards pagination uses `limit` (default 10, max 100) and `offset` (0-based record skip count). Using `page` returns an empty response with no error. To get records 11-20: `limit=10&offset=10`. To get records 21-30: `limit=10&offset=20`. The API does NOT validate negative limit values (silently returns data), so always validate client-side.

### 25. Contract Awards Date Format: MM/dd/yyyy with Brackets
Same MM/DD/YYYY format as other SAM endpoints. Single value: `dateSigned=01/15/2026`. Range: `dateSigned=[01/01/2026,03/01/2026]`. ISO 8601 dates are rejected with a clear error: "dateSigned has invalid format. Expected MM/dd/yyyy or [MM/dd/yyyy,MM/dd/yyyy]". Date parameters: `dateSigned`, `lastModifiedDate`, `createdDate`, `approvedDate`, `closedDate`, `periodOfPerformanceStartDate`, `currentCompletionDate`, `ultimateCompletionDate`, `solicitationDate`.

### 26. Contract Awards Response Wrapper Is INCONSISTENT (CRITICAL)
Populated results return `awardSummary` at root level with integer pagination values: `{"awardSummary": [...], "totalRecords": 123, "limit": 10, "offset": 0}`. Empty results wrap in `awardResponse` with STRING values: `{"awardResponse": {"totalRecords": "0", "limit": "2", "offset": "1"}, "message": "No Data found..."}`. Always check for both `awardSummary` and `awardResponse` keys. Never assume `totalRecords` is an integer.

### 27. Contract Awards Error Responses Are NOT JSON
Unlike Entity Management and Exclusions which return JSON errors, Contract Awards returns plain text for most errors: bad parameter name returns "The search parameter(s),[paramname]does not exist."; bad date format returns "dateSigned has invalid format. Expected MM/dd/yyyy..."; limit > 100 returns "Max value allowed for parameter \"limit\" is 100". Bad API key returns HTML: `<h1>API_KEY_INVALID</h1>`. Missing API key returns an empty body. Always handle non-JSON responses.

### 28. Contract Awards Deleted Records Use deletedStatus, Not a Separate Endpoint
To find deleted contract records, pass `deletedStatus=Y` on the same `/contract-awards/v1/search` endpoint. There is no separate deleted-contracts path. Without this parameter, deleted records are excluded from results.

### 29. Contract Awards includeSections Controls Response Size
Three sections: `contractId`, `coreData`, `awardDetails`. Default returns all three. Use `includeSections=contractId` for lightweight PIID-only queries. Use `includeSections=contractId,awardDetails` to skip the large coreData section when you only need dates, dollars, and awardee info.

### 30. Vendor Name Parameter Is awardeeLegalBusinessName, NOT vendorName
The Contract Awards API uses `awardeeLegalBusinessName` for vendor name search. `vendorName`, `awardeeName`, and `contractorName` do NOT exist as parameters and return an error. Other vendor identifiers: `awardeeUniqueEntityId` (UEI), `awardeeCageCode` (CAGE), `ultimateParentUniqueEntityId`, `ultimateParentLegalBusinessName`.

---

## Core Helper

```python
import urllib.request, urllib.parse, json

# IMPORTANT: The API key is NOT bundled with this skill.
# Users must supply their own SAM.gov API key.
# Get one free at https://sam.gov/profile/details (Public API Key section).
# Claude: check the user's memory or conversation context for their SAM.gov API key.
# If no key is available, instruct the user to request one from their SAM.gov profile.
API_KEY = None  # Set from user context

def sam_entity(params, version="v3"):
    """Query SAM.gov Entity Management API."""
    if not API_KEY:
        raise ValueError("SAM.gov API key not set. User must provide their key.")
    params["api_key"] = API_KEY
    url = f"https://api.sam.gov/entity-information/{version}/entities?" + urllib.parse.urlencode(params, safe='[],~/!')
    req = urllib.request.Request(url)
    with urllib.request.urlopen(req, timeout=20) as resp:
        return json.loads(resp.read().decode())

def sam_exclusions(params, version="v4"):
    """Query SAM.gov Exclusions API."""
    params["api_key"] = API_KEY
    url = f"https://api.sam.gov/entity-information/{version}/exclusions?" + urllib.parse.urlencode(params, safe='[],~/!*')
    req = urllib.request.Request(url)
    with urllib.request.urlopen(req, timeout=20) as resp:
        return json.loads(resp.read().decode())

def sam_opportunities(params):
    """Query SAM.gov Get Opportunities API (v2)."""
    params["api_key"] = API_KEY
    url = "https://api.sam.gov/opportunities/v2/search?" + urllib.parse.urlencode(params, safe='[],~/!')
    req = urllib.request.Request(url)
    with urllib.request.urlopen(req, timeout=20) as resp:
        return json.loads(resp.read().decode())

def sam_awards(params, version="v1"):
    """Query SAM.gov Contract Awards API (FPDS replacement). Returns awardSummary list."""
    params["api_key"] = API_KEY
    url = f"https://api.sam.gov/contract-awards/{version}/search?" + urllib.parse.urlencode(params, safe='[],~/!')
    req = urllib.request.Request(url)
    with urllib.request.urlopen(req, timeout=30) as resp:
        data = json.loads(resp.read().decode())
    # Normalize inconsistent response wrapper (see Rule 26)
    if "awardResponse" in data and "awardSummary" not in data:
        ar = data["awardResponse"]
        data["totalRecords"] = int(ar.get("totalRecords", 0))
        data["awardSummary"] = []
    return data

def sam_psc(query, search_by=None):
    """Query SAM.gov PSC lookup. Use search_by='psc' for code lookup, or omit for free text."""
    params = {"api_key": API_KEY, "q": query}
    if search_by:
        params["searchby"] = search_by
    url = "https://api.sam.gov/prod/locationservices/v1/api/publicpscdetails?" + urllib.parse.urlencode(params)
    req = urllib.request.Request(url)
    with urllib.request.urlopen(req, timeout=15) as resp:
        return json.loads(resp.read().decode())

def safe_sam(func, params, **kwargs):
    """Wrapper for graceful error handling."""
    try:
        return func(params, **kwargs)
    except urllib.error.HTTPError as e:
        body = e.read().decode() if hasattr(e, 'read') else ""
        try:
            err = json.loads(body)
        except Exception:
            err = {"raw": body}
        return {"error": e.code, "detail": err}
```

---

## Endpoint Map

### Entity Management (v3)

| Endpoint | Description | Key Use |
|----------|-------------|---------|
| `/entity-information/v3/entities` | Entity registration data | UEI/CAGE lookup, registration status, business types, POCs, reps & certs |

**Key Search Parameters:**

| Parameter | Description | Example |
|---|---|---|
| `ueiSAM` | UEI (single or up to 100 with `~` OR) | `ueiSAM=ZQGGHJH74DW7` |
| `cageCode` | CAGE code (single or up to 100) | `cageCode=855J5` |
| `legalBusinessName` | Partial or complete name | `legalBusinessName=BOOZ ALLEN` |
| `registrationStatus` | A=Active, E=Expired | `registrationStatus=A` |
| `primaryNaics` | 6-digit primary NAICS | `primaryNaics=541512` |
| `naicsCode` | Any NAICS on record | `naicsCode=541512` |
| `pscCode` | 4-char PSC | `pscCode=R425` |
| `businessTypeCode` | 2-char business type code | `businessTypeCode=23` (minority-owned) |
| `physicalAddressProvinceOrStateCode` | 2-char state | `physicalAddressProvinceOrStateCode=TX` |
| `purposeOfRegistrationCode` | Z1=Fed Assistance, Z2=All Awards | `purposeOfRegistrationCode=Z2` |
| `includeSections` | Filter response sections | `includeSections=entityRegistration,coreData` |
| `q` | Free text search | `q=cybersecurity` |

**includeSections values:** `entityRegistration`, `coreData`, `assertions`, `pointsOfContact`, `repsAndCerts`, `integrityInformation`, `All`. Note: `repsAndCerts` and `integrityInformation` require explicit request; `All` does not include them.

### Exclusions (v4)

| Endpoint | Description | Key Use |
|----------|-------------|---------|
| `/entity-information/v4/exclusions` | Debarment/suspension records | Responsibility determination, excluded party check |

**Key Search Parameters:**

| Parameter | Description | Example |
|---|---|---|
| `ueiSAM` | UEI of excluded entity | `ueiSAM=E1K4E4A29SU5` |
| `cageCode` | CAGE code | `cageCode=12345` |
| `classification` | Firm, Individual, Vessel, Special Entity Designation | `classification=Firm` |
| `exclusionType` | Type of exclusion | `exclusionType=Ineligible (Proceedings Completed)` |
| `exclusionProgram` | Reciprocal, NonProcurement, Procurement | `exclusionProgram=Reciprocal` |
| `excludingAgencyCode` / `excludingAgencyName` | Agency that excluded | `excludingAgencyCode=DOD` |
| `stateProvince` | State code | `stateProvince=VA` |
| `country` | Country code | `country=USA` |
| `q` | Free text (supports `*` wildcard, AND, OR operators) | `q=acme*` |
| `activationDate` | Date range | `activationDate=[01/01/2024,12/31/2024]` |

**Response structure:** `totalRecords` + `excludedEntity[]` array. Each entry has `exclusionDetails`, `exclusionIdentification`, `exclusionActions`, `exclusionAddress`, `exclusionOtherInformation`, `vesselDetails`.

### Get Opportunities (v2)

| Endpoint | Description | Key Use |
|----------|-------------|---------|
| `/opportunities/v2/search` | Contract opportunity notices | Active solicitations, sources sought, pre-sols, award notices |

**Key Search Parameters:**

| Parameter | Description | Example |
|---|---|---|
| `postedFrom` / `postedTo` | **MANDATORY** date range (MM/DD/YYYY) | `postedFrom=01/01/2026&postedTo=04/04/2026` |
| `ptype` | Notice type code (repeatable) | `ptype=o&ptype=k` |
| `title` | Search in title | `title=cybersecurity` |
| `solnum` | Solicitation number | `solnum=75FCMC25R0001` |
| `noticeid` | Specific notice ID | `noticeid=abc123def456` |
| `ncode` | NAICS code | `ncode=541512` |
| `ccode` | PSC/classification code | `ccode=R425` |
| `typeOfSetAside` | Set-aside code | `typeOfSetAside=SBA` |
| `deptname` | **BROKEN: silently ignored, does not filter** | Do not use |
| `subtier` | **BROKEN: silently ignored, does not filter** | Do not use |
| `state` | Place of performance state | `state=MD` |
| `rdlfrom` / `rdlto` | Response deadline range | `rdlfrom=04/01/2026&rdlto=04/30/2026` |
| `limit` / `offset` | Pagination | `limit=25&offset=0` |

**Notice Type Codes (`ptype`):**

| Code | Type |
|---|---|
| `p` | Presolicitation |
| `o` | Solicitation |
| `k` | Combined Synopsis/Solicitation |
| `r` | Sources Sought |
| `g` | Sale of Surplus Property |
| `s` | Special Notice |
| `i` | Intent to Bundle |
| `a` | Award Notice |
| `u` | Justification (J&A) |

### PSC Lookup (Convenience)

| Endpoint | Description | Key Use |
|----------|-------------|---------|
| `/prod/locationservices/v1/api/publicpscdetails` | Product Service Code lookup | PSC code/name resolution, category hierarchy |

**Parameters:** `searchby=psc` with `q` (code prefix search), or omit `searchby` and use `q` alone for free text search across all fields. Also: `active` (Y, N, ALL). Returns `pscCode`, `pscName`, `pscFullName`, `pscInclude`, `pscExclude`, `parentPscCode`, `level1Category`, `level1CategoryName`, `level2Category`, `level2CategoryName`.

### Contract Awards (v1) -- FPDS Replacement

| Endpoint | Description | Key Use |
|----------|-------------|---------|
| `/contract-awards/v1/search` | Contract award transaction data | PIID lookup, vendor award history, modification tracking, award analysis |

**Key Search Parameters:**

| Parameter | Description | Example |
|---|---|---|
| `piid` | Procurement Instrument Identifier | `piid=W52P1J20C0001` |
| `awardeeLegalBusinessName` | Vendor name (partial match) | `awardeeLegalBusinessName=BOOZ ALLEN` |
| `awardeeUniqueEntityId` | Vendor UEI | `awardeeUniqueEntityId=CLD7RLD1Y828` |
| `awardeeCageCode` | Vendor CAGE code | `awardeeCageCode=2X264` |
| `contractingDepartmentCode` | 4-digit contracting agency code | `contractingDepartmentCode=9700` (DoD) |
| `contractingSubtierCode` | Sub-agency code | `contractingSubtierCode=97AS` (DLA) |
| `contractingOfficeCode` | Contracting office | `contractingOfficeCode=SPE7M1` |
| `fundingDepartmentCode` | Funding agency code | `fundingDepartmentCode=7500` (HHS) |
| `naicsCode` | 6-digit NAICS | `naicsCode=541330` |
| `productOrServiceCode` | 4-char PSC | `productOrServiceCode=R425` |
| `typeOfContractPricingCode` | Pricing type code | `typeOfContractPricingCode=J` (FFP) |
| `typeOfSetAsideCode` | Set-aside type | `typeOfSetAsideCode=SBA` |
| `extentCompetedCode` | Competition extent | `extentCompetedCode=F` (SAP competed) |
| `awardOrIDV` | AWARD or IDV | `awardOrIDV=AWARD` |
| `dateSigned` | Award date (MM/dd/yyyy or bracket range) | `dateSigned=[01/01/2026,03/01/2026]` |
| `lastModifiedDate` | Last modified (bracket range) | `lastModifiedDate=[01/01/2026,04/01/2026]` |
| `dollarsObligated` | Action obligation range | `dollarsObligated=[1000000,5000000]` |
| `totalDollarsObligated` | Total obligation range | `totalDollarsObligated=[10000000,]` |
| `fiscalYear` | Fiscal year | `fiscalYear=2026` |
| `modificationNumber` | Specific mod number | `modificationNumber=P00003` |
| `closedStatus` | Y or N | `closedStatus=N` |
| `deletedStatus` | Y to include deleted records | `deletedStatus=Y` |
| `piidAggregation` | Y/N group mods under parent PIID | `piidAggregation=Y` |
| `q` | Free text search | `q=cybersecurity` |
| `includeSections` | Limit response sections | `includeSections=contractId,awardDetails` |
| `limit` / `offset` | Pagination (max limit=100) | `limit=50&offset=0` |

**Additional Parameters:** `solicitationID`, `awardOrIDVTypeCode`, `numberOfOffersReceived`, `coBusSizeDeterminationCode`, `reasonForModificationCode`, `referencedIdvPiid`, `awardeeStateCode`, `awardeeZipCode`, `placeOfPerformStateCode`, `placeOfPerformCountryCode`, `ultimateParentUniqueEntityId`, `ultimateParentLegalBusinessName`, `format` (csv/json for async extract), `emailId` (for async delivery).

**Response structure:** `totalRecords` (int) + `awardSummary[]` array. Each record has three sections: `contractId` (PIID, mod number, transaction number, agency), `coreData` (solicitation, organization, acquisition data, legislative mandates, place of performance, product/service info, competition info), `awardDetails` (dates, dollars, total contract dollars, awardee data with business types and socioeconomic flags, transaction metadata).

---

## Quick Reference: Common Queries

```python
# Entity: Look up a vendor by UEI (basic registration info)
sam_entity({"ueiSAM": "ZQGGHJH74DW7", "includeSections": "entityRegistration,coreData"})

# Entity: Look up by CAGE code
sam_entity({"cageCode": "855J5", "includeSections": "entityRegistration"})

# Entity: Search by name
sam_entity({"legalBusinessName": "BOOZ ALLEN", "registrationStatus": "A"})

# Entity: Find active SDVOSB firms in Virginia doing IT
sam_entity({
    "businessTypeCode": "QF",
    "physicalAddressProvinceOrStateCode": "VA",
    "primaryNaics": "541512",
    "registrationStatus": "A",
    "includeSections": "entityRegistration,coreData,assertions"
})

# Entity: Get reps & certs for a specific vendor
sam_entity({"ueiSAM": "ZQGGHJH74DW7", "includeSections": "repsAndCerts"})

# Entity: Get integrity/proceedings information
sam_entity({
    "ueiSAM": "ZQGGHJH74DW7",
    "includeSections": "integrityInformation",
    "proceedingsData": "Yes"
})

# Exclusion: Check if a vendor is excluded
sam_exclusions({"ueiSAM": "E1K4E4A29SU5"})

# Exclusion: Search excluded firms in Virginia
sam_exclusions({"classification": "Firm", "stateProvince": "TX"})

# Exclusion: Recent exclusion actions
sam_exclusions({"activationDate": "[01/01/2026,04/04/2026]"})

# Exclusion: Free text search with wildcard
sam_exclusions({"q": "acme*"})

# Opportunities: Active solicitations posted this month
sam_opportunities({
    "postedFrom": "04/01/2026",
    "postedTo": "04/04/2026",
    "ptype": "o",
    "limit": "25"
})

# Opportunities: Sources sought for IT services
sam_opportunities({
    "postedFrom": "01/01/2026",
    "postedTo": "04/04/2026",
    "ptype": "r",
    "ncode": "541512",
    "limit": "25"
})

# Opportunities: SDVOSB solicitations (set-aside filter works)
sam_opportunities({
    "postedFrom": "10/01/2025",
    "postedTo": "04/04/2026",
    "typeOfSetAside": "SDVOSBC",
    "ptype": "o",
    "limit": "25"
})

# Opportunities: Filter by agency (post-filter; deptname param is broken)
r = sam_opportunities({
    "postedFrom": "10/01/2025",
    "postedTo": "04/04/2026",
    "ptype": "o",
    "limit": "100"
})
hhs_opps = [o for o in r.get("opportunitiesData", [])
            if "DEFENSE" in o.get("fullParentPathName", "").upper()]

# Opportunities: Specific solicitation number (date range max 364 days)
sam_opportunities({
    "postedFrom": "10/01/2025",
    "postedTo": "04/04/2026",
    "solnum": "75FCMC25R0001",
    "limit": "10"
})

# Opportunities: Recent award notices (post-filter by agency if needed)
r = sam_opportunities({
    "postedFrom": "10/01/2025",
    "postedTo": "04/04/2026",
    "ptype": "a",
    "limit": "25"
})

# Fetch full opportunity description (requires API key appended)
import urllib.request
def get_opp_description(notice_id):
    url = f"https://api.sam.gov/prod/opportunities/v1/noticedesc?noticeid={notice_id}&api_key={API_KEY}"
    req = urllib.request.Request(url)
    with urllib.request.urlopen(req, timeout=15) as resp:
        return json.loads(resp.read().decode()).get("description", "")

# Awards: Look up a specific contract by PIID (all modifications)
sam_awards({"piid": "W52P1J20C0001", "limit": "100"})

# Awards: Find awards to a vendor by UEI
sam_awards({"awardeeUniqueEntityId": "CLD7RLD1Y828", "limit": "50"})

# Awards: Search by vendor name
sam_awards({"awardeeLegalBusinessName": "BOOZ ALLEN", "limit": "25"})

# Awards: HHS engineering awards signed in Q1 FY2026
sam_awards({
    "contractingDepartmentCode": "7500",
    "naicsCode": "541330",
    "dateSigned": "[10/01/2025,12/31/2025]",
    "limit": "50"
})

# Awards: Recent DoD awards with small business set-aside
sam_awards({
    "contractingDepartmentCode": "9700",
    "typeOfSetAsideCode": "SBA",
    "fiscalYear": "2026",
    "limit": "50"
})

# Awards: Modifications to a specific PIID
sam_awards({"piid": "SPE7M125P2263", "modificationNumber": "P00003"})

# Awards: Free text search
sam_awards({"q": "cybersecurity", "limit": "25"})

# Awards: Deleted records only
sam_awards({"deletedStatus": "Y", "limit": "10"})

# Awards: FFP awards over $1M in FY2026
sam_awards({
    "typeOfContractPricingCode": "J",
    "dollarsObligated": "[1000000,]",
    "fiscalYear": "2026",
    "limit": "50"
})

# Awards: Lightweight PIID-only query (skip coreData and awardDetails)
sam_awards({"naicsCode": "541512", "includeSections": "contractId", "limit": "100"})

# PSC: Look up a code
sam_psc("R425", search_by="psc")

# PSC: Free text search (no searchby param)
sam_psc("engineering")
```

---

## Rate Limiting

1. API key required on every call; pass as `api_key=` query parameter
2. Personal keys: 1,000/day (non-fed with role or federal); system accounts: 10,000/day
3. Add `time.sleep(0.5)` between batch requests
4. Prefer single-entity lookups over broad searches to conserve daily quota
5. Contract Awards calls count against the same daily limit as Entity/Exclusion/Opportunity calls. A vendor responsibility check + award history lookup = 3+ calls minimum. Plan accordingly.
6. For aggregated spending analysis and obligation tracking, use the USASpending skill (no key, no limits). Reserve SAM.gov Contract Awards for individual award record lookups, PIID-specific research, and modification tracking.

---

## Additional Resources

For entity section schemas (full field reference), exclusion response fields, contract awards response schema, opportunity set-aside codes, business type codes, composite workflows (vendor responsibility check, market research entity scan, opportunity monitoring, contract award history lookup, vendor award profile), and troubleshooting: `view` the **REFERENCE.md** file in this skill directory.
