# 1102tools Claude Skills

Claude Skills for federal acquisition professionals.

Website: [1102tools.com](https://1102tools.com)

![Architecture diagram showing how each instrument chains scope, pricing, and data sources. FAR contracts: SOW/PWS Builder feeds three IGCE Builders (FFP, LH/T&M, Cost-Reimbursement) pulling from BLS OEWS, GSA CALC+, and GSA Per Diem. Other Transactions: OT Project Description Builder feeds OT Cost Analysis pulling from the same three data sources.](docs/architecture-v4.png)

**Before you build:** Not every acquisition capability should be an AI tool. Dozens of potential skills were evaluated and several were intentionally excluded because they cross the line from data assembly into professional judgment.

See **[AI-BOUNDARIES.md](AI-BOUNDARIES.md)**.

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
| [SOW/PWS Builder](skills/sow-pws-builder) | No key | -- | Structured scope decision tree producing contract-file-ready SOW or PWS. FAR 37.102(d) compliant: staffing handoff for the IGCE Builder is delivered as chat output, never embedded in the document body. Independently tested across 8 scenarios and 2 Claude models (83 of 84 assertions passed). [Testing Record](skills/sow-pws-builder/TESTING.md). |
| [IGCE Builder: FFP](skills/igce-builder-ffp) | No key* | BLS OEWS, GSA CALC+, GSA Per Diem | Firm-fixed-price IGCEs with layered wrap rate model (fringe, overhead, G&A, profit). |
| [IGCE Builder: LH/T&M](skills/igce-builder-lh-tm) | No key* | BLS OEWS, GSA CALC+, GSA Per Diem | Labor Hour and T&M IGCEs with burden multiplier pricing. |
| [IGCE Builder: Cost-Reimbursement](skills/igce-builder-cr) | No key* | BLS OEWS, GSA CALC+, GSA Per Diem | CPFF, CPAF, CPIF IGCEs with fee structure analysis and statutory fee caps. |

### Other Transaction (OT) Skills

| Skill | Key | Requires | Description |
|-------|-----|----------|-------------|
| [OT Project Description Builder](skills/ot-project-description-builder) | No key | -- | Milestone-based project descriptions for prototype OT agreements under 10 USC 4021/4022. Replaces the SOW/PWS for OTs: structures work around TRL progression phases and go/no-go gates instead of task/subtask CLINs. Handles NDC, small business, traditional (with cost sharing), and consortium-brokered agreements. Produces a .docx agreement attachment and a chat-only milestone handoff table for the OT Cost Analysis. |
| [OT Cost Analysis](skills/ot-cost-analysis) | No key* | BLS OEWS, GSA CALC+, GSA Per Diem | Should-cost estimates and price reasonableness analyses for OT agreements. Milestone-based pricing citing 10 USC 4021 instead of FAR 15.404. Handles cost-sharing math (10 USC 4022(d)), consortium management fees, fixed-price and cost-type milestone payments, and pre-solicitation budget planning. Produces a formula-driven .xlsx workbook with scenario analysis and a price reasonableness memo for the agreement file. |

*Uses keys from other installed skills. No additional key needed.

## MCP Servers

Want more consistent results across runs? Install the MCP servers. Each of the 8 API data source skills above also ships as a Model Context Protocol (MCP) server on PyPI. When installed in Claude Desktop or another MCP client, orchestration skills call these servers directly for deterministic tool calls instead of letting Claude generate API request code through SKILL.md instructions alone.

### What is MCP?

Model Context Protocol is Anthropic's open standard for LLM applications to communicate with external tools and data sources. MCP servers run locally, expose a set of callable tools, and return structured responses. They work in any MCP-compatible client including [Claude Desktop](https://claude.ai/download), Claude Code, Cursor, Cline, Continue, and Zed.

### Why use the MCP servers?

- **Deterministic tool calls.** Every call to `search_awards()` produces the same structured output, rather than Claude generating slightly different API request code across runs.
- **Orchestration skills prefer them.** If the SAM.gov MCP is installed, the SOW/PWS Builder and IGCE Builder skills use it for vendor lookups instead of falling back to API-call generation through SKILL.md.
- **Independently hardened.** Each server went through 3 to 6 audit rounds with live testing against the production API. See each TESTING.md for the full record.
- **Published to PyPI with Trusted Publisher.** Install via `uvx` with no manual Python environment setup.

### Install

1. Install [Claude Desktop](https://claude.ai/download) on Mac or Windows.
2. Open `claude_desktop_config.json`:
   - Mac: `~/Library/Application Support/Claude/claude_desktop_config.json`
   - Windows: `%APPDATA%\Claude\claude_desktop_config.json`
3. Paste an `mcpServers` block for each server you want. Example config for all 8:

```json
{
  "mcpServers": {
    "bls-oews": {
      "command": "uvx",
      "args": ["bls-oews-mcp"],
      "env": {"BLS_API_KEY": "your-key-here"}
    },
    "ecfr": {
      "command": "uvx",
      "args": ["ecfr-mcp"]
    },
    "federal-register": {
      "command": "uvx",
      "args": ["federal-register-mcp"]
    },
    "gsa-calc": {
      "command": "uvx",
      "args": ["gsa-calc-mcp"]
    },
    "gsa-perdiem": {
      "command": "uvx",
      "args": ["gsa-perdiem-mcp"],
      "env": {"PERDIEM_API_KEY": "your-key-here"}
    },
    "regulations-gov": {
      "command": "uvx",
      "args": ["regulationsgov-mcp"],
      "env": {"REGULATIONS_GOV_API_KEY": "your-key-here"}
    },
    "sam-gov": {
      "command": "uvx",
      "args": ["sam-gov-mcp"],
      "env": {"SAM_API_KEY": "your-key-here"}
    },
    "usaspending": {
      "command": "uvx",
      "args": ["usaspending-gov-mcp"]
    }
  }
}
```

4. Restart Claude Desktop.
5. Orchestration skills will use the installed MCP servers automatically when available.

See [API Keys](#api-keys) below for where to get free keys for the 4 servers that need them. `ecfr`, `federal-register`, `gsa-calc`, and `usaspending` need no key.

### All 8 MCP servers

Each repo ships with a TESTING.md documenting its hardening record. Click through for bug counts, audit rounds, and signature bug stories.

| MCP | Version | Tests | Source | Testing Record |
|---|---|---|---|---|
| bls-oews-mcp | 0.2.2 | 60 | [GitHub](https://github.com/1102tools/bls-oews-mcp) | [TESTING.md](https://github.com/1102tools/bls-oews-mcp/blob/main/TESTING.md) |
| ecfr-mcp | 0.2.1 | 101 | [GitHub](https://github.com/1102tools/ecfr-mcp) | [TESTING.md](https://github.com/1102tools/ecfr-mcp/blob/main/TESTING.md) |
| federal-register-mcp | 0.2.2 | 77 | [GitHub](https://github.com/1102tools/federal-register-mcp) | [TESTING.md](https://github.com/1102tools/federal-register-mcp/blob/main/TESTING.md) |
| gsa-calc-mcp | 0.2.2 | 117 | [GitHub](https://github.com/1102tools/gsa-calc-mcp) | [TESTING.md](https://github.com/1102tools/gsa-calc-mcp/blob/main/TESTING.md) |
| gsa-perdiem-mcp | 0.2.1 | 172 | [GitHub](https://github.com/1102tools/gsa-perdiem-mcp) | [TESTING.md](https://github.com/1102tools/gsa-perdiem-mcp/blob/main/TESTING.md) |
| regulationsgov-mcp | 0.2.0 | 51 | [GitHub](https://github.com/1102tools/regulationsgov-mcp) | [TESTING.md](https://github.com/1102tools/regulationsgov-mcp/blob/main/TESTING.md) |
| sam-gov-mcp | 0.3.1 | 79 | [GitHub](https://github.com/1102tools/sam-gov-mcp) | [TESTING.md](https://github.com/1102tools/sam-gov-mcp/blob/main/TESTING.md) |
| usaspending-gov-mcp | 0.2.3 | 62 | [GitHub](https://github.com/1102tools/usaspending-gov-mcp) | [TESTING.md](https://github.com/1102tools/usaspending-gov-mcp/blob/main/TESTING.md) |

Across all 8 MCPs: 719 regression tests covering roughly 350 bugs fixed during hardening.

## Testing and Validation

Skills in this repo are progressively being run through independent end-to-end validation. Each testing wave uses the following methodology:

- Scenarios are run on claude.ai web chat, the same environment real users run in.
- Each worker output is graded by a separate Claude instance (Claude Code Opus 4.7 with 1M context, max effort) reading only the final artifact and chat transcript, with no access to the worker's drafting reasoning.
- Grading is binary pass/fail against a 14-point assertion matrix covering FAR compliance, document structure, voice consistency, and handoff format.
- Findings that surface real bugs or consistent behavioral gaps become skill patches.
- Every tested skill ships with a public Testing Record documenting methodology, results, and changes made.

### Orchestration Skills Testing Records

| Skill | Scenarios | Models tested | Assertion result | Testing Record |
|-------|-----------|---------------|------------------|----------------|
| [SOW/PWS Builder](skills/sow-pws-builder) | 8 | Opus 4.7 + Sonnet 4.6 | 83 of 84 passed, 1 fixed | [Testing Record](skills/sow-pws-builder/TESTING.md) |
| [IGCE Builder: FFP](skills/igce-builder-ffp) | Pending | -- | -- | Not yet published |
| [IGCE Builder: LH/T&M](skills/igce-builder-lh-tm) | Pending | -- | -- | Not yet published |
| [IGCE Builder: Cost-Reimbursement](skills/igce-builder-cr) | Pending | -- | -- | Not yet published |

### Other Transaction (OT) Skills Testing Records

| Skill | Scenarios | Models tested | Assertion result | Testing Record |
|-------|-----------|---------------|------------------|----------------|
| [OT Project Description Builder](skills/ot-project-description-builder) | Pending | -- | -- | Not yet published |
| [OT Cost Analysis](skills/ot-cost-analysis) | Pending | -- | -- | Not yet published |

### MCP Servers Testing Records

MCP servers use a different testing methodology from the skills above: instead of scenario-graded assertions, each server was hardened through multi-round live audits targeting input validation, response-shape fragility, and silent-wrong-data bugs. Each was probed against its production API with edge cases, injection attempts, boundary values, and unusual response shapes. See each TESTING.md for the round-by-round breakdown.

| MCP | Audit rounds | Regression tests | Bugs fixed | Testing Record |
|---|---|---|---|---|
| [bls-oews-mcp](https://github.com/1102tools/bls-oews-mcp) | 5 | 60 | 22 | [TESTING.md](https://github.com/1102tools/bls-oews-mcp/blob/main/TESTING.md) |
| [ecfr-mcp](https://github.com/1102tools/ecfr-mcp) | 5 | 101 | 72 | [TESTING.md](https://github.com/1102tools/ecfr-mcp/blob/main/TESTING.md) |
| [federal-register-mcp](https://github.com/1102tools/federal-register-mcp) | 4 | 77 | ~30 | [TESTING.md](https://github.com/1102tools/federal-register-mcp/blob/main/TESTING.md) |
| [gsa-calc-mcp](https://github.com/1102tools/gsa-calc-mcp) | 4 | 117 | 86 | [TESTING.md](https://github.com/1102tools/gsa-calc-mcp/blob/main/TESTING.md) |
| [gsa-perdiem-mcp](https://github.com/1102tools/gsa-perdiem-mcp) | 6 | 172 | 55 | [TESTING.md](https://github.com/1102tools/gsa-perdiem-mcp/blob/main/TESTING.md) |
| [regulationsgov-mcp](https://github.com/1102tools/regulationsgov-mcp) | 3 | 51 | 22 | [TESTING.md](https://github.com/1102tools/regulationsgov-mcp/blob/main/TESTING.md) |
| [sam-gov-mcp](https://github.com/1102tools/sam-gov-mcp) | 4 | 79 | 46 | [TESTING.md](https://github.com/1102tools/sam-gov-mcp/blob/main/TESTING.md) |
| [usaspending-gov-mcp](https://github.com/1102tools/usaspending-gov-mcp) | 4 | 62 | 15 | [TESTING.md](https://github.com/1102tools/usaspending-gov-mcp/blob/main/TESTING.md) |

Combined across all 8 MCPs: 719 regression tests covering roughly 350 bugs fixed.

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
