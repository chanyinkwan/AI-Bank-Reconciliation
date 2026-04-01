# AI-Powered Bank Statement Reconciliation

> Automated rental payment reconciliation using AI agents, reducing manual review from 15+ hours/month to under 30 minutes.

## Client

**Meridian Property Management** (anonymised) -- A London-based property management company overseeing 200+ residential rental units across South East England.

| Detail | Value |
|--------|-------|
| Industry | Property Management / Real Estate |
| Size | 12 staff, 200+ units |
| Location | London, UK |
| Monthly transactions | ~800 rental payments |

## The Problem

Meridian's finance team manually reconciled bank statements against tenant records every month. The process involved:

- Downloading CSV bank statements from their business account
- Cross-referencing each transaction against tenant lease agreements
- Flagging discrepancies: underpayments, late payments, unrecognised transfers
- Creating a summary report for the property managers

**Pain:** 15+ hours of manual work per month. Two staff members dedicated 2 full days each. Error rate was ~5% due to human fatigue. Late payment follow-ups were delayed by 1-2 weeks because reconciliation was always behind.

## The Solution

An n8n workflow that watches for new bank statement files, feeds them to an AI agent (GPT-4o) with access to tenant and property records, and produces a structured reconciliation report with flagged anomalies.

### How It Works

```
Bank Statement (CSV)
       |
       v
[File Watcher] --> [Read & Parse CSV]
       |
       v
[AI Reconciliation Agent]
  |-- Tool: Get Tenant Details
  |-- Tool: Get Property Details
  |-- Structured Output Parser
       |
       v
[Reconciliation Report]
  |-- Matched payments
  |-- Flagged discrepancies
  |-- Unrecognised transactions
       |
       v
[Append to Master Spreadsheet]
```

### Architecture

| Component | Technology |
|-----------|-----------|
| Orchestration | n8n (self-hosted) |
| AI Model | OpenAI GPT-4o |
| Data Source | CSV bank statements |
| Output | Structured JSON -> Google Sheets |
| Triggers | Local file watcher (new CSV) |

## Key Design Decisions

- **Why an AI agent instead of rule-based matching?** Tenant names on bank transfers rarely match exactly (e.g., "J. Smith" vs "John Smith", reference numbers in wrong fields). The AI agent handles fuzzy matching that would require complex regex otherwise.
- **Why structured output parsing?** The reconciliation report must be machine-readable for downstream processing. Structured output ensures consistent schema regardless of input variation.
- **Why file-based trigger?** Meridian downloads statements manually from their bank portal. Automating the bank API integration was out of scope for Phase 1.

## Results

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Time per reconciliation | 15+ hours/month | 20-35 minutes | ~95% reduction |
| Error rate | ~5% | ~0.8% (human-reviewed) | Significantly reduced |
| Late payment follow-up delay | 1-2 weeks | Same day | Noticeable improvement in tenant satisfaction |
| Staff hours freed | 0 | ~28-30 hrs/month | Reallocated to tenant relations |

*Note: Results measured over 3-month pilot period. Error rate varies by bank format -- Barclays CSVs had cleaner data than Lloyds.*

## Setup

### Prerequisites
- Docker & Docker Compose
- OpenAI API key

### Quick Start
```bash
docker compose up -d
# Open n8n at http://localhost:5678
# Import workflow/workflow.json
# Set OpenAI API key in credentials
# Drop a test CSV into the watched folder
```

### Environment Variables
| Variable | Description |
|----------|-------------|
| `OPENAI_API_KEY` | Your OpenAI API key |
| `WATCH_FOLDER` | Path to folder for incoming bank statements |

## Example Run

### Input: `sample-data/sample_bank_statement.csv`

| Date | Description | Amount | Reference | Balance |
|------|-------------|--------|-----------|---------|
| 01/02/2025 | SMITH J RENT FEB | 950.00 | FLAT-4B | 24,350.00 |
| 04/02/2025 | A KOWALSKI | 1,350.00 | WEST-1F | 31,762.68 |
| 06/02/2025 | J SMITH | 900.00 | | 34,255.18 |
| 10/02/2025 | TRANSFER FROM KENSINGTON | 2,500.00 | TXN-90821 | 35,730.18 |

### AI Reconciliation Output

The agent processes each transaction and returns structured JSON:

```json
[
  {
    "transaction": "SMITH J RENT FEB",
    "amount": 950.00,
    "matched_tenant": "John Smith",
    "property": "14 Albion Road, Flat 4B",
    "confidence": 0.95,
    "status": "matched",
    "reason": "Name matches tenant record, reference FLAT-4B matches expected pattern, amount matches expected rent of £950.00"
  },
  {
    "transaction": "A KOWALSKI",
    "amount": 1350.00,
    "matched_tenant": "Anya Kowalski",
    "property": "55 Westfield Avenue, First Floor",
    "confidence": 0.72,
    "status": "review",
    "reason": "Surname match on 'Kowalski', initial 'A' consistent with 'Anya'. No reference provided but amount matches expected rent of £1,350.00. Flagged for review due to missing reference."
  },
  {
    "transaction": "J SMITH",
    "amount": 900.00,
    "matched_tenant": "John Smith",
    "property": "14 Albion Road, Flat 4B",
    "confidence": 0.68,
    "status": "flagged",
    "reason": "Likely match to John Smith (Flat 4B) but amount £900.00 is £50.00 below expected rent of £950.00. Possible underpayment. No reference provided."
  },
  {
    "transaction": "TRANSFER FROM KENSINGTON",
    "amount": 2500.00,
    "matched_tenant": null,
    "property": null,
    "confidence": 0.00,
    "status": "unrecognised",
    "reason": "No tenant or property match found. Description does not match any known tenant name or reference pattern. Requires manual review."
  }
]
```

### Summary Report

```
===============================================
  RECONCILIATION SUMMARY -- February 2025
===============================================

  Total transactions:     20
  Rent payments:          14

  Matched (auto):         10  (71%)
  Review queue:            2  (14%)
  Unrecognised:            2  (14%)

  FLAGGED ITEMS:
  - Underpayment:  J SMITH -- £900.00 paid, £950.00 expected (Flat 4B)
  - Duplicate:     PATEL PRIYA FEB RENT -- second payment of £1,100.00
                   on 07/02 (first received 01/02, ref FLAT-2A)

  Non-rent transactions:   6  (utilities, maintenance, insurance)
===============================================
```

## Challenges & Iteration

### V1 failed: monolithic prompt
The first version sent the entire bank statement + all tenant records to GPT-4o in a single prompt. It worked for ~50 transactions but hallucinated matches above 100 rows (context window saturation). **Fix:** Switched to batch processing -- 20 transactions at a time, each with only the top-5 candidate tenant matches from a pre-filter step.

### CSV format hell
Meridian's tenants paid from 4 different banks. Each bank exported CSVs with different column names, date formats, and encoding. Barclays used `DD/MM/YYYY`, Lloyds used `YYYY-MM-DD`, and one tenant's bank exported with BOM characters that broke the parser silently. **Fix:** Added a format detection node that normalises CSVs before feeding to the AI. Took 2 days of debugging to find the BOM issue.

### Fuzzy matching confidence threshold
Initial threshold of 0.8 confidence missed ~12% of valid matches (tenants who put flat numbers in different formats: "Flat 2A" vs "2a" vs "F2A"). Lowering to 0.6 caught them but introduced false positives. **Fix:** Two-pass approach -- 0.8 threshold for auto-match, 0.6-0.8 goes to a "review queue" for human verification.

### Duplicate payments
Two tenants set up standing orders twice. The system matched both payments to the same lease and didn't flag the overpayment. **Fix:** Added a post-reconciliation check that flags when total payments per tenant exceed expected rent by >5%.

## Constraints & Trade-offs

- **Why CSV file drop instead of bank API?** Meridian's bank (NatWest Business) charges for API access. CSV download is free and takes 2 minutes. Automating the bank connection wasn't worth the cost for their volume.
- **Why Google Sheets output instead of a database?** The property managers live in Sheets. They rejected a dashboard prototype ("we just want it in the spreadsheet we already use"). Lesson: meet users where they are.
- **Why not rule-based matching first?** We tried. Rules covered ~70% of cases. The remaining 30% had too many variations in payment references to codify. AI handles the long tail.

## Edge Cases & Error Handling

| Scenario | What happens | Recovery |
|----------|-------------|----------|
| CSV parsing fails (corrupt file) | Workflow halts, Slack alert sent to finance team | Manual review, re-download statement |
| OpenAI API timeout | 3 retries with exponential backoff, then Slack alert | Falls back to queue for next run |
| Unrecognised transaction (no match above 0.6) | Flagged in "Unknown" tab of output sheet | Finance team manually categorises |
| Duplicate tenant names (e.g., two "J. Smith") | Property address used as tiebreaker | If still ambiguous, flagged for human review |
| Statement with zero transactions | Skipped with info log (holiday months) | No action needed |

## Monitoring & Maintenance

- **Slack alerts** on workflow failure, parsing errors, or >10% unmatched transactions (anomaly signal)
- **Weekly summary** auto-sent: total matched, total flagged, total unrecognised
- **Monthly review:** Finance team checks the "Unknown" tab. Persistent unknowns get added to the tenant lookup as aliases (e.g., "J Smith" -> "John Smith, Flat 4B")
- **Knowledge base update:** When a new tenant moves in, they add them to the tenant sheet. The workflow picks up the new data on next run.

## Customisation

- **Different bank format:** Modify the CSV parsing node to match your bank's export format
- **Different property management system:** Replace the tenant/property lookup tools with your own data source
- **Add email alerts:** Connect a notification node after the flagged discrepancies output

## Tech Stack

`n8n` `OpenAI GPT-4o` `Structured Output Parsing` `CSV Processing` `Google Sheets` `Docker`

## Lessons Learned

1. AI agents excel at fuzzy matching tasks that are tedious for humans but hard to codify as rules
2. Structured output parsing is essential for any workflow where AI output feeds into downstream systems
3. The "human reviews AI output" pattern builds trust faster than full automation

---

*Built by [Kessog Chan](https://linkedin.com/in/kessogchan) -- AI Solutions | Workflow Automation | Finance & Banking*
