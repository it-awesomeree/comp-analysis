# `Shopee-mylinks-sales-data-merged.py` — Complete Script Documentation

**Location:** `VM3 → C:\Users\Admin\Desktop\Shopee Comp My links Api\Shopee-mylinks-sales-data-merged.py`
**Size:** 2,006 lines, 81,274 bytes | **Last modified:** 2026-03-05 13:20:27
**Language:** Python (self-contained, no local imports)
**Dependencies:** `mysql-connector-python`, `requests`, `hmac`, `hashlib`, `json`, `re`, `os`, `time`, `datetime`, `logging`, `traceback`, `concurrent.futures`

---

## 1. Purpose

This script is **Job 5** of a 7-script nightly Shopee Competitive Analysis pipeline orchestrated by `scheduler.py` on VM3. It is a **2-phase updater** that enriches the `AllBots.Shopee_Comp` table:

1. **Shopee Product Info** — Fetches product details (name, description, price, stock, rating, images, SKU) from the Shopee Partner API and writes them to existing rows
2. **SiteGiant SKU Sales** — Fetches order data from SiteGiant's multi-channel OMS API, aggregates units sold by SKU across rolling periods, and writes 7d/14d/30d/60d/90d sales figures

**In simple terms:** This script answers two questions:
- *"What are the current details (price, stock, rating, images) of our products on Shopee?"*
- *"How many units of each SKU did we sell across all SiteGiant-connected channels?"*

**Critical distinction:** This script populates the `our_sales_*` column family (sourced from SiteGiant orders, matched by SKU). These are **independent** from the `shopee_var_sales_*` and `shopee_var_value_*` columns populated by Job 4 (sourced from Shopee's own order API, matched by model ID). The `our_sales` column (single, no suffix) is a third metric — lifetime total sales from the Shopee product listing.

---

## 2. Pipeline Position & Data Dependencies

```
scheduler.py (orchestrator, runs nightly via Windows Task Scheduler)
│
├─ Job 1: ca_shopee_listing_to_db.py          (SEED — imports listings)
├─ Job 2: our_variation_preprocessing.py       (backfills our_variation)
├─ Job 3: ca_ai_variation_match.py             (AI variation matching)
├─ Job 4: shopee_comp_shopee_sales.py          (Shopee order sales sync)
├─ Job 5: Shopee-mylinks-sales-data-merged.py  ← THIS SCRIPT
├─ Job 6: ca_shopee_ads_metrics.py             (ads metrics)
└─ Job 7: ca_similarity_check.py               (AI similarity scoring)
```

- **Depends on:** Jobs 1-3 must have populated `our_link`, `our_variation`, `our_shop_id`, `our_item_id` for this script to find candidate rows and resolve models.
- **Depends on:** Job 4's model resolution (Phase 2) helps this script's SKU matching, though this script can also resolve SKUs independently via Shopee API.
- **Scheduling:** Triggered nightly via `scheduler.py`. Has a 6-hour safety timeout. On failure, retried up to 2 times (3 total attempts) with token refresh between retries.

---

## 3. Configuration Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `ONLY_UPDATE_WHEN_EMPTY` | `True` | COALESCE mode — only fill blank fields for normal rows |
| `NONE_SKIP_DAYS` | `5` | Skip NONE rows that were updated within 5 days |
| `PARALLEL_WORKERS` | `3` | SiteGiant parallel order detail fetchers |
| `SG_BATCH_SIZE` | `100` | Orders per SiteGiant API page |
| `SG_REQUEST_TIMEOUT` | `90` | Seconds per SiteGiant API call |
| `SG_MAX_RETRIES` | `3` | SiteGiant API retry limit |
| `DB_MAX_RETRIES` | `6` | Database write retry limit |
| `DB_LOCK_WAIT_TIMEOUT` | `120` | InnoDB lock wait timeout (seconds) |
| `TMP_INSERT_CHUNK_SIZE` | `1000` | Rows per INSERT into temp table |

---

## 4. Execution Flow (Step by Step)

```
main()
├── Phase 1: run_shopee_api_updates()           → Shopee product info enrichment
│   ├── fetch_candidate_rows()                  → Get rows needing updates
│   ├── For each (shop_id, item_id) group:
│   │   ├── URL canonicalization + writeback
│   │   ├── Shopee API: get_item_base_info
│   │   ├── Shopee API: get_item_extra_info
│   │   ├── Shopee API: get_model_list
│   │   ├── Product-level field write
│   │   ├── Model-level matching + field write
│   │   └── NONE row date_taken update
│   └── Summary logging
│
└── Phase 2: run_sitegiant_sales_updates()      → SiteGiant SKU sales
    ├── Load SiteGiant tokens
    ├── For each SiteGiant channel:
    │   ├── Phase A: Collect order IDs (30-day chunks, sequential)
    │   ├── Phase B: Fetch order details (parallel, 3 workers)
    │   └── Aggregate by SKU into SKUSalesData
    ├── update_sales_by_sku()                   → Temp table + UPDATE JOIN
    └── Append report to Jan_report.txt
```

---

## 5. Phase-by-Phase Breakdown

### Phase 1 — Shopee Product Info (`run_shopee_api_updates`, line 1605)

**Purpose:** Enrich `Shopee_Comp` rows with current product details from the Shopee Partner API — names, descriptions, prices, stock levels, ratings, images, and SKUs.

#### Candidate Selection (`fetch_candidate_rows`, line 1097)

Two separate queries run back-to-back:

**Query 1 — Normal filtered rows:**
```sql
SELECT id, our_link, our_variation, sku, sheet_name, date_taken,
       our_shop_id, our_item_id, our_model_id,
       product_name, our_price, our_stock, shopee_stock
FROM AllBots.Shopee_Comp
WHERE our_link IS NOT NULL AND TRIM(our_link) <> ''
  AND our_variation IS NOT NULL AND TRIM(our_variation) <> ''
  AND (
    (sheet_name IN ('VVIP','VIP','NEW_ITEMS') AND date_taken >= NOW() - INTERVAL 10 DAY)
    OR (sheet_name = 'LINKS_INPUT' AND date_taken >= NOW() - INTERVAL 4 WEEK)
  )
  -- When ONLY_UPDATE_WHEN_EMPTY = True, adds:
  AND (product_name IS NULL OR our_price IS NULL OR our_stock IS NULL
       OR shopee_stock IS NULL OR sku IS NULL OR TRIM(sku) = '' OR sku = '-')
ORDER BY id DESC
LIMIT 50000
```

**Key:** Requires both `our_link` AND `our_variation` to be non-null. This is stricter than Script 1's Phase 2 which accepts `our_variation OR sku`.

**Query 2 — NONE rows (separate pool):**
```sql
SELECT id, our_link, our_variation, sku, sheet_name, date_taken,
       our_shop_id, our_item_id, our_model_id,
       product_name, our_price, our_stock, shopee_stock
FROM AllBots.Shopee_Comp
WHERE sheet_name = 'NONE'
  AND our_link IS NOT NULL AND TRIM(our_link) <> ''
  AND our_variation IS NOT NULL AND TRIM(our_variation) <> ''
  AND (date_taken IS NULL OR date_taken < NOW() - INTERVAL 5 DAY)
ORDER BY id DESC
LIMIT 10000
```

**Key:** NONE rows are skipped if updated within 5 days (`NONE_SKIP_DAYS`).

Returns `(rows, none_row_ids)` tuple — the set of NONE row IDs is tracked separately for special overwrite logic.

#### Processing per (shop_id, item_id) Group

**Step 1 — URL canonicalization:**
Same regex patterns as Script 1's Phase 1. Cleaned URL written back to `our_link`.

**Step 2 — Shopee API calls (3 per product):**

| Endpoint | What it returns |
|----------|----------------|
| `GET /api/v2/product/get_item_base_info` | `item_name`, `description`/`extended_description`, `image.image_url_list`, `price_info` |
| `GET /api/v2/product/get_item_extra_info` | `rating_star`, `sale` (lifetime total) |
| `GET /api/v2/product/get_model_list` | All model/variation data (SKUs, prices, stock, tier options) |

**Step 3 — Field extraction helpers:**

| Helper | Source | Extraction Logic |
|--------|--------|-----------------|
| `extract_description` | `get_item_base_info` | Tries `description`, then `description_info.extended_description.field_list` (text fields only). Strips URLs via regex. |
| `extract_main_images` | `get_item_base_info` | Reads `image.image_url_list`, returns JSON array of URLs |
| `extract_item_price_text` | `get_item_base_info` | Tries `current_price`, then `price_info[0].current_price` |
| `extract_model_price` | `get_model_list` | From `price_info[0].current_price` or direct `current_price` |
| `extract_product_price_from_models` | All models | Fallback: computes min/max across models, returns range like `"12.50 - 25.00"` |
| `sum_seller_stock` | `get_model_list` | Sums `stock_info_v2.seller_stock` where `if_saleable` is True |
| `sum_shopee_stock` | `get_model_list` | Sums `stock_info_v2.shopee_stock` (all entries, handles string values) |
| `build_option_image_lookup` | `get_model_list` | Maps tier indices to variation image URLs |
| `resolve_model_option_image_url` | Per model | Looks up model's `tier_index` in the image lookup |

**Step 4 — Product-level write (per item, affects all rows sharing that item_id):**

For **normal rows** (COALESCE — fill blanks only):
```sql
UPDATE AllBots.Shopee_Comp
SET product_name     = COALESCE(%s, product_name),
    product_description = COALESCE(%s, product_description),
    our_sales        = COALESCE(%s, our_sales),
    our_rating       = COALESCE(%s, our_rating),
    our_product_price = COALESCE(%s, our_product_price),
    our_main_images  = COALESCE(%s, our_main_images)
WHERE id IN (...)
```

For **NONE rows** (direct overwrite):
```sql
UPDATE AllBots.Shopee_Comp
SET product_name     = %s,
    product_description = %s,
    our_sales        = %s,
    our_rating       = %s,
    our_product_price = %s,
    our_main_images  = %s
WHERE id IN (...)
```

**Step 5 — Model-level matching:**

| Priority | Condition | Match Strategy |
|----------|-----------|----------------|
| 1 | Row has existing SKU | Match by exact SKU against API models |
| 2 | Row has no SKU | Match by normalized variation name |
| 3 | Single model | If product has exactly 1 model, use it |

**Variation normalization (simpler than Script 1):**
```python
# Only replaces: , - _
text.replace(",", " ").replace("-", " ").replace("_", " ")
```

Does NOT replace `.`, `/`, `&`, `(`, `)` — less aggressive than Script 1's model patch normalization.

**Step 6 — Model-level write (per model, per row):**

**SKU — NEVER overwritten regardless of row type or overwrite flag:**
```sql
sku = CASE WHEN (sku IS NULL OR sku = '' OR sku = '-')
      THEN COALESCE(%s, sku) ELSE sku END
```

**Other model-level fields:**

For **normal rows** (COALESCE):
```sql
UPDATE AllBots.Shopee_Comp
SET sku = (CASE WHEN guard above),
    our_price            = COALESCE(%s, our_price),
    our_stock            = COALESCE(%s, our_stock),
    shopee_stock         = COALESCE(%s, shopee_stock),
    our_variation_images = COALESCE(%s, our_variation_images)
WHERE id = %s
```

For **NONE rows** (direct overwrite, except SKU):
```sql
UPDATE AllBots.Shopee_Comp
SET sku = (CASE WHEN guard above),
    our_price            = %s,
    our_stock            = %s,
    shopee_stock         = %s,
    our_variation_images = %s
WHERE id = %s
```

**Step 7 — Post-processing for NONE rows:**
```sql
UPDATE AllBots.Shopee_Comp SET date_taken = NOW()
WHERE id IN (... successfully processed NONE row IDs ...)
```

This resets the NONE row's `date_taken` so it won't be re-processed for another 5 days (`NONE_SKIP_DAYS`).

---

### Phase 2 — SiteGiant SKU Sales (`run_sitegiant_sales_updates`, line 1816)

**Purpose:** Fetch order data from SiteGiant's multi-channel order management system, aggregate units sold by SKU across rolling time periods, and write back to `Shopee_Comp`.

#### SiteGiant API Class (`SiteGiantAPI`, line ~700)

```python
base_url = "https://opensgapi.sitegiant.co/api/v1"
Auth: Access-Token header (static Bearer token)
Connection pool: 20 connections, 20 max size
Timeout: 90s per request

Endpoints used:
  GET  /channels              → List all connected sales channels
  GET  /orders                → List orders with filters
  POST /order/multipleOrder   → Fetch multiple order details
```

#### Channel Processing (`process_sg_channel_sku`, line 999)

**Phase A — Collect order IDs (sequential):**
- Iterates backwards in 30-day chunks from today to 90 days ago
- Filters by order status
- Collects order IDs into a list

**Phase B — Fetch order details (parallel, 3 workers):**
- Uses `concurrent.futures.ThreadPoolExecutor(max_workers=3)`
- Fetches order details via `POST /order/multipleOrder`
- Retry logic per batch on failure
- `BatchProcessingStats` tracks: success count, failure count, retry count, data loss metrics

**Valid order statuses:** `["Paid", "Completed", "Shipped", "Processed", "Processing", "Ready to Ship"]`
**Skipped statuses:** `["Unpaid", "Cancelled", "Refunded", "Failed"]`

**Note on status differences:** These are SiteGiant statuses, NOT Shopee statuses. Script 1 uses Shopee statuses (`READY_TO_SHIP`, `PROCESSED`, `SHIPPED`, `COMPLETED`). The two systems have different status naming conventions.

#### Order Parsing

**`parse_order_date`:**
- Tries fields in order: `order_time`, `created_at`, `order_date`, `paid_time`, `create_time`
- **Silent fallback:** If ALL fields are missing/unparseable, defaults to **60 days ago** with NO warning logged
- This is a potential data quality issue — orders with unparseable dates silently land in the 60-day bucket

**`extract_order_products`:**
- Tries keys in order: `products`, `items`, `item_list`, `line_items`
- Extracts `(name, sku, qty)` tuples
- Handles nested product structures from different channel formats

**`SKUSalesData` class:**
- Aggregates units sold by SKU across time periods
- Buckets: 7d, 14d, 30d, 60d, 90d (based on `parse_order_date` result)
- Each order's items increment the appropriate period counters

#### Database Update (`update_sales_by_sku`, line 1311)

Uses a **temp table + UPDATE JOIN** pattern with advisory locking:

```sql
-- 1. Acquire advisory lock (prevents concurrent SiteGiant sales writers)
SELECT GET_LOCK('ShopeeCompSalesUpdate', 10)

-- 2. Set session lock wait timeout
SET SESSION innodb_lock_wait_timeout = 120

-- 3. Create MEMORY temp table
CREATE TEMPORARY TABLE tmp_sku_sales (
    sku VARCHAR(255) PRIMARY KEY,
    s7 INT, s14 INT, s30 INT, s60 INT, s90 INT
) ENGINE=MEMORY

-- 4. Bulk insert matched SKU sales (chunked, 1000 rows per INSERT)
INSERT INTO tmp_sku_sales (sku, s7, s14, s30, s60, s90)
VALUES (%s, %s, %s, %s, %s, %s)
ON DUPLICATE KEY UPDATE
    s7 = s7 + VALUES(s7), s14 = s14 + VALUES(s14),
    s30 = s30 + VALUES(s30), s60 = s60 + VALUES(s60),
    s90 = s90 + VALUES(s90)

-- 5. Single UPDATE JOIN (writes all SKU sales in one statement)
UPDATE AllBots.Shopee_Comp c
JOIN tmp_sku_sales t ON c.sku = t.sku
SET c.our_sales_7d  = t.s7,
    c.our_sales_14d = t.s14,
    c.our_sales_30d = t.s30,
    c.our_sales_60d = t.s60,
    c.our_sales_90d = t.s90
WHERE (
    (c.sheet_name IN ('VVIP','VIP','NEW_ITEMS')
     AND c.date_taken >= NOW() - INTERVAL 10 DAY)
    OR (c.sheet_name = 'LINKS_INPUT'
        AND c.date_taken >= NOW() - INTERVAL 4 WEEK)
    OR c.sheet_name = 'NONE'
)

-- 6. Drop temp table
DROP TEMPORARY TABLE IF EXISTS tmp_sku_sales

-- 7. Release advisory lock
SELECT RELEASE_LOCK('ShopeeCompSalesUpdate')
```

**Retry logic:** Up to 6 attempts (`DB_MAX_RETRIES`) with exponential backoff + random jitter on any database error.

**Important:** The UPDATE JOIN matches on `sku` (exact string match). If a row's `sku` column is NULL, empty, or doesn't match any SiteGiant order SKU, its `our_sales_*` columns are NOT updated (they retain their previous values).

#### Report Output

Appends to `Jan_report.txt` (currently ~19MB) with:
- Per-channel SKU breakdown
- Batch processing stats (success/failure/retry counts)
- Data loss warnings (orders with unparseable dates, missing SKUs)
- Timestamp and run summary

**Note:** This file grows without bound — it is never truncated or rotated. Manual cleanup may be needed periodically.

---

## 6. Database Reads & Writes (Complete)

### Reads

| # | Database.Table | Query Summary | When | Phase |
|---|---------------|---------------|------|-------|
| 1 | `AllBots.Shopee_Comp` | Normal rows: `our_link + our_variation` non-null + sheet/date filter + empty field check | Phase 1 start | Candidate selection |
| 2 | `AllBots.Shopee_Comp` | NONE rows: `sheet_name='NONE'` + not updated within 5 days | Phase 1 start | Candidate selection |
| 3 | `requestDatabase.ShopeeTokens` | `SELECT shop_id, access_token, refresh_token` | Phase 1 start | API auth tokens |
| 4 | `requestDatabase.ShopeeTokens` | `SELECT refresh_token WHERE shop_id = %s` | On HTTP 403 | Token refresh |
| 5 | SiteGiant token source (DB or config) | SiteGiant API access tokens | Phase 2 start | SiteGiant auth |

### Writes

| # | Database.Table | Operation | When | Phase |
|---|---------------|-----------|------|-------|
| 1 | `AllBots.Shopee_Comp` | `UPDATE SET our_link` | Per item group | Phase 1 (URL cleanup) |
| 2 | `AllBots.Shopee_Comp` | `UPDATE SET product_name, product_description, our_sales, our_rating, our_product_price, our_main_images` | Per item group | Phase 1 (product-level) |
| 3 | `AllBots.Shopee_Comp` | `UPDATE SET sku, our_price, our_stock, shopee_stock, our_variation_images` | Per model/row | Phase 1 (model-level) |
| 4 | `AllBots.Shopee_Comp` | `UPDATE SET date_taken = NOW()` | Per NONE row batch | Phase 1 (post-process) |
| 5 | `AllBots.Shopee_Comp` | `UPDATE JOIN SET our_sales_7d/14d/30d/60d/90d` via temp table | After all channels | Phase 2 (SKU sales) |
| 6 | `requestDatabase.ShopeeTokens` | `UPDATE SET access_token, refresh_token, expires_at, updated_at` | On token refresh | Phase 1 |

### Columns Written to `Shopee_Comp`

| Column | Phase | Description | Write Mode |
|--------|-------|-------------|------------|
| `our_link` | 1 | Canonical Shopee URL | Always overwrite |
| `product_name` | 1 | Product listing name | COALESCE (normal) / Overwrite (NONE) |
| `product_description` | 1 | Product description text | COALESCE (normal) / Overwrite (NONE) |
| `our_sales` | 1 | Lifetime total sales (from Shopee listing) | COALESCE (normal) / Overwrite (NONE) |
| `our_rating` | 1 | Star rating (float) | COALESCE (normal) / Overwrite (NONE) |
| `our_product_price` | 1 | Product-level price (string, may be range) | COALESCE (normal) / Overwrite (NONE) |
| `our_main_images` | 1 | JSON array of product image URLs | COALESCE (normal) / Overwrite (NONE) |
| `sku` | 1 | Seller SKU | **NEVER overwritten** (special CASE WHEN guard) |
| `our_price` | 1 | Model-level price (float) | COALESCE (normal) / Overwrite (NONE) |
| `our_stock` | 1 | Seller stock count | COALESCE (normal) / Overwrite (NONE) |
| `shopee_stock` | 1 | Shopee warehouse stock count | COALESCE (normal) / Overwrite (NONE) |
| `our_variation_images` | 1 | JSON array of variation image URLs | COALESCE (normal) / Overwrite (NONE) |
| `date_taken` | 1 | Reset to NOW() for processed NONE rows | Only NONE rows |
| `our_sales_7d` | 2 | Units sold via SiteGiant in last 7 days | Direct SET |
| `our_sales_14d` | 2 | Units sold via SiteGiant in last 14 days | Direct SET |
| `our_sales_30d` | 2 | Units sold via SiteGiant in last 30 days | Direct SET |
| `our_sales_60d` | 2 | Units sold via SiteGiant in last 60 days | Direct SET |
| `our_sales_90d` | 2 | Units sold via SiteGiant in last 90 days | Direct SET |

---

## 7. NONE Row Special Handling

NONE rows (imported from competitor analysis with `sheet_name='NONE'`) receive different treatment:

| Behavior | Normal Rows | NONE Rows |
|----------|-------------|-----------|
| **Candidate selection** | Based on sheet/date filter | Skipped if `date_taken` within 5 days |
| **Product field writes** | COALESCE (fill blanks only) | Direct overwrite |
| **SKU write** | COALESCE (fill blanks only) | **NEVER overwritten** (same guard) |
| **Model field writes** | COALESCE (fill blanks only) | Direct overwrite |
| **`date_taken` update** | Not updated | Set to `NOW()` after processing |
| **SiteGiant sales scope** | Included (based on sheet/date filter) | Always included |
| **Row limit** | 50,000 | 10,000 (separate pool) |

**Why SKU is never overwritten:** The SKU is often manually set or comes from a trusted source. Overwriting it with an API-derived value could break SiteGiant sales matching (Phase 2), which relies on exact SKU string matches.

---

## 8. Shopee Partner API Usage

| Endpoint | Granularity | Batching | Rate Limit |
|----------|------------|----------|------------|
| `/api/v2/product/get_item_base_info` | 1 call per item batch | Up to 50 item_ids | 150ms sleep |
| `/api/v2/product/get_item_extra_info` | 1 call per item batch | Up to 50 item_ids | 150ms sleep |
| `/api/v2/product/get_model_list` | **1 call per item** | NOT batched | 150ms sleep |
| `/api/v2/auth/access_token/get` | 1 call per shop (on demand) | N/A | Only on 403 |

**API authentication:** HMAC-SHA256 signing. Signature = `SHA256(partner_id + api_path + timestamp + access_token + shop_id)`. Partner ID: `2012161`.

---

## 9. SiteGiant API Usage

| Endpoint | Method | Purpose | Pagination |
|----------|--------|---------|------------|
| `/channels` | GET | List all connected sales channels | No |
| `/orders` | GET | List order IDs with date/status filters | Yes (offset-based, 100/page) |
| `/order/multipleOrder` | POST | Fetch multiple order details | Batched (100 order IDs) |

**Authentication:** Static `Access-Token` header (Bearer token). No refresh mechanism — tokens are long-lived.

**Connection pool:** 20 connections, 20 max size.

**Order fetching strategy:**
- Phase A (sequential): Walk backwards in 30-day chunks, collecting order IDs
- Phase B (parallel): Fetch order details using 3 concurrent workers

---

## 10. Token Management & 403 Recovery

Same pattern as Script 1 (Job 4):

```
On Shopee API HTTP 403:
  1. Read refresh_token from requestDatabase.ShopeeTokens
  2. Call POST /api/v2/auth/access_token/get
  3. Update DB with new access_token + refresh_token + expires_at
  4. Retry failed API call
```

SiteGiant API uses static tokens — no refresh mechanism needed.

---

## 11. Error Handling & Resilience

### Advisory Lock (Phase 2 DB write)

```sql
SELECT GET_LOCK('ShopeeCompSalesUpdate', 10)
-- ... all temp table + UPDATE JOIN operations ...
SELECT RELEASE_LOCK('ShopeeCompSalesUpdate')
```

Prevents concurrent instances of this script (or any other script using the same lock name) from writing SiteGiant sales simultaneously.

### InnoDB Lock Wait Timeout

```sql
SET SESSION innodb_lock_wait_timeout = 120
```

Set per-session before the UPDATE JOIN to allow longer waits during heavy concurrent DB activity.

### Database Retry Logic (Phase 2)

```python
# DB_MAX_RETRIES = 6
# Exponential backoff with random jitter
# Retries on any database error during the temp table + UPDATE JOIN sequence
```

### SiteGiant API Retry Logic

```python
# SG_MAX_RETRIES = 3
# Per-batch retry on HTTP errors or timeouts
# BatchProcessingStats tracks retry counts for reporting
```

### Batch Processing Stats

The `BatchProcessingStats` class tracks per-channel metrics:
- `orders_fetched`: Total order IDs collected
- `details_success`: Order details fetched successfully
- `details_failed`: Order details that failed all retries
- `details_retried`: Order details that required retries
- `skus_extracted`: Unique SKUs found
- `data_loss_orders`: Orders where date parsing failed (silent 60-day fallback)

### Retry Summary

| Component | Max Retries | Backoff | Trigger |
|-----------|-------------|---------|---------|
| Shopee API | 3 | Fixed delay | HTTP errors, 403 |
| SiteGiant API | 3 (`SG_MAX_RETRIES`) | Per-batch | HTTP errors, timeouts |
| DB write (Phase 2) | 6 (`DB_MAX_RETRIES`) | Exponential + random jitter | Any DB error |
| Scheduler per-job | 2 | Token refresh between | Non-zero exit code |

---

## 12. The Three Sales Column Sets — Critical Reference

This script writes TWO of the three sales-related column families on `Shopee_Comp`. Understanding the difference is essential for debugging:

| Column | Source | Match Key | Written By | Meaning |
|--------|--------|-----------|-----------|---------|
| `our_sales` (single) | Shopee API `get_item_extra_info` → `sale` field | Per product (item_id) | This script (Phase 1) | Lifetime total units sold on Shopee listing page |
| `our_sales_7d` through `our_sales_90d` | SiteGiant API orders | `sku` (exact string match) | This script (Phase 2) | Units sold across SiteGiant channels in rolling periods |
| `shopee_var_sales_7d` through `shopee_var_sales_90d` | Shopee API orders | `(our_shop_id, our_item_id, our_model_id)` | Script 1 / Job 4 | Units sold on Shopee marketplace in rolling periods |
| `shopee_var_value_7d` through `shopee_var_value_90d` | Shopee API orders | `(our_shop_id, our_item_id, our_model_id)` | Script 1 / Job 4 | Revenue (RM) on Shopee marketplace in rolling periods |

**Why they can differ:**
- `our_sales_*` (SiteGiant) may include non-Shopee channel sales
- `shopee_var_sales_*` (Shopee) only includes Shopee marketplace orders
- `our_sales` (lifetime) is a cumulative total, not a rolling window
- Match keys differ — SiteGiant matches by SKU string, Shopee matches by model ID triple

---

## 13. Variation Normalization Differences

This script uses simpler normalization than Script 1 (Job 4):

| Script | Replacements | Example: `Red (Large)` |
|--------|-------------|----------------------|
| This script (Job 5) | `, - _` → space | `Red (Large)` (parens preserved) |
| Script 1 (Job 4) model_patch | `, - _ . / & ( )` → space | `Red Large` (parens removed) |

**Implication:** If a variation name contains `.`, `/`, `&`, `(`, or `)`, the two scripts may normalize it differently, potentially leading to different model matching results.

---

## 14. Date Filter Reference

| Context | Sheet Types | Lookback |
|---------|-------------|----------|
| Phase 1 candidate selection | VVIP/VIP/NEW_ITEMS | 10 days |
| Phase 1 candidate selection | LINKS_INPUT | 4 weeks |
| Phase 1 NONE row selection | NONE | All (but skip if updated within 5 days) |
| Phase 2 UPDATE JOIN WHERE clause | VVIP/VIP/NEW_ITEMS | 10 days |
| Phase 2 UPDATE JOIN WHERE clause | LINKS_INPUT | 4 weeks |
| Phase 2 UPDATE JOIN WHERE clause | NONE | All (always included) |

---

## 15. Common Failure Modes

| Symptom | Likely Cause | How to Debug |
|---------|-------------|--------------|
| `our_sales_*` all zeros for a product | SKU not populated or doesn't match SiteGiant order SKUs | Check `sku` column; compare with SiteGiant order data |
| Product info not updating (price/stock/name still null) | Row doesn't meet candidate criteria — `our_variation` null, or `ONLY_UPDATE_WHEN_EMPTY=True` and fields already populated | Check `our_variation` is non-null; check if fields already have values |
| NONE rows not getting updated | Updated within 5 days (`NONE_SKIP_DAYS`) | Check `date_taken` — must be >5 days ago or NULL |
| SiteGiant sales seem too low | Orders with unparseable dates silently default to "60 days ago" | Check `Jan_report.txt` for data loss warnings; investigate SiteGiant API response format |
| `Jan_report.txt` growing unbounded | Report appends every run, never truncated | Currently ~19MB; needs manual cleanup |
| SKU not being written even though API has it | SKU write protection — existing non-empty SKU is NEVER overwritten | This is by design; clear the SKU manually if it's wrong |
| Advisory lock timeout | Another instance of this script (or a script using `ShopeeCompSalesUpdate` lock) is still running | Check for concurrent processes on VM3 |
| SiteGiant API returning empty orders | Channel may be disconnected or API token expired | Verify channel status in SiteGiant dashboard; check token validity |
| Model-level fields (price/stock) not matching | Variation normalization mismatch — this script's simpler normalization missed a match | Compare variation names with and without `.`, `/`, `&`, `()` characters |

---

## 16. External Dependencies

| Dependency | Impact if Down |
|------------|----------------|
| MySQL at `34.142.159.230` (Kaushal SQL Machine) | Complete failure — no reads or writes possible |
| Shopee Partner API v2 | Phase 1 fails — product info not updated |
| SiteGiant API (`opensgapi.sitegiant.co`) | Phase 2 fails — `our_sales_*` columns not updated |
| `employee.awesomeree.com.my` | Pre-job token refresh by scheduler fails; Phase 1 may fail on 403 |

---

## 17. Logging & Reporting

- **Log module:** Python `logging`
- **Phase 1:** Logs per-item progress, API errors, write counts
- **Phase 2:** Logs per-channel processing, batch stats
- **Report file:** `Jan_report.txt` (append-only, ~19MB)
  - Per-SKU breakdown of sales by period
  - Batch processing stats per channel
  - Data loss warnings
  - Run timestamp and summary
- **Warning:** `Jan_report.txt` is never rotated or truncated — grows ~200KB+ per run
