[schema-mapping-guide.md](https://github.com/user-attachments/files/28590714/schema-mapping-guide.md)
# DetQ — Schema Mapping Guide

**Adapting the Framework to Your Database**
*Integration Reference v1.0*

---

## Overview

DetQ's query pipeline is schema-agnostic by design. The framework does not hardcode your database's column names — instead, a mapping layer in `criteria.js` translates normalized field names (used throughout the pipeline) to your firm's actual database column names.

This guide explains how to perform that mapping for a new deployment covering IG, HY, and ABS markets.

---

## How Schema Mapping Works

The AI extraction stage always returns criteria using DetQ's **normalized field names** — a consistent vocabulary defined in the extraction prompt. For example:

```json
{
  "spread_min": 80,
  "spread_max": 150,
  "rating": "BBB",
  "sector": "Financials",
  "date_from": "2025-01-01"
}
```

Before SQL generation, `criteria.js` passes these normalized names through a **field mapping object** that resolves them to your actual database column names:

```javascript
// Example field map — replace right-hand values with your column names
const fieldMap = {
  spread_min:   "oas_spread",         // your column name for spread
  spread_max:   "oas_spread",
  rating:       "sp_rating",          // your S&P rating column
  sector:       "industry_sector",    // your sector classification column
  date_from:    "pricing_date",       // your primary date column
  size:         "deal_size_usd_mm",   // your deal size column
};
```

The SQL generator never sees normalized names — only your column names.

---

## Step-by-Step Schema Mapping Process

### Step 1: Inventory Your Database Schema

Run a schema inspection on your target table(s):

```sql
-- SQLite
PRAGMA table_info(your_deals_table);

-- PostgreSQL
SELECT column_name, data_type
FROM information_schema.columns
WHERE table_name = 'your_deals_table'
ORDER BY ordinal_position;
```

Export the result to a spreadsheet. You'll be matching each DetQ normalized field to one of your column names.

---

### Step 2: Map DetQ Normalized Fields

Use the table below as your mapping worksheet. For each DetQ field, find the equivalent column in your database and record it in the right column.

#### IG / HY Core Fields

| DetQ Normalized Field | Type | Description | Your Column Name |
|---|---|---|---|
| `date_from` | DATE | Start of date filter range | |
| `date_to` | DATE | End of date filter range | |
| `pricing_date` | DATE | Exact pricing date | |
| `issuer` | STRING | Issuer name | |
| `ticker` | STRING | Issuer ticker symbol | |
| `sector` | STRING | Industry sector | |
| `rating` | STRING | Credit rating (any agency) | |
| `rating_agency` | STRING | Specific rating agency | |
| `rating_sp` | STRING | S&P specific rating | |
| `rating_moodys` | STRING | Moody's specific rating | |
| `rating_fitch` | STRING | Fitch specific rating | |
| `spread_min` | NUMBER | Minimum spread (bps) | |
| `spread_max` | NUMBER | Maximum spread (bps) | |
| `size_min` | NUMBER | Minimum deal size ($mm) | |
| `size_max` | NUMBER | Maximum deal size ($mm) | |
| `maturity_years_min` | NUMBER | Minimum tenor (years) | |
| `maturity_years_max` | NUMBER | Maximum tenor (years) | |
| `book_size_min` | NUMBER | Minimum order book size | |
| `x_covered_min` | NUMBER | Minimum oversubscription | |
| `ipt_pxd` | NUMBER | IPT to pricing difference (bps) | |
| `coupon_min` | NUMBER | Minimum coupon | |
| `coupon_max` | NUMBER | Maximum coupon | |
| `is_green` | BOOLEAN | Green/ESG bond flag | |
| `is_tlac` | BOOLEAN | TLAC-eligible bond flag | |
| `is_manda` | BOOLEAN | M&A financing flag | |
| `is_ipo` | BOOLEAN | IPO financing flag | |
| `is_shr_repo` | BOOLEAN | Share repurchase flag | |
| `is_hybrid` | BOOLEAN | Hybrid instrument flag | |
| `is_frn` | BOOLEAN | Floating rate note flag | |
| `is_em` | BOOLEAN | Emerging market flag | |
| `uop_bucket` | STRING | Use of proceeds category | |
| `priced_through` | BOOLEAN | Priced through guidance flag | |
| `priced_at_guidance` | BOOLEAN | Priced at guidance flag | |
| `priced_wider` | BOOLEAN | Priced wider than guidance flag | |

#### ABS-Specific Fields

| DetQ Normalized Field | Type | Description | Your Column Name |
|---|---|---|---|
| `deal_id` | STRING | Unique deal identifier (grouping key) | |
| `announced_date` | DATE | Deal announcement date | |
| `wal_min` | NUMBER | Minimum weighted average life | |
| `wal_max` | NUMBER | Maximum weighted average life | |
| `deal_size_mm` | NUMBER | Total deal size ($mm) | |
| `tranche_size_mm` | NUMBER | Individual tranche size ($mm) | |
| `collateral` | STRING | Collateral type | |
| `reinvestment_period` | NUMBER | Reinvestment period (months) | |
| `backend_spread` | STRING | Raw spread string (e.g. "S+120") | |
| `spread_raw` | STRING | Fallback spread field | |
| `rating_dbrs` | STRING | DBRS rating | |
| `rating_kroll` | STRING | Kroll rating | |
| `rating_morningstar` | STRING | Morningstar rating | |
| `cut_off_balance_mm` | NUMBER | Cut-off balance ($mm) | |

---

### Step 3: Handle Common Naming Variations

Bond databases use inconsistent naming conventions across vendors and internal systems. The table below covers the most common variations DetQ encounters and how to handle them.

#### Spread Fields

Your database may call spread any of the following:

| Possible Column Name | Notes |
|---|---|
| `spread` | Most common in IG databases |
| `oas_spread` | Option-adjusted spread — common in Bloomberg exports |
| `g_spread` | Government spread |
| `z_spread` | Z-spread |
| `dm` | Discount margin — common for FRNs |
| `backend_spread` | Raw string format — requires `parseNumLoose()` |

**Recommendation:** If your spread is stored as a string (e.g., `"T+120"`, `"+120bps"`), implement the `parseNumLoose()` utility from the ABS module. It extracts the first numeric value from any spread string format.

#### Date Fields

| Possible Column Name | Notes |
|---|---|
| `pricing_date` | Most common |
| `trade_date` | Some platforms use trade date as primary |
| `settlement_date` | Occasionally used as primary |
| `announced_date` | ABS primary — may need fallback to `pricing_date` |

**Date fallback pattern:** If your primary date field has nulls, configure a fallback:
```javascript
const dateField = row.pricing_date ?? row.announced_date ?? row.trade_date;
```

#### Rating Fields

| Pattern | Example column names |
|---|---|
| Separate columns per agency | `sp_rating`, `moodys_rating`, `fitch_rating` |
| Combined rating column | `credit_rating`, `rating` |
| Long-form agency names | `standard_and_poors`, `moodys`, `fitch_ibca` |

If your database has a single combined rating column with an agency identifier column, the rating query logic will need a minor adaptation — documented in the integration plan.

#### Boolean / Flag Fields

Deal flags may be stored as:
- `INTEGER` (0/1) — most SQLite databases
- `BOOLEAN` (true/false) — PostgreSQL
- `VARCHAR` ('Y'/'N') — some legacy systems
- Separate lookup table — requires a JOIN

If your flags are stored as 'Y'/'N' strings, update the SQL generation to use `WHERE is_green = 'Y'` rather than `WHERE is_green = 1`.

---

### Step 4: Update the Field Map in `criteria.js`

Once your mapping is complete, update the `fieldMap` object in `criteria.js`:

```javascript
// criteria.js — update this object for your database
export const fieldMap = {

  // Dates
  date_from:              "pricing_date",
  date_to:                "pricing_date",

  // Identifiers
  issuer:                 "issuer_name",
  ticker:                 "issuer_ticker",

  // Market data
  spread_min:             "oas_spread",
  spread_max:             "oas_spread",
  size_min:               "deal_size_usd_mm",
  size_max:               "deal_size_usd_mm",

  // Ratings
  rating:                 "sp_rating",
  rating_sp:              "sp_rating",
  rating_moodys:          "moodys_rating",
  rating_fitch:           "fitch_rating",

  // Sector
  sector:                 "industry_sector",

  // Deal flags (stored as INTEGER 0/1)
  is_green:               "green_bond_flag",
  is_tlac:                "tlac_flag",
  is_manda:               "ma_financing_flag",

  // Add remaining fields here...
};
```

---

### Step 5: Configure the Grouping Key

For IG/HY markets, the default grouping key is `issuer + date + capital structure bucket`. Update this in `dealGrouping.js` to match your database's deal identifier pattern:

```javascript
// dealGrouping.js — update grouping key
const groupKey = (row) =>
  `${row.issuer_name}__${row.pricing_date}__${row.capital_structure}`;
  // Replace field names with your actual column names
```

For ABS, the grouping key must be a **unique deal identifier** — typically a deal ID, URL ID, or CUSIP prefix. Update in `postProcessing.js`:

```javascript
const absGroupKey = (row) => row.deal_id; // Replace with your unique deal identifier
```

---

### Step 6: Validate with Test Queries

Run the following validation queries after schema mapping is complete. These are designed to surface the most common mapping errors:

| Test query | What it validates |
|---|---|
| `"Top 10 largest deals this year"` | Date field, size field, deal grouping |
| `"Tightest spread BBB deals in Q3"` | Spread field, rating field, date parsing |
| `"Green bonds over $500mm"` | Boolean flag, size with explicit numeric signal |
| `"AAA rated Moody's deals"` | Rating agency guard, agency-specific field |
| `"M&A financings last month"` | Stop token protection, deal flag mapping |
| `"Deals excluding financials sector"` | Exclusion guard, sector field |
| `"Deals 10 years or more"` | Maturity field, numeric intent gate |

Expected behavior for each: correct field mapping, correct guard rule activation, no hallucinated filters.

---

## Common Mapping Errors

| Symptom | Likely cause |
|---|---|
| All queries return 0 results | Date field mapped to wrong column or wrong format |
| Rating filter not working | Rating column name mismatch or agency column missing |
| Spread filter returns wrong results | Spread stored as string — needs `parseNumLoose()` |
| Deal grouping produces duplicates | Grouping key not unique — add additional key field |
| Boolean flags not filtering | Flag stored as 'Y'/'N' — update SQL comparison |
| ABS deals missing tranches | Wrong `deal_id` column — check unique identifier mapping |

---

*For pipeline context, see `architecture-overview.md`.*
*For guard rule specifications, see `guard-rules-reference.md`.*
*For deployment phases, see `/integration-plan/phased-delivery-plan.md`.*
