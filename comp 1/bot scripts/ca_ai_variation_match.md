# ca_ai_variation_match.py — Documentation

**Location**: VM3 — `C:\Users\Admin\Desktop\Shopee Comp My links Api\ca_ai_variation_match.py`
**Size**: 57,838 bytes | 1,681 lines | Last modified: 2026-03-18 15:48:35
**Language**: Python (self-contained, no local imports)
**Dependencies**: `requests`, `mysql-connector-python`, `hmac`, `hashlib`, `json`, `re`, `os`, `time`, `datetime`, `logging`, `traceback`

---

## Purpose

A **post-scrape data enrichment pipeline** for the Shopee MY Competitive Analysis (CA) system. Upstream scrapers populate the `AllBots.Shopee_Comp` table with raw competitor product data — messy URLs, free-text variation names, partial SKUs. This script runs after scraping to **resolve those messy fields into structured Shopee product identifiers** (shop_id, item_id, model_id, SKU, shop_name) so the CA dashboard can properly link competitor products to your own catalog.

It uses **deterministic matching first** (exact variation/SKU lookups against the Shopee Partner API), then falls back to **OpenAI-powered fuzzy matching** (o4-mini) for ambiguous variation names that couldn't be resolved deterministically.

On VM3, `scheduler.py` runs this as Job 3 of the current 5-script daily chain, started by the 4:00 PM Task Scheduler trigger.

---

## Live Verification (2026-03-19)

- Verified against VM3 live file: `C:\Users\Admin\Desktop\Shopee Comp My links Api\ca_ai_variation_match.py`
- Scheduler context unchanged: Job #3 in the 4:00 PM VM3 pipeline
- Critical documentation corrections from live code:
  - Phase 1 URL patch is constrained to rows with `date_taken >= NOW() - INTERVAL 4 DAY`
  - Phase 2 (`model_patch`) is date/sheet filtered:
    - `VVIP` / `VIP` / `NEW_ITEMS`: 7 days
    - `LINKS_INPUT`: 28 days
  - Phase 2B (`ai_patch`) uses the same 7-day / 28-day filter window
  - Phase 3 (`shop_name_patch`) also uses the same 7-day / 28-day filter window
  - Older statements that Phase 2/2B run without date filters are no longer accurate
  - Phase 0 zero-sales cleanup remains disabled in `main()`
  - `DRY_RUN` constant is currently `False` in live code (despite stale comments implying otherwise)

## Architecture Overview

Sequential pipeline — `main()` calls each phase in order:

```
main()
├── fetch_tokens_by_shop_id()        # Load OAuth tokens from DB
├── url_patch_main()                 # Phase 1: URL normalization
├── model_patch_main(shop_names)     # Phase 2: Deterministic model resolution
├── ai_patch_main()                  # Phase 2B: AI fuzzy variation matching
└── shop_name_patch_main(shop_names) # Phase 3: Shop name enrichment
```

Phase 0 (delete zero-sales rows) exists in code but is **currently disabled** (commented out in `main()` as of March 5, 2026).

---

## Configuration Constants (lines ~80–155)

| Constant | Value | Notes |
|---|---|---|
| `OPENAI_KEY` | `os.environ["OPENAI_API_KEY"]` | Crashes at import if missing |
| `OPENAI_MODEL` | `o4-mini` | |
| `SHOPEE_API_HOST` | `https://partner.shopeemobile.com` | |
| `SHOPEE_PARTNER_ID` | `2012161` | |
| `DB_PASSWORD` | `os.environ["DB_PASSWORD"]` | Crashes at import if missing |
| `DB_CFG_ALLBOTS` | host: `34.142.159.230`, db: `AllBots` | GCP MySQL |
| `DB_CFG_TOKENS` | host: `34.142.159.230`, db: `requestDatabase` | Same host, different DB |
| `TABLE_NAME` | `Shopee_Comp` | |
| `DRY_RUN` | `False` | Currently live mode |
| `AI_PATCH_CONFIDENCE_THRESHOLD` | `90` | Minimum confidence to apply AI match |
| `AI_PATCH_MAX_VARIATIONS_PER_PROMPT` | `20` | Max variations per OpenAI call |

---

## Phase-by-Phase Breakdown

### Phase 0 — Zero-Sales Cleanup (DISABLED)

**Status**: Code exists but is commented out in `main()`. Was active on March 5 (deleted 1,435 rows), disabled by March 5 13:09 code update.

**SQL**:
```sql
-- Read: count total and zero-sales
SELECT COUNT(*) FROM AllBots.Shopee_Comp
SELECT COUNT(*) FROM AllBots.Shopee_Comp WHERE comp_monthly_sales = 0

-- Write: delete zero-sales rows
DELETE FROM AllBots.Shopee_Comp WHERE comp_monthly_sales = 0
```

**Critical note**: No date filter — hits the **entire table**. This is why it was likely disabled.

---

### Phase 1 — URL Normalization (lines ~580–750)

**Purpose**: Canonicalize messy Shopee product URLs into the standard format `https://shopee.com.my/product/<shop_id>/<item_id>`.

**SQL Read**:
```sql
SELECT id, our_link, date_taken FROM Shopee_Comp
WHERE date_taken IS NOT NULL
  AND date_taken >= (NOW() - INTERVAL 4 DAY)
  AND our_link IS NOT NULL AND TRIM(our_link) <> ''
```

**SQL Write**:
```sql
UPDATE Shopee_Comp SET our_link = %s WHERE id = %s
```

**URL transformations performed**:
- `i.<shop_id>.<item_id>` shorthand → full canonical URL
- Query string removal (`?sp_atk=...`, `?xptdk=...`)
- Fragment removal (`#...`)
- Domain normalization
- Path cleanup (trailing slashes, double slashes)
- Extracts `shop_id` and `item_id` from the URL for downstream phases

**Date filter**: Last 4 days only.

---

### Phase 2 — Deterministic Model Resolution (lines ~750–1100)

**Purpose**: For rows that have a variation name and/or SKU but are missing `our_model_id`, call the Shopee Partner API to get the product's model list and match deterministically.

**SQL Read**:
```sql
SELECT id, our_link, our_variation, sku, sheet_name, date_taken,
       our_shop_id, our_item_id, our_model_id
FROM AllBots.Shopee_Comp
WHERE (our_link IS NOT NULL AND TRIM(our_link) <> '')
  AND ((our_variation IS NOT NULL AND TRIM(our_variation) <> '')
       OR (sku IS NOT NULL AND TRIM(sku) <> '' AND sku <> '-'))
  AND (our_shop_id IS NULL OR our_item_id IS NULL
       OR our_model_id IS NULL OR sku IS NULL
       OR TRIM(sku) = '' OR sku = '-')
  AND date_taken IS NOT NULL
ORDER BY date_taken DESC, id DESC
LIMIT 50000
```

**Important (live script)**: Phase 2 now enforces sheet/date filtering:
- `VVIP` / `VIP` / `NEW_ITEMS`: last 7 days
- `LINKS_INPUT`: last 28 days

**SQL Write** (conditional — only fills empty fields):
```sql
UPDATE Shopee_Comp
SET our_shop_id  = CASE WHEN our_shop_id IS NULL THEN %s ELSE our_shop_id END,
    our_item_id  = CASE WHEN our_item_id IS NULL THEN %s ELSE our_item_id END,
    our_model_id = CASE WHEN our_model_id IS NULL OR our_model_id = 0 THEN %s ELSE our_model_id END,
    sku          = CASE WHEN sku IS NULL OR TRIM(sku) = '' OR sku = '-' THEN %s ELSE sku END
WHERE id = %s
```

**Matching priority** (for each product's model list from API):
1. **variation + SKU** — both match a model (highest confidence)
2. **variation only** — variation text matches a model name
3. **SKU only** — SKU matches a model's seller_sku
4. **single model fallback** — product has only one model, assign it
5. **no match** — skip row

**Variation text normalization**: Lowercased, stripped, tier values joined with ` ` (space separator).

**API call**: `GET /api/v2/product/get_model_list?shop_id=X&item_id=Y` with HMAC-SHA256 signature.

**Grouping**: Rows grouped by `(shop_id, item_id)` to avoid redundant API calls.

---

### Phase 2B — AI Fuzzy Variation Matching (lines ~1100–1500)

**Purpose**: For rows that Phase 2 couldn't resolve (still missing `our_model_id`), use OpenAI to fuzzy-match the scraped variation text against the Shopee API's known variations.

**SQL Read**:
```sql
SELECT id, our_link, our_variation, sku, sheet_name, date_taken,
       our_shop_id, our_item_id, our_model_id
FROM AllBots.Shopee_Comp
WHERE (our_link IS NOT NULL AND TRIM(our_link) <> '')
  AND (our_variation IS NOT NULL AND TRIM(our_variation) <> ''
       AND our_variation <> '-')
  AND (our_model_id IS NULL OR our_model_id = 0)
  AND date_taken IS NOT NULL
ORDER BY date_taken DESC, id DESC
LIMIT 50000
```

**Key difference from Phase 2**: Explicitly excludes `our_variation = '-'` rows. Only targets rows that Phase 2 left unresolved.

**SQL Write**: Same conditional CASE WHEN pattern as Phase 2.

**OpenAI prompt structure**:
```
System: You are a product-variation matcher. Given a list of
        known variations and a list of scraped variation texts,
        return JSON mapping each scraped text to its best match
        with a confidence score 0-100.

User: Known variations: [list from Shopee API]
      Scraped variations: [list from DB rows]
```

**Confidence threshold**: Only applies matches with confidence >= 90.

**Batching**: Up to 20 variations per OpenAI call (`AI_PATCH_MAX_VARIATIONS_PER_PROMPT`).

**Variation text handling difference from Phase 2**: Phase 2 normalizes variation text (lowercase, strip). Phase 2B uses raw text for the lookup dictionary but sends normalized text to OpenAI, creating a potential mismatch.

---

### Phase 3 — Shop Name Enrichment (lines ~1500–1700)

**Purpose**: Fill in the `shop_name` column for rows that have a `our_shop_id` but no shop name.

**SQL Read**:
```sql
SELECT id, our_shop_id FROM AllBots.Shopee_Comp
WHERE (shop_name IS NULL OR TRIM(shop_name) = '')
  AND our_shop_id IS NOT NULL AND our_shop_id <> 0
  AND date_taken IS NOT NULL
  AND ((sheet_name IN ('VVIP','VIP','NEW_ITEMS')
        AND date_taken >= (NOW() - INTERVAL 7 DAY))
       OR (sheet_name = 'LINKS_INPUT'
           AND date_taken >= (NOW() - INTERVAL 28 DAY)))
```

**Live note**: Phase 2, Phase 2B, and Phase 3 all enforce sheet/date windows in the current VM3 script.

**SQL Write (live behavior)**:
Live script update: the current VM3 version performs row-wise updates with periodic batch commits; the older JOIN safety-net pass is no longer present.

---

## Token Management (lines ~320–500)

**`fetch_tokens_by_shop_id()`**:
```sql
SELECT shop_id, access_token, shop_name FROM requestDatabase.ShopeeTokens
```
Returns two dictionaries: `tokens_by_shop_id` and `shop_names_by_shop_id`.

**`refresh_shop_token_via_api(shop_id)`**:
- Reads current `refresh_token` from `requestDatabase.ShopeeTokens`
- Calls `POST /api/v2/auth/access_token/get` with partner credentials
- Updates `access_token`, `refresh_token`, `expires_at` in DB

**`run_with_token_retry(shop_id, tokens, work_fn, retries=1)`**:
- Executes `work_fn` with current token
- On `ShopeeForbiddenError` (HTTP 403): refreshes token, retries once
- Returns `work_fn` result or `None` on failure

---

## Shopee API Authentication (lines ~280–310)

- HMAC-SHA256 signature: `hmac_sha256_hex(partner_id + path + timestamp + access_token + shop_id)`
- Signature appended as query parameter `sign` to every API request
- Custom exception `ShopeeForbiddenError` raised on HTTP 403 to trigger token refresh

---

## Complete Database Read/Write Summary

| Database.Table | Operation | Phase | Columns/Purpose |
|---|---|---|---|
| `AllBots.Shopee_Comp` | DELETE | 0 (disabled) | `WHERE comp_monthly_sales = 0` |
| `AllBots.Shopee_Comp` | SELECT | 1 | id, our_link, date_taken |
| `AllBots.Shopee_Comp` | UPDATE | 1 | our_link (canonicalized URL) |
| `AllBots.Shopee_Comp` | SELECT | 2 | id, our_link, our_variation, sku, sheet_name, date_taken, our_shop_id, our_item_id, our_model_id |
| `AllBots.Shopee_Comp` | UPDATE | 2 | our_shop_id, our_item_id, our_model_id, sku (conditional fill) |
| `AllBots.Shopee_Comp` | SELECT | 2B | Same as Phase 2, filtered to unresolved rows |
| `AllBots.Shopee_Comp` | UPDATE | 2B | our_model_id, sku (conditional fill) |
| `AllBots.Shopee_Comp` | SELECT | 3 | id, our_shop_id (with sheet_name + date filters) |
| `AllBots.Shopee_Comp` | UPDATE | 3 | shop_name (loop + JOIN) |
| `requestDatabase.ShopeeTokens` | SELECT | All | shop_id, access_token, shop_name, refresh_token |
| `requestDatabase.ShopeeTokens` | UPDATE | Token refresh | access_token, refresh_token, expires_at |

---

## External API Calls

| API | Method | Endpoint | Purpose |
|---|---|---|---|
| Shopee Partner API | GET | `/api/v2/product/get_model_list` | Get product variations/models for a shop_id + item_id |
| Shopee Partner API | POST | `/api/v2/auth/access_token/get` | OAuth token refresh on 403 |
| OpenAI API | POST | `/v1/chat/completions` | Fuzzy variation matching (o4-mini model) |

---

## Data Flow

```
Upstream Scrapers
       │
       ▼
AllBots.Shopee_Comp (raw data)
       │
       ▼
Phase 1: URL Normalization
  messy URLs → canonical shopee.com.my/product/shop_id/item_id
       │
       ▼
Phase 2: Deterministic Model Resolution
  variation/SKU text → Shopee API → model_id, sku match
       │
       ▼
Phase 2B: AI Fuzzy Matching (unresolved from Phase 2)
  variation text → OpenAI o4-mini → model_id (confidence >= 90)
       │
       ▼
Phase 3: Shop Name Enrichment
  shop_id → shop_name (from tokens DB + API)
       │
       ▼
AllBots.Shopee_Comp (enriched data)
       │
       ▼
CA Dashboard (downstream consumer)
```

---

## Known Issues & Edge Cases

### 1. AI Match Lookup Bug (Tier Separator Mismatch)
The lookup dictionary in `ai_patch_build_variation_lookup` uses **raw** (un-normalized) variation text as keys, joining tier values with a space. But OpenAI sometimes returns matches with ` - ` as the tier separator. This causes valid matches to fail lookup.

**Observed in production**:
```
[WARNING] AI matched 'Shorts - Black M (40-55 kg)' not found in lookup
```

### 2. `our_variation = '-'` Processed Unnecessarily
Phase 2 processes these rows every run but they can never match any variation. Phase 2B correctly excludes them, but Phase 2 doesn't, wasting API calls and processing time.

### 3. Dead Parameter in `model_patch_main`
`model_patch_main(shop_names_by_shop_id)` accepts the parameter but never uses it internally.

### 4. Docstring vs Reality (Filter Mismatch)
Live script behavior now applies sheet/date windows across enrichment phases: Phase 1 uses a 4-day date filter, and Phases 2/2B/3 use `7d` (`VVIP`/`VIP`/`NEW_ITEMS`) plus `28d` (`LINKS_INPUT`).

### 5. MySQL Deadlocks
Observed in production (March 5 log — shop_id 422263284). Caused by concurrent pipeline scripts updating the same table. The script catches these as general exceptions and continues processing.

### 6. OpenAI Non-Determinism
Same variation can get different confidence scores across runs. "Beige" → "ColorBlock Yellow" returned confidence 70 (correctly rejected), but edge cases near the 90 threshold could flip between runs.

### 7. Phase 0 Scope (When Enabled)
The DELETE has no date filter — it hits the entire `Shopee_Comp` table.

---

## Logging

Logs go to both console and rotating file:
```
variation_match/shopee_sync_YYYYMMDD_HHMMSS_live.log
```

Log levels used:
- **INFO**: Progress updates, phase start/end, row counts
- **WARNING**: Match failures, skipped rows, lookup misses
- **ERROR**: API failures, DB errors, deadlocks

---

## Production Runtime Statistics

### March 6, 2026 (light run — mostly clean data)
| Phase | Candidates | Changes | Notes |
|---|---|---|---|
| Phase 1 | 43,985 | 0 | All URLs already clean |
| Phase 2 | 1,489 (291 groups) | 21 | |
| Phase 2B | 2 OpenAI calls | 0 | 1 match rejected (confidence 70) |
| Phase 3 | — | 21 | |
| **Total runtime** | | | **~1.75 minutes** |

### March 5, 2026 (heavy run — fresh data load)
| Phase | Candidates | Changes | Notes |
|---|---|---|---|
| Phase 0 | — | 1,435 deleted | Still active this day |
| Phase 1 | — | 5,104 | |
| Phase 2 | 6,939 (489 groups) | — | 1 deadlock (shop_id 422263284) |
| Phase 2B | 4 OpenAI calls | 49 rows (33 matched) | |
| Phase 3 | — | 5,467 | |
| **Total runtime** | | | **~8.5 minutes** |

**Note**: Phase 0 was disabled later on March 5, 2026. The 1,435 deletions above are from the last active run before the disablement.

---

## Dry Run Mode

Set `DRY_RUN = True` at the top of the script to enable safe testing. In dry-run mode:
- All SELECT queries execute normally
- All UPDATE/DELETE queries are logged but **not executed**
- API calls still fire (to validate responses)
- Log file suffix changes from `_live.log` to `_dryrun.log`

---

*Last updated: 2026-03-06*
