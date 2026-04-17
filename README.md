# 1102tools Claude Skills

Claude Skills for federal acquisition professionals.

Website: [1102tools.com](https://1102tools.com)

![Architecture diagram showing how each instrument chains scope, pricing, and data sources. Market Research Builder pulls from USASpending and SAM.gov. FAR contracts: SOW/PWS Builder feeds three IGCE Builders (FFP, LH/T&M, Cost-Reimbursement) pulling from BLS OEWS, GSA CALC+, and GSA Per Diem. Other Transactions: OT Project Description Builder feeds OT Cost Analysis pulling from the same three data sources. Grants and Cooperative Agreements: Grants Budget Builder pulling from BLS OEWS and GSA Per Diem. Also available as standalone reference skills: eCFR, Federal Register, and Regulations.gov.](docs/architecture.png?v=2026-04-17c)

**Before you build:** Not every acquisition capability should be an AI tool. Dozens of potential skills were evaluated and several were intentionally excluded because they cross the line from data assembly into professional judgment. See **[AI-BOUNDARIES.md](AI-BOUNDARIES.md)**.

## Skills

### API Data Sources

| Skill | Key | Description |
|-------|-----|-------------|
| [USASpending API](skills/usaspending-api) | No key | Federal contract and award data. PIIDs, vendor awards, transaction histories, agency spending. |
| [USASpending Reference](skills/usaspending-api-reference) | No key | Filter tables, composite workflows, bulk download, vendor dedup. Install alongside main skill. |
| [GSA CALC+ Ceiling Rates](skills/gsa-calc-ceilingrates) | No key | Awarded NTE hourly rates from GSA MAS contracts (230K+ records). |
| [GSA CALC+ Reference](skills/gsa-calc-ceilingrates-reference) | No key | Aggregation schemas, IGCE benchmarking, price reasonableness. Install alongside main skill. |
| [BLS OEWS Wages](skills/bls-oews-api) | BLS key | Market wage data covering ~830 occupations across 530+ metro areas. |
| [BLS OEWS Reference](skills/bls-oews-api-reference) | No key | Query recipes, SOC code lookup, IGCE rate derivation. Install alongside main skill. |
| [GSA Per Diem Rates](skills/gsa-perdiem-rates) | api.data.gov | Federal travel per diem (lodging + M&IE) for all CONUS locations. |
| [GSA Per Diem Reference](skills/gsa-perdiem-rates-reference) | No key | Travel cost recipes, common rates table, IGCE travel formula. Install alongside main skill. |
| [Federal Register API](skills/federalregister-api) | No key | All Federal Register documents since 1994. Proposed rules, final rules, notices, executive orders. |
| [Federal Register Reference](skills/federalregister-api-reference) | No key | Composite workflows, FAR case history, regulatory timeline. Install alongside main skill. |
| [eCFR Lookup](skills/ecfr-api) | No key | Full current CFR text, updated daily. FAR/DFARS clauses, version comparison back to 2017. |
| [eCFR Reference](skills/ecfr-api-reference) | No key | Title 48 chapter map, common FAR sections, composite workflows. Install alongside main skill. |
| [Regulations.gov](skills/regulationsgov-api) | api.data.gov | Federal rulemaking dockets, proposed rules, public comments, docket histories. |
| [Regulations.gov Reference](skills/regulationsgov-api-reference) | No key | Comment tracker, FAR case history, regulatory monitor. Install alongside main skill. |
| [SAM.gov API](skills/sam-gov-api) | SAM.gov | Entity registration (UEI/CAGE), exclusion/debarment records, contract opportunities, contract awards (FPDS replacement). |
| [SAM.gov Reference](skills/sam-gov-api-reference) | No key | Entity and award schemas, business type codes, composite workflows. Install alongside main skill. |

### Orchestration Skills

| Skill | Key | Requires | Description |
|-------|-----|----------|-------------|
| [SOW/PWS Builder](skills/sow-pws-builder) | No key | -- | Structured scope decision tree producing contract-file-ready SOW or PWS. FAR 37.102(d) compliant: staffing handoff for the IGCE Builder is delivered as chat output, never embedded in the document body. |
| [IGCE Builder: FFP](skills/igce-builder-ffp) | No key* | BLS OEWS, GSA CALC+, GSA Per Diem | Firm-fixed-price IGCEs with layered wrap rate model (fringe, overhead, G&A, profit). |
| [IGCE Builder: LH/T&M](skills/igce-builder-lh-tm) | No key* | BLS OEWS, GSA CALC+, GSA Per Diem | Labor Hour and T&M IGCEs with burden multiplier pricing. |
| [IGCE Builder: Cost-Reimbursement](skills/igce-builder-cr) | No key* | BLS OEWS, GSA CALC+, GSA Per Diem | CPFF, CPAF, CPIF IGCEs with fee structure analysis and statutory fee caps. |
| [Market Research Builder](skills/market-research-builder) | No key* | USASpending API | FAR Part 10 market research report from USASpending data. |

### Other Transaction (OT) Skills

| Skill | Key | Requires | Description |
|-------|-----|----------|-------------|
| [OT Project Description Builder](skills/ot-project-description-builder) | No key | -- | Milestone-based project descriptions for prototype OT agreements under 10 USC 4021/4022. Replaces the SOW/PWS for OTs: structures work around TRL progression phases and go/no-go gates instead of task/subtask CLINs. Handles NDC, small business, traditional (with cost sharing), and consortium-brokered agreements. Produces a .docx agreement attachment and a chat-only milestone handoff table for the OT Cost Analysis. |
| [OT Cost Analysis](skills/ot-cost-analysis) | No key* | BLS OEWS, GSA CALC+, GSA Per Diem | Should-cost estimates and price reasonableness analyses for OT agreements. Milestone-based pricing citing 10 USC 4021 instead of FAR 15.404. Handles cost-sharing math (10 USC 4022(d)), consortium management fees, fixed-price and cost-type milestone payments, and pre-solicitation budget planning. Produces a formula-driven .xlsx workbook with scenario analysis and a price reasonableness memo for the agreement file. |

### Grants and Cooperative Agreement Skill

| Skill | Key | Requires | Description |
|-------|-----|----------|-------------|
| [Grants Budget Builder](skills/grants-budget-builder) | No key* | BLS OEWS, GSA Per Diem | Federal grant budget estimates aligned to 2 CFR 200 / SF-424A. Calculates personnel costs with percent-effort, fringe by appointment type, travel via Per Diem, and indirect costs via NICRA or de minimis rate. |

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

## License

MIT
