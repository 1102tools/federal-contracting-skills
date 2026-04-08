---
name: sam-gov-api-reference
description: >
  Reference file for the SAM.gov API skill. Do not trigger directly. Contains entity section schemas, exclusion response fields, contract awards response schema, opportunity set-aside codes, business type codes, composite workflows, and troubleshooting. Loaded on demand by the main sam-gov-api skill.
---

# SAM.gov API Reference

Supplemental reference for the SAM.gov API skill. Load on demand when you need field schemas, code tables, composite workflows, or troubleshooting.

## Table of Contents
1. Entity Management Response Schema
2. Exclusion Response Schema
3. Opportunity Response Schema
4. Contract Awards Response Schema
5. Business Type Codes
6. Set-Aside Type Codes
7. Composite Workflows
8. PSC Category Hierarchy
9. Troubleshooting

---

## 1. Entity Management Response Schema

### entityRegistration Section
- `samRegistered` - "Yes" or "No"
- `ueiSAM` - 12-character Unique Entity Identifier
- `entityEFTIndicator` - EFT indicator (null for most)
- `cageCode` - 5-character CAGE code
- `dodaac` - DoDAAC (if applicable)
- `legalBusinessName` - Legal business name
- `dbaName` - Doing Business As name
- `purposeOfRegistrationCode` / `purposeOfRegistrationDesc` - Z1=Federal Assistance Awards, Z2=All Awards, Z5=Supplemental grants only
- `registrationStatus` - Active, Inactive, ID Assigned
- `evsSource` - Entity Validation Service source (D&B, etc.)
- `registrationDate` - Initial registration date
- `lastUpdateDate` - Last update
- `registrationExpirationDate` - Expiration date (registration must be renewed annually)
- `activationDate` - Date activated
- `ueiStatus` - Active or Inactive
- `ueiCreationDate` - UEI assignment date
- `publicDisplayFlag` - "Y" = opted into public display, "N" = opted out (FOUO key needed for opted-out)
- `exclusionStatusFlag` - "Y" if entity has active exclusion, "N" if not
- `exclusionURL` - URL to exclusion record if flagged
- `dnbOpenData` - "Y" if D&B data is available

### coreData Section

**entityHierarchyInformation:**
- `immediateParentEntity.ueiSAM` / `.legalBusinessName` / `.physicalAddress`
- `ultimateParentEntity.ueiSAM` / `.legalBusinessName` / `.physicalAddress`
- `evsMonitoring.legalBusinessName` / `.dbaName` / `.outOfBusinessFlag` / `.monitoringStatus` (EVS monitoring changes)

**federalHierarchy:**
- `source` - Hierarchy source
- `hierarchyDepartmentCode` / `hierarchyDepartmentName`
- `hierarchyAgencyCode` / `hierarchyAgencyName`
- `hierarchyOfficeCode`

**entityInformation:**
- `entityURL` - Company website
- `entityDivisionName` / `entityDivisionNumber`
- `entityStartDate` - Business start date
- `fiscalYearEndCloseDate` - Fiscal year end
- `submissionDate` - Last submission

**physicalAddress / mailingAddress:**
- `addressLine1` / `addressLine2`
- `city`
- `stateOrProvinceCode` - 2-character code
- `zipCode` / `zipCodePlus4`
- `countryCode` - 3-character code (USA, CAN, etc.)

**generalInformation:**
- `agencyBusinessPurposeCode` / `agencyBusinessPurposeDesc` - 1=Buyer and Seller, etc.
- `entityStructureCode` / `entityStructureDesc` - 2L=Partnership, 2J=S Corp, 2K=C Corp, etc.
- `organizationStructureCode` / `organizationStructureDesc` - MF=Manufacturer, etc.
- `entityTypeCode` / `entityTypeDesc` - Business or Individual
- `profitStructureCode` / `profitStructureDesc` - 2X=For Profit, A8=Non-Profit, etc.
- `stateOfIncorporationCode` / `countryOfIncorporationCode`

**businessTypes:**
- `businessTypeList[]` - Array of `businessTypeCode` / `businessTypeDesc` pairs
- `sbaBusinessTypeList[]` - SBA-specific certifications

**financialInformation (FOUO/Sensitive only):**
- `creditCardUsage` / `debtSubjectToOffset`

NOTE: NAICS and PSC codes are NOT in coreData. They are in `assertions.goodsAndServices`. See assertions section below.

### assertions Section
- `goodsAndServices.primaryNaics` - 6-digit primary NAICS code (string)
- `goodsAndServices.naicsList[]` - All NAICS codes on record
  - `naicsCode` - 6-digit NAICS
  - `naicsDescription`
  - `sbaSmallBusiness` - "Y" or "N" (NOTE: not "isSmallBusiness")
  - `naicsException` - Exception details if applicable
- `goodsAndServices.pscList[]` - All PSC codes on record
  - `pscCode` - 4-character PSC
  - `pscDescription`
- `disasterReliefData` - Disaster relief registration data
- `ediInformation` - EDI capability

### pointsOfContact Section
- `governmentBusinessPOC` - Primary government business POC
- `alternateGovernmentBusinessPOC`
- `electronicBusinessPOC` - E-business POC
- `alternateElectronicBusinessPOC`
- `pastPerformancePOC`
- `alternatePastPerformancePOC`

Each POC contains: `firstName`, `middleInitial`, `lastName`, `title`, `addressLine1/2`, `city`, `stateOrProvinceCode`, `zipCode`, `countryCode`. FOUO adds: `email`, `phone`, `fax`.

### repsAndCerts Section (must be explicitly requested)
FAR and DFARS certification responses. Extensive; see https://open.gsa.gov/api/entity-api/ for full schema.

Key areas: FAR 52.204-17 (ownership/control), FAR 52.209-2 (FAPIIS proceedings), FAR 52.209-11 (representation by entity that has been debarred), FAR 52.212-3 (offeror representations), FAR 52.219-1 (small business program), FAR 52.222-18 (certification regarding knowledge of child labor), FAR 52.225-2 (Buy American), DFARS 252.204-7016 (covered defense info).

### integrityInformation Section (must be explicitly requested, v3+)
- `proceedingsData` - Proceedings disclosures per FAR 52.209-7/9
- `proceedingsPointsOfContact` - POCs for proceedings information
- Requires `proceedingsData=Yes` filter in conjunction with `includeSections=integrityInformation`

---

## 2. Exclusion Response Schema

### excludedEntity[] Array

**exclusionDetails:**
- `classificationType` - Firm, Individual, Vessel, Special Entity Designation
- `exclusionType` - "Ineligible (Proceedings Completed)", "Ineligible (Proceedings Pending)", "Prohibition/Restriction", "Voluntary Exclusion"
- `exclusionProgram` - Reciprocal, NonProcurement, Procurement
- `excludingAgencyCode` / `excludingAgencyName`

**exclusionIdentification:**
- `ueiSAM` - Excluded entity UEI
- `cageCode`
- `npi` - National Provider Identifier (healthcare)
- `prefix` / `firstName` / `middleName` / `lastName` / `suffix` - Individual name fields
- `entityName` - Firm name (if classification=Firm)

**exclusionActions:**
- `listOfActions[]` - Array of action records
  - `createDate` - When exclusion was created
  - `updateDate` - Last update
  - `activateDate` - When exclusion became active
  - `terminationDate` - When exclusion ends/ended
  - `terminationType` - How termination occurred
  - `recordStatus` - Active or Inactive

**exclusionAddress:**
- `addressLine1/2`, `city`, `stateOrProvinceCode`, `zipCode`, `countryCode`

**exclusionOtherInformation:**
- `additionalComments` - Free text comments
- `ctCode` - Cross-reference type code
- `dnbInvestigationStatus`

**vesselDetails (if classification=Vessel):**
- `callSign`, `type`, `tonnage`, `grt`, `flag`, `owner`

**moreLocations[] (if entity has multiple addresses):**
- Each entry has `exclusionName`, `ueiSAM`, `cageCode`, `primaryAddress`, `secondaryAddress`

---

## 3. Opportunity Response Schema

### opportunitiesData[] Array

- `noticeId` - Unique notice identifier (used for description fetch)
- `title` - Notice title
- `solicitationNumber` - Solicitation number
- `fullParentPathName` - Dept.SubTier.Office hierarchy (dot-delimited)
- `fullParentPathCode` - Numeric hierarchy codes (dot-delimited)
- `department` / `subTier` / `office` - Individual org level names (v1 style, may be null in v2)
- `postedDate` - Date posted (YYYY-MM-DD)
- `type` - Current notice type name (e.g., "Solicitation", "Award Notice")
- `baseType` - Original notice type name
- `archiveType` - auto15, auto30, autocustom, manual
- `archiveDate` - When notice archives
- `typeOfSetAsideDescription` / `typeOfSetAside` - Set-aside info
- `responseDeadLine` - Response deadline (ISO 8601 with timezone)
- `naicsCode` - Primary NAICS (string)
- `naicsCodes[]` - All NAICS codes (array, v2+)
- `classificationCode` - PSC code
- `active` - "Yes" or "No"

**award (if award notice):**
- `date` - Award date
- `number` - Contract number
- `amount` - Award amount (string)
- `awardee.name` / `.ueiSAM`
- `awardee.location` (streetAddress, city, state, zip, country)

**pointOfContact[]:**
- `type` - "primary" or "secondary"
- `fullName` - Contact name
- `title` - Contact title
- `email` - Contact email
- `phone` / `fax`

**description** - URL to full description HTML (NOT inline text). Fetch separately.

**officeAddress:**
- `zipcode`, `city`, `countryCode`, `state`

**placeOfPerformance:**
- `city`, `state`, `country` (each with `code` and `name`)

**uiLink** - Direct SAM.gov UI link to the opportunity

**resourceLinks[]** - Array of download URLs for attached solicitation documents (SOW, RFQ, amendments, etc.)

---

## 4. Contract Awards Response Schema

Top-level response (populated): `totalRecords` (int), `limit` (int), `offset` (int), `awardSummary[]` array.
Top-level response (empty): `awardResponse.totalRecords` (string "0"), `awardResponse.limit` (string), `awardResponse.offset` (string), `message` (string). See Rule 26 in SKILL.md.

### contractId Section
- `subtier.code` / `.name` - Agency subtier (e.g., "9700" / "DEPT OF DEFENSE")
- `piid` - Procurement Instrument Identifier
- `modificationNumber` - Modification number ("0" for base award, "P00001" etc. for mods)
- `transactionNumber` - Transaction number within the modification
- `reasonForModification.code` / `.name` - Modification reason (e.g., "M" / "OTHER ADMINISTRATIVE ACTION")
- `referencedIDVSubtier.code` / `.name` - Referenced IDV agency (for delivery/task orders)
- `referencedIDVPiid` - Referenced IDV PIID (parent contract for orders)
- `referencedIDVModificationNumber` - Referenced IDV mod number

### coreData Section

**solicitationId** - Associated solicitation number
**solicitationDate** - Solicitation issue date
**awardOrIDV** - "AWARD" or "IDV"
**awardOrIDVType.code / .name** - B=Purchase Order, C=Delivery Order, A=BPA Call, D=Definitive Contract, E=BPA, F=Indefinite Delivery Contract, G=Basic Ordering Agreement

**federalOrganization:**
- `contractingInformation.contractingDepartment.code/.name` - Contracting department
- `contractingInformation.contractingSubtier.code/.name` - Contracting sub-agency
- `contractingInformation.contractingOffice.code/.name/.country` - Contracting office
- `fundingInformation.fundingDepartment.code/.name` - Funding department
- `fundingInformation.fundingSubtier.code/.name` - Funding sub-agency
- `fundingInformation.fundingOffice.code/.name` - Funding office
- `fundingInformation.foreignFunding.code/.name` - Foreign funding flag

**acquisitionData:**
- `typeOfContractPricing.code/.name` - J=FFP, K=FP-EPA, L=FP-Incentive, R=Cost Plus Award Fee, S=Cost No Fee, T=Cost Sharing, U=Cost Plus Fixed Fee, V=Cost Plus Incentive Fee, Y=T&M, Z=LH
- `multiyearContract` - YES/NO
- `performanceBasedServiceContract.code/.name`
- `consolidatedContract.code/.name`
- `contractFinancing.code/.name`

**principalPlaceOfPerformance:**
- `city.name`, `county.code/.name`, `state.code/.name`, `zipCode`, `congressionalDistrict`, `country.code/.name`

**productOrServiceInformation:**
- `productOrService.type` - PRODUCT or SERVICE
- `productOrService.code/.name` - PSC code and name
- `principalNaics[].code/.name` - NAICS code(s)
- `contractBundling.code/.name`
- `countryOfOrigin.code/.name`

**competitionInformation:**
- `extentCompeted.code/.name` - A=Full and Open, B=Not Available, C=Not Competed, D=Full and Open After Exclusion, E=Follow On, F=Competed Under SAP, G=Not Competed Under SAP
- `typeOfSetAside.code/.name` - NONE, SBA, 8A, HZC, SDVOSBC, WOSB, etc.
- `solicitationProcedures.code/.name`
- `numberOfOffersSource.code/.name`

### awardDetails Section

**dates:**
- `dateSigned` - Award signature date (ISO 8601: 2026-01-07T00:00:00Z)
- `periodOfPerformanceStartDate` - PoP start
- `currentCompletionDate` - Current PoP end
- `ultimateCompletionDate` - Ultimate PoP end (with all options)
- `fiscalYear` - Fiscal year of action

**dollars:**
- `actionObligation` - Dollars obligated on this specific action/modification
- `baseDollarsObligated` - Base obligation
- `baseAndExercisedOptionsValue` - Base + exercised options
- `baseAndAllOptionsValue` - Base + all options (ceiling)
- `feePaidForUseOfService` - Fee amount

**totalContractDollars:**
- `totalActionObligation` - Cumulative obligation across all mods
- `totalBaseAndExercisedOptionsValue` - Cumulative base + exercised
- `totalBaseAndAllOptionsValue` - Cumulative ceiling

**awardeeData:**
- `awardeeHeader.awardeeName` - Vendor name
- `awardeeHeader.awardeeNameFromContract` - Name as it appears on the contract
- `awardeeUEIInformation.uniqueEntityId` - UEI
- `awardeeUEIInformation.cageCode` - CAGE code
- `awardeeUEIInformation.awardeeUltimateParentUniqueEntityId` - Parent UEI
- `awardeeUEIInformation.awardeeUltimateParentName` - Parent name
- `awardeeLocation` - streetAddress1, city, state.code/.name, zip, country.code/.name, congressionalDistrict, phoneNumber, faxNumber
- `awardeeBusinessTypes` - Nested structure with YES/NO flags: `isUsFederalGovernment`, `usStateGovernment`, `foreignGovernment`, `businessOrOrganization` (corporateEntityNotTaxExempt, soleProprietorship, etc.)
- `socioEconomicData` - YES/NO flags: `veteranOwnedBusiness`, `serviceDisabledVeteranOwnedBusiness`, `minorityOwnedBusiness` (with sub-flags), `womenOwnedBusiness`, `womenOwnedSmallBusiness`, `emergingSmallBusiness`
- `certifications` - `sbaCertified8aProgramParticipant`, `sbaCertifiedHubZoneFirm`, `sbaCertifiedWomenOwnedSmallBusiness`, etc.

**competitionInformation:**
- `numberOfOffersReceived` - Number of offers
- `commercialProductsAndServicesAcquisitionProcedures.code/.name`
- `evaluatedPreference.code/.name`

**preferenceProgramsInformation:**
- `contractingOfficerBusinessSizeDetermination[].code/.name` - S=Small Business, O=Other Than Small

**transactionData:**
- `status.code/.name` - F=FINAL, D=DRAFT
- `version` - Record version
- `createdBy` / `createdDate` - Creator and timestamp
- `lastModifiedBy` / `lastModifiedDate` - Last modifier and timestamp
- `approvedBy` / `approvedDate` - Approver and timestamp
- `closedStatus` - Y/N

---

## 5. Business Type Codes

### Small Business and Ownership Types (businessTypeCode)

All codes below validated against the live API with entity counts.

| Code | Description | Entities |
|---|---|---|
| 23 | Minority-Owned Business | 140,568 |
| 27 | Self Certified Small Disadvantaged Business | 186,767 |
| 2X | For Profit Organization | 521,255 |
| 8W | Women-Owned Small Business | 100,412 |
| A2 | Women-Owned Business | 128,312 |
| A5 | Veteran-Owned Business | 65,509 |
| A8 | Non-Profit Organization | 133,518 |
| LJ | Limited Liability Company | 204,668 |
| MF | Manufacturer of Goods | 58,961 |
| OY | Black American Owned | 79,140 |
| PI | Hispanic American Owned | ~5,000 |
| QF | Service-Disabled Veteran-Owned Business | 43,034 |
| XS | Subchapter S Corporation | 83,038 |

CAUTION: These codes are NOT intuitive. Common mistakes:
- SDVOSB is `QF`, not `XS` (XS = S-Corp)
- Women-Owned is `A2`, not `8W` (8W = WOSB specifically)
- Hispanic-Owned is `PI`, not `QF` (QF = SDVOSB)

### SBA Business Types (sbaBusinessTypeCode)

NOTE: `sbaBusinessTypeCode` uses DIFFERENT codes than `businessTypeCode`. Only codes validated against the live API are listed. For 8(a) filtering, use `sbaBusinessTypeCode=XX` (4,690 results). `businessTypeCode` has no 8(a)-specific code because 8(a) is an SBA certification tracked separately.

| Code | Description | Validated |
|---|---|---|
| XX | 8(a) Certified | Yes (4,690 results) |
| JT | Joint Venture | Yes (797 results) |

Other SBA codes (A4, A7, 12) returned 0 results when tested against `sbaBusinessTypeCode`. Use `businessTypeCode` instead for HUBZone (filter by NAICS with SB size standard) or WOSB (code 8W).

### Entity Structure Codes (entityStructureCode, NOT businessTypeCode)

These use the `entityStructureCode` parameter, not `businessTypeCode`. They describe the legal entity form.

| Code | Description |
|---|---|
| 2J | S Corporation |
| 2K | C Corporation |
| 2L | Partnership or LLP |
| 8H | Corporate Entity (Tax Exempt) |
| CY | City |
| X6 | Federal Government |
| ZZ | Other |

---

## 6. Set-Aside Type Codes (Opportunities)

| Code | Description |
|---|---|
| SBA | Total Small Business Set-Aside |
| SBP | Partial Small Business Set-Aside |
| 8A | 8(a) Competed |
| 8AN | 8(a) Sole Source |
| HZC | HUBZone Set-Aside |
| HZS | HUBZone Sole Source |
| SDVOSBC | SDVOSB Set-Aside |
| SDVOSBS | SDVOSB Sole Source |
| WOSB | WOSB Set-Aside |
| WOSBSS | WOSB Sole Source |
| EDWOSB | EDWOSB Set-Aside |
| EDWOSBSS | EDWOSB Sole Source |
| VSA | Veteran Set-Aside |
| VSS | Veteran Sole Source |

---

## 7. Composite Workflows

### Vendor Responsibility Check (Entity + Exclusion)

Supports FAR 9.104-1 responsibility determination. Two API calls: entity registration status and exclusion check on the same UEI.

```python
import time

def vendor_responsibility_check(uei):
    """Check vendor registration status and exclusion status for responsibility determination."""
    result = {"uei": uei, "registration": None, "exclusion": None, "flags": []}

    # Step 1: Entity registration check
    entity = safe_sam(sam_entity, {
        "ueiSAM": uei,
        "includeSections": "entityRegistration,coreData"
    })

    if "error" in entity:
        result["registration"] = {"error": entity}
        result["flags"].append("ENTITY_LOOKUP_FAILED")
    elif entity.get("totalRecords", 0) == 0:
        result["registration"] = None
        result["flags"].append("NOT_REGISTERED")
    else:
        e = entity["entityData"][0]
        reg = e.get("entityRegistration", {})
        result["registration"] = {
            "legalBusinessName": reg.get("legalBusinessName"),
            "status": reg.get("registrationStatus"),
            "activationDate": reg.get("activationDate"),
            "expirationDate": reg.get("registrationExpirationDate"),
            "cageCode": reg.get("cageCode"),
            "exclusionStatusFlag": reg.get("exclusionStatusFlag"),
            "publicDisplayFlag": reg.get("publicDisplayFlag")
        }
        if reg.get("registrationStatus") != "Active":
            result["flags"].append("REGISTRATION_NOT_ACTIVE")
        if reg.get("exclusionStatusFlag") == "Y":
            result["flags"].append("EXCLUSION_FLAG_ON_ENTITY")

        # Extract business types from coreData
        core = e.get("coreData", {})
        biz_types = core.get("businessTypes", {})
        result["registration"]["businessTypes"] = [
            bt.get("businessTypeDesc") for bt in biz_types.get("businessTypeList", [])
        ]
        result["registration"]["sbaTypes"] = [
            st.get("sbaBusinessTypeDesc") for st in biz_types.get("sbaBusinessTypeList", [])
        ]

    time.sleep(0.5)

    # Step 2: Exclusion check
    exclusion = safe_sam(sam_exclusions, {"ueiSAM": uei})

    if "error" in exclusion:
        result["exclusion"] = {"error": exclusion}
        result["flags"].append("EXCLUSION_LOOKUP_FAILED")
    elif exclusion.get("totalRecords", 0) == 0:
        result["exclusion"] = {"totalRecords": 0, "status": "NO_ACTIVE_EXCLUSIONS"}
    else:
        records = exclusion.get("excludedEntity", [])
        active = [r for r in records if any(
            a.get("recordStatus") == "Active"
            for a in r.get("exclusionActions", {}).get("listOfActions", [])
        )]
        result["exclusion"] = {
            "totalRecords": exclusion["totalRecords"],
            "activeRecords": len(active),
            "details": [{
                "type": r.get("exclusionDetails", {}).get("exclusionType"),
                "program": r.get("exclusionDetails", {}).get("exclusionProgram"),
                "agency": r.get("exclusionDetails", {}).get("excludingAgencyName")
            } for r in active]
        }
        if active:
            result["flags"].append("ACTIVE_EXCLUSION_FOUND")

    return result
```

**Interpreting the flags:**
- No flags = clear for responsibility (registration active, no exclusions)
- `NOT_REGISTERED` = entity has no SAM registration; cannot receive awards per FAR 4.1102
- `REGISTRATION_NOT_ACTIVE` = registration expired or inactive; vendor must renew before award
- `EXCLUSION_FLAG_ON_ENTITY` = entity record itself indicates an exclusion exists
- `ACTIVE_EXCLUSION_FOUND` = confirmed active debarment/suspension; FAR 9.405 prohibits award

### Market Research Entity Scan

Find vendors by NAICS, geography, and small business type for market research supporting set-aside decisions.

```python
def market_research_entity_scan(naics, state=None, business_type=None, limit_pages=3):
    """Scan SAM entities for market research. Returns up to limit_pages * 10 entities."""
    params = {
        "primaryNaics": naics,
        "registrationStatus": "A",
        "purposeOfRegistrationCode": "Z2",
        "includeSections": "entityRegistration,coreData"
    }
    if state:
        params["physicalAddressProvinceOrStateCode"] = state
    if business_type:
        params["businessTypeCode"] = business_type

    all_entities = []
    for page in range(limit_pages):
        params["page"] = str(page)
        result = safe_sam(sam_entity, params)
        if "error" in result or result.get("totalRecords", 0) == 0:
            break
        all_entities.extend(result.get("entityData", []))
        if len(result.get("entityData", [])) < 10:
            break
        time.sleep(0.5)

    return {
        "totalFound": result.get("totalRecords", 0) if "error" not in result else 0,
        "retrieved": len(all_entities),
        "entities": [{
            "ueiSAM": e.get("entityRegistration", {}).get("ueiSAM"),
            "legalBusinessName": e.get("entityRegistration", {}).get("legalBusinessName"),
            "cageCode": e.get("entityRegistration", {}).get("cageCode"),
            "status": e.get("entityRegistration", {}).get("registrationStatus"),
            "businessTypes": [
                bt.get("businessTypeDesc")
                for bt in e.get("coreData", {}).get("businessTypes", {}).get("businessTypeList", [])
            ],
            "city": e.get("coreData", {}).get("physicalAddress", {}).get("city"),
            "state": e.get("coreData", {}).get("physicalAddress", {}).get("stateOrProvinceCode")
        } for e in all_entities]
    }
```

### Opportunity Monitoring

Search for new opportunities by NAICS, set-aside, or agency within a recent window.

```python
from datetime import datetime, timedelta

def check_recent_opportunities(naics=None, set_aside=None, agency_keyword=None, days_back=7):
    """Check for opportunities posted in the last N days.
    NOTE: deptname/subtier API params are silently ignored.
    Use agency_keyword to post-filter by fullParentPathName instead."""
    today = datetime.now()
    from_date = (today - timedelta(days=days_back)).strftime("%m/%d/%Y")
    to_date = today.strftime("%m/%d/%Y")

    params = {
        "postedFrom": from_date,
        "postedTo": to_date,
        "limit": "100"
    }
    # Search solicitations and combined synopsis/solicitations
    params["ptype"] = "o"

    if naics:
        params["ncode"] = naics
    if set_aside:
        params["typeOfSetAside"] = set_aside
    # Do NOT use deptname or subtier - they are silently ignored by the API

    result = safe_sam(sam_opportunities, params)
    if "error" in result:
        return result

    # Also check combined synopsis/solicitations
    params2 = dict(params)
    params2["ptype"] = "k"
    time.sleep(0.5)
    result2 = safe_sam(sam_opportunities, params2)

    opps = result.get("opportunitiesData", []) + result2.get("opportunitiesData", [])

    # Post-filter by agency if requested (API filter is broken)
    if agency_keyword:
        agency_upper = agency_keyword.upper()
        opps = [o for o in opps if agency_upper in o.get("fullParentPathName", "").upper()]

    return {
        "totalFound": len(opps),
        "opportunities": [{
            "title": o.get("title"),
            "solicitationNumber": o.get("solicitationNumber"),
            "agency": o.get("fullParentPathName"),
            "postedDate": o.get("postedDate"),
            "responseDeadLine": o.get("responseDeadLine"),
            "setAside": o.get("typeOfSetAsideDescription"),
            "naics": o.get("naicsCode"),
            "psc": o.get("classificationCode"),
            "type": o.get("type"),
            "uiLink": o.get("uiLink"),
            "pocEmail": o.get("pointOfContact", [{}])[0].get("email") if o.get("pointOfContact") else None
        } for o in opps]
    }
```

### Contract Award History Lookup

Retrieve all modifications for a PIID to build a complete award history timeline.

```python
def contract_award_history(piid, include_deleted=False):
    """Pull all modifications for a PIID. Returns chronological award history."""
    params = {"piid": piid, "limit": "100", "offset": "0"}
    if include_deleted:
        params["deletedStatus"] = "Y"

    all_records = []
    while True:
        result = safe_sam(sam_awards, params)
        if "error" in result:
            return result
        records = result.get("awardSummary", [])
        all_records.extend(records)
        total = result.get("totalRecords", 0)
        if len(all_records) >= total or not records:
            break
        params["offset"] = str(len(all_records))
        time.sleep(0.5)

    # Sort by mod number then transaction number
    all_records.sort(key=lambda r: (
        r.get("contractId", {}).get("modificationNumber", ""),
        r.get("contractId", {}).get("transactionNumber", "0")
    ))

    return {
        "piid": piid,
        "totalRecords": len(all_records),
        "history": [{
            "modificationNumber": r.get("contractId", {}).get("modificationNumber"),
            "transactionNumber": r.get("contractId", {}).get("transactionNumber"),
            "reasonForModification": r.get("contractId", {}).get("reasonForModification", {}).get("name"),
            "dateSigned": r.get("awardDetails", {}).get("dates", {}).get("dateSigned"),
            "actionObligation": r.get("awardDetails", {}).get("dollars", {}).get("actionObligation"),
            "totalObligation": r.get("awardDetails", {}).get("totalContractDollars", {}).get("totalActionObligation"),
            "vendor": r.get("awardDetails", {}).get("awardeeData", {}).get("awardeeHeader", {}).get("awardeeName"),
            "status": r.get("awardDetails", {}).get("transactionData", {}).get("status", {}).get("name")
        } for r in all_records]
    }
```

### Vendor Award Profile

Combines Entity Management (registration/business types) with Contract Awards (recent awards) to build a vendor intelligence profile. Three API calls.

```python
def vendor_award_profile(uei, fiscal_year=None):
    """Build vendor profile combining entity registration + recent contract awards."""
    result = {"uei": uei, "entity": None, "awards": None, "flags": []}

    # Step 1: Entity registration
    entity = safe_sam(sam_entity, {
        "ueiSAM": uei,
        "includeSections": "entityRegistration,coreData,assertions"
    })
    if "error" in entity:
        result["entity"] = {"error": entity}
        result["flags"].append("ENTITY_LOOKUP_FAILED")
    elif entity.get("totalRecords", 0) == 0:
        result["flags"].append("NOT_REGISTERED")
    else:
        e = entity["entityData"][0]
        reg = e.get("entityRegistration", {})
        core = e.get("coreData", {})
        result["entity"] = {
            "legalBusinessName": reg.get("legalBusinessName"),
            "status": reg.get("registrationStatus"),
            "cageCode": reg.get("cageCode"),
            "expirationDate": reg.get("registrationExpirationDate"),
            "businessTypes": [bt.get("businessTypeDesc") for bt in core.get("businessTypes", {}).get("businessTypeList", [])],
            "primaryNaics": e.get("assertions", {}).get("goodsAndServices", {}).get("primaryNaics")
        }

    time.sleep(0.5)

    # Step 2: Recent contract awards
    award_params = {"awardeeUniqueEntityId": uei, "limit": "100"}
    if fiscal_year:
        award_params["fiscalYear"] = str(fiscal_year)

    awards = safe_sam(sam_awards, award_params)
    if "error" in awards:
        result["awards"] = {"error": awards}
        result["flags"].append("AWARDS_LOOKUP_FAILED")
    else:
        records = awards.get("awardSummary", [])
        total_obligation = sum(
            float(r.get("awardDetails", {}).get("dollars", {}).get("actionObligation") or 0)
            for r in records
        )
        result["awards"] = {
            "totalRecords": awards.get("totalRecords", 0),
            "recordsReturned": len(records),
            "totalActionObligation": total_obligation,
            "agencies": list(set(
                r.get("coreData", {}).get("federalOrganization", {}).get("contractingInformation", {}).get("contractingDepartment", {}).get("name", "UNKNOWN")
                for r in records
            )),
            "recentAwards": [{
                "piid": r.get("contractId", {}).get("piid"),
                "dateSigned": r.get("awardDetails", {}).get("dates", {}).get("dateSigned"),
                "obligation": r.get("awardDetails", {}).get("dollars", {}).get("actionObligation"),
                "department": r.get("coreData", {}).get("federalOrganization", {}).get("contractingInformation", {}).get("contractingDepartment", {}).get("name"),
                "naics": (r.get("coreData", {}).get("productOrServiceInformation", {}).get("principalNaics") or [{}])[0].get("code")
            } for r in records[:10]]
        }

    return result
```

---

## 8. PSC Category Hierarchy

The PSC system organizes into two levels of categories above the 4-character PSC code:

| Level 1 Code | Level 1 Name |
|---|---|
| 1 | IT & Telecom |
| 2 | Professional Services |
| 3 | Support Services (non-IT) |
| 4 | Construction & Building |
| 5 | Industrial Products/Services |
| 6 | Logistics |
| 7 | Office Management |
| 8 | Medical |
| 9 | Travel |
| 10 | Facilities |
| 11 | Aircraft, Ships & Land Vehicles |
| 12 | Weapons & Ammunition |
| 13 | Social |
| 14 | Sustainment S&E |

Common PSC codes for professional services acquisitions:

| Code | Description |
|---|---|
| R425 | Engineering/Technical Support |
| R499 | Other Professional Services |
| R408 | Program Management/Support |
| R706 | Management Support Services |
| R707 | Contract/Procurement/Acquisition Support |
| D302 | IT Systems Development |
| D306 | IT Systems Analysis |
| D307 | Automated Information Systems Design |
| D399 | Other IT and Telecom |
| DA01 | Application Development Support |

---

## 9. Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `deptname`/`subtier` not filtering | These params are SILENTLY IGNORED by the API; any value including garbage returns unfiltered results | Do NOT use. Post-filter in Python by checking `fullParentPathName` in results. Working filters: title, solnum, noticeid, ptype, ncode, ccode, typeOfSetAside, state, zip, rdlfrom/rdlto |
| Opportunity description returns 404 | Description URL requires API key appended | Append `&api_key=KEY` to the description URL; use `get_opp_description()` helper |
| Entity name search returns too many results | `legalBusinessName` does partial matching with no relevance ranking; JVs and subsidiaries appear before the parent | Use UEI or CAGE for exact lookup; if using name, scan results for best match on exact name |
| Opportunity date range > 364 days | `postedFrom` to `postedTo` must span less than 1 calendar year | Use a 6-month window; for older notices, chain multiple calls with sequential date ranges |
| NAICS/PSC missing from entity response | NAICS/PSC are in `assertions.goodsAndServices`, NOT in `coreData` | Include `assertions` in `includeSections`; use `includeSections=entityRegistration,coreData,assertions` |
| Entity response missing name/UEI/status | Requested only `coreData` or `pointsOfContact` without `entityRegistration` | Always include `entityRegistration` alongside any other section |
| POC email/phone fields missing | Public API key only returns name/address; email/phone require FOUO | Need Federal System Account with "Read FOUO" permission for POC contact details |
| Multi-ptype returns same as single ptype | `sam_opportunities()` helper only passes one ptype value | Make separate calls per ptype and merge, or build URL manually with `&ptype=o&ptype=r` |
| Entity name with `&` returns 0 | API strips `&` from search terms; `JOHNSON & JOHNSON` becomes broken two-word query | Search by UEI/CAGE, or use single word from name (e.g., `legalBusinessName=JOHNSON`) and scan results |
| Entity `size` > 10 returns 400 | Hard cap: "Size Cannot Exceed 10 Records" | Cannot override; paginate with `page` param (10 per page) or use extract mode for bulk |
| Opportunity `limit=1000` works fine | Opportunities has no practical limit cap (unlike Entity's hard 10) | Use large limit values for Opportunities to minimize calls |
| 401 API_KEY_INVALID (HTML response) | Expired or invalid API key; response is HTML, not JSON | Regenerate key at sam.gov/profile/details; `safe_sam()` wrapper handles HTML responses gracefully |
| Entity country code returns 0 | Used 2-char code (US, CA) instead of 3-char (USA, CAN) | All SAM.gov country codes are ISO alpha-3 (3 chars). Same applies to Exclusion `country` param |
| exclusionURL returns 404 | URL contains literal `REPLACE_WITH_API_KEY` placeholder | String-replace the placeholder with your actual API key before fetching |
| Opportunity ccode=R4 returns 0 | PSC filter requires exact 4-char code, no prefix matching | Use full PSC code (R425 not R4); look up child codes via PSC API if needed |
| businessTypeCode=XS returns S-Corps, not SDVOSB | XS = Subchapter S Corporation; SDVOSB = QF | See validated business type code table. QF = SDVOSB, A2 = Women-Owned, PI = Hispanic-Owned |
| Typo'd param on Opportunities returns unfiltered results | Opportunities API silently ignores unknown params (same as deptname/subtier); Entity and Exclusions properly reject with 400 | Double-check Opportunities param names; a typo gives full unfiltered results with no error |
| Entity name with `()` returns 0 | Parentheses in `legalBusinessName` break matching | Strip parenthetical portions and search main name only (e.g., `GENERAL DYNAMICS` not `GENERAL DYNAMICS (GD)`) |
| 406 Not Acceptable | Accept: application/json header on Exclusions endpoint | Do NOT set Accept header on any SAM.gov request; omit it entirely and let it default |
| 400 `INVALID_SEARCH_PARAMETER` | Unknown parameter name (e.g., `limit` on Exclusions) | Check parameter names against API docs; Exclusions uses `size` not `limit` |
| 400 bad request on dates | Wrong date format | Use `MM/DD/YYYY`, not ISO 8601 |
| 403 Forbidden | Invalid or expired API key | Check key format (`SAM-xxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`), regenerate if expired (keys expire every 90 days) |
| 429 rate limit | Exceeded daily request limit | Wait until next day; switch to system account for 10K/day |
| Empty `entityData` | Entity not registered or wrong `samRegistered` flag | Try `samRegistered=No` for ID-assigned entities; verify UEI spelling |
| Opportunities returns 0 | Missing `postedFrom`/`postedTo` or too-narrow date range | Both date params are mandatory; widen the range |
| Opportunity `description` is a URL | Normal behavior; description is not inline | Fetch the URL separately with `get_opp_description()` |
| `totalRecords` > 0 but no data | Pagination issue | Check `page` and `size`/`offset` params |
| `repsAndCerts` missing from response | Not included in `All` | Explicitly request `includeSections=repsAndCerts` |
| `integrityInformation` missing | Must be explicitly requested with filter | Use `includeSections=integrityInformation` AND `proceedingsData=Yes` |
| Exclusion `recordStatus` confusion | Exclusion can be Active or Inactive | Active = currently excluded; Inactive = exclusion expired or lifted |
| Entity with `exclusionStatusFlag=Y` but no exclusion found | Flag can lag or reference historical exclusion | Check exclusion by UEI to confirm current status |
| `publicDisplayFlag=N` | Entity opted out of public display | Public API key only returns limited data for opted-out entities; need FOUO system account for full data |
| Forbidden characters in query | Params contain `& \| { } ^ \` | URL-encode or remove these characters |
| PSC endpoint 404 | Wrong base path | PSC is at `/prod/locationservices/v1/api/publicpscdetails`, not under `/entity-information/` |
| Contract Awards `vendorName` rejected | Parameter does not exist | Use `awardeeLegalBusinessName` for vendor name search, `awardeeUniqueEntityId` for UEI, `awardeeCageCode` for CAGE |
| Contract Awards returns empty `awardResponse` wrapper | 0 results for query; response uses different wrapper than populated results | Check for both `awardSummary` (populated) and `awardResponse` (empty) keys; see Rule 26 |
| Contract Awards error is plain text, not JSON | API returns plain text for param errors, HTML for auth errors | Parse content-type or catch JSON decode errors; do not assume JSON error bodies |
| Contract Awards `limit=101` rejected | Max limit is 100 | Validate client-side; use offset-based pagination for more records |
| Contract Awards `page` param returns empty | `page` is not a valid param; API silently returns nothing | Use `offset` for pagination, not `page`. offset=0 + limit=10 for first page, offset=10 for second page |
| Contract Awards date format rejected | Used ISO 8601 (2026-01-01) | Use MM/dd/yyyy format: `01/01/2026` or bracket range `[01/01/2026,03/01/2026]` |
| Contract Awards deleted records missing | Deleted records excluded by default | Pass `deletedStatus=Y` to include deleted records; there is no separate endpoint |
| Contract Awards `totalRecords` type varies | Integer when populated, string when empty (in `awardResponse` wrapper) | Always convert to int: `int(d.get("totalRecords", 0))` |
| FPDS.gov is gone | FPDS ezSearch decommissioned Feb 24, 2026; ATOM feed retires July 31, 2026 | Use the SAM.gov Contract Awards API at `/contract-awards/v1/search` as the replacement |
