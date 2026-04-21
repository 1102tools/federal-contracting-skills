---
name: bls-oews-api
description: >
  Query the BLS Occupational Employment and Wage Statistics (OEWS) API for market wage data by occupation, geography, and industry. Trigger for any mention of BLS, Bureau of Labor Statistics, OEWS, OES, occupational wages, market wages, salary data, wage percentiles, median wage, mean wage, labor market rates, SOC codes, or geographic wage differentials. Also trigger when the user needs to compare wages across metro areas, benchmark contractor labor rates against market data, support IGCE development with market wage research, or validate price proposals against BLS data. Complements the GSA CALC+ skill (ceiling rates from awarded contracts) by providing independent market wage data from employer surveys. Together they form a complete pricing toolkit - BLS OEWS for what the market pays, CALC+ for what GSA contractors charge.
---

# BLS OEWS API Skill

Query the U.S. Bureau of Labor Statistics Occupational Employment and Wage Statistics program via the public Timeseries API.

## Quick Start

```python
import urllib.request, json

OEWS_CURRENT_YEAR = "2024"  # May 2024 estimates, released April 2025
BLS_API_KEY = None  # Set to v2 key for 500 queries/day, or None for v1 (25/day)

def quick_wage_lookup(occ_code, prefix, area, industry="000000"):
    """Get the nine IGCE-relevant wage measures (including RSE for data quality) in one call.

    Parameters are intentionally required (no silent national default).
    prefix: 'OEUN' (national), 'OEUS' (state), or 'OEUM' (metro)
    area:   7-char area code (e.g., '0000000' national, '1100000' DC state, '0047900' DC metro)
    """
    # 9 key datatypes: employment + RSEs (data quality) + annual mean + all 5 annual percentiles
    datatypes = {
        "01": "Employment",
        "02": "Employment RSE %",
        "04": "Annual Mean",
        "05": "Wage RSE %",
        "11": "Annual P10",
        "12": "Annual P25",
        "13": "Annual Median",
        "14": "Annual P75",
        "15": "Annual P90",
    }
    series_ids = [f"{prefix}{area}{industry}{occ_code}{dt}" for dt in datatypes]
    for sid in series_ids:
        assert len(sid) == 25, f"Bad series ID length {len(sid)}: {sid}"
    url = "https://api.bls.gov/publicAPI/v2/timeseries/data/" if BLS_API_KEY else "https://api.bls.gov/publicAPI/v1/timeseries/data/"
    payload = {"seriesid": series_ids, "startyear": OEWS_CURRENT_YEAR, "endyear": OEWS_CURRENT_YEAR}
    if BLS_API_KEY:
        payload["registrationkey"] = BLS_API_KEY
    req = urllib.request.Request(url, data=json.dumps(payload).encode('utf-8'),
                                 headers={'Content-Type': 'application/json'})
    with urllib.request.urlopen(req, timeout=30) as resp:
        data = json.loads(resp.read().decode())
    results = {}
    for s in data["Results"]["series"]:
        dt = s["seriesID"][-2:]
        if s["data"]:
            entry = s["data"][0]
            val = entry["value"]
            # Pass full footnote dicts (code + text) for durable cap detection
            footnotes = entry.get("footnotes", [])
            results[datatypes[dt]] = interpret_value(val, footnotes, dt)
    return results

def interpret_value(val, footnotes, datatype):
    """Distinguish CAP (footnote code 5) from SUPPRESSION. Matters for IGCE defensibility.

    footnotes accepts list of dicts (with 'code' and 'text' keys) OR list of strings (legacy).
    Code-match (5 = cap) is the durable heuristic; text-match is the fallback if codes absent.
    """
    if val not in ("-", "#", "*"):
        if datatype in ("01",):
            return f"{int(float(val)):,}"
        if datatype in ("02", "05"):
            return f"{float(val):.1f}%"
        return f"${int(float(val)):,}"

    codes, texts = [], []
    for f in footnotes or []:
        if isinstance(f, dict):
            if f.get("code"):
                codes.append(str(f["code"]))
            if f.get("text"):
                texts.append(f["text"])
        else:
            texts.append(str(f))

    # Code 5 is the durable cap signal; text match is fallback if API ever drops codes
    if val == "-" and ("5" in codes or any("115.00" in t or "239,200" in t for t in texts)):
        return "CAP: wage >= $239,200/yr ($115/hr). Use as lower bound."
    if val == "-" and any("not released" in t.lower() for t in texts):
        return "SUPPRESSED: estimate not released (confidentiality or small sample)."
    return f"SUPPRESSED: {texts[0] if texts else val}"

# Example: Software Developers, DC metro
# print(quick_wage_lookup("151252", prefix="OEUM", area="0047900"))
```

**What you get back:** a dict with 7 measures. Numeric values are formatted strings. Capped and suppressed values are returned as distinct labeled strings (critical for IGCE defensibility; see "Interpreting Capped Wages" below).

---

## Overview

BLS Public Data API provides occupational wage data from the OEWS program (~1.1M establishments, ~830 occupations). National, state, and metro area levels with optional industry breakdowns.

Base URL (v2): `https://api.bls.gov/publicAPI/v2/timeseries/data/`
Base URL (v1): `https://api.bls.gov/publicAPI/v1/timeseries/data/`

| Feature | v1 (No Key) | v2 (Registered) |
|---------|-------------|-----------------|
| Daily queries | 25 | 500 |
| Series/query | 25 | 50 |
| Years/query | 10 | 20 |

**Register for a v2 key:** https://data.bls.gov/registrationEngine/ (free, email-verified). Store as env var or in memory.

**What this data is:** Employer-reported base wages. Does NOT include fringe, overhead, G&A, or profit. To build an IGCE, apply a burden/wrap rate (typically 1.5x-2.5x).

**CRITICAL:** Always use `OEWS_CURRENT_YEAR = "2024"`. Do NOT use `datetime.date.today().year`. OEWS data lags ~2 years. Querying 2025 or 2026 returns nothing. Next release: May 15, 2026 (May 2025 estimates).

---

## Series ID Format (CRITICAL)

Every query requires a **25-character** series ID:

```
Position:  [1-4]   [5-11]    [12-17]    [18-23]    [24-25]
Field:     PREFIX  AREA_CODE INDUSTRY   OCC_CODE   DATATYPE
Length:    4       7         6          6          2
Example:   OEUN    0000000   000000     151252     13
```

### PREFIX (4 chars)

| Prefix | Scope | Area Code format |
|--------|-------|------------------|
| `OEUN` | National | `0000000` |
| `OEUS` | State | FIPS + `00000` (e.g., `5100000` = VA) |
| `OEUM` | Metro | `00` + 5-digit MSA (e.g., `0047900` = DC) |

### DATATYPE (2 chars)

Full table. All codes verified against live API May 2024 data.

| Code | Measure | Unit |
|------|---------|------|
| `01` | Employment | Integer count |
| `02` | Employment % relative standard error | Percent |
| `03` | Hourly mean | Float |
| `04` | Annual mean | Integer |
| `05` | Wage % relative standard error | Percent |
| `06` | Hourly P10 | Float |
| `07` | Hourly P25 | Float |
| `08` | Hourly median (P50) | Float |
| `09` | Hourly P75 | Float |
| `10` | Hourly P90 | Float |
| `11` | Annual P10 | Integer |
| `12` | Annual P25 | Integer |
| `13` | Annual median (P50) | Integer |
| `14` | Annual P75 | Integer |
| `15` | Annual P90 | Integer |

**For IGCE work:** use 04 (mean), 12 (P25), 13 (median), 14 (P75). P10/P90 hit the reporting cap in high-wage occupations and metros. Hourly percentiles (06/07/09/10) are available at the same percentile points; BLS calculates them as annual / 2080, but queries for the direct hourly series return the computed value.

### INDUSTRY (6 chars)

`000000` = all industries (default for any place-of-performance query).

**Industry-specific estimates are national only (`OEUN`).** Cannot combine `OEUM` or `OEUS` with non-zero industry codes. 2-digit NAICS does NOT work; use 3-digit.

Common industries:
- `541000` = Professional, scientific, and technical services
- `541500` = Computer systems design and related
- `541600` = Management, scientific, and technical consulting
- `999100` = Federal government (excluding postal)

### OCCUPATION (6 chars)

SOC code without the dash. See "Common SOC Codes" below and the full federal-use table further down.

---

## Common Area Codes

### States (FIPS + 00000)

```
0600000 = CA    1100000 = DC    1200000 = FL    1300000 = GA
2400000 = MD    3400000 = NJ    3600000 = NY    4200000 = PA
4700000 = TN    4800000 = TX    5100000 = VA    5300000 = WA
```

### Metros (00 + 5-digit MSA)

Order: federal acquisition markets first. Commercial metros follow for cross-metro comparisons.

**Federal capital region / East Coast:**
```
0047900 = Washington-Arlington-Alexandria DC-VA-MD-WV MSA
0012580 = Baltimore-Columbia-Towson
0047260 = Virginia Beach-Norfolk-Newport News (NavAir, NavSea, Langley-Eustis)
```

**Defense and intelligence hubs:**
```
0017820 = Colorado Springs (Peterson/Schriever SFB, NORAD, USSPACECOM, Fort Carson)
0026620 = Huntsville, AL (Redstone Arsenal, MDA, MSFC)
0019380 = Dayton, OH (Wright-Patterson AFB, AFRL)
0036420 = Oklahoma City (Tinker AFB)
0041700 = San Antonio (JBSA, 24th AF, NSA Texas)
0036260 = Ogden-Clearfield, UT (Hill AFB, Ogden ALC)
0049660 = Warner Robins, GA (Robins AFB, WR-ALC)
```

**DOE national lab metros:**
```
0028940 = Knoxville, TN (ORNL, Y-12 NSC, TVA)
0042140 = Santa Fe, NM (LANL, adjacent Los Alamos)
0028420 = Kennewick-Richland, WA (PNNL, Hanford)
0026820 = Idaho Falls, ID (INL)
0010740 = Albuquerque, NM (Sandia, Kirtland AFB)
```

**Major commercial (for cross-metro comparisons):**
```
0041860 = San Francisco-Oakland-Berkeley   0042660 = Seattle
0035620 = NYC                              0031080 = Los Angeles
0014460 = Boston                           0019100 = Dallas
0026420 = Houston                          0016980 = Chicago
0037980 = Philadelphia                     0038060 = Phoenix
0012420 = Austin                           0033100 = Miami
0033460 = Minneapolis-St Paul              0041740 = San Diego
0045300 = Tampa                            0019820 = Detroit
```

**Southeast and other:**
```
0034980 = Nashville-Davidson-Murfreesboro
0032820 = Memphis
0017140 = Cincinnati
0035380 = New Orleans
0040140 = Riverside-San Bernardino
```

### CSA vs MSA

Some area codes resolve to Combined Statistical Areas (CSAs) with different county coverage vs. Core-Based Statistical Areas (CBSAs / MSAs) of the same name. BLS OEWS queries use CBSA codes. Example: Knoxville MSA is CBSA 28940; Knoxville CSA is 34100. If you need CSA-level aggregation, query the component MSAs separately and combine.

### 2024 MSA realignment note

OMB Bulletin 23-01 (2020 census) realigned MSA boundaries starting with May 2024 OEWS data. Same-code composition may differ from pre-2024 data. When comparing to older OEWS releases, verify the MSA's county composition hasn't changed. Pre-2024 trend analysis across the boundary is not reliable without explicit code-and-composition verification.

Full current MSA list: https://www.bls.gov/oes/current/msa_def.htm

### When your metro isn't in the table

1. Look up the MSA code at https://www.bls.gov/oes/current/oessrcma.htm or the OMB delineation file.
2. Format as `00` + 5-digit MSA = 7-char area code.
3. **Before assuming BLS doesn't publish an occupation at that metro**, verify your area code is correct. A wrong code returns "series does not exist" (same response pattern as real non-publication). See "Fallback Pattern" below.

---

## Common SOC Codes

### IT and software

| Title | SOC | Code |
|-------|-----|------|
| IT Manager | Computer and Information Systems Managers | 113021 |
| Computer & IS Manager | Computer and Information Systems Managers | 113021 |
| Software Developer | Software Developers | 151252 |
| Systems Analyst / Systems Engineer (IT) | Computer Systems Analysts | 151211 |
| Cybersecurity / InfoSec Analyst | Information Security Analysts | 151212 |
| Network Architect | Computer Network Architects | 151241 |
| Database Administrator | Database Administrators | 151242 |
| Systems Administrator | Network and Computer Systems Administrators | 151244 |
| QA Tester | Software QA Analysts and Testers | 151253 |
| Help Desk / Tier 1 Support | Computer User Support Specialists | 151232 |
| Data Scientist | Data Scientists | 152051 |
| Computer Programmer | Computer Programmers | 151251 |
| Web Developer | Web Developers | 151254 |

### Engineering (non-IT)

Use these for physical, aerospace, and defense engineering roles. Do NOT default to 151211 Computer Systems Analyst for non-IT "Systems Engineer" roles.

| Title | SOC | Code |
|-------|-----|------|
| Aerospace Engineer | Aerospace Engineers | 172011 |
| Biomedical Engineer | Biomedical Engineers | 172031 |
| Chemical Engineer | Chemical Engineers | 172041 |
| Civil Engineer | Civil Engineers | 172051 |
| Electrical Engineer | Electrical Engineers | 172071 |
| Electronics Engineer | Electronics Engineers, Except Computer | 172072 |
| Environmental Engineer | Environmental Engineers | 172081 |
| Industrial Engineer | Industrial Engineers | 172112 |
| Mechanical Engineer | Mechanical Engineers | 172141 |
| Nuclear Engineer | Nuclear Engineers | 172161 |
| Systems Engineer (non-IT) | Engineers, All Other | 172199 |
| Petroleum Engineer | Petroleum Engineers | 172171 |

### Management, analyst, and support

| Title | SOC | Code |
|-------|-----|------|
| Program Manager (technology) | Computer and Information Systems Managers | 113021 |
| Program Manager (general ops) | General and Operations Managers | 111021 |
| Project Manager | Project Management Specialists | 131082 |
| Management Analyst | Management Analysts | 131111 |
| Accountant | Accountants and Auditors | 132011 |
| Contracting Specialist (1102) | Buyers and Purchasing Agents | 131020 |
| Technical Writer | Technical Writers | 273042 |
| Admin Assistant | Secretaries and Admin Assistants | 436014 |
| Executive Secretary | Executive Secretaries | 436011 |
| Office Clerk | Office Clerks, General | 439061 |
| First-Line Office Supervisor | First-Line Supervisors of Office Workers | 431011 |

When mapping is ambiguous (e.g., "Program Manager" could be 113021 or 111021), query both and present the range. Document the choice in the output.

Full SOC list: https://www.bls.gov/oes/current/oes_stru.htm

---

## Critical Rules

1. **Series ID must be exactly 25 characters.** Use `assert len(sid) == 25`. Print component lengths to debug.
2. **Always query `OEWS_CURRENT_YEAR = "2024"`**, not calendar year. Most common failure.
3. **Industry estimates are national only.** Cannot combine `OEUM`/`OEUS` with non-zero industry codes.
4. **Not all occupation x area combos exist.** "Series does not exist" means the combination was not published. This is NOT the same as suppression (see next rule).
5. **Three distinct failure modes, all return `-` or empty, but mean different things:**
   - **`-` with footnote code 5:** wage >= $239,200/yr or $115/hr. **Use as lower bound.** Capped but present.
   - **`-` with "Estimate not released":** confidentiality/sample-size suppression. **Value is unknown.** Not usable.
   - **"Series does not exist":** your series ID doesn't exist in BLS's database. Either you typed the code wrong OR BLS legitimately doesn't publish this occ-area combo. **Verify your area and SOC codes before assuming non-publication.**
6. **Special values (`-`, `*`, `#`) must be checked before `float()`.** `float("-")` throws ValueError.
7. **Batch series into one call** (up to 50 for v2, 25 for v1). One query = one slot against your daily quota regardless of series count.
8. **2-digit NAICS codes don't work.** Use 3-digit (e.g., `541000` not `540000`).
9. **MSA definitions changed in May 2024 data** (2020 census, OMB Bulletin 23-01). Same-code composition may differ from pre-2024 data.
10. **Verify MSA code before assuming suppression.** If a metro query returns "series does not exist" for EVERY datatype, check the area code against the current BLS MSA list before falling back.

---

## Interpreting Capped Wages

BLS OEWS top-codes wages at **$239,200/year** or **$115.00/hour**. This is not a soft ceiling; it is the upper bound of the highest wage bracket in the survey instrument. Any percentile whose true value falls above this ceiling is reported as `-` with footnote code 5: "This wage is equal to or greater than $115.00 per hour or $239,200 per year."

### What the cap means

- **The cap is a floor on the true value, not an estimate.** A capped P90 means the 90th percentile is at least $239,200 but could be materially higher.
- **For high-wage occupations in high-wage metros**, P90 (and sometimes P75) often cap out. Examples: Software Developers in SF, Physicians in most major metros, Nuclear Engineers at specialty labs.
- **The cap has been $239,200 since May 2023 data.** It has not been updated for inflation. Tech and specialty labor markets have outgrown it. Expect more capped P90s in 2024 and 2025 data than in prior years.

### Handling capped values in IGCE work

1. **When P90 is capped, fall back to P75 as the highest reliable BLS figure.** P75 is usually uncapped and is a defensible ceiling for price reasonableness.
2. **Do not compute P90/P50 ratios from capped numbers.** $239,200 / median is a floor on the ratio, not the actual value.
3. **For contract estimate defensibility**, document "BLS OEWS caps reporting at $239,200/yr; actual market P90 may be higher. Using P75 as comparison point." Peer reviewers familiar with the cap will accept this language.
4. **Cross-reference with GSA CALC+** for labor categories where BLS P90 is capped. CALC+ ceiling rates have their own reporting conventions and can provide an independent upper bound for senior/principal labor.

### When P90 is capped and you need a point estimate

If the IGCE requires an actual P90 figure (e.g., competitive range analysis for senior or principal roles), follow this decision tree:

1. **Use P75 as your uncapped ceiling** for the rate range. Cite it as the highest reliable BLS figure. Document that P90 is capped and P75 is the defensibility anchor.
2. **Cross-reference GSA CALC+ ceiling rates** for comparable labor categories. CALC+ uses different reporting conventions and may provide an independent upper bound for senior labor.
3. **Apply the national P75/median ratio to your local median** as a derived P75 estimate when local P75 is also capped. Document as inference, not BLS figure.
4. **Use commercial compensation surveys** (Levels.fyi, Radford, Mercer) for tech and specialty markets where BLS caps bind well below true market P90. Cite the source explicitly.
5. **Convert capped value to hourly and apply burden multiplier** as a sanity check. $239,200/yr = $115/hr direct; at 2.0x burden = $230/hr loaded. This is a defensible LOWER bound for the top decile, not the value.

Do NOT use $239,200 as the point estimate. It is a floor, not the value.

### Cap vs. suppression: do not conflate them

Both return `-` in the API response. Both look the same to a naive parser. They are different:

| Pattern | Meaning | Usable? |
|---------|---------|---------|
| `-` with footnote code 5 (text contains "115.00" or "239,200") | CAP (top-coded) | Yes, as lower bound |
| `-` with footnote "Estimate not released" | SUPPRESSED | No, unknown value |
| `-` with footnote about RSE/sample size | SUPPRESSED | No, unknown value |
| Empty `data` array, or "Series does not exist" | NOT PUBLISHED | Check area code, then try broader geography |

The `interpret_value()` and `format_oes_value()` helpers handle all four cases. Both prefer footnote CODE matching (5 = cap) over text matching for durability against future BLS text reformatting; text match is a fallback. A naive parser that returns "Suppressed" for every `-` loses the cap vs. suppression distinction, which is the most common defensibility question in a BLS-based IGCE.

---

## Fallback Pattern: Metro to State to National

When a specific occupation at a specific metro returns no data, you may need to fall back to broader geography. This is common for specialty occupations (nuclear engineering, cleared cyber roles, etc.) in smaller metros.

### Fallback order

1. **Metro (OEUM):** Most geographically precise. Preferred.
2. **State (OEUS):** Broader pool. May be suppressed if the state's workforce for that occupation is concentrated in a few employers (nuclear in TN, certain intel SOCs in VA).
3. **National (OEUN):** Always available but least locally accurate.

### Counterintuitive pattern

State data is NOT always more reliable than metro data. When the state's workforce for an occupation is concentrated in a small number of large employers, BLS suppresses state estimates to protect confidentiality, but may still publish metro-level data.

Example: Nuclear Engineers in Tennessee. State-level wage data is suppressed for every percentile (ORNL/Y-12/TVA dominate the sample). Knoxville metro returns clean data for most percentiles, with P75 at the reporting cap.

### Single-employer-dominated occupations (predict suppression before querying)

When an occupation's state-level workforce is concentrated in a small number of dominant employers, BLS will suppress state-level estimates to protect confidentiality even when metro-level data publishes cleanly. Known patterns:

- **Nuclear Engineers in TN** (ORNL, Y-12 NSC, TVA dominate)
- **Nuclear physicists / engineers in NM, ID, WA** (LANL, INL, PNNL)
- **Cleared intel SOCs in VA and MD** (IC contractor clusters in NoVA and Fort Meade)
- **Aerospace engineers in AL Madison County** (Redstone, MDA, MSFC)
- **Astronomers in MD** (NASA Goddard cluster)

For these occupation/state combinations, query metro first; expect state to suppress; do not assume state is a safer fallback.

### Helper function

```python
def query_with_fallback(occ_code, metro_area, state_area, datatype="13"):
    """Query metro first, fall back to state, then national.

    Returns: (geography_used, value, footnotes)
    """
    # Try metro
    sid = build_oes_series_id("OEUM", normalize_area_code(metro_area), "000000", occ_code, datatype)
    result = _single_query(sid)
    if result["is_numeric"]:
        return ("metro", result["value"], result["footnotes"])

    # Try state
    sid = build_oes_series_id("OEUS", normalize_area_code(state_area), "000000", occ_code, datatype)
    result = _single_query(sid)
    if result["is_numeric"]:
        return ("state", result["value"], result["footnotes"])

    # Fall back to national
    sid = build_oes_series_id("OEUN", "0000000", "000000", occ_code, datatype)
    result = _single_query(sid)
    return ("national", result["value"], result["footnotes"])

def _single_query(sid):
    response = query_bls([sid])
    parsed = parse_oes_results(response)
    if sid in parsed:
        data = parsed[sid]
        return {"is_numeric": data["is_numeric"], "value": data["value"], "footnotes": data["footnotes"]}
    return {"is_numeric": False, "value": None, "footnotes": ["Series does not exist"]}
```

**Always document which geography the final answer came from.** A $127,520 national figure and a $174,380 metro figure for the same occupation tell different stories; don't let the reader assume you used metro when you actually fell back to national.

---

## Core Functions

### Build Series ID

```python
def build_oes_series_id(prefix="OEUN", area="0000000", industry="000000", occ="000000", datatype="04"):
    sid = f"{prefix}{area}{industry}{occ}{datatype}"
    assert len(sid) == 25, f"Series ID must be 25 chars, got {len(sid)}: {sid}"
    return sid
```

### Normalize Area Code

```python
def normalize_area_code(area_input):
    """Convert 2-digit FIPS, 5-digit MSA, or 7-char full code to 7-char format."""
    area = str(area_input).strip()
    if len(area) == 7: return area
    elif len(area) == 5: return f"00{area}"
    elif len(area) == 2: return f"{area}00000"
    else: raise ValueError(f"Unrecognized area code: '{area}' (len={len(area)})")
```

### Query BLS API

```python
OEWS_CURRENT_YEAR = "2024"

def query_bls(series_ids, start_year=None, end_year=None, registration_key=None):
    url = "https://api.bls.gov/publicAPI/v2/timeseries/data/" if registration_key else "https://api.bls.gov/publicAPI/v1/timeseries/data/"
    year = start_year or OEWS_CURRENT_YEAR
    payload = {"seriesid": series_ids, "startyear": year, "endyear": end_year or year}
    if registration_key:
        payload["registrationkey"] = registration_key
    req = urllib.request.Request(url, data=json.dumps(payload).encode('utf-8'),
                                 headers={'Content-Type': 'application/json'})
    with urllib.request.urlopen(req, timeout=30) as resp:
        return json.loads(resp.read().decode())
```

### Parse Results

```python
BLS_SPECIAL_VALUES = {"-", "#", "*", "N/A"}

def parse_oes_results(api_response):
    """Parse response into {series_id: {year, value, is_numeric, footnotes}}."""
    results = {}
    for series in api_response.get("Results", {}).get("series", []):
        sid = series["seriesID"]
        if series["data"]:
            latest = series["data"][0]
            footnote_texts = [f.get("text", "") for f in latest.get("footnotes", []) if f.get("text")]
            val = latest["value"]
            results[sid] = {
                "year": latest["year"],
                "period": latest["periodName"],
                "value": val,
                "is_numeric": val not in BLS_SPECIAL_VALUES,
                "footnotes": footnote_texts,
            }
    return results
```

### Format Value (cap vs. suppression aware)

```python
DATATYPE_LABELS = {
    "01": "Employment", "02": "Emp RSE", "03": "Hourly Mean", "04": "Annual Mean",
    "05": "Wage RSE", "06": "Hourly P10", "07": "Hourly P25", "08": "Hourly Median",
    "09": "Hourly P75", "10": "Hourly P90", "11": "Annual P10", "12": "Annual P25",
    "13": "Annual Median", "14": "Annual P75", "15": "Annual P90",
}
HOURLY_DATATYPES = {"03", "06", "07", "08", "09", "10"}

def format_oes_value(value_str, datatype_code, footnotes=None):
    """Format with cap vs suppression distinction.

    footnotes accepts list of dicts (with 'code'/'text' keys) OR list of strings (legacy).
    Code 5 is the durable cap signal; text-match is fallback.
    """
    if value_str in BLS_SPECIAL_VALUES:
        codes, texts = [], []
        for f in footnotes or []:
            if isinstance(f, dict):
                if f.get("code"):
                    codes.append(str(f["code"]))
                if f.get("text"):
                    texts.append(f["text"])
            else:
                texts.append(str(f))
        first_text = texts[0] if texts else value_str
        if "5" in codes or any("115.00" in t or "239,200" in t for t in texts):
            return f"[CAP >=$239,200] {first_text}"
        if any("not released" in t.lower() for t in texts):
            return f"[SUPPRESSED: not released] {first_text}"
        return f"[SUPPRESSED: {first_text}]"
    try:
        if datatype_code == "01":
            return f"{int(float(value_str)):,}"
        elif datatype_code in ("02", "05"):
            return f"{float(value_str):.1f}%"
        elif datatype_code in HOURLY_DATATYPES:
            return f"${float(value_str):,.2f}/hr"
        else:
            return f"${int(float(value_str)):,}"
    except (ValueError, TypeError):
        return f"[Unparseable: {value_str}]"
```

### Detect Latest OEWS Year

Optional. Call once per session if your work may run close to the May release date.

```python
def detect_oews_year(api_key=None):
    """Probe API for newer data year. May be superseded by new release after April each year."""
    probe_series = "OEUN000000000000000000004"  # national all-occs annual mean
    candidate = str(int(OEWS_CURRENT_YEAR) + 1)
    url = "https://api.bls.gov/publicAPI/v2/timeseries/data/" if api_key else "https://api.bls.gov/publicAPI/v1/timeseries/data/"
    payload = {"seriesid": [probe_series], "startyear": candidate, "endyear": candidate}
    if api_key: payload["registrationkey"] = api_key
    try:
        req = urllib.request.Request(url, data=json.dumps(payload).encode('utf-8'),
                                     headers={'Content-Type': 'application/json'})
        with urllib.request.urlopen(req, timeout=15) as resp:
            data = json.loads(resp.read().decode())
        for s in data.get("Results", {}).get("series", []):
            if s.get("data") and s["data"][0].get("value") not in ("-", "#", "*", ""):
                return candidate
    except Exception:
        pass
    return OEWS_CURRENT_YEAR
```

**When to call:** at the start of a session occurring after April 15 of any year. Skip for single-shot lookups before then.

---

## Query Recipes

### Full Wage Profile for One Occupation

Pulls all 9 IGCE-relevant datatypes (employment + 2 means + 5 annual percentiles + hourly median) in one batched call.

```python
def get_wage_profile(occ_code, prefix, area, industry="000000"):
    """Get employment + full wage distribution for an occupation."""
    datatypes = ["01", "03", "04", "08", "11", "12", "13", "14", "15"]
    series_ids = [build_oes_series_id(prefix, area, industry, occ_code, dt) for dt in datatypes]
    response = query_bls(series_ids)
    results = parse_oes_results(response)
    profile = {}
    all_footnotes = []
    for sid, data in results.items():
        dt = sid[-2:]
        profile[DATATYPE_LABELS.get(dt, dt)] = format_oes_value(data["value"], dt, data.get("footnotes"))
        profile["_year"] = data["year"]
        if data.get("footnotes"):
            all_footnotes.extend(data["footnotes"])
    if all_footnotes:
        profile["_footnotes"] = list(set(all_footnotes))
    return profile

# National: get_wage_profile("151252", "OEUN", "0000000")
# Seattle:  get_wage_profile("151252", "OEUM", "0042660")
# Virginia: get_wage_profile("151252", "OEUS", "5100000")
# Federal govt, national: get_wage_profile("151252", "OEUN", "0000000", industry="999100")
```

### Full Hourly Profile

Hourly percentiles are available via datatypes 06/07/09/10 alongside hourly mean 03 and median 08.

```python
def get_hourly_profile(occ_code, prefix, area, industry="000000"):
    """Get full hourly wage distribution."""
    datatypes = ["03", "06", "07", "08", "09", "10"]
    series_ids = [build_oes_series_id(prefix, area, industry, occ_code, dt) for dt in datatypes]
    response = query_bls(series_ids)
    return parse_oes_results(response)
```

BLS computes hourly percentiles as annual / 2080. The direct hourly series returns the same value to the penny.

### Worked Example: Capped Wage Response

When a percentile returns at the $239,200 cap, here's what the API response looks like and how to handle it:

```python
# Request: Software Developer (15-1252) P90, San Francisco MSA (0041860)
sid = build_oes_series_id("OEUM", "0041860", "000000", "151252", "15")
# Series ID: OEUM004186000000015125215

# Response from API:
# {
#   "seriesID": "OEUM004186000000015125215",
#   "data": [{
#       "year": "2024",
#       "value": "-",
#       "footnotes": [{
#           "code": "5",
#           "text": "This wage is equal to or greater than $115.00 per hour or $239,200 per year."
#       }]
#   }]
# }

# interpret_value(val="-", footnotes=[...], datatype="15") returns:
# "CAP: wage >= $239,200/yr ($115/hr). Use as lower bound."

# IGCE narrative format:
# "P90 for Software Developers in SF-Oakland-Berkeley MSA is top-coded at >= $239,200/yr
#  per BLS OEWS footnote code 5. True P90 is unknown but greater than this figure.
#  Using P75 of $213,420 as the upper anchor for the labor category rate range."
```

### Compare Wages Across Metros

```python
def compare_metros(occ_code, metro_codes, datatype="13"):
    """Compare a wage measure across multiple metro areas."""
    series_ids = [build_oes_series_id("OEUM", normalize_area_code(area), "000000", occ_code, datatype)
                  for area in metro_codes]
    return parse_oes_results(query_bls(series_ids))

# metros = ["0047900", "0042660", "0012580"]  # DC, Seattle, Baltimore
```

### Compare Multiple Occupations in One Location

```python
def compare_occupations(occ_codes, prefix, area, datatype="13"):
    """Compare median wage across occupations in one location."""
    series_ids = [build_oes_series_id(prefix, area, "000000", occ, datatype) for occ in occ_codes]
    return parse_oes_results(query_bls(series_ids))
```

### Industry-Specific Wages (National Only)

```python
# Federal govt vs all-industry vs computer systems design for Systems Analysts
series_ids = [
    build_oes_series_id("OEUN", "0000000", "000000", "151211", "04"),  # All industries
    build_oes_series_id("OEUN", "0000000", "999100", "151211", "04"),  # Federal govt
    build_oes_series_id("OEUN", "0000000", "541500", "151211", "04"),  # Computer systems
]
```

---

## BLS vs CALC+ Comparison

| Dimension | BLS OEWS | GSA CALC+ Ceiling Rates |
|-----------|----------|------------------------|
| Data source | Employer wage surveys (~1.1M establishments) | Awarded GSA MAS contract rates |
| What it measures | Base wages paid to employees | Fully burdened ceiling hourly rates |
| Includes overhead/profit | No (base wage only) | Yes (fully loaded) |
| Geographic detail | National, state, 530+ metros | Worldwide (no geo breakdown) |
| Industry detail | National industry breakdowns | By SIN/category |
| Occupation taxonomy | SOC codes (~830 occupations) | Vendor-defined labor categories |
| Percentile data | Yes (10/25/50/75/90, annual + hourly) | Yes (via aggregation) |
| Update frequency | Annual (~April) | Nightly |
| Best for | Market wage benchmarking, IGCE base rates | Price reasonableness for GSA orders |
| IGCE role | Base wage x burden factor | Validate proposed rates |

### Using Both Together

1. Start with BLS OEWS: look up occupation at performance location (median + range).
2. Calculate burdened range: apply 1.5x-2.5x multiplier per contract type.
3. Cross-reference CALC+: GSA ceiling rates should fall within or near the burdened BLS range.
4. If CALC+ >> burdened BLS: specialized role, clearance, or high overhead. Document the gap.
5. If CALC+ << burdened BLS: aggressive pricing or BLS category is broader than the labor cat.

---

## IGCE Rate Derivation

BLS annual wages assume 2,080 hours/year (standard work year). To convert to burdened hourly rate:

```python
def bls_to_igce_rate(annual_wage, burden_multiplier=2.0, productive_hours=2080):
    """Convert BLS annual wage to estimated fully burdened hourly rate.

    productive_hours: 2080 = standard BLS work year (default; use for direct labor rate basis).
                      1880 = contractor billable hours (accounts for ~200 hrs of PTO,
                             holidays, training, sick). Use for contractor pricing.

    Typical burden multipliers:
      1.5x-1.7x = Lean contractor
      1.8x-2.2x = Mid-range professional services
      2.0x-2.5x = Large contractor with clearance overhead
      2.5x-3.0x = High-overhead (SCIF, deployed, specialty)
    """
    return round((annual_wage / productive_hours) * burden_multiplier, 2)

# Software Developer DC median $138,410 at 2.0x burden, BLS 2080 hr basis = $133.09/hr
# Same wage at 2.0x burden, contractor 1880 hr basis = $147.24/hr
```

Burden multiplier ranges above are practitioner consensus, not DCAA-certified. For defensibility in contested price negotiations, cite an actual vendor-specific wrap rate or published DCAA guidance, not just the skill default.

---

## Complete IGCE Workflow Example

```python
OEWS_CURRENT_YEAR = "2024"
BLS_API_KEY = None  # set to your v2 key

occupations = {
    "151211": "Systems Analyst",
    "151252": "Software Developer",
    "151232": "Help Desk Specialist",
    "151244": "Network Administrator",
}
datatypes = ["04", "11", "12", "13", "14", "15"]  # full annual distribution
target_metro = "0042660"  # Seattle

all_series = []
for occ in occupations:
    for dt in datatypes:
        all_series.append(build_oes_series_id("OEUM", target_metro, "000000", occ, dt))

response = query_bls(all_series, registration_key=BLS_API_KEY)

data_year = "N/A"
for s in response['Results']['series']:
    if s['data']:
        data_year = s['data'][0]['year']
        break

print(f"=== IGCE Market Wage Research: Seattle Metro ===")
print(f"    Source: BLS OEWS, May {data_year}\n")

labels = {"04": "Mean", "11": "P10", "12": "P25", "13": "Median", "14": "P75", "15": "P90"}
for occ_code, title in occupations.items():
    print(f"  {title} (SOC {occ_code[:2]}-{occ_code[2:]}):")
    for dt in datatypes:
        sid = build_oes_series_id("OEUM", target_metro, "000000", occ_code, dt)
        for s in response['Results']['series']:
            if s['seriesID'] == sid and s['data']:
                entry = s['data'][0]
                val = entry['value']
                footnotes = [f.get("text", "") for f in entry.get("footnotes", []) if f.get("text")]
                formatted = format_oes_value(val, dt, footnotes)
                if val in BLS_SPECIAL_VALUES:
                    print(f"    {labels[dt]:>8}: {formatted}")
                else:
                    wage = int(float(val))
                    hourly = wage / 2080
                    burdened_low, burdened_high = hourly * 1.8, hourly * 2.2
                    print(f"    {labels[dt]:>8}: ${wage:>9,}/yr | ${hourly:>6.2f}/hr | ${burdened_low:>6.2f}-${burdened_high:>6.2f}/hr burdened")
    print()
```

---

## Error Handling

| Response | Meaning | Action |
|----------|---------|--------|
| `"Series does not exist"` | Invalid series ID OR occ-area combo not published | Verify 25-char format, check area code against current MSA list, verify SOC code |
| `"No Data Available for Series X Year: Y"` | Valid series, wrong year | Use `OEWS_CURRENT_YEAR = "2024"`, not calendar year |
| Empty `data` array | Same as "does not exist" in v1 API | Try fallback geography |
| Value `"-"` with footnote text containing "115.00" or "239,200" | CAP (top-coded) | Use $239,200/yr or $115/hr as lower bound. Document in IGCE narrative. |
| Value `"-"` with "Estimate not released" footnote | SUPPRESSED | Value is unknown. Try broader geography or parent SOC. |
| Value `"*"` (footnote 8) | RSE > 50% or <50 observations | Try broader geography. Value is unreliable. |
| Value `"#"` | Estimate withheld | Try broader geography. |
| `"REQUEST_NOT_PROCESSED"` | Rate limit or malformed request | Wait and retry. Check series ID construction. |
| `"REQUEST_SUCCEEDED"` with messages | Partial success | Check individual series; some may have failed while others succeeded. |

### Key distinction: cap vs. non-publication

Both can return `-` or empty data, but they require different responses:

- **CAP**: the value IS at least $239,200; you have a usable lower bound. Keep building the estimate and document the cap.
- **NOT PUBLISHED / SUPPRESSED**: the value is unknown. Fall back to broader geography (state, then national), or try a related SOC code. Do not guess.

`format_oes_value()` distinguishes these cases based on footnote text. A naive parser that treats every `-` as "capped" will misreport suppression as a usable lower bound, which is wrong for defensibility.

---

## Appendix: Full State FIPS Codes

```
0100000=AL  0200000=AK  0400000=AZ  0500000=AR  0600000=CA
0800000=CO  0900000=CT  1000000=DE  1100000=DC  1200000=FL
1300000=GA  1500000=HI  1600000=ID  1700000=IL  1800000=IN
1900000=IA  2000000=KS  2100000=KY  2200000=LA  2300000=ME
2400000=MD  2500000=MA  2600000=MI  2700000=MN  2800000=MS
2900000=MO  3000000=MT  3100000=NE  3200000=NV  3300000=NH
3400000=NJ  3500000=NM  3600000=NY  3700000=NC  3800000=ND
3900000=OH  4000000=OK  4100000=OR  4200000=PA  4400000=RI
4500000=SC  4600000=SD  4700000=TN  4800000=TX  4900000=UT
5000000=VT  5100000=VA  5300000=WA  5400000=WV  5500000=WI
5600000=WY  7200000=PR
```

Full state list: https://www.bls.gov/oes/current/oes_stru.htm

---

*MIT © James Jenrette / 1102tools. Source: github.com/1102tools/federal-contracting-skills*
