[guard-rules-reference.md](https://github.com/user-attachments/files/28590695/guard-rules-reference.md)
# DetQ — Guard Rules Reference

**Deterministic Logic Layer Specification**
*Framework Specification v1.0*

---

## Overview

Guard rules are the core of DetQ's determinism guarantee. They are enforced in code after every AI extraction pass, before SQL generation begins. They cannot be overridden by query phrasing, model confidence, or contextual inference.

The principle behind every guard rule is the same: **if the user did not explicitly say it, the system does not apply it.**

This is a stricter standard than most NLP systems follow. It means DetQ will occasionally return broader results than a human analyst might expect — but it will never return results filtered by criteria the user did not state.

---

## Guard Rule Index

| Rule | Module | Purpose |
|---|---|---|
| G-01: Numeric Intent Gate | `enforceNumericIntent.js` | Strips ungrounded numeric filters |
| G-02: Entity Guard | `criteria.js` | Wipes hallucinated issuer/ticker filters |
| G-03: Stop Token Protection | `criteria.js` | Prevents deal flags from becoming ticker matches |
| G-04: Ranking Pre-Extraction | `intentTopN.js` | Extracts sort intent before AI runs |
| G-05: UOP Trigger Lock | `criteria.js` | Restricts use-of-proceeds bucket activation |
| G-06: Demand Filter Guard | `criteria.js` | Controls book/coverage filter activation |
| G-07: Exclusion Guard | `criteria.js` | Requires explicit exclusion language |
| G-08: Pricing Flag Guard | `criteria.js` | Requires explicit pricing language |
| G-09: Rating Agency Guard | `criteria.js` | Requires explicit agency naming |
| G-10: Sector Sanity Check | `postProcessing.js` | Prevents year tokens in sector field |

---

## Rule Specifications

### G-01: Numeric Intent Gate

**Module:** `enforceNumericIntent.js`

**Problem it solves:** AI models frequently infer numeric thresholds from context. A query like "show me large deals" might produce `size_min: 500` in the extracted criteria — even though the user never specified a number.

**Rule:** Numeric filter fields are nullified unless the user's original query contained at least one of the following explicit signals:

| Signal type | Examples |
|---|---|
| Comparison words | `over`, `under`, `above`, `below`, `more than`, `less than`, `at least`, `greater than` |
| Financial units | `mm`, `bn`, `%`, `bp`, `bps` |
| Currency symbol | `$` |

**Fields governed by this rule:**
`spread_min`, `spread_max`, `size_min`, `size_max`, `book_size_min`, `x_covered_min`, `peak_min`, `wal_min`, `wal_max`, `ipt_pxd_min`, `ipt_pxd_max`

**Behavior:** If none of the explicit signals are present in the original query string, all fields in the above list are set to `null` — regardless of what the AI extracted.

---

### G-02: Entity Guard

**Module:** `criteria.js`

**Problem it solves:** AI models will sometimes extract an issuer or ticker from context, even when the user asked a general question. "Show me the largest deals from Apple" is correct extraction. "Show me the largest deals" should not produce any issuer filter.

**Rule:** `issuer` and `ticker` filter fields are only retained if the user's query contained a recognizable company name or ticker symbol. If the AI extracts an issuer but no company name appears in the original query string, the issuer field is wiped.

**Implementation note:** Entity matching uses a simple presence check against the original query — not a confidence score. If in doubt, wipe.

---

### G-03: Stop Token Protection

**Module:** `criteria.js`

**Problem it solves:** Common bond market keywords are frequently misidentified as tickers or issuers by general-purpose language models.

**Protected stop tokens:**

| Token | Correct mapping | Incorrect mapping (blocked) |
|---|---|---|
| `TLAC` | `is_tlac = true` (deal flag) | ticker/issuer filter |
| `GREEN` | `is_green = true` (deal flag) | ticker/issuer filter |
| `MANDA` / `M&A` | `is_manda = true` (deal flag) | ticker/issuer filter |
| `UOP` | triggers UOP bucket (G-05) | ticker/issuer filter |
| `A`, `M` | — | must not match from M&A tokenization |

**Rule:** Any value in `issuer` or `ticker` that matches a stop token is wiped and the appropriate deal flag is set instead.

---

### G-04: Ranking Pre-Extraction

**Module:** `intentTopN.js`

**Problem it solves:** Ranking language ("largest", "tightest spread") is the most commonly misinterpreted signal in bond market NLP. If the AI processes it, it may treat ranking words as filter criteria rather than sort instructions.

**Rule:** Ranking intent is extracted from the user query **before the AI model is called**, using deterministic keyword matching.

| User phrase | Sort field | Direction |
|---|---|---|
| `largest` / `top` / `biggest` | `size` | Descending |
| `smallest` / `lowest size` | `size` | Ascending |
| `tightest spread` / `lowest spread` | `spread` | Ascending |
| `widest spread` / `highest spread` | `spread` | Descending |
| `shortest WAL` / `lowest WAL` | `wal` | Ascending |
| `longest WAL` / `highest WAL` | `wal` | Descending |

**Top N defaults:**
- Default: 10 results
- User can specify: "top 5", "top 20", etc. — extracted pre-AI
- Maximum enforced cap: configurable per deployment (default: 50)
- Unlimited result sets are never returned

---

### G-05: UOP Trigger Lock

**Module:** `criteria.js`

**Problem it solves:** "Green bonds", "TLAC issuance", and "M&A financings" are common query topics. These map to deal-flag boolean fields — not to the use-of-proceeds classification bucket. Activating `uop_bucket` for these queries would return incorrect results.

**Rule:** The `uop_bucket` filter is only activated if the user's query explicitly contains `"UOP"` or `"use of proceeds"` (case-insensitive).

**Correct mappings for common terms:**

| User says | Correct field | Not `uop_bucket` |
|---|---|---|
| `green` / `green bond` | `is_green = true` | ✓ |
| `TLAC` | `is_tlac = true` | ✓ |
| `M&A` / `manda` / `acquisition` | `is_manda = true` | ✓ |
| `IPO financing` | `is_ipo = true` | ✓ |
| `share repurchase` / `buyback` | `is_shr_repo = true` | ✓ |
| `use of proceeds` / `UOP` | `uop_bucket` filter activated | — |

---

### G-06: Demand Filter Guard

**Module:** `criteria.js`

**Problem it solves:** Demand metrics (book size, oversubscription) are specialist filters. Applying them without explicit user intent would silently exclude many valid deals from results.

**Rule:** The following fields are only applied if the user's query contained explicit demand language:

| Field | Required trigger language |
|---|---|
| `book_size_min` | `book`, `order book`, `demand` |
| `x_covered_min` | `x covered`, `times covered`, `oversubscribed`, `covered` |
| `peak_min` | `peak`, `peak book` |

If none of the trigger phrases are present, all three fields are nullified.

---

### G-07: Exclusion Guard

**Module:** `criteria.js`

**Problem it solves:** Exclusion criteria are meaningfully different from inclusion criteria. The AI may extract exclusion-style filters when the user is simply describing a category they want to see.

**Rule:** `issuer_exclude` and `sector_exclude` fields are only retained if the user's query contained explicit exclusion language: `exclude`, `except`, `not`, `without`, `non-`, `other than`, `excluding`.

---

### G-08: Pricing Flag Guard

**Module:** `criteria.js`

**Problem it solves:** Pricing outcomes (priced through guidance, priced wider) are specific analytical filters. They should not be inferred from general spread language.

**Rule:** Pricing flag fields are only applied when the user explicitly used the corresponding phrase:

| Field | Required trigger phrase |
|---|---|
| `priced_through` | `priced through`, `through guidance` |
| `priced_at_guidance` | `priced at guidance`, `at guidance` |
| `priced_wider` | `priced wider`, `wider than guidance` |

---

### G-09: Rating Agency Guard

**Module:** `criteria.js`

**Problem it solves:** When a user says "AAA rated deals", they typically mean any AAA rating, not a specific agency. The AI may assume a default agency, producing a more restrictive filter than intended.

**Rule:** The `rating_agency` field is only set if the user explicitly named a rating agency: `Moody's`, `S&P`, `Standard & Poor's`, `Fitch`, `DBRS`, `Kroll`, `Morningstar`.

If no agency is named, `rating_agency` is null and the rating filter applies across all agencies.

---

### G-10: Sector Sanity Check

**Module:** `postProcessing.js`

**Problem it solves:** Year strings (`2024`, `Q3 2025`) in queries can occasionally be misextracted into the sector field by the AI model.

**Rule:** If `criteria.sector` contains no alphabetic characters (i.e., it is purely numeric or contains only numbers and punctuation), the field is wiped.

**Examples:**
- `"Financials"` → retained
- `"2025"` → wiped
- `"Q3"` → wiped
- `"Tech & Media"` → retained

---

## Adding New Guard Rules

When extending DetQ for a new deployment, new guard rules should follow this pattern:

1. **Define the trigger condition** — what explicit language must the user have used?
2. **Define the failure mode** — what does the AI incorrectly extract without this guard?
3. **Implement as a post-AI pass** — never modify pre-AI extraction logic to compensate
4. **Test with known failure queries** — document the queries that motivated the rule
5. **Add to this reference** — include the rule in the index and spec

Guard rules should be **additive and independent**. Each rule guards one specific failure mode. Rules should not have dependencies on each other.

---

## Testing Guard Rules

Each guard rule should have a corresponding test suite covering:

| Test type | Description |
|---|---|
| Trigger positive | Query containing explicit signal — filter should be applied |
| Trigger negative | Query without signal — filter should be null |
| Edge cases | Partial matches, case variations, punctuation |
| Stop token isolation | Confirm stop tokens never bleed into entity fields |

A failing guard rule test means the system can produce incorrect financial output. Treat guard rule failures with the same severity as SQL injection vulnerabilities.

---

*For pipeline context, see `architecture-overview.md`.*
*For schema adaptation, see `schema-mapping-guide.md`.*
