[architecture-overview.md](https://github.com/user-attachments/files/28590650/architecture-overview.md)
# DetQ — Architecture Overview

**Deterministic Query Framework for Corporate Bond Analytics**
*Framework Specification v1.0*

---

## Purpose of This Document

This document describes the DetQ processing pipeline, component responsibilities, and the architectural decisions that make deterministic financial query output possible. It is intended for technical evaluators, integration engineers, and AI/ML practitioners assessing the framework for adoption.

---

## Architectural Philosophy

Most NLP-to-SQL systems rely on the language model to understand intent, map fields, and generate queries. DetQ deliberately inverts this. The AI model is scoped to one narrow task — structured field extraction from natural language — and every other decision is made deterministically in code.

This distinction matters in financial contexts because:

- **Hallucinated filters are dangerous.** A system that infers "the user probably meant deals over $500M" can silently exclude valid results.
- **Reproducibility is a compliance concern.** Analysts need to demonstrate that the same query produces the same output on different days.
- **Domain logic is not general knowledge.** A general-purpose LLM does not natively know that `TLAC` is a deal flag, not a ticker — or that `M&A` should never tokenize into single-letter issuer matches.

DetQ's answer to these constraints: use AI minimally, guard aggressively, and enforce everything in code.

---

## Processing Pipeline

```
┌─────────────────────────────────────────────────────────┐
│                     User Query (plain text)             │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│  STAGE 1: Pre-AI Intent Extraction                      │
│  • Ranking intent detection (largest, tightest, etc.)   │
│  • Top N extraction (default: 10)                       │
│  • Market selection (IG / ABS)                          │
│  • All deterministic — no model involved                │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│  STAGE 2: AI Extraction (Structured JSON Only)          │
│  • Model: local LLM (phi-3 via Ollama) or API           │
│  • Input: structured prompt with field schema           │
│  • Output: raw criteria JSON — filters only             │
│  • Model NEVER generates logic, rankings, or output     │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│  STAGE 3: Criteria Cleaning (cleanCriteria)             │
│  • Field name normalization → your DB column names      │
│  • Type coercion and format standardization             │
│  • Null-safety: unknown fields wiped, not guessed       │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│  STAGE 4: Guard Rule Enforcement                        │
│  • enforceNumericIntent — strips ungrounded numbers     │
│  • Entity guard — wipes hallucinated issuer/ticker      │
│  • Demand filter guard, pricing flag guard, UOP guard   │
│  • Rating agency guard, sector sanity check             │
│  • See guard-rules-reference.md for full specification  │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│  STAGE 5: SQL Generation                                │
│  • IG/HY: buildSQLQuery() with standard deal grouping  │
│  • ABS: raw SELECT + in-memory JS aggregation           │
│  • Market routing is deterministic — no fallback logic  │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│  STAGE 6: Post-Processing & Output                      │
│  • Deal grouping, tranche assembly, Top N enforcement   │
│  • formatResponse() → structured JSON + table          │
│  • Same output schema regardless of query type          │
└─────────────────────────────────────────────────────────┘
```

---

## Component Responsibilities

### `route.js` — Pipeline Orchestrator
The main API handler. Receives the query, sequences all pipeline stages, handles errors, and returns the final response. Does not contain business logic — delegates to specialized modules.

### `criteria.js` — AI Output Cleaning
Receives raw JSON from the AI model. Normalizes field names to match the target database schema, enforces type safety, and applies the domain-specific guard rules. This is where hallucinated or unsafe criteria are removed before they reach SQL generation.

This file is the primary integration point for schema adaptation — see `schema-mapping-guide.md`.

### `criteria_enforceNumericIntent.js` — Numeric Guard
A dedicated module that nullifies any numeric filter (spread range, size threshold, WAL bounds) unless the user's original query contained an explicit numeric signal — a comparison word, a unit, or a dollar sign. Prevents the AI from inventing thresholds based on contextual inference.

### `sql.js` — Query Builder
Constructs parameterized SQL from the cleaned criteria object. For IG/HY markets, generates grouped deal-level queries with standard ordering. For ABS, generates a broad row-fetch query and delegates aggregation to JavaScript.

### `intentTopN.js` — Pre-AI Ranking Extraction
Runs before the AI model is called. Detects ranking intent (largest, tightest, widest, shortest WAL) and sets the sort field and direction. This prevents the AI from misinterpreting ranking language as filter criteria.

### `dealGrouping.js` — IG Deal Assembly
Groups individual tranche rows into deal objects for display. Uses issuer + date + capital structure bucket as the grouping key. Ensures the UI always shows deal-level results with tranches as sub-rows.

### `postProcessing.js` — Final Criteria Pass
A cleanup pass after all guards have run. Catches any edge cases that slipped through earlier stages. Enforces field integrity before SQL generation begins.

### `formatResponse.js` — Output Formatter
Converts the raw query result into the structured JSON contract expected by the UI. Ensures consistent column presence and ordering regardless of which market or query type produced the result.

### `prompt.js` — AI Prompt Builder
Constructs the structured extraction prompt sent to the AI model. The prompt constrains the model to return only a specific JSON schema — it cannot return explanations, reasoning, or free text. Field definitions in the prompt are the single source of truth for what the AI is allowed to extract.

---

## AI Integration Design

DetQ's AI integration is intentionally narrow. The model receives:

```
System: You are a structured data extractor. Return only valid JSON matching this schema: [field definitions]. Do not infer, assume, or add fields not explicitly mentioned in the query.

User: [natural language query]
```

The model returns a flat JSON object — for example:

```json
{
  "market": "ig",
  "rating": "AAA",
  "rating_agency": null,
  "sector": null,
  "spread_min": null,
  "spread_max": null,
  "size_min": null,
  "date_from": "2025-10-01",
  "date_to": "2025-10-31",
  "is_green": false,
  "is_manda": false
}
```

Every field has a defined null state. The model is never asked to decide whether to apply a filter — it only extracts what the user said. The guard rules decide whether to keep it.

**Model choice is swappable.** The extraction prompt is model-agnostic. DetQ was validated with a local phi-3 model via Ollama for air-gapped environments, but any instruction-following model with JSON output mode (GPT-4o, Claude, Gemini) can be substituted without changing pipeline logic.

---

## Data Architecture

### IG / HY Market
- Storage: SQLite (`finsight.db`) — swappable to PostgreSQL for production scale
- Primary table: `investment_grade_deals`
- Grouping: issuer + date + capital structure bucket
- Query pattern: SQL-level grouping and filtering, JavaScript-level Top N enforcement

### ABS Market
- Storage: SQLite (`abs.db`)
- Primary table: `abs_clean` (derived from raw ingestion table)
- Grouping key: `url_id` (unique per deal — not issuer/date, which can be non-unique in ABS)
- Query pattern: broad SQL fetch (up to 5,000 rows), full aggregation in JavaScript
- Spread logic: parse from `backend_spread` first, fallback to `spread_raw`, using `parseNumLoose()` for string formats like `"S+120"` or `"L+135"`

---

## Error Handling and Fallback Behavior

| Failure Point | Behavior |
|---|---|
| AI returns malformed JSON (attempt 1) | Retry once with same prompt |
| AI returns malformed JSON (attempt 2) | Fall back to empty criteria object — return all recent deals with no filters |
| Unknown market string | Return 400 error immediately — no fallback routing |
| Criteria field not in schema | Silently wiped — never passed to SQL |
| SQL execution error | Return structured error response — never crash |
| Zero results | Return empty result set with query echo — never fabricate results |

The system never guesses. When uncertain, it returns less — not more.

---

## Key Architectural Decisions

**Why local LLM by default?**
Financial data environments often restrict external API calls. Ollama + phi-3 runs entirely on-premises with no data egress. API-based models are supported as a configuration option for firms without air-gap requirements.

**Why SQLite for the prototype?**
SQLite requires zero infrastructure, runs identically on any machine, and is sufficient for datasets up to ~10M rows with proper indexing. The query interface is fully compatible with PostgreSQL — migration requires only a connection string change and minor dialect adjustments.

**Why in-memory aggregation for ABS?**
ABS data has a complex many-to-one tranche-to-deal relationship that is expensive to express cleanly in SQL across a 400-field schema. JavaScript aggregation gives full control over spread parsing, WAL weighting, and deal assembly without complex SQL joins.

**Why pre-AI ranking extraction?**
Ranking intent ("largest", "tightest spread") is the most commonly misinterpreted signal in financial NLP. By extracting it deterministically before the AI runs, we prevent it from being confused with filter criteria — a known failure mode in general-purpose NL-to-SQL systems.

---

*For guard rule specifications, see `guard-rules-reference.md`.*
*For schema adaptation instructions, see `schema-mapping-guide.md`.*
