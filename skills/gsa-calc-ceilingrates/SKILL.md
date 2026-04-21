---
name: gsa-calc-ceilingrates
description: >
  Query the GSA CALC+ Labor Ceiling Rates API for awarded GSA MAS schedule hourly rates. Use this skill whenever the user asks about GSA labor rates, MAS ceiling prices, IGCE development, price reasonableness analysis, labor category comparisons, GSA schedule pricing, or market research for professional services acquisitions. Trigger for any mention of CALC, CALC+, ceiling rates, GSA rates, MAS rates, labor categories, GSA pricing, schedule pricing, IGCE support, price analysis, or rate benchmarking. Also trigger when the user needs to compare vendor pricing, look up rates by education/experience level, find rates for a specific SIN, or support price negotiations for task orders under GSA MAS contracts. This skill is essential for IGCEs, price reasonableness determinations, market research memos, and competitive range analysis.
---

# GSA CALC+ Labor Ceiling Rates API Skill

Query GSA CALC+ (Contract-Awarded Labor Category) for awarded NTE hourly ceiling rates from GSA MAS contracts.

## Overview

The CALC+ API provides free, no-auth access to awarded NTE hourly ceiling rates from GSA MAS contracts. Data refreshes nightly.

Base URL: `https://api.gsa.gov/acquisition/calc/v3/api/ceilingrates/`
Web UI: https://buy.gsa.gov/pricing/qr/mas

**What this data is:** Fully burdened hourly ceiling rates (the max a contractor can charge). Worldwide, master contract-level pricing.

**What this data is NOT:** Order-level actuals, BLS wages, or prices-paid data.

## Response Shape (READ THIS FIRST)

Every response follows this envelope. **All aggregation statistics are nested under `aggregations`, NOT at the top level.** A naive `response["wage_stats"]` returns `None` and breaks silently.

```
response
├── took                               (request duration in ms)
├── timed_out                          (check this; retry if true)
├── _shards                            (Elasticsearch internals)
├── hits
│   ├── total
│   │   ├── value                      (cap 10,000; see hits.total note below)
│   │   └── relation                   ("eq" or "gte")
│   └── hits                           (paginated record array)
│       └── [{_id, _source: {labor_category, current_price, ...}}, ...]
└── aggregations                       <-- ALL stats live here
    ├── wage_stats
    │   ├── count                      (TRUE record count; trust over hits.total)
    │   ├── min, max, avg
    │   ├── std_deviation
    │   └── std_deviation_bounds       ({upper, lower})
    ├── histogram_percentiles
    │   └── values                     ({"10.0": ..., "25.0": ..., "50.0": ..., "75.0": ..., "90.0": ...})
    ├── median_price
    │   └── values                     ({"50.0": ...}; see "median vs median" below)
    ├── education_level_counts
    │   └── buckets                    ([{key: "BA", doc_count: N}, ...])
    ├── business_size
    │   └── buckets
    ├── worksite
    │   └── buckets
    ├── security_clearance
    │   └── buckets                    (MULTIPLE variant keys; not normalized)
    ├── wage_histogram
    │   └── buckets
    └── labor_category
        └── buckets                    (top term aggregations)
```

**Canonical JSON paths for the 7 IGCE-relevant statistics:**

```python
response["aggregations"]["wage_stats"]["count"]                       # N
response["aggregations"]["wage_stats"]["min"]                         # min
response["aggregations"]["wage_stats"]["max"]                         # max
response["aggregations"]["wage_stats"]["avg"]                         # mean
response["aggregations"]["histogram_percentiles"]["values"]["25.0"]   # P25
response["aggregations"]["histogram_percentiles"]["values"]["50.0"]   # P50 (median)
response["aggregations"]["histogram_percentiles"]["values"]["75.0"]   # P75
```

## Quick Start: Distribution Snapshot

The most common IGCE use case in one function. Pulls the canonical 7-statistic table.

```python
import urllib.request, urllib.parse, json

BASE_URL = "https://api.gsa.gov/acquisition/calc/v3/api/ceilingrates/"

def distribution_snapshot(query_mode, query_value, filters=None):
    """Pull count/min/max/mean/P25/P50/P75 for a labor category population.

    query_mode: 'keyword' (wildcard across 3 fields; DO NOT use for final stats),
                'search' (exact labor_category match; USE THIS for rate stats),
                'filter_only' (no search term, filter-based only)
    query_value: the keyword, or 'labor_category:<exact value>' for search mode
    filters: list of 'field:value' filter strings (optional)

    Returns dict with the 7 stats plus record_count. aggregations are computed
    over the FULL result set regardless of page_size.
    """
    if query_mode == "keyword":
        params = f"keyword={urllib.parse.quote_plus(query_value)}"
    elif query_mode == "search":
        params = f"search={query_value}"
    elif query_mode == "filter_only":
        params = ""
    else:
        raise ValueError(f"Unknown query_mode: {query_mode}")

    # page_size=1 minimizes payload for aggregation-only queries (stats cover full set regardless)
    params += "&page=1&page_size=1"
    if filters:
        for f in filters:
            params += f"&filter={f}"

    url = f"{BASE_URL}?{params}"
    with urllib.request.urlopen(urllib.request.Request(url), timeout=15) as resp:
        data = json.loads(resp.read().decode())

    if data.get("timed_out"):
        return {"error": "API timed_out; retry query", "query_value": query_value}

    aggs = data.get("aggregations", {})
    stats = aggs.get("wage_stats", {})
    pct = aggs.get("histogram_percentiles", {}).get("values", {})

    return {
        "query_value": query_value,
        "query_mode": query_mode,
        "filters": filters or [],
        "record_count": stats.get("count"),
        "min": stats.get("min"),
        "max": stats.get("max"),
        "mean": round(stats.get("avg", 0), 2) if stats.get("avg") else None,
        "std_deviation": round(stats.get("std_deviation", 0), 2) if stats.get("std_deviation") else None,
        "p25": pct.get("25.0"),
        "p50": pct.get("50.0"),
        "p75": pct.get("75.0"),
        "p10": pct.get("10.0"),
        "p90": pct.get("90.0"),
    }

# Example: Information Security Analyst baseline
# print(distribution_snapshot("keyword", "information security analyst"))
```

---

## Critical Rules

### 1. Use the correct aggregation path (silent-wrong-answer bug)

Every stat lives under `aggregations`. A naive `data.get("wage_stats")` returns `None`. Print the response's `list(data.keys())` to verify envelope shape before indexing. Reading from the wrong path is the most common failure mode and it fails silently (returns `None`, not an error).

### 2. Never use `keyword=` for rate statistics

The `keyword=` parameter does wildcard matching across THREE fields simultaneously: `labor_category`, `vendor_name`, AND `idv_piid`. Use it only for scoping and discovery. For actual rate analysis, contamination is material.

**Example of contamination:** `keyword=Program Manager` returns 7,763 records. The sum of exact-match queries across all real Program Manager LCAT tiers (PM I, PM II, PM III, Senior PM, IT PM, Cloud PM, etc.) totals about 3,354. Over half the keyword hits are vendor-name matches or PIID token matches, not Program Manager labor categories. The keyword's $187 median is polluted; the exact-match median is $177 (PM exact) or $151 (PM I) or $203 (Senior PM) depending on seniority.

**Use this pattern instead:**
```
1. suggest-contains=labor_category:<term>       # discover real LCAT titles
2. search=labor_category:<exact bucket name>    # pull rate stats on each
3. Compare across tiers; do NOT average the keyword= noise
```

### 3. Discover Labor Categories First (Before Anything Else)

Labor category titles are the noisiest field in CALC+. Vendor PPT submissions produce dozens of seniority variants for a single occupation. Always use `suggest-contains=labor_category:<term>` to discover real bucket names before computing statistics.

```python
def suggest_labor_category(term):
    """List real LCAT bucket names matching a search term."""
    url = f"{BASE_URL}?suggest-contains=labor_category:{urllib.parse.quote_plus(term)}"
    with urllib.request.urlopen(urllib.request.Request(url), timeout=15) as resp:
        data = json.loads(resp.read().decode())
    buckets = data.get('aggregations', {}).get('labor_category', {}).get('buckets', [])
    return [{"bucket": b["key"], "count": b["doc_count"]} for b in buckets]
```

**Known default bucket cap:** `suggest-contains` appears to return at most 100 buckets. If you hit exactly 100, assume silent truncation and narrow your discovery term. (Undocumented in the API itself; observed empirically.)

### 4. security_clearance filter is fuzzy, not normalized

The skill previously claimed `security_clearance:yes` gets normalized. It does NOT. The filter does truthy/falsy partial matching but returns MULTIPLE variant bucket keys:

- Truthy variants: `true`, `Yes`, `SECRET or TS/SCI eligible`
- Falsy variants: `false`, `No`, `N/A`

Many records have clearance info buried in the LCAT name itself rather than this field. The `security_clearance:yes` filter COUNT IS A FLOOR, not a true count of cleared records. For IGCE defensibility on cleared contracts, document the filter's limitations and consider the all-clearance pool as the more defensible benchmark.

### 5. hits.total caps at 10,000

When results exceed 10,000, `hits.total.value` reads as 10000 with `relation: "gte"`. **`aggregations.wage_stats.count` is the true record count.** Trust it. Below 10,000, `hits.total.value` and `wage_stats.count` match exactly.

### 6. Messy Field Values (tribal knowledge)

Data comes from vendor PPTs. Values are not standardized.

- **education_level:** ~15 distinct values; filter with abbreviated form (`BA`, `MA`, `HS`, `AA`, `PHD`); API does partial matching. Prefer `aggregations.education_level_counts` (6 normalized buckets) over raw `education_level` aggregation.
- **worksite:** ~20 values; filter with `Customer`, `Contractor`, or `Both`; API partial-matches.
- **business_size:** Clean. `S` = Small, `O` = Other.
- **SIN:** Case-sensitive. Use official GSA casing (`54151HACS`, not `54151hacs`).
- **labor_category:** See Rule #3; use discovery first.
- **vendor_name:** Inconsistent spellings; use `suggest-contains` for discovery.

### 7. Use `page_size=1` for aggregation-only queries

Aggregations are computed over the FULL result set regardless of page_size. For distribution/stats queries, `page_size=1` minimizes payload and tokens. Only increase page_size when you need the actual records.

### 8. Check `timed_out`

If `timed_out=true`, results may be incomplete. Retry or narrow the query.

### 9. Rate limit: 1,000 requests/hour per IP (no auth)

Add `time.sleep(0.3)` between batch queries. No API key needed but IP-level limit applies.

---

## Core Endpoints

All queries are GET against the base URL with query parameters. No authentication.

### Three Search Modes

| Mode | Parameter | Use For |
|------|-----------|---------|
| Keyword wildcard | `keyword=term` | Scoping and discovery ONLY. Pollutes rate stats. |
| Exact match | `search=field:value` | Rate statistics. Must use exact field value. |
| Autocomplete | `suggest-contains=field:term` | Discover exact values for `search=`. 2-char min. |

### Keyword Search (for discovery/scoping only)

```python
def keyword_search(keyword, page=1, page_size=1, filters=None,
                   ordering="current_price", sort="asc"):
    """Wildcard across labor_category + vendor_name + idv_piid. For scoping only."""
    params = f"keyword={urllib.parse.quote_plus(keyword)}&page={page}&page_size={page_size}"
    params += f"&ordering={ordering}&sort={sort}"
    if filters:
        for f in filters:
            params += f"&filter={f}"
    with urllib.request.urlopen(urllib.request.Request(f"{BASE_URL}?{params}"), timeout=15) as resp:
        return json.loads(resp.read().decode())
```

### Exact Match Search (PRIMARY for rate stats)

```python
def exact_search(field, value, page=1, page_size=1, filters=None,
                 ordering="current_price", sort="asc"):
    """Exact-match on labor_category, vendor_name, or idv_piid.

    Use suggest-contains to discover the exact value first.
    """
    encoded = urllib.parse.quote_plus(value)
    params = f"search={field}:{encoded}&page={page}&page_size={page_size}"
    params += f"&ordering={ordering}&sort={sort}"
    if filters:
        for f in filters:
            params += f"&filter={f}"
    with urllib.request.urlopen(urllib.request.Request(f"{BASE_URL}?{params}"), timeout=15) as resp:
        return json.loads(resp.read().decode())
```

### Suggest-Contains (discovery)

```python
def suggest_contains(field, term):
    """Return aggregation buckets for autocomplete discovery. 2-char min.

    Returns ~100 buckets max (default cap). If you hit exactly 100, narrow the term.
    """
    encoded = urllib.parse.quote_plus(term)
    url = f"{BASE_URL}?suggest-contains={field}:{encoded}"
    with urllib.request.urlopen(urllib.request.Request(url), timeout=15) as resp:
        data = json.loads(resp.read().decode())
    buckets = data.get('aggregations', {}).get(field, {}).get('buckets', [])
    total = data.get('hits', {}).get('total', {}).get('value', 0)
    return {
        'suggestions': [{"value": b["key"], "count": b["doc_count"]} for b in buckets],
        'total_matching_records': total,
        'likely_truncated': len(buckets) >= 100,
    }
```

### Filtered Browse (no search term)

```python
def filtered_browse(filters, page=1, page_size=1, ordering="current_price", sort="asc"):
    """Browse with filters only. Good for SIN-based or filter-only market segment stats."""
    params = f"page={page}&page_size={page_size}&ordering={ordering}&sort={sort}"
    for f in filters:
        params += f"&filter={f}"
    with urllib.request.urlopen(urllib.request.Request(f"{BASE_URL}?{params}"), timeout=15) as resp:
        return json.loads(resp.read().decode())
```

---

## Filter Reference

Format: `filter=field:value`. Multiple filters AND together. Pipe (`|`) within a field = OR.

| Filter | Format | Example |
|--------|--------|---------|
| `education_level` | Code or pipe-OR | `education_level:BA` or `education_level:BA\|MA` |
| `experience_range` | min,max | `experience_range:5,15` |
| `min_years_experience` | exact | `min_years_experience:10` |
| `price_range` | min,max | `price_range:50,500` |
| `business_size` | S or O | `business_size:S` |
| `security_clearance` | yes or no (partial-match; see Rule #4) | `security_clearance:yes` |
| `sin` | SIN code (case-sensitive) | `sin:54151HACS` |
| `worksite` | value (partial-match) | `worksite:Customer` |

**Combining example:** `filter=education_level:BA|MA&filter=experience_range:5,15&filter=business_size:S`

**Filter encoding guidance:**
- Pipe (`|`) and comma (`,`) within filter values: the API accepts them unescaped. If you URL-encode these characters, queries may fail.
- Quote/encode the filter VALUE only when it contains spaces or special chars (e.g., `filter=labor_category:%22Software%20Engineer%22`).
- Pass filter parameters raw to the URL builder; do NOT use `urllib.parse.quote_plus` on the full `filter=` string.

**Outlier-strip convention for IGCE work:** Apply `filter=price_range:30,500` by default for professional-services labor to exclude clerical floor and unrealistic ceiling outliers. Document the bound in the memo. Adjust up for SCIF/cleared work (`price_range:50,600`) or down for routine administrative (`price_range:15,200`).

---

## Ordering

| Field | Description |
|-------|-------------|
| `current_price` | Hourly ceiling rate (default) |
| `labor_category` | Alphabetical |
| `vendor_name` | Alphabetical |
| `education_level` | By education |
| `min_years_experience` | By experience |

Sort: `sort=asc` or `sort=desc`.

---

## Composite Recipes

### Price Reasonableness Check (dual-pool for senior LCATs)

For any "Senior X" LCAT, there are TWO legitimate benchmark pools that often disagree materially:
- **Title-match pool:** `keyword="Senior X"` + exact-match discovery + `search=labor_category:<each senior tier>`
- **Experience-match pool:** `keyword="X"` + `min_years_experience:8` filter (or similar senior threshold)

These can differ by 10-15% at the median. Always pull both and present side-by-side. Do not pre-resolve which is correct; the CO decides based on the vendor's LCAT description (title-weighted vs. experience-weighted).

```python
def price_reasonableness(lcat_keyword, proposed_rate, senior_title_keyword=None,
                         min_experience=8, price_range="30,500"):
    """Compare a proposed rate against two benchmark pools.

    lcat_keyword: the base LCAT keyword (e.g., "Software Developer")
    senior_title_keyword: optional title-match variant (e.g., "Senior Software Developer")
    min_experience: threshold for experience-match pool (default 8 yrs)
    """
    filters_exp = [f"price_range:{price_range}", f"min_years_experience:{min_experience}"]
    filters_title = [f"price_range:{price_range}"]

    # Experience-match pool (broad base keyword + experience filter)
    exp_pool = distribution_snapshot("keyword", lcat_keyword, filters=filters_exp)

    # Title-match pool (if senior_title_keyword given)
    title_pool = None
    if senior_title_keyword:
        title_pool = distribution_snapshot("keyword", senior_title_keyword, filters=filters_title)

    def position(rate, pool):
        if not pool or not pool.get("p50"):
            return None
        p25, p50, p75, p90 = pool.get("p25"), pool.get("p50"), pool.get("p75"), pool.get("p90")
        divergence = round(((rate - p50) / p50) * 100, 1) if p50 else None
        if rate < p25:
            band = "below P25"
        elif rate < p50:
            band = "P25-P50"
        elif rate < p75:
            band = "P50-P75"
        elif p90 and rate < p90:
            band = "P75-P90"
        elif p90:
            band = "above P90"
        else:
            band = "above P75"
        return {"band": band, "divergence_vs_median_pct": divergence}

    return {
        "proposed_rate": proposed_rate,
        "experience_match_pool": {**exp_pool, "position": position(proposed_rate, exp_pool)},
        "title_match_pool": ({**title_pool, "position": position(proposed_rate, title_pool)}
                             if title_pool else None),
        "note": "Pools often differ by 10-15% at median. CO decides which better matches the vendor's LCAT description.",
    }
```

### Tiered Rate Card

Auto-stratify by seniority tier for IGCE rate tables.

```python
def tiered_rate_card(lcat_family, price_range="30,500"):
    """Build a rate card across seniority tiers of a labor category family.

    Discovers real LCAT bucket names containing the family term (e.g., "Software Developer"),
    then pulls distribution_snapshot for each. Returns a table indexed by tier.
    """
    discovery = suggest_contains("labor_category", lcat_family)
    if not discovery['suggestions']:
        return {"error": f"No LCAT buckets match '{lcat_family}'."}

    card = {}
    for entry in discovery['suggestions'][:15]:  # top 15 tiers by count
        bucket_name = entry["value"]
        snap = distribution_snapshot(
            "search",
            f"labor_category:{urllib.parse.quote_plus(bucket_name)}",
            filters=[f"price_range:{price_range}"],
        )
        card[bucket_name] = {
            "count": snap.get("record_count"),
            "p25": snap.get("p25"),
            "p50": snap.get("p50"),
            "p75": snap.get("p75"),
        }
        time.sleep(0.3)
    return {
        "family": lcat_family,
        "filter_applied": f"price_range:{price_range}",
        "tiers": card,
        "total_discovered_buckets": len(discovery['suggestions']),
        "discovery_truncated": discovery.get('likely_truncated', False),
    }
```

### SIN Analysis

```python
def sin_analysis(sin_code, price_range="30,500"):
    """Pull rate distribution for a specific SIN."""
    return distribution_snapshot(
        "filter_only", None,
        filters=[f"sin:{sin_code}", f"price_range:{price_range}"],
    )
```

### Vendor Rate Card

```python
def vendor_rate_card(vendor_keyword, page_size=500):
    """Get all ceiling rates for a vendor. Discovers exact name first."""
    discovery = suggest_contains("vendor_name", vendor_keyword)
    if not discovery['suggestions']:
        return {'error': f'No vendor found matching "{vendor_keyword}".'}
    exact_name = discovery['suggestions'][0]['value']
    result = exact_search("vendor_name", exact_name, page_size=page_size,
                          ordering="labor_category", sort="asc")
    rates = [
        {'labor_category': h['_source']['labor_category'],
         'rate': h['_source']['current_price'],
         'education': h['_source']['education_level'],
         'experience': h['_source']['min_years_experience'],
         'sin': h['_source']['sin'],
         'contract': h['_source']['idv_piid']}
        for h in result.get('hits', {}).get('hits', [])
    ]
    return {'vendor': exact_name,
            'total_categories': result['hits']['total']['value'],
            'rates': rates}
```

### Multi-Category IGCE Builder

```python
import time

def build_igce(labor_categories, price_range="30,500", senior_title_prefix="Senior",
               min_experience=8):
    """Build dual-pool IGCE benchmarks for multiple labor categories.

    For each LCAT, returns both title-match and experience-match pools.
    """
    igce = {}
    for cat in labor_categories:
        try:
            igce[cat] = price_reasonableness(
                lcat_keyword=cat,
                proposed_rate=None,  # just benchmark, no comparison
                senior_title_keyword=f"{senior_title_prefix} {cat}",
                min_experience=min_experience,
                price_range=price_range,
            )
        except Exception as e:
            igce[cat] = {'error': str(e)}
        time.sleep(0.3)
    return igce
```

---

## Record Fields (for exact-search)

| Field | Type | Description |
|-------|------|-------------|
| `labor_category` | string | Awarded labor category title |
| `current_price` | float | Current NTE ceiling rate ($/hr) |
| `next_year_price` | float | Next option year rate |
| `vendor_name` | string | Contractor name |
| `idv_piid` | string | GSA MAS contract number |
| `education_level` | string | Min education requirement (messy) |
| `min_years_experience` | int | Min years experience |
| `business_size` | string | `S` (small) or `O` (other) |
| `security_clearance` | mixed | Boolean or string (messy; see Rule #4) |
| `worksite` | string | Where work is performed |
| `sin` | string | Special Item Number (case-sensitive) |
| `contract_start` / `contract_end` | string | YYYY-MM-DD |

Hit-level `_id` (string) is used for the `exclude` parameter. Source-level `id` (int) is the record ID.

---

## Aggregation Schemas (full reference)

### wage_stats

```json
{
  "count": 10927,
  "min": 28.31,
  "max": 534.25,
  "avg": 155.17,
  "std_deviation": 52.09,
  "std_deviation_bounds": {"upper": 259.36, "lower": 50.99}
}
```

`count` is the true record count; trust over `hits.total.value` when `relation="gte"`.

### histogram_percentiles

```json
{
  "values": {
    "10.0": 96.80, "25.0": 121.45, "30.0": 127.23, "35.0": 133.51,
    "50.0": 149.69, "65.0": 166.97, "70.0": 173.32, "75.0": 180.79, "90.0": 221.80
  }
}
```

Ten percentile buckets returned (10, 25, 30, 35, 50, 65, 70, 75, 90). P25, P50, P75 and P10, P90 are the IGCE-relevant ones. Use P25-P75 for the interquartile range, P10-P90 for defensibility bounds.

### median_price vs histogram_percentiles["50.0"]

Both fields present a "median," but they use different interpolation methods:
- `histogram_percentiles.values["50.0"]`: linear interpolation between the two adjacent ranked values (standard Elasticsearch percentile computation).
- `median_price.values["50.0"]`: exact-value selection (picks the observed value at rank n/2).

Numerically they usually differ by less than $1. For IGCE purposes, **use `histogram_percentiles["50.0"]` as the canonical median** (consistent with P25/P75 from the same aggregation). If a CO asks why two "medians" differ by a dollar, cite this difference in interpolation.

### education_level_counts (normalized buckets)

```json
{
  "buckets": [
    {"key": "AA", "doc_count": 2206},
    {"key": "BA", "doc_count": 32183},
    {"key": "HS", "doc_count": 2102},
    {"key": "MA", "doc_count": 1722},
    {"key": "PHD", "doc_count": 109},
    {"key": "TEC", "doc_count": 21}
  ]
}
```

Prefer this over raw `education_level` aggregation (which splits "BA"/"Bachelors" into separate buckets).

### security_clearance (NOT normalized)

Returns variant bucket keys (`true`, `Yes`, `SECRET or TS/SCI eligible`, `false`, `No`, `N/A`). Counts with `security_clearance:yes` filter are a FLOOR, not a true cleared-record count.

### Other aggregations

`business_size`, `worksite`, `min_years_experience`: standard term buckets.
`wage_histogram`: rate distribution histogram for charting.
`current_price`: price point buckets.
`labor_category`: top term buckets.

---

## Common SINs

| SIN | Description |
|-----|-------------|
| `54151S` | IT Professional Services (most general IT; very noisy) |
| `54151HACS` | Highly Adaptive Cybersecurity Services (cleared cyber) |
| `541611` | Management and Financial Consulting |
| `541715` | Engineering Research and Development |
| `541330ENG` | Engineering Services |
| `541519` | Other Computer Related Services |
| `541690` | Other Scientific and Technical Consulting |
| `541511` | Custom Computer Programming |
| `541512` | Computer Systems Design |
| `541513` | Computer Facilities Management |
| `541610` | Management Consulting |
| `561210FAC` | Facilities Maintenance and Management |
| `611430` | Training |

SIN codes are case-sensitive. Use official GSA casing exactly (`54151HACS` not `54151hacs`).

---

## Excluding Outliers

Use `exclude` with hit-level `_id` values (pipe-delimited): `&exclude=6275099|6275111`

Excluded records are removed from both hits and aggregation stats.

For systematic outlier removal in IGCE work, use `price_range` filter instead (e.g., `price_range:30,500`) and document the bound in the memo.

---

## CSV Export

Append `&export=y` to any query URL. CSV includes metadata header rows before actual data; skip to the line starting with `Contract #,`.

---

## Pagination

Max `page_size=500`. For aggregation-only queries use `page_size=1` (see Rule #7). Paginate only when you need actual records.

```python
def paginate_all(keyword, filters=None, page_size=500):
    """Retrieve all results by paginating. For record-level analysis only."""
    all_hits, page = [], 1
    while True:
        result = keyword_search(keyword, page=page, page_size=page_size, filters=filters)
        hits = result.get('hits', {}).get('hits', [])
        all_hits.extend(hits)
        total_info = result['hits']['total']
        # aggregations.wage_stats.count is truth when hits.total caps at 10K
        true_total = (result.get('aggregations', {}).get('wage_stats', {}).get('count',
                      total_info['value']) if total_info['relation'] == 'gte'
                      else total_info['value'])
        if len(all_hits) >= true_total or not hits:
            break
        page += 1
        time.sleep(0.3)
    return all_hits, result.get('aggregations', {})
```

---

## Pagination caps and `hits.total`

Reading record counts correctly:

| Condition | `hits.total.value` | `aggregations.wage_stats.count` | Which to trust |
|---|---|---|---|
| Results ≤ 10,000 | Exact count | Exact count | Either; they match |
| Results > 10,000 | 10000 | Actual count | **wage_stats.count** |

If `hits.total.relation == "gte"`, results exceeded the 10K cap. Always read `aggregations.wage_stats.count` for IGCE defensibility — a memo citing "10,000" when the true population is 45,000 misrepresents the sample size.

---

## Disclaimers (include in every output)

1. **Ceiling rates, not actuals.** Order-level prices should be lower per FAR 8.405-2(d).
2. **Worldwide rates.** No geographic cost adjustment.
3. **Master contract-level.** Not task-order rates.
4. **Fair and reasonable determination still required** per FAR 15.4. The CO makes the determination; this data informs it.
5. **Sample size matters.** Note the `wage_stats.count` in any analysis. Samples under 30 are directional only.

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `wage_stats` returns `None` | Reading from top-level instead of `aggregations.wage_stats` | Use `response["aggregations"]["wage_stats"]["count"]`; see Response Shape section |
| 0 results for known labor category | Using `search=labor_category:...` with a title that doesn't exactly match any bucket | Use `suggest-contains=labor_category:<term>` first to find the real bucket name |
| 0 results for known vendor | Name variation | `suggest-contains=vendor_name:<partial>` to discover exact string |
| Inconsistent education values | Vendor PPT data quality | Use `education_level_counts` aggregation (normalized buckets) |
| `security_clearance:yes` under-counts | Filter is fuzzy partial-match, not normalized | See Rule #4; use all-clearance pool for IGCE unless the cleared subset is large enough to be defensible |
| `hits.total.relation == "gte"` | Results exceed 10K cap | Use `wage_stats.count` for true count; add filters to narrow |
| `suggest-contains` returns empty | 1-char search term | Minimum 2 characters |
| `suggest-contains` returns exactly 100 buckets | Likely truncated at default cap | Narrow the discovery term |
| Stats seem wrong or skewed | Outliers (clerical rates $10-20/hr; fake $9,999/hr sentinels) | Add `price_range` filter (e.g., `30,500`); document the bound |
| CSV starts with junk rows | Metadata header before data | Skip to line starting with `Contract #,` |
| Median from API is $1 off from your calc | `median_price` vs. `histogram_percentiles["50.0"]` interpolation differ | Use `histogram_percentiles["50.0"]` as the canonical median |
| Keyword search returns too-wide rate range | `keyword=` contaminates across labor_category + vendor_name + idv_piid | See Rule #2; switch to `suggest-contains` + `search=labor_category:<exact>` |
| 404 or non-2xx on base URL | Transient infra issue or wrong URL | Verify base URL: `https://api.gsa.gov/acquisition/calc/v3/api/ceilingrates/`. No auth. Retry once. If persistent, fall back to manual validation at https://buy.gsa.gov/pricing/qr/mas. |

---

## Quick Reference: Recommended Workflow for IGCE Rate Analysis

```
1. Discover:   suggest_contains("labor_category", "Software Developer")
               → returns [Software Developer, Senior SWD, SWD I, II, III, ...]

2. Stratify:   tiered_rate_card("Software Developer")
               → dual-pool by tier; filters price_range:30,500

3. Position:   price_reasonableness(
                   lcat_keyword="Software Developer",
                   proposed_rate=185.00,
                   senior_title_keyword="Senior Software Developer",
                   min_experience=8,
                   price_range="30,500"
               )
               → returns both pools, percentile positions, divergence from median

4. Narrate:    Cite CALC+, record count, P50 of chosen pool, percentile position,
               and standard disclaimers. Note which pool (title vs experience match)
               was chosen and why.
```

---

*MIT © James Jenrette / 1102tools. Source: github.com/1102tools/federal-contracting-skills*
