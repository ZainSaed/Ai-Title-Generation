# AI Title Generation Engine for eBay Listings (n8n Workflow)

An autonomous n8n automation that generates accurate, SEO-optimized eBay Australia listing titles for Breville replacement parts. The workflow cross-references live eBay competitor listings with official Breville product data, then uses an LLM agent with a strict multi-step verification chain to synthesize a single, structurally validated title — falling back to manual review whenever the available data cannot be verified.

Built to eliminate manual title-writing for high-volume eBay product listings while avoiding the two most common failure modes of AI-generated content: hallucinated compatibility claims and inconsistent formatting.

---

## Overview

Writing accurate, keyword-rich eBay titles at scale is a bottleneck for sellers managing large parts catalogs. Manually researching part numbers, verified compatibility, and competitor SEO wording for every SKU does not scale, and naive LLM prompting introduces a real risk: models will confidently invent compatible machine models or part numbers that were never verified, creating listings that mislead buyers and risk platform strikes.

This project solves that with a **source-of-truth pipeline**: every fact used in a generated title must trace back to either the input SKU data, live eBay competitor consensus, or official Breville product data — never model inference alone. A dedicated LLM validation step checks the generated title against a structural and factual checklist before it is accepted, and rejects it back to a manual-review flag if it fails.

## Key Features

- **Dual-source data pipeline** — parallel eBay Browse API searches and live Breville product-page scraping run concurrently per SKU
- **Source-priority resolution logic** — eBay competitor consensus is treated as primary for naming and compatibility, with official Breville data as a verified fallback when eBay results are absent
- **Identifier whitelisting** — part and model numbers are only ever pulled from the original SKU input or confirmed Breville data, never inferred from unverified eBay listings
- **Deduplication logic** — prevents repeated identifiers, phrases, or compatibility claims across merged sources
- **Structural self-validation** — the LLM agent checks its own output against a formatting and factuality checklist before returning a result
- **Manual-review fallback** — any row with insufficient verified data, or a title that fails validation, is flagged instead of silently publishing a guess
- **Batch processing with paired-item tracking** — iterates through an entire Google Sheet of SKUs in a single execution using n8n's loop pattern, with correct per-row data pairing across iterations
- **Google Sheets as source and sink** — reads pending SKUs and writes back generated titles or manual-review flags directly

## Architecture

```
Google Sheet (SKU input)
        |
        v
  Batch Loop (per row)
        |
        +---> eBay Browse API search (SKU 1)
        +---> eBay Browse API search (SKU 2)
        +---> Breville product page scrape
        |
        v
  Data normalization (JavaScript)
        |
        v
  LLM Agent (Gemini)
    - Source priority resolution
    - Identifier whitelisting
    - Deduplication
    - Structural validation
        |
        v
  Google Sheet (title written back, or flagged for manual review)
        |
        v
  Loop continues to next row
```

## Tech Stack

| Layer | Technology |
|---|---|
| Orchestration | n8n |
| Data source / sink | Google Sheets API |
| Market data | eBay Browse API (OAuth2 client credentials) |
| Brand data | HTTP scraping (Breville product pages) |
| Title generation | Google Gemini (LangChain LLM Chain) |
| Logic / normalization | JavaScript (n8n Code nodes) |

## How It Works

1. **Read pending rows** — the workflow pulls all rows with SKU 1 / SKU 2 values from a connected Google Sheet.
2. **Batch loop** — each row is processed individually using n8n's Split In Batches pattern, with paired-item references ensuring the correct SKU data flows through every downstream node for that specific iteration.
3. **Parallel research** — for each row, the workflow concurrently searches eBay for both SKUs and scrapes the corresponding Breville product page.
4. **Normalization** — a JavaScript node deduplicates and structures eBay titles, Breville product details, and confirmed compatibility data into a clean payload.
5. **Conditional routing** — rows with no usable data from either source are routed directly to a manual-review write, skipping the LLM call entirely.
6. **LLM synthesis** — the remaining rows are passed to Gemini with a system prompt enforcing a strict, ordered decision procedure: verify facts, prioritize eBay over Breville, resolve conflicts, deduplicate, construct the title, validate structure, and only then output.
7. **Write-back** — the final title (or a manual-review flag) is written back to the originating row using row-number matching for reliability.
8. **Loop continuation** — the workflow proceeds automatically to the next row until the sheet is fully processed.

## Prompt Engineering Highlights

The LLM system prompt is structured as a sequential, auditable decision procedure rather than a flat instruction list:

- Explicit source-priority rules with a defined verification bar (agreement across multiple independent listings before a fact is treated as confirmed)
- A hard separation between identifier data (never eBay-sourced) and naming/compatibility data (eBay-prioritized, Breville-fallback)
- A mandatory pre-output structural validation checklist, forcing the model to reject its own output and fall back to a safe default rather than emit a malformed or unverifiable title
- Zero tolerance for invented compatibility claims — the single most common failure mode in unguarded LLM-generated e-commerce content

## Setup

1. Import `ai-title-generation.json` into an n8n instance (self-hosted or cloud).
2. Configure credentials:
   - Google Sheets OAuth2
   - eBay API HTTP Basic Auth (client credentials)
   - Google Gemini API key
3. Point the Google Sheets nodes at a sheet with `SKU 1`, `SKU 2`, and `Titles` columns.
4. Run manually via the trigger node, or attach a schedule trigger for recurring batch processing.

## Repository Contents

```
ai-title-generation.json   n8n workflow export
README.md                  project documentation
```

Built as a production-grade eBay listing automation exercise in reliable, verifiable LLM pipeline design — prioritizing factual accuracy over generation speed.
