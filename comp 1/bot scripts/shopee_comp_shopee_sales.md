# `shopee_comp_shopee_sales.py` — Complete Script Documentation

**Location:** `VM3 → C:\Users\Admin\Desktop\Shopee Comp My links Api\shopee_comp_shopee_sales.py`
**Size:** 2,743 lines, 98,794 bytes | **Last modified:** 2026-03-06 16:18:29
**Language:** Python (self-contained, no local imports)
**Dependencies:** `mysql-connector-python`, `requests`, `hmac`, `hashlib`, `json`, `re`, `os`, `time`, `datetime`, `logging`, `threading`, `traceback`

---

## Live Verification (2026-03-19)

- Verified against VM3 live file: `C:\Users\Admin\Desktop\Shopee Comp My links Api\shopee_comp_shopee_sales.py`
- Scheduler trigger remains in the VM3 4:00 PM daily chain (`Sales Data` task)
- Live behavior checks:
  - Phase 1 URL patch still uses `sheet_name='NONE'` OR `date_taken >= NOW() - INTERVAL 4 DAY`
  - Phase 2 model patch still uses `NONE` (all rows) + date windows for other sheets (`7d` for `VVIP/VIP/NEW_ITEMS`, `28d` for `LINKS_INPUT`)
  - Phase 3 daily sales sync still writes rolling units and rolling revenue (`shopee_var_sales_*` + `shopee_var_value_*`)
- No major logic drift was found beyond metadata/line-count growth

## 1. Purpose

This script is **Job 4** of the current 5-script daily Shopee Competitive Analysis pipeline orchestrated by `scheduler.py` on VM3. The chain starts from the daily 4:00 PM Task Scheduler trigger. It is a **3-phase data maintenance and synchronization utility** for the `AllBots.Shopee_Comp` table:

1. **URL Normalization** — Canonicalizes messy Shopee product URLs into a standard format
2. **Model Resolution** — Resolves model IDs by matching variation names/SKUs against the Shopee API
3. **Daily Sales Sync** — Syncs order-level sales data from the Shopee Partner API into daily granularity, then rolls up into 7d/14d/30d/60d/90d period windows

**In simple terms:** This script answers the question *"How many units of each product variation did we sell on Shopee each day, and what's the rolling sales total?"* It fetches real Shopee orders, breaks them down by product variation per day, and writes rolling sales + revenue windows back to the competitive analysis table.

**What it writes:** This script populates the `shopee_var_sales_*` and `shopee_var_value_*` column families on `Shopee_Comp`, sourced directly from Shopee's own order API.

---

## 2. Pipeline Position & Data Dependencies

```
scheduler.py (orchestrator, runs daily at 4:00 PM via Windows Task Scheduler)
│
├─ Job 1: ca_shopee_listing_to_db.py          (SEED — imports listings)
├─ Job 2: our_variation_preprocessing.py       (backfills our_variation)
├─ Job 3: ca_ai_variation_match.py             (AI variation matching)
├─ Job 4: shopee_comp_shopee_sales.py          ← THIS SCRIPT
├─ Job 5: Shopee-mylinks-sales-data-merged.py  (product info + SiteGiant sales)
└─ Jobs 6-7: scheduled out of the current chain (ads metrics and similarity)
```

- **Depends on:** Jobs 1-3 must have populated `our_shop_id`, `our_item_id`, and `our_model_id` for this script's Phase 3 rolling update to match rows.
- **Scheduling:** Triggered daily at 4:00 PM by Windows Task Scheduler via `scheduler.py`. Has a 6-hour safety timeout. On failure, retried up to 2 times (3 total attempts) with token refresh between retries.
- **All 5 jobs run sequentially** with 5-minute gaps between them.

---

## 3. Configuration Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `DRY_RUN` | `False` | If `True`, logs actions without DB writes |
| `SHOP_ID_FILTER` | `None` | Set to a shop_id int to process only one shop (for testing) |
| `BACKFILL_DAYS` | `90` | How far back to backfill on first sync for a shop |
| `BACKFILL_CHUNK_DAYS` | `0` | Sub-chunking for backfill (0 = disabled) |
| `MAX_WINDOW_DAYS` | `15` | Max days per Shopee API time window |
| `OVERLAP_DAYS` | `2` | Days of overlap from last sync point for boundary safety |
| `COMP_UPDATE_BATCH_SIZE` | `20000` | ID range per batch for rolling sales UPDATE |
| `PER_SHOP_TIMEOUT_SECONDS` | `1200` | 20-minute max per shop |
| `HARD_REQUEST_TIMEOUT_SECONDS` | `120` | 2-minute max per API call (threading-based) |
| `MAX_RETRIES` | `3` | API call retry limit |
| `MAX_DEADLOCK_RETRIES` | `3` | MySQL deadlock retry limit |
| `DELETE_BATCH_SIZE` | `5000` | Batch size for old record deletion |
| `URL_PATCH_BATCH_SIZE` | `50000` | Batch size for URL normalization reads |
| `MODEL_PATCH_ONLY_UPDATE_WHEN_EMPTY` | `True` | Only fill blank fields, don't overwrite |
| `LOCAL_TZ` | `UTC+8` (MYT) | Timezone for all date calculations |

---

## 4. Execution Flow (Step by Step)

```
main()
├── Phase 1: url_patch_main()           → URL normalization
├── Phase 2: model_patch_main()         → Model ID resolution
└── Phase 3: daily_sales_sync_main()    → Daily sales sync + rolling update
    ├── Step 3.1: delete_old_daily_sales()
    ├── Step 3.2: get_eligible_shops()
    ├── Step 3.3: load_tokens()
    ├── Step 3.4: sync_shop_orders() [per shop]
    │   ├── determine sync window
    │   ├── fetch orders (paginated, 15-day chunks)
    │   ├── fetch order details (batches of 50)
    │   ├── aggregate by (shop_id, item_id, model_id, sale_date)
    │   └── sync_shop_daily_sales_atomic() [atomic write]
    └── Step 3.5: update_shopee_comp_rolling_sales()
```

---

## 5. Phase-by-Phase Breakdown

### Phase 1 — URL Normalization (`url_patch_main`, line 796)

**Purpose:** Canonicalizes messy Shopee product URLs into the standard format `https://shopee.com.my/product/{shop_id}/{item_id}`.

**SQL Read:**
```sql
SELECT id, our_link FROM AllBots.Shopee_Comp
WHERE our_link IS NOT NULL
  AND (sheet_name = 'NONE' OR date_taken >= NOW() - INTERVAL 4 DAY)
LIMIT 50000
```

**Date filter:** 4-day lookback for non-NONE rows. All NONE rows included regardless of age.

**URL transformation patterns:**
```python
I_DOT_PATTERN = re.compile(r"i\.(\d+)\.(\d+)", re.IGNORECASE)           # i.123.456 slug
PRODUCT_PATH_PATTERN = re.compile(r"^/product/(\d+)/(\d+)(?:$|[/?#])")  # /product/shop/item
CANONICAL_URL_PATTERN = re.compile(r"shopee\.com\.my/product/(\d+)/(\d+)") # Already canonical
```

**Transformations performed:**
- `shopee.com.my/Some-Product-i.12345.67890` → `shopee.com.my/product/12345/67890`
- Query string removal (`?sp_atk=xxx`, `?xptdk=xxx`)
- Fragment removal (`#...`)
- Domain and path cleanup

**SQL Write:**
```sql
UPDATE AllBots.Shopee_Comp SET our_link = %s WHERE id = %s
```

**Notable:** Uses `print()` for output, NOT `logging` — output goes to stdout only, NOT to the log file.

---

### Phase 2 — Model Resolution (`model_patch_main`, line 1308)

**Purpose:** Resolves `our_model_id` by matching variation names and/or SKUs against Shopee API model lists.

**SQL Read (candidate selection):**
```sql
SELECT id, our_link, our_variation, sku, sheet_name, date_taken,
       our_shop_id, our_item_id, our_model_id
FROM AllBots.Shopee_Comp
WHERE our_link IS NOT NULL
  AND (our_variation IS NOT NULL OR sku IS NOT NULL)
  AND (sheet/date filter — see table below)
ORDER BY id DESC
LIMIT 50000
```

**Important:** Candidate selection requires `our_variation OR sku` — uses OR, meaning it can attempt matching by SKU alone even without a variation name.

**Date filters (hardcoded — different from shared config):**

| Sheet Name | Lookback | Notes |
|------------|----------|-------|
| VVIP / VIP / NEW_ITEMS | **7 days** | Shared config says 10 days — this is hardcoded differently |
| LINKS_INPUT | **28 days** | Matches shared config |
| NONE | **All rows** | No date filter |

**3-tier matching fallback:**
1. **Variation text + SKU match** — both must match a model (highest confidence)
2. **Variation text only** — first matching model by normalized name
3. **SKU only** — match against all models by SKU
4. **Single model fallback** — if product has exactly 1 model, assign it
5. **No match** — skip row

**Variation normalization (aggressive):**
```python
def model_patch_normalize_variation(variation):
    text = variation.lower()
    text = text.replace(",", " ").replace("-", " ").replace("_", " ")
              .replace(".", " ").replace("/", " ").replace("&", " ")
              .replace("(", " ").replace(")", " ")
    return " ".join(text.split())
```

Replaces `, - _ . / & ( )` with space, then collapses whitespace.

**API call:** `GET /api/v2/product/get_model_list` with 0.15s rate limiting per call.

**SQL Write (conditional — only fills empty fields when `MODEL_PATCH_ONLY_UPDATE_WHEN_EMPTY = True`):**
```sql
UPDATE AllBots.Shopee_Comp
SET our_shop_id  = CASE WHEN our_shop_id IS NULL THEN %s ELSE our_shop_id END,
    our_item_id  = CASE WHEN our_item_id IS NULL THEN %s ELSE our_item_id END,
    our_model_id = CASE WHEN our_model_id IS NULL OR our_model_id = 0 THEN %s ELSE our_model_id END,
    sku          = CASE WHEN sku IS NULL OR TRIM(sku) = '' OR sku = '-' THEN %s ELSE sku END
WHERE id = %s
```

---

### Phase 3 — Daily Sales Sync (`daily_sales_sync_main`, line 2536)

The core of this script. Syncs order-level sales data from the Shopee Partner API into a daily granularity table, then rolls up into period windows on the main table.

#### Step 3.1 — Cleanup (`delete_old_daily_sales`)

Deletes records older than 90 days from `Shopee_VariationSalesDaily`:

```sql
DELETE FROM AllBots.Shopee_VariationSalesDaily
WHERE sale_date < CURDATE() - INTERVAL 90 DAY
LIMIT 5000
```

Runs in a loop until no more rows match. Decorated with `@retry_on_deadlock`.

#### Step 3.2 — Get Eligible Shops

```sql
SELECT DISTINCT our_shop_id FROM AllBots.Shopee_Comp
WHERE our_shop_id IS NOT NULL
```

**Important:** No sheet/date filter — gets ALL shops that have ever been mapped to the table.

#### Step 3.3 — Load Tokens

```sql
SELECT shop_id, access_token, refresh_token FROM requestDatabase.ShopeeTokens
```

#### Step 3.4 — Per-Shop Sync (`sync_shop_orders`)

For each shop:

**1. Determine sync window:**
```sql
SELECT last_time_to FROM AllBots.Shopee_VariationSalesSyncState
WHERE shop_id = %s
```

- If no prior sync: backfills 90 days (`BACKFILL_DAYS`)
- Otherwise: resumes from `last_time_to - 2 days` (overlap for boundary safety)
- End: always "end of yesterday" — today's orders are never synced

**2. Fetch orders from Shopee API:**
- Endpoint: `GET /api/v2/order/get_order_list`
- Statuses: `["READY_TO_SHIP", "PROCESSED", "SHIPPED", "COMPLETED"]`
- Broken into 15-day chunks (`MAX_WINDOW_DAYS`)
- Paginated: 100 orders/page, max 500 pages = 50,000 orders safety limit
- Each call signed with HMAC-SHA256

**3. Fetch order details in batches of 50:**
- Endpoint: `GET /api/v2/order/get_order_detail`
- Parameter: `response_optional_fields=item_list`
- Hard timeout: 120s per request (threading-based wrapper)

**4. Aggregate by `(shop_id, item_id, model_id, sale_date)`:**
```python
aggregates[key]["units_sold"] += qty
aggregates[key]["revenue"] += round(qty * unit_price, 2)
aggregates[key]["orders_count"] += 1  # per unique model per order
```

Revenue calculation: `qty * model_discounted_price` (fallback: `model_original_price`)

**5. Atomic DB write (`sync_shop_daily_sales_atomic`):**
```
BEGIN TRANSACTION
  → DELETE existing data in scan window for this shop
  → INSERT/UPSERT aggregated records (sorted by PK for deadlock prevention)
  → UPSERT sync state (last_time_to = window end)
COMMIT   (or ROLLBACK on any failure)
```

**SQL — Delete existing window:**
```sql
DELETE FROM AllBots.Shopee_VariationSalesDaily
WHERE shop_id = %s AND sale_date BETWEEN %s AND %s
```

**SQL — Insert/Upsert daily records:**
```sql
INSERT INTO AllBots.Shopee_VariationSalesDaily
  (shop_id, item_id, model_id, sku, sale_date, units_sold, orders_count, revenue, last_synced_at)
VALUES (%s, %s, %s, %s, %s, %s, %s, %s, NOW())
ON DUPLICATE KEY UPDATE
  units_sold = VALUES(units_sold),
  orders_count = VALUES(orders_count),
  revenue = VALUES(revenue),
  sku = VALUES(sku),
  last_synced_at = NOW()
```

**SQL — Upsert sync state:**
```sql
INSERT INTO AllBots.Shopee_VariationSalesSyncState
  (shop_id, last_time_to, last_synced_at)
VALUES (%s, %s, NOW())
ON DUPLICATE KEY UPDATE
  last_time_to = VALUES(last_time_to),
  last_synced_at = NOW()
```

Decorated with `@retry_on_deadlock`. Records sorted by PK before insert to minimize deadlock risk.

#### Step 3.5 — Rolling Sales Update (`update_shopee_comp_rolling_sales`)

Updates the main `Shopee_Comp` table with rolling period totals computed from `Shopee_VariationSalesDaily`:

```sql
UPDATE AllBots.Shopee_Comp sc
LEFT JOIN (
    SELECT shop_id, item_id, model_id,
        SUM(CASE WHEN sale_date >= CURDATE()-7  AND sale_date <= CURDATE()-1
            THEN units_sold ELSE 0 END) AS s7,
        SUM(CASE WHEN sale_date >= CURDATE()-14 AND sale_date <= CURDATE()-1
            THEN units_sold ELSE 0 END) AS s14,
        SUM(CASE WHEN sale_date >= CURDATE()-30 AND sale_date <= CURDATE()-1
            THEN units_sold ELSE 0 END) AS s30,
        SUM(CASE WHEN sale_date >= CURDATE()-60 AND sale_date <= CURDATE()-1
            THEN units_sold ELSE 0 END) AS s60,
        SUM(CASE WHEN sale_date >= CURDATE()-90 AND sale_date <= CURDATE()-1
            THEN units_sold ELSE 0 END) AS s90,
        -- Same pattern for revenue: v7, v14, v30, v60, v90
        SUM(CASE WHEN sale_date >= CURDATE()-7  AND sale_date <= CURDATE()-1
            THEN revenue ELSE 0 END) AS v7,
        SUM(CASE WHEN sale_date >= CURDATE()-14 AND sale_date <= CURDATE()-1
            THEN revenue ELSE 0 END) AS v14,
        SUM(CASE WHEN sale_date >= CURDATE()-30 AND sale_date <= CURDATE()-1
            THEN revenue ELSE 0 END) AS v30,
        SUM(CASE WHEN sale_date >= CURDATE()-60 AND sale_date <= CURDATE()-1
            THEN revenue ELSE 0 END) AS v60,
        SUM(CASE WHEN sale_date >= CURDATE()-90 AND sale_date <= CURDATE()-1
            THEN revenue ELSE 0 END) AS v90
    FROM AllBots.Shopee_VariationSalesDaily
    WHERE sale_date >= CURDATE()-90 AND sale_date <= CURDATE()-1
    GROUP BY shop_id, item_id, model_id
) agg ON sc.our_shop_id = agg.shop_id
     AND sc.our_item_id = agg.item_id
     AND sc.our_model_id = agg.model_id
SET sc.shopee_var_sales_7d  = IFNULL(agg.s7, 0),
    sc.shopee_var_sales_14d = IFNULL(agg.s14, 0),
    sc.shopee_var_sales_30d = IFNULL(agg.s30, 0),
    sc.shopee_var_sales_60d = IFNULL(agg.s60, 0),
    sc.shopee_var_sales_90d = IFNULL(agg.s90, 0),
    sc.shopee_var_value_7d  = IFNULL(agg.v7, 0),
    sc.shopee_var_value_14d = IFNULL(agg.v14, 0),
    sc.shopee_var_value_30d = IFNULL(agg.v30, 0),
    sc.shopee_var_value_60d = IFNULL(agg.v60, 0),
    sc.shopee_var_value_90d = IFNULL(agg.v90, 0),
    sc.shopee_var_sales_last_synced = NOW()
WHERE sc.id BETWEEN %s AND %s
  AND sc.our_shop_id IS NOT NULL
  AND sc.our_item_id IS NOT NULL
  AND sc.our_model_id IS NOT NULL
```

- Batched by 20,000 ID range (`COMP_UPDATE_BATCH_SIZE`)
- Each batch committed separately
- Decorated with `@retry_on_deadlock`
- Uses `IFNULL(agg.*, 0)` — rows without matching sales data are set to 0, not left NULL

---

## 6. Database Reads & Writes (Complete)

### Reads

| # | Database.Table | Query Summary | When | Phase |
|---|---------------|---------------|------|-------|
| 1 | `AllBots.Shopee_Comp` | Rows with `our_link IS NOT NULL` + date/sheet filter | Startup | Phase 1 (URL patch) |
| 2 | `AllBots.Shopee_Comp` | Rows with `our_link` + `our_variation OR sku` + date/sheet filter | After Phase 1 | Phase 2 (model patch) |
| 3 | `AllBots.Shopee_Comp` | `SELECT DISTINCT our_shop_id WHERE NOT NULL` | Phase 3 start | Phase 3 (eligible shops) |
| 4 | `AllBots.Shopee_VariationSalesSyncState` | `SELECT last_time_to WHERE shop_id = %s` | Per-shop | Phase 3 (sync cursor) |
| 5 | `requestDatabase.ShopeeTokens` | `SELECT shop_id, access_token, refresh_token` | Phase 3 start | Phase 3 (auth tokens) |
| 6 | `requestDatabase.ShopeeTokens` | `SELECT refresh_token WHERE shop_id = %s` | On HTTP 403 | Token refresh |

### Writes

| # | Database.Table | Operation | When | Phase |
|---|---------------|-----------|------|-------|
| 1 | `AllBots.Shopee_Comp` | `UPDATE SET our_link` | Per batch | Phase 1 (URL cleanup) |
| 2 | `AllBots.Shopee_Comp` | `UPDATE SET our_shop_id, our_item_id, our_model_id, sku` (conditional CASE WHEN) | Per row | Phase 2 (model IDs) |
| 3 | `AllBots.Shopee_VariationSalesDaily` | `DELETE WHERE sale_date < 90 days ago` | Phase 3 start | Phase 3.1 (cleanup) |
| 4 | `AllBots.Shopee_VariationSalesDaily` | `DELETE + INSERT (atomic)` per shop scan window | Per shop | Phase 3.4 (daily sync) |
| 5 | `AllBots.Shopee_VariationSalesSyncState` | `INSERT ... ON DUPLICATE KEY UPDATE` | Per shop | Phase 3.4 (sync state) |
| 6 | `AllBots.Shopee_Comp` | `UPDATE SET shopee_var_sales_*, shopee_var_value_*, shopee_var_sales_last_synced` | After all shops | Phase 3.5 (rolling update) |
| 7 | `requestDatabase.ShopeeTokens` | `UPDATE SET access_token, refresh_token, expires_at, updated_at` | On token refresh | Any phase |

### Columns Written to `Shopee_Comp`

| Column | Phase | Description |
|--------|-------|-------------|
| `our_link` | 1 | Canonical URL |
| `our_shop_id` | 2 | Shopee shop ID |
| `our_item_id` | 2 | Shopee item ID |
| `our_model_id` | 2 | Shopee model/variation ID |
| `sku` | 2 | Seller SKU |
| `shopee_var_sales_7d` | 3 | Units sold in last 7 days |
| `shopee_var_sales_14d` | 3 | Units sold in last 14 days |
| `shopee_var_sales_30d` | 3 | Units sold in last 30 days |
| `shopee_var_sales_60d` | 3 | Units sold in last 60 days |
| `shopee_var_sales_90d` | 3 | Units sold in last 90 days |
| `shopee_var_value_7d` | 3 | Revenue in last 7 days |
| `shopee_var_value_14d` | 3 | Revenue in last 14 days |
| `shopee_var_value_30d` | 3 | Revenue in last 30 days |
| `shopee_var_value_60d` | 3 | Revenue in last 60 days |
| `shopee_var_value_90d` | 3 | Revenue in last 90 days |
| `shopee_var_sales_last_synced` | 3 | Timestamp of last rolling update |

---

## 7. Shopee Partner API Usage

| Endpoint | Phase | Granularity | Batching | Rate Limit |
|----------|-------|------------|----------|------------|
| `/api/v2/product/get_model_list` | 2 | 1 call per item | NOT batched | 150ms sleep |
| `/api/v2/order/get_order_list` | 3 | 1 call per page per time chunk | 100 orders/page, paginated | Standard |
| `/api/v2/order/get_order_detail` | 3 | 1 call per batch | 50 order_sn per call | Standard |
| `/api/v2/auth/access_token/get` | Any | 1 call per shop (on demand) | N/A | Only on 403 |

**API authentication:** HMAC-SHA256 signing. Signature = `SHA256(partner_id + api_path + timestamp + access_token + shop_id)`. Partner ID: `2012161`.

**Order list parameters:**
- `time_range_field`: `create_time`
- `order_status`: `READY_TO_SHIP`, `PROCESSED`, `SHIPPED`, `COMPLETED`
- Max 15-day window per request

**Order detail parameters:**
- `response_optional_fields`: `item_list`
- Max 50 order_sn per request

---

## 8. Token Management & 403 Recovery

```
On HTTP 403 (token expired/invalid):
  1. Read refresh_token from requestDatabase.ShopeeTokens
  2. Call POST /api/v2/auth/access_token/get with refresh_token
  3. If refresh succeeds:
     → UPDATE ShopeeTokens with new access_token, refresh_token, expires_at
     → expires_at = current_time + expire_in (default 14400s = 4 hours)
     → Retry the failed API call with new token
  4. If refresh fails:
     → Reload ALL tokens from DB (in case another process refreshed)
     → Retry with reloaded token
  5. Max retries per call: MAX_RETRIES (3)
```

The scheduler also performs a pre-flight token refresh via `employee.awesomeree.com.my/api/shopee/refresh-tokens` before each pipeline job, so tokens are typically fresh at script start.

---

## 9. Error Handling & Resilience

### Deadlock Handling (`@retry_on_deadlock` decorator)

Applied to all critical DB write functions:
- `delete_old_daily_sales`
- `sync_shop_daily_sales_atomic`
- `update_shopee_comp_rolling_sales`

```python
# Catches MySQL errno 1213 (deadlock) and 1205 (lock wait timeout)
# Exponential backoff: 0.5s * 2^attempt + random jitter
# MAX_DEADLOCK_RETRIES = 3
```

### Hard Timeout (threading-based)

```python
# Wraps API calls in a thread with a 120-second timeout
# HARD_REQUEST_TIMEOUT_SECONDS = 120
# If the API call hangs beyond 120s, the thread is abandoned
# Prevents stuck connections from blocking the entire pipeline
```

### Per-Shop Timeout

```python
# PER_SHOP_TIMEOUT_SECONDS = 1200 (20 minutes)
# If a single shop's sync exceeds 20 minutes, it is aborted
# Prevents a problematic shop from blocking all other shops
```

### Atomic Transactions

Phase 3.4 uses atomic write pattern:
```
BEGIN → DELETE old window → INSERT new data → UPSERT sync state → COMMIT
```
On ANY failure within the transaction: `ROLLBACK` — no partial data is written. Either all data for a shop's scan window is updated, or none of it is.

### Retry Summary

| Component | Max Retries | Backoff | Trigger |
|-----------|-------------|---------|---------|
| `@retry_on_deadlock` | 3 | Exponential (0.5s base) + jitter | MySQL errno 1213/1205 |
| API requests | 3 (`MAX_RETRIES`) | Fixed delay | HTTP errors, timeouts |
| Scheduler per-job | 2 | Token refresh between | Non-zero exit code |

---

## 10. Incremental Sync Strategy

This script uses **incremental sync with overlap** to avoid re-fetching all historical orders:

1. **Sync state table** (`Shopee_VariationSalesSyncState`) stores `last_time_to` per shop — the end timestamp of the last successfully synced window.

2. **On each run**, the sync window starts at `last_time_to - 2 days` (overlap) and ends at "end of yesterday" (23:59:59 MYT).

3. **The 2-day overlap** (`OVERLAP_DAYS`) ensures orders that were created near the boundary of the last sync window are not missed due to timing issues or delayed order status updates.

4. **Window chunking**: The total sync range is broken into 15-day API chunks (`MAX_WINDOW_DAYS`) because the Shopee API limits time range queries.

5. **First sync**: If no `last_time_to` exists for a shop, the script backfills 90 days (`BACKFILL_DAYS`).

6. **Today is never synced**: The end boundary is always "end of yesterday" because today's orders are still in flux (new orders, status changes).

---

## 11. Date Filter Discrepancies

Different phases use different lookback windows — important for debugging:

| Phase | Sheet Types | Lookback | Notes |
|-------|-------------|----------|-------|
| Phase 1 (URL patch) | VVIP/VIP/NEW_ITEMS | **4 days** | Shortest window |
| Phase 2 (model patch) | VVIP/VIP/NEW_ITEMS | **7 days** | Hardcoded, differs from shared config (10 days) |
| Phase 2 (model patch) | LINKS_INPUT | **28 days** | Matches other scripts |
| Phase 3 (rolling update) | All | N/A | Requires `our_shop_id`, `our_item_id`, `our_model_id` all non-null |
| All phases | NONE | **All rows** | No date filter for NONE rows |

**Implication:** A row added 5 days ago will be URL-patched (4-day window includes it) but might not yet be model-patched (7-day window includes it, but 4-day doesn't overlap cleanly). A row added 8 days ago won't be URL-patched but will be model-patched.

---

## 12. Common Failure Modes

| Symptom | Likely Cause | How to Debug |
|---------|-------------|--------------|
| `shopee_var_sales_*` all zeros for a product | Model IDs not resolved — Phase 3.5 rolling update can't match `(shop_id, item_id, model_id)` | Check `our_shop_id`, `our_item_id`, `our_model_id` are all populated for the row |
| Sales not updating for recent rows | Date filter too narrow — 7-day model patch window excludes the row | Check `date_taken` vs the 7-day lookback |
| "Deadlock found" in logs | Concurrent writers on `Shopee_Comp` or `Shopee_VariationSalesDaily` | Verify no overlapping script instances; check advisory locks |
| Token refresh loop | Shopee refresh token expired (30-day TTL) | Check `requestDatabase.ShopeeTokens.expires_at`; may need manual re-auth via Shopee OAuth |
| Per-shop timeout (20 min exceeded) | Very large shop with many orders in the sync window | Check the shop's order volume; consider `SHOP_ID_FILTER` for targeted re-runs |
| Hard timeout (120s exceeded) | Shopee API unresponsive or network issue | Check VM3 network connectivity; the script will retry up to 3 times |
| Rolling update setting sales to 0 | Shop had sales but `our_model_id` doesn't match `Shopee_VariationSalesDaily.model_id` | Compare model IDs between the two tables — may indicate model resolution mismatch |
| Sync state shows future date | Clock skew or manual DB modification | Check `Shopee_VariationSalesSyncState.last_time_to` for the affected shop |

---

## 13. External Dependencies

| Dependency | Impact if Down |
|------------|----------------|
| MySQL at `34.142.159.230` (Kaushal SQL Machine) | Complete failure — no reads or writes possible |
| Shopee Partner API v2 | All three phases fail (URL patch still runs DB-side, but model patch and sales sync require API) |
| `employee.awesomeree.com.my` | Pre-job token refresh by scheduler fails; script may fail on 403 if tokens are stale |

---

## 14. Logging

- **Log module:** Python `logging` (except Phase 1 which uses `print()`)
- **Log file:** `shopee_comp_shopee_sales.log` (rotating)
- **Log contents:** Per-shop sync progress, API call counts, aggregation stats, deadlock retries, rolling update batch progress
- **Phase 1 output:** Goes to stdout only (not captured in log file)
