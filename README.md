# 1102tools Claude Skills

Claude Skills for federal contracting professionals. No subscriptions, no paywalls. Every skill in this collection uses a public federal API that's free to query.

Website: [1102tools.com](https://1102tools.com)

![Architecture diagram showing how the 1102tools Claude Skills connect. SOW/PWS Builder feeds three IGCE Builders (FFP, LH/T&M, Cost-Reimbursement) which pull from BLS OEWS, GSA CALC+, and GSA Per Diem APIs. Also in the collection: Market Research Builder uses USASpending, Grants Builder uses BLS OEWS and Per Diem, Vendor Intelligence uses SAM.gov, USASpending, and eCFR, and Regulatory Intelligence uses Federal Register, eCFR, and Regulations.gov.](docs/architecture.png)

> **Before you build:** Not every acquisition capability should be an AI tool. Dozens of potential skills were evaluated and several were intentionally excluded. Some are planned and coming. Others will never be built because they cross the line from data assembly into professional judgment -- the kind of output that would not survive a protest, would not be adopted by the workforce, and would not be worth the time to develop. Read **[AI-BOUNDARIES.md](AI-BOUNDARIES.md)** for the full reasoning. It will save you development time and your users the backlash.

## Skills

### API Data Sources

| Skill | Version | Key | Description |
|-------|---------|-----|-------------|
| [USASpending API](skills/usaspending-api) | v1.6 | No key | Federal contract and award data. PIIDs, vendor awards, transaction histories, agency spending. |
| [USASpending Reference](skills/usaspending-api-reference) | v1.6 | No key | Filter tables, composite workflows, bulk download, vendor dedup. Install alongside main skill. |
| [GSA CALC+ Ceiling Rates](skills/gsa-calc-ceilingrates) | v1.3 | No key | Awarded NTE hourly rates from GSA MAS contracts (230K+ records). |
| [GSA CALC+ Reference](skills/gsa-calc-ceilingrates-reference) | v1.3 | No key | Aggregation schemas, IGCE benchmarking, price reasonableness. Install alongside main skill. |
| [BLS OEWS Wages](skills/bls-oews-api) | v1.3 | BLS key | Market wage data covering ~830 occupations across 530+ metro areas. |
| [BLS OEWS Reference](skills/bls-oews-api-reference) | v1.3 | No key | Query recipes, SOC code lookup, IGCE rate derivation. Install alongside main skill. |
| [GSA Per Diem Rates](skills/gsa-perdiem-rates) | v1.3 | api.data.gov | Federal travel per diem (lodging + M&IE) for all CONUS locations. |
| [GSA Per Diem Reference](skills/gsa-perdiem-rates-reference) | v1.3 | No key | Travel cost recipes, common rates table, IGCE travel formula. Install alongside main skill. |
| [Federal Register API](skills/federalregister-api) | v1.3 | No key | All Federal Register documents since 1994. Proposed rules, final rules, notices, executive orders. |
| [Federal Register Reference](skills/federalregister-api-reference) | v1.3 | No key | Composite workflows, FAR case history, regulatory timeline. Install alongside main skill. |
| [eCFR Lookup](skills/ecfr-api) | v1.2 | No key | Full current CFR text, updated daily. FAR/DFARS clauses, version comparison back to 2017. |
| [eCFR Reference](skills/ecfr-api-reference) | v1.2 | No key | Title 48 chapter map, common FAR sections, composite workflows. Install alongside main skill. |
| [Regulations.gov](skills/regulationsgov-api) | v1.3 | api.data.gov | Federal rulemaking dockets, proposed rules, public comments, docket histories. |
| [Regulations.gov Reference](skills/regulationsgov-api-reference) | v1.3 | No key | Comment tracker, FAR case history, regulatory monitor. Install alongside main skill. |
| [SAM.gov API](skills/sam-gov-api) | v1.1 | SAM.gov | Entity registration (UEI/CAGE), exclusion/debarment records, contract opportunities, contract awards (FPDS replacement). |
| [SAM.gov Reference](skills/sam-gov-api-reference) | v1.1 | No key | Entity and award schemas, business type codes, composite workflows. Install alongside main skill. |

### Orchestration Skills

| Skill | Version | Key | Description |
|-------|---------|-----|-------------|
| [SOW/PWS Builder](skills/sow-pws-builder) | v1.0 | No key | Structured scope decision tree producing contract-file-ready SOW or PWS with staffing table. |
| [IGCE Builder: FFP](skills/igce-builder-ffp) | v1.1 | No key* | Firm-fixed-price IGCEs with layered wrap rate model (fringe, overhead, G&A, profit). |
| [IGCE Builder: LH/T&M](skills/igce-builder-lh-tm) | v1.1 | No key* | Labor Hour and T&M IGCEs with burden multiplier pricing. |
| [IGCE Builder: Cost-Reimbursement](skills/igce-builder-cr) | v1.1 | No key* | CPFF, CPAF, CPIF IGCEs with fee structure analysis and statutory fee caps. |
| [Market Research Builder](skills/market-research-builder) | v1.3 | No key* | FAR Part 10 market research report from USASpending data. |
| [Grants Builder](skills/grants-builder) | v1.0 | No key* | Federal grant budgets aligned to 2 CFR 200 / SF-424A. |

*Uses keys from other installed skills. No additional key needed.

## API Keys

Three free keys cover everything:

- **BLS**: Register at [data.bls.gov/registrationEngine](https://data.bls.gov/registrationEngine/) (500 queries/day)
- **api.data.gov**: Register at [api.data.gov/signup](https://api.data.gov/signup/) (1,000 req/hr; covers Per Diem + Regulations.gov)
- **SAM.gov**: Register at [SAM.gov](https://sam.gov/) (free account; API key in your profile; keys expire every 90 days)

Tell Claude to remember your keys and it will use them automatically.

## Install

1. Download and unzip a skill
2. In Claude: Customize > Skills > + > Create skill > Upload a skill > drag in each SKILL.md file individually
3. For skills with a reference companion, install both
4. Ask a question naturally. Claude reads the instructions and makes the API call.

## What's Not Here

If you're looking for a capability that doesn't exist in this collection, there are two possible reasons: it's planned, or it was evaluated and deliberately excluded. Proposal scoring, best value tradeoff analysis, responsibility determinations, negotiation strategy, OCI analysis, and several other capabilities were considered and rejected because they require warrant-level judgment that AI cannot replicate and the acquisition workforce will not accept. See **[AI-BOUNDARIES.md](AI-BOUNDARIES.md)** for the complete list and the reasoning behind each decision.

## License

MIT
