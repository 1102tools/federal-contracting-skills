---
name: usaspending-api
description: Query the USASpending.gov REST API for federal contract and award data. Use this skill whenever the user asks about federal spending data, contract descriptions, award details, transaction histories, vendor/recipient lookups, obligation amounts, agency spending breakdowns, PSC/NAICS code lookups, or any procurement research that involves USASpending, FPDS, or FAADC data. Also trigger when the user needs to enrich internal contract data (PRISM, FPDS, ALP, Contract Court) with base award descriptions, transaction histories, or award metadata. Trigger for any mention of USASpending, FPDS, federal awards, PIIDs, FAINs, award IDs, or federal spending analysis. This skill is essential for daily contract portfolio management, description enrichment, and procurement intelligence work.
---

# USASpending.gov API Skill

## Overview

The USASpending.gov API (https://api.usaspending.gov) provides free, no-auth access to all federal spending data sourced from FPDS, FAADC, and agency DATA Act submissions. No API key required.

Base URL: `https://api.usaspending.gov`

This skill is self-contained. Filter value tables, composite workflows, bulk download, loan search, reference endpoints, PSC codes, vendor dedup notes, and troubleshooting are all below in the same file.

## Critical Rules

### 1. Award Type Code Groups (MOST COMMON ERROR)
Award type codes MUST NOT be mixed across groups. The API returns HTTP 422 if you mix them.

- **Contracts:** `["A", "B", "C", "D"]` (A=BPA Call, B=PO, C=DO, D=Definitive)
- **IDVs:** `["IDV_A", "IDV_B", "IDV_B_A", "IDV_B_B", "IDV_B_C", "IDV_C", "IDV_D", "IDV_E"]`
- **Grants:** `["02", "03", "04", "05"]`
- **Loans:** `["07", "08"]` (use "Loan Value" not "Award Amount")
- **Direct Payments:** `["06", "10"]`
- **Other:** `["09", "11", "-1"]`

### 2. PIID Type Detection
PIID formats vary by agency. Example: NAVSEA PIIDs start with N00024 (e.g., N0002425CXXXX). Other DoD prefixes: W91CRB (Army Contracting Command), FA8650 (AFRL). If the agency format is unknown, try contracts first, then IDVs if 0 results.

```python
def detect_award_type(piid):
    if piid.startswith("N00024") and len(piid) >= 9:
        return ["IDV_B_B"] if piid[8] == 'D' else ["A", "B", "C", "D"]
    return ["A", "B", "C", "D"]
```

### 3. Sort Field Must Be in Fields Array
The `sort` value MUST appear in `fields` or the API returns HTTP 400.

### 4. Endpoint Limit Caps
- Search endpoints: max `limit` = 100
- Transactions endpoint: max `limit` = 5000
- Paginate with `page` parameter

### 5. No Total Count in page_metadata
`spending_by_award` has no `total` field. Use `spending_by_award_count` for totals.

### 6. PSC/NAICS Filter Format
Do NOT include empty `"exclude": []`. Use simple arrays: `"psc_codes": ["R499"]`.

### 7. Contract Award Type vs Pricing Type
`Contract Award Type` in search = how issued (BPA Call, PO, DO, Definitive). For pricing type (FFP, T&M, etc.), use award detail endpoint or `contract_pricing_type_codes` filter.

### 8. Generated Award IDs
Format: `CONT_AWD_[PIID]_[AGENCY]_[PARENT_PIID]_[PARENT_AGENCY]` or `CONT_IDV_[PIID]_[AGENCY]`.

---

## Core Endpoints

### 1. Award Search (PRIMARY WORKHORSE)

**Endpoint:** `POST /api/v2/search/spending_by_award/`

```python
import urllib.request, json

def search_awards(keywords, award_type_codes, fields=None, limit=10, page=1, sort="Award Amount"):
    if fields is None:
        fields = ["Award ID", "Description", "Award Amount", "Recipient Name",
                  "Start Date", "End Date", "generated_internal_id",
                  "Awarding Agency", "Awarding Sub Agency", "PSC", "NAICS"]
    if sort not in fields:
        fields.append(sort)
    payload = json.dumps({
        "subawards": False, "limit": min(limit, 100), "page": page,
        "sort": sort, "order": "desc",
        "filters": {
            "keywords": keywords if isinstance(keywords, list) else [keywords],
            "award_type_codes": award_type_codes
        }, "fields": fields
    })
    url = "https://api.usaspending.gov/api/v2/search/spending_by_award/"
    req = urllib.request.Request(url, data=payload.encode('utf-8'),
                                 headers={'Content-Type': 'application/json'})
    with urllib.request.urlopen(req, timeout=15) as resp:
        return json.loads(resp.read().decode())
```

**Contract fields:** Award ID, Recipient Name, Recipient UEI, Description, Awarding Agency, Awarding Sub Agency, Funding Agency, Funding Sub Agency, generated_internal_id, Start Date, End Date, Award Amount, Total Outlays, Contract Award Type, NAICS, PSC, def_codes, Last Modified Date, Base Obligation Date

**IDV fields:** Same base plus Last Date to Order (no End Date).

**Loan fields:** Use "Loan Value" and "Subsidy Cost" instead of "Award Amount". Sort by "Loan Value".

**Note on `award_ids` filter:** Use plain strings: `"award_ids": ["N0002424C0085"]`. Performs prefix match.

### 2. Award Detail

**Endpoint:** `GET /api/v2/awards/<GENERATED_AWARD_ID>/`

```python
def get_award_detail(generated_id):
    url = f"https://api.usaspending.gov/api/v2/awards/{generated_id}/"
    req = urllib.request.Request(url)
    with urllib.request.urlopen(req, timeout=15) as resp:
        return json.loads(resp.read().decode())
```

Returns: piid, description, total_obligation, date_signed, recipient, parent_award, latest_transaction_contract_data (competition, set-aside, pricing type), period_of_performance, place_of_performance, naics_hierarchy, psc_hierarchy, base_and_all_options, total_subaward_amount.

### 3. Transaction History

**Endpoint:** `POST /api/v2/transactions/`

```python
def get_transactions(generated_award_id, limit=100, sort="action_date", order="asc"):
    payload = json.dumps({
        "award_id": generated_award_id, "page": 1,
        "sort": sort, "order": order, "limit": min(limit, 5000)
    })
    url = "https://api.usaspending.gov/api/v2/transactions/"
    req = urllib.request.Request(url, data=payload.encode('utf-8'),
                                 headers={'Content-Type': 'application/json'})
    with urllib.request.urlopen(req, timeout=15) as resp:
        return json.loads(resp.read().decode())
```

Returns per transaction: id, type, action_date, action_type, modification_number, description, federal_action_obligation. Mod 0 = original base award description.

### 4. Agency-Filtered Search

```python
filters = {
    "award_type_codes": ["A", "B", "C", "D"],
    "agencies": [{"type": "awarding", "tier": "subtier",
                  "name": "Naval Sea Systems Command",
                  "toptier_name": "Department of Defense"}],
    "time_period": [{"start_date": "2024-10-01", "end_date": "2025-09-30"}]
}
```

DoD constants: toptier code 097 (Defense). Navy subtier 1700. Example NAVSEA PIID format N00024[YY][type][NNNN].

### 5. IDV Child Awards

**Endpoint:** `POST /api/v2/idvs/awards/`

```python
def get_idv_children(generated_idv_id, award_type="child_awards"):
    payload = json.dumps({
        "award_id": generated_idv_id, "type": award_type,
        "limit": 50, "page": 1,
        "sort": "period_of_performance_start_date", "order": "desc"
    })
    url = "https://api.usaspending.gov/api/v2/idvs/awards/"
    req = urllib.request.Request(url, data=payload.encode('utf-8'),
                                 headers={'Content-Type': 'application/json'})
    with urllib.request.urlopen(req, timeout=15) as resp:
        return json.loads(resp.read().decode())
```

**Field name differences:** IDV children use `piid` (not "Award ID"), `obligated_amount` (not "Award Amount"), `generated_unique_award_id` (not "generated_internal_id").

### 6. Spending by Category

**Endpoint:** `POST /api/v2/search/spending_by_category/<CATEGORY>/`

Category MUST be in URL path. Categories: awarding_agency, awarding_subagency, funding_agency, funding_subagency, recipient, cfda, naics, psc, country, county, district, state_territory, federal_account, defc.

```python
def spending_by_category(category, filters, limit=10):
    payload = json.dumps({"filters": filters, "limit": limit, "page": 1})
    url = f"https://api.usaspending.gov/api/v2/search/spending_by_category/{category}/"
    req = urllib.request.Request(url, data=payload.encode('utf-8'),
                                 headers={'Content-Type': 'application/json'})
    with urllib.request.urlopen(req, timeout=15) as resp:
        return json.loads(resp.read().decode())
```

Recipient results: `amount` (net obligations, can be negative), `name` (ALL CAPS), `recipient_id`, `uei`. See Vendor Deduplication section below for caveats on large-company multi-UEI entries.

### 7. Spending Over Time

**Endpoint:** `POST /api/v2/search/spending_over_time/`

```python
def spending_over_time(filters, group="fiscal_year"):
    payload = json.dumps({"group": group, "filters": filters})
    url = "https://api.usaspending.gov/api/v2/search/spending_over_time/"
    req = urllib.request.Request(url, data=payload.encode('utf-8'),
                                 headers={'Content-Type': 'application/json'})
    with urllib.request.urlopen(req, timeout=15) as resp:
        return json.loads(resp.read().decode())
```

**CRITICAL:** `fiscal_year` is returned as a **string**. Cast to int before numeric comparison.

### 8. Award Count

**Endpoint:** `POST /api/v2/search/spending_by_award_count/`

```python
def get_award_counts(filters):
    payload = json.dumps({"filters": filters})
    url = "https://api.usaspending.gov/api/v2/search/spending_by_award_count/"
    req = urllib.request.Request(url, data=payload.encode('utf-8'),
                                 headers={'Content-Type': 'application/json'})
    with urllib.request.urlopen(req, timeout=15) as resp:
        return json.loads(resp.read().decode())
```

Returns counts by group: `contracts`, `idvs`, `grants`, `loans`, `direct_payments`, `other`. Use with dimensional filters (contract_pricing_type_codes, extent_competed_type_codes, set_aside_type_codes) for aggregate analysis.

### 9. Autocomplete Endpoints

```python
def autocomplete_psc(search_text):
    payload = json.dumps({"search_text": search_text, "limit": 10})
    url = "https://api.usaspending.gov/api/v2/autocomplete/psc/"
    req = urllib.request.Request(url, data=payload.encode('utf-8'),
                                 headers={'Content-Type': 'application/json'})
    with urllib.request.urlopen(req, timeout=10) as resp:
        return json.loads(resp.read().decode())

def autocomplete_naics(search_text):
    payload = json.dumps({"search_text": search_text, "limit": 10})
    url = "https://api.usaspending.gov/api/v2/autocomplete/naics/"
    req = urllib.request.Request(url, data=payload.encode('utf-8'),
                                 headers={'Content-Type': 'application/json'})
    with urllib.request.urlopen(req, timeout=10) as resp:
        return json.loads(resp.read().decode())
```

PSC autocomplete works best with code prefixes ("R499") or single keywords ("professional").

---

## Common Filter Object

```python
filters = {
    "award_type_codes": ["A","B","C","D"],  # MUST be from same group
    "keywords": ["search term"],
    "time_period": [{"start_date": "2024-10-01", "end_date": "2025-09-30"}],
    "agencies": [{"type": "awarding", "tier": "subtier", "name": "..."}],
    "recipient_search_text": ["vendor name"],
    "award_ids": ["PIID1"],  # Plain strings, prefix match
    "psc_codes": ["R499"],  # Simple array, no empty exclude
    "naics_codes": ["541512"],
    "place_of_performance_locations": [{"country": "USA", "state": "MD"}],
    "award_amounts": [{"lower_bound": 1000000, "upper_bound": 10000000}],
    "set_aside_type_codes": ["SBA"],
    "extent_competed_type_codes": ["A"],
    "contract_pricing_type_codes": ["J"],
    "def_codes": ["L","M","N","O","P"]
}
```

---

## Filter Value Reference Tables

### contract_pricing_type_codes

| Code | Description |
|------|-------------|
| A | Fixed Price Redetermination |
| B | Fixed Price Level of Effort |
| J | Firm Fixed Price |
| K | Fixed Price with Economic Price Adjustment |
| L | Fixed Price Incentive |
| M | Fixed Price Award Fee |
| R | Cost Plus Award Fee |
| S | Cost No Fee |
| T | Cost Sharing |
| U | Cost Plus Fixed Fee |
| V | Cost Plus Incentive Fee |
| Y | Time and Materials |
| Z | Labor Hours |

### extent_competed_type_codes

| Code | Description |
|------|-------------|
| A | Full and Open Competition |
| B | Not Available for Competition |
| C | Not Competed |
| D | Full and Open after Exclusion of Sources |
| E | Follow On to Competed Action |
| F | Competed under SAP |
| G | Not Competed under SAP |
| CDO | Competitive Delivery Order |
| NDO | Non-Competitive Delivery Order |

Useful groupings: competed = `["A","D","F","CDO"]`, not competed = `["B","C","E","G","NDO"]`.

### set_aside_type_codes

| Code | Description |
|------|-------------|
| SBA | Small Business Set-Aside (Total) |
| SBP | Small Business Set-Aside (Partial) |
| 8A | 8(a) Competitive |
| 8AN | 8(a) Sole Source |
| HZC | HUBZone Set-Aside |
| HZS | HUBZone Sole Source |
| SDVOSBS | SDVOSB Set-Aside |
| SDVOSBC | SDVOSB Sole Source |
| WOSB | WOSB Set-Aside |
| WOSBSS | WOSB Sole Source |
| EDWOSB | EDWOSB Set-Aside |
| EDWOSBSS | EDWOSB Sole Source |
| VSA | Veteran Set-Aside |

Useful groupings: all SB = `["SBA","SBP","8A","8AN","HZC","HZS","SDVOSBS","SDVOSBC","WOSB","WOSBSS","EDWOSB","EDWOSBSS","VSA"]`, 8(a) = `["8A","8AN"]`, SDVOSB = `["SDVOSBS","SDVOSBC"]`, WOSB/EDWOSB = `["WOSB","WOSBSS","EDWOSB","EDWOSBSS"]`.

---

## Composite Workflows

### Enrich Contract Court / PRISM Descriptions

Problem: PRISM overwrites the description field with the latest modification text.

```python
def get_base_description(piid):
    """Get the original base award description for a PIID."""
    award_types = detect_award_type(piid)
    result = search_awards(piid, award_types,
                           fields=["Award ID", "Description", "Award Amount",
                                   "Recipient Name", "generated_internal_id"])
    awards = result.get('results', [])
    if not awards and 'IDV' not in str(award_types):
        award_types = ["IDV_A","IDV_B","IDV_B_A","IDV_B_B","IDV_B_C","IDV_C","IDV_D","IDV_E"]
        result = search_awards(piid, award_types,
                               fields=["Award ID", "Description", "Award Amount",
                                        "Recipient Name", "generated_internal_id"])
        awards = result.get('results', [])
    return awards[0] if awards else None
```

### Batch PIID Lookup

```python
import time

def batch_lookup_descriptions(piids):
    """Look up base descriptions for a list of PIIDs."""
    results = {}
    for piid in piids:
        try:
            award = get_base_description(piid)
            if award:
                results[piid] = {'description': award.get('Description'),
                    'amount': award.get('Award Amount'),
                    'recipient': award.get('Recipient Name'),
                    'internal_id': award.get('generated_internal_id')}
            else:
                results[piid] = None
        except Exception as e:
            results[piid] = {'error': str(e)}
        time.sleep(0.3)
    return results
```

### Get Full Award Lifecycle

```python
def get_award_lifecycle(piid):
    """Get complete award info: base description + all modifications."""
    award = get_base_description(piid)
    if not award:
        return None
    internal_id = award.get('generated_internal_id')
    detail = get_award_detail(internal_id)
    transactions = get_transactions(internal_id, limit=100)
    return {'award': detail, 'transactions': transactions.get('results', [])}
```

### Dimensional Analysis with Award Counts

```python
# Count FFP awards
ffp_filters = {**base_filters, "contract_pricing_type_codes": ["J"]}
ffp_count = get_award_counts(ffp_filters)["results"]["contracts"]

# Count competed awards
comp_filters = {**base_filters, "extent_competed_type_codes": ["A","D","F","CDO"]}
comp_count = get_award_counts(comp_filters)["results"]["contracts"]

# Count small business set-aside awards
sb_filters = {**base_filters, "set_aside_type_codes": ["SBA","SBP","8A","8AN"]}
sb_count = get_award_counts(sb_filters)["results"]["contracts"]
```

### Top Small Business Vendors

```python
sb_vendor_filters = {**base_filters, "set_aside_type_codes": ["SBA","SBP","8A","8AN","HZC","HZS","SDVOSBS","SDVOSBC","WOSB","WOSBSS","EDWOSB","EDWOSBSS","VSA"]}
sb_vendors = spending_by_category("recipient", sb_vendor_filters, limit=15)
```

---

## Bulk Download

**Endpoint:** `POST /api/v2/bulk_download/awards/`

Async: returns download URL. Uses DIFFERENT filter structure than search endpoints.

```python
def request_bulk_download(filters):
    payload = json.dumps({"filters": filters, "columns": [], "file_format": "csv"})
    url = "https://api.usaspending.gov/api/v2/bulk_download/awards/"
    req = urllib.request.Request(url, data=payload.encode('utf-8'),
                                 headers={'Content-Type': 'application/json'})
    with urllib.request.urlopen(req, timeout=30) as resp:
        return json.loads(resp.read().decode())
```

Key differences from search filters:
- `"prime_award_types"` instead of `"award_type_codes"`
- `"date_range": {"start_date": "...", "end_date": "..."}` (object) instead of `"time_period"` (array)
- Requires `"date_type"` field (e.g., `"action_date"`)

---

## Loan Search

Loans use different field names. "Award Amount" returns HTTP 400; use "Loan Value".

```python
def search_loans(keywords, limit=10):
    payload = json.dumps({
        "subawards": False, "limit": min(limit, 100), "page": 1,
        "sort": "Loan Value", "order": "desc",
        "filters": {
            "keywords": keywords if isinstance(keywords, list) else [keywords],
            "award_type_codes": ["07", "08"]
        },
        "fields": ["Award ID", "Recipient Name", "Recipient UEI", "Description",
                    "Loan Value", "Subsidy Cost", "Awarding Agency", "Awarding Sub Agency",
                    "generated_internal_id", "Last Modified Date"]
    })
    url = "https://api.usaspending.gov/api/v2/search/spending_by_award/"
    req = urllib.request.Request(url, data=payload.encode('utf-8'),
                                 headers={'Content-Type': 'application/json'})
    with urllib.request.urlopen(req, timeout=15) as resp:
        return json.loads(resp.read().decode())
```

---

## Award Funding (File C Data)

**Endpoint:** `POST /api/v2/awards/funding/`

Federal account, object class, and program activity data linked to an award.

```python
def get_award_funding(generated_award_id):
    payload = json.dumps({"award_id": generated_award_id, "limit": 50, "page": 1})
    url = "https://api.usaspending.gov/api/v2/awards/funding/"
    req = urllib.request.Request(url, data=payload.encode('utf-8'),
                                 headers={'Content-Type': 'application/json'})
    with urllib.request.urlopen(req, timeout=15) as resp:
        return json.loads(resp.read().decode())
```

Sort fields: reporting_fiscal_date, account_title, transaction_obligated_amount, object_class.

---

## Reference Endpoints (GET)

| Endpoint | Use |
|----------|-----|
| `/api/v2/references/toptier_agencies/` | All top-tier agencies |
| `/api/v2/references/naics/<CODE>/` | NAICS code details (2-6 digit) |
| `/api/v2/references/filter_tree/psc/` | PSC hierarchy (drillable: `/psc/Service/R/`) |
| `/api/v2/references/submission_periods/` | Available reporting periods |
| `/api/v2/references/def_codes/` | Disaster/emergency fund codes |
| `/api/v2/references/data_dictionary/` | Full data dictionary |
| `/api/v2/references/glossary/` | Glossary terms |
| `/api/v2/recipient/state/<FIPS>/` | State spending profile |
| `/api/v2/agency/<CODE>/` | Agency overview (?fiscal_year=YYYY) |
| `/api/v2/agency/<CODE>/awards/` | Agency award summary |

---

## PSC Code Reference (Common IT/Services Codes)

| Code | Description |
|------|-------------|
| AN11 | R&D: Health Care Services, Basic Research |
| AN91 | R&D: Other Research and Development, Basic Research |
| AJ11 | R&D: General Science/Technology, Basic Research |
| AJ12 | R&D: General Science/Technology, Applied Research |
| AZ11 | R&D: Other R&D, Basic Research |
| R499 | Other Professional Services |
| D399 | IT and Telecom: Other IT and Telecommunications |
| DA01 | IT and Telecom: Application Development Support |
| S112 | Utilities: Electric |

---

## Vendor Deduplication Notes

The `recipient` category returns results by unique entity (UEI). Large companies often appear multiple times due to subsidiaries, divisions, or re-registrations. When using for vendor landscape analysis:
- Vendor counts may overstate unique companies
- Market share may be split across multiple entries
- No built-in dedup; apply case-insensitive name matching if precision matters
- Name normalization catches suffix variations but not acquisitions/rebrands (e.g., Cerner to Oracle Health Government Services; Northrop Grumman acquired Peraton)
- Vendor names return in ALL CAPS; title-case for reports but preserve LLC/INC/LLP in uppercase

---

## Rate Limiting

1. No auth needed; API is public and read-only
2. Add 0.3s delays between batch requests
3. Search max limit = 100; transactions max limit = 5000
4. API silently accepts invalid field names (returns null); verify spelling
5. Empty `keywords: []` returns HTTP 400; omit entirely if not needed

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| 422 "award_type_codes must only contain types from one group" | Mixed contract + IDV codes | Separate into two requests |
| 422 "Invalid value in filters\|psc_codes" | Empty `"exclude": []` | Remove exclude key or use simple array |
| 422 "Field 'limit' value above max" | limit > 100 (search) or > 5000 (transactions) | Reduce limit; paginate |
| 400 "Sort value not found in requested fields" | Sort field not in fields array | Add sort field to fields |
| 400 "Sort value not found in Loan Award mappings" | "Award Amount" with loan codes | Use "Loan Value" for loans |
| 400 "Invalid value in filters\|keywords" | Empty keywords array | Omit keywords filter entirely |
| Empty results for known PIID | Wrong award type group | Try contracts then IDVs |
| Null values for valid fields | Typo in field name (API accepts anything) | Verify field spelling |
| Truncated description | FPDS 4000-char limit | Normal; use what's available |
| Stale data (non-DoD) | FPDS update lag | Non-DoD typically available within 5 business days |
| Stale data (DoD/USACE) | 90-day FPDS reporting delay | DoD and USACE procurement data has a 90-day lag; plan accordingly |


---

*MIT © James Jenrette / 1102tools. Source: github.com/1102tools/federal-contracting-skills*
