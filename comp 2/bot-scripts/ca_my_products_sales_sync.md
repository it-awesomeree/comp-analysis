# ca_my_products_sales_sync.py — Comprehensive Script Analysis

## Section 1: Script Identity

| Field | Value |
|-------|-------|
| **Script** | `ca_my_products_sales_sync.py` |
| **Pipeline Position** | #2 of 3 (job ID: `sales_sync`) |
| **Location** | `C:\Users\Admin\Desktop\ca_sg\ca_my_products_sales_sync.py` |
| **VM** | VM TT (`DESKTOP-KD0HLJU`, port 65504) |
| **Pipeline Timeout** | 2 hours |
| **File Size** | 160,941 bytes (~4,244 lines) |
| **Code Structure** | 24 numbered sections |
| **Dependencies** | `requests`, `mysql-connector-python` (pip) |
| **Imports** | `hashlib, os, hmac, json, logging, re, sys, time, random, threading, collections, concurrent.futures, datetime, typing, urllib.parse` |

## Live Verification (2026-03-19)

- Verified against VM TT live file: `C:\Users\Admin\Desktop\ca_sg\ca_my_products_sales_sync.py`
- Scheduler context unchanged: Job #2 (`sales_sync`) in the midnight `ca_sg_pipeline.py` chain
- Live-script checks confirmed:
  - Four-phase structure remains intact (URL patch -> model patch -> Shopee daily sales sync -> SiteGiant sales sync)
  - SiteGiant token refresh flow is active (`requestDatabase.SitegiantToken`, sentinel sync state shop_id `0`)
  - No major logic drift found versus this documentation beyond metadata/size growth

---

## Section 2: Purpose & Overview

A **combined data maintenance and synchronization script** that performs 4 sequential phases in a single execution. Despite the pipeline job ID being "sales_sync", this script handles far more than sales — it is the primary workhorse that maintains data integrity, ID resolution, and sales metrics in `AllBots.Shopee_My_Products`.

The script ensures that:

- All product URLs are in a canonical format
- Every product row has its Shopee shop/item/model IDs and SKU populated
- Daily variation-level sales from Shopee are synced and aggregated into rolling windows
- SKU-level sales from SiteGiant are synced and written to the same product table

---

## Section 3: 4 Phases (Execution Order)

| Phase | Function | Label | Purpose |
|-------|----------|-------|---------|
| 1 | `url_patch_main()` | URL Normalization | Converts various Shopee URL formats to canonical `https://shopee.sg/product/<shop_id>/<item_id>` |
| 2 | `model_patch_main()` | Model Resolution | Populates `our_shop_id`, `our_item_id`, `our_model_id`, `sku` by parsing URLs + querying Shopee Partner API |
| 3 | `daily_sales_sync_main()` | Daily Sales Sync | Syncs completed Shopee orders into daily variation sales buckets, then updates rolling totals |
| 4 | `sitegiant_sales_main()` | SiteGiant Sales Sync | Fetches orders from SiteGiant API, aggregates by SKU, updates `our_sales_*` columns |

**Phase dependencies (within this script):**

- Phase 2 depends on Phase 1 — URL normalization must happen first so model resolution can parse canonical URLs
- Phase 3 depends on Phase 2 — needs `our_shop_id` populated to know which shops to sync
- Phase 3 and Phase 4 are independent — they update different column sets from different data sources

---

## Section 4: Database Connections

| Database | Host | Config Variable | Purpose |
|----------|------|-----------------|---------|
| `AllBots` | `34.142.159.230` (GCP) | `DB_CFG_MAIN` | Main data — all product/sales tables |
| `requestDatabase` | `34.142.159.230` (GCP) | `DB_CFG_TOKENS` | Token storage — `ShopeeTokens` table |

- **Auth:** `root` user, password from `os.environ["DB_PASSWORD"]`
- Both databases are on the same GCP MySQL instance

---

## Section 5: Table — `AllBots.Shopee_My_Products` (Primary Target)

This is the central table. All 4 phases read from and/or write to it.

### Phase 1 — URL Normalization

| Operation | Columns | Details |
|-----------|---------|---------|
| **READ** | `id, our_link` | Scans all rows where `our_link IS NOT NULL AND TRIM(our_link) <> ''` |
| **WRITE** | `our_link` | Converts i-dot format (`shopee.sg/<slug>-i.<shop>.<item>`) to canonical form. Removes query strings, fragments, trailing slashes. Batch size: 50,000 |

### Phase 2 — Model Resolution

| Operation | Columns | Details |
|-----------|---------|---------|
| **READ** | `id, our_link, our_variation, sku, sheet_name, our_shop_id, our_item_id, our_model_id` | Fetches rows where at least one of the 4 ID fields is missing AND the row has `our_link` + either `our_variation` or `sku` |
| **WRITE** | `our_shop_id, our_item_id, our_model_id, sku` | Fills IDs from Shopee API model list. When `MODEL_PATCH_ONLY_UPDATE_WHEN_EMPTY=True` (default), only fills NULL/0/empty fields — does not overwrite existing values |

**Model matching priority:**

1. Match normalized `our_variation` text against model tier variation text
2. If variation matches found, narrow further by existing SKU if possible
3. If no variation match, try exact SKU match across all models
4. If still no match and product has exactly 1 model, use it as fallback

### Phase 3 — Daily Sales Sync (Rolling Update)

| Operation | Columns | Details |
|-----------|---------|---------|
| **READ** | `our_shop_id` (DISTINCT) | Gets list of eligible shop IDs where `our_shop_id IS NOT NULL` |
| **WRITE** | `shopee_var_sales_7d, shopee_var_sales_14d, shopee_var_sales_30d, shopee_var_sales_60d, shopee_var_sales_90d` | Rolling unit sales from `Shopee_VariationSalesDaily` aggregation |
| **WRITE** | `shopee_var_value_7d, shopee_var_value_14d, shopee_var_value_30d, shopee_var_value_60d, shopee_var_value_90d` | Rolling revenue from `Shopee_VariationSalesDaily` aggregation |
| **WRITE** | `shopee_var_sales_last_synced` | Timestamp of last sync (`NOW()`) |

- **Total: 11 columns written**
- Uses `LEFT JOIN` on aggregated `Shopee_VariationSalesDaily` grouped by `(shop_id, item_id, model_id)`
- Joined on `our_shop_id = shop_id AND our_item_id = item_id AND our_model_id = model_id`
- Processed in batches of 20,000 by ID range with per-batch commits

### Phase 4 — SiteGiant Sales Sync

| Operation | Columns | Details |
|-----------|---------|---------|
| **READ** | `id, sku` | Loads SKU to row ID mapping for all rows where `sku IS NOT NULL AND sku != '' AND sku != '-'` |
| **WRITE** | `our_sales_7d, our_sales_14d, our_sales_30d, our_sales_60d, our_sales_90d` | SiteGiant sales counts matched by exact SKU |

- **Total: 5 columns written**
- Uses `CREATE TEMPORARY TABLE tmp_sku_sales (ENGINE=MEMORY)` then bulk INSERT then single `UPDATE ... JOIN`
- Advisory lock `GET_LOCK('MyProductsSiteGiantSalesUpdate')` prevents concurrent script overlap
- Session-level `innodb_lock_wait_timeout = 120`

---

## Section 6: Table — `AllBots.Shopee_VariationSalesDaily`

Phase 3 manages this table. It stores daily sales buckets per variation.

| Operation | Details |
|-----------|---------|
| **DELETE** (cleanup) | Removes records older than 90 days: `WHERE sale_date < CURDATE() - 90 days`. Batched with `LIMIT 5000` per DELETE, commits after each batch |
| **DELETE** (re-sync) | Removes records in the sync window for idempotent re-insert: `WHERE shop_id = %s AND sale_date >= %s` |
| **INSERT/UPSERT** | `INSERT ... ON DUPLICATE KEY UPDATE` with columns: `shop_id, item_id, model_id, sku, sale_date, units_sold, orders_count, revenue, last_synced_at` |
| **READ** (aggregation) | Aggregated via `LEFT JOIN` subquery for rolling window calculation, grouped by `(shop_id, item_id, model_id)` with conditional `SUM(CASE WHEN sale_date >= CURDATE() - INTERVAL N DAY ...)` |

Records are sorted by `(shop_id, model_id, sale_date)` before insert for consistent lock ordering to prevent deadlocks.

---

## Section 7: Table — `AllBots.Shopee_VariationSalesSyncState`

Tracks incremental sync progress. Used by both Phase 3 and Phase 4.

| Operation | Phase | Details |
|-----------|-------|---------|
| **READ** | Phase 3 | `SELECT last_time_to WHERE shop_id = %s` — determines where to resume sync for each shop |
| **UPSERT** | Phase 3 | `INSERT ... ON DUPLICATE KEY UPDATE` — updates `last_time_to, last_synced_at` after successful shop sync |
| **READ** | Phase 4 | `SELECT last_time_to WHERE shop_id = 0` — sentinel row for SiteGiant sync timestamp |
| **UPSERT** | Phase 4 | `INSERT ... ON DUPLICATE KEY UPDATE` with `shop_id = 0` — updates SiteGiant sync timestamp after successful run |

- `shop_id = 0` is a **sentinel value** used exclusively for SiteGiant sync state (defined as `SG_SYNC_SENTINEL_SHOP_ID = 0`)

---

## Section 8: Table — `requestDatabase.ShopeeTokens`

Token management for Shopee Partner API authentication. Used across all phases that call Shopee APIs.

| Operation | Details |
|-----------|---------|
| **READ** | `SELECT shop_id, access_token FROM requestDatabase.ShopeeTokens` — loads all tokens into memory as `Dict[int, str]` |
| **READ** | `SELECT refresh_token FROM requestDatabase.ShopeeTokens WHERE shop_id = %s` — retrieves refresh token on HTTP 403 |
| **WRITE** | `UPDATE requestDatabase.ShopeeTokens SET access_token, refresh_token, expires_at, updated_at` — persists new tokens after OAuth refresh |

---

## Section 9: Table — `tmp_sku_sales` (Temporary)

Phase 4 only. Created as an in-memory temporary table within a single connection.

```sql
CREATE TEMPORARY TABLE IF NOT EXISTS tmp_sku_sales (
    sku VARCHAR(255) PRIMARY KEY,
    s7  INT NOT NULL,
    s14 INT NOT NULL,
    s30 INT NOT NULL,
    s60 INT NOT NULL,
    s90 INT NOT NULL
) ENGINE=MEMORY
```

- Truncated and re-populated each run
- Inserted in chunks of 1,000 (`SG_TMP_INSERT_CHUNK_SIZE`)
- Used in a single `UPDATE Shopee_My_Products c JOIN tmp_sku_sales t ON c.sku = t.sku`

---

## Section 10: External APIs — Shopee Partner API v2

**Base URL:** `https://partner.shopeemobile.com`

**Auth:** HMAC-SHA256 signed URLs — signature base string: `partner_id + path + timestamp + access_token + shop_id`

| Endpoint | Phase | Method | Purpose |
|----------|-------|--------|---------|
| `/api/v2/product/get_model_list` | 2 | GET | Fetches variations/models for a product to resolve `model_id` |
| `/api/v2/order/get_order_list` | 3 | GET | Lists orders by time window + status, paginated (100/page, cursor-based) |
| `/api/v2/order/get_order_detail` | 3 | GET | Fetches item-level details for up to 50 orders per request |
| `/api/v2/auth/access_token/get` | Any | POST | Refreshes expired tokens using `refresh_token` (triggered on HTTP 403) |

**Order list filters:**

- `time_range_field=create_time` (order placement date, not update time)
- Statuses fetched: `READY_TO_SHIP`, `PROCESSED`, `SHIPPED`, `COMPLETED`
- Max 500 pages safety limit per window

**Rate limiting:** 0.15s sleep between API calls (`SLEEP_BETWEEN_CALLS_SECONDS`)

---

## Section 11: External APIs — SiteGiant API v1

**Base URL:** `https://opensgapi.sitegiant.co/api/v1`

**Auth:** Static `Access-Token` header

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/channels` | GET | Lists all SiteGiant sales channels |
| `/orders` | GET | Fetches orders by `channel_id`, date range, page (100/page) |
| `/order/multipleOrder` | POST | Batch order detail retrieval by `order_id` list |

**Order status handling:**

- **Valid (counted):** Paid, Completed, Shipped, Processed, Processing, Ready to Ship
- **Skipped:** Unpaid, Cancelled, Refunded, Failed

**Channel filtering:**

- `OpenApi` type channels are skipped
- `TEST_SPECIFIC_SHOPS` and `TEST_CHANNEL_LIMIT` available for testing

**Parallel processing:** 3 workers (`PARALLEL_WORKERS`), 0.15s delay between batch submissions

---

## Section 12: Date/Time Logic

**Timezone:** `UTC+8` (Asia/Kuala_Lumpur), defined as `LOCAL_TZ = timezone(timedelta(hours=8))`

**Core rules:**

- Sync window always ends at **end of yesterday** (midnight today local time)
- **Today's orders are NEVER synced** — today is an incomplete day
- Rolling windows count complete days only: N days ago to yesterday = N days total

**Example (today = Jan 14, `BACKFILL_DAYS=90`):**

- Yesterday = Jan 13
- 90 days ago = Oct 16
- 90d window = Oct 16 to Jan 13 = 90 complete days

**Incremental sync (Phase 3):**

- Uses `last_time_to` from `Shopee_VariationSalesSyncState` as resume point
- Applies 2-day overlap (`OVERLAP_DAYS`) to avoid boundary issues
- If already synced through yesterday, skips the shop
- API time windows chunked to 15-day max (`MAX_WINDOW_DAYS`)

**Incremental sync (Phase 4):**

- Uses local JSON cache + `last_fetch_date` to determine fetch window
- Fetches `days_since_last_fetch + 2` days of overlap
- If cache is missing/corrupt or gap > 90 days, performs full backfill

---

## Section 13: Configuration Constants

### Global

| Constant | Value | Purpose |
|----------|-------|---------|
| `DRY_RUN` | `False` | Global flag — preview mode without DB writes (affects all phases) |
| `LOCAL_TZ` | `UTC+8` | Malaysia timezone for all date math |

### Phase 1 — URL Patch

| Constant | Value | Purpose |
|----------|-------|---------|
| `URL_PATCH_LIMIT` | `None` | Max rows to process (None = all) |
| `URL_PATCH_BATCH_SIZE` | `50000` | Batch size for URL updates |

### Phase 2 — Model Patch

| Constant | Value | Purpose |
|----------|-------|---------|
| `MODEL_PATCH_BATCH_SIZE` | `50000` | Max candidate rows to fetch |
| `MODEL_PATCH_ONLY_UPDATE_WHEN_EMPTY` | `True` | Only fill NULL/0/empty fields |
| `MODEL_PATCH_SLEEP_BETWEEN_CALLS_SECONDS` | `0.15` | Rate limit between API calls |

### Phase 3 — Daily Sales Sync

| Constant | Value | Purpose |
|----------|-------|---------|
| `BACKFILL_DAYS` | `90` | Total days to backfill for new shops |
| `BACKFILL_CHUNK_DAYS` | `0` | Max days per run during backfill (0 = unlimited) |
| `SHOP_ID_FILTER` | `None` | Restrict to single shop (None = all) |
| `OVERLAP_DAYS` | `2` | Re-sync overlap to avoid boundary issues |
| `MAX_WINDOW_DAYS` | `15` | Shopee API max time range per request |
| `ORDER_LIST_PAGE_SIZE` | `100` | Max orders per page |
| `ORDER_DETAIL_BATCH_SIZE` | `50` | Max orders per detail request |
| `COMP_UPDATE_BATCH_SIZE` | `20000` | Batch size for rolling sales UPDATE by ID range |
| `PER_SHOP_TIMEOUT_SECONDS` | `1200` | 20 min max per shop |
| `HARD_REQUEST_TIMEOUT_SECONDS` | `120` | Hard kill timeout per API call |
| `SLEEP_BETWEEN_CALLS_SECONDS` | `0.15` | Rate limiting between API calls |
| `DELETE_BATCH_SIZE` | `5000` | Batch size for cleanup DELETEs |
| `MAX_RETRIES` | `3` | API retry attempts |
| `RETRY_BASE_DELAY_SECONDS` | `1.0` | Exponential backoff base |
| `MAX_DEADLOCK_RETRIES` | `3` | DB deadlock retry attempts |
| `DEADLOCK_RETRY_BASE_DELAY` | `0.5` | Deadlock backoff base |

### Phase 4 — SiteGiant Sales

| Constant | Value | Purpose |
|----------|-------|---------|
| `ENABLE_SITEGIANT_SALES` | `True` | Toggle Phase 4 on/off |
| `DAYS_BACK` | `90` | SiteGiant lookback window |
| `TIME_PERIODS` | `[7, 14, 30, 60, 90]` | Rolling periods to track |
| `PARALLEL_WORKERS` | `3` | Thread pool size |
| `SG_BATCH_SIZE` | `100` | Orders per API request |
| `SG_BATCH_DELAY` | `0.15` | Delay between batch submissions |
| `SG_REQUEST_TIMEOUT` | `90` | API request timeout (seconds) |
| `SG_MAX_RETRIES` | `3` | Batch retry attempts |
| `SG_RETRY_BASE_DELAY` | `2` | Exponential backoff base |
| `SG_DB_MAX_RETRIES` | `6` | DB lock contention retries |
| `SG_DB_RETRY_BASE_DELAY` | `2.0` | DB backoff base |
| `SG_DB_LOCK_WAIT_TIMEOUT` | `120` | Session-level lock wait (seconds) |
| `SG_TMP_INSERT_CHUNK_SIZE` | `1000` | Temp table insert batch size |
| `SG_FETCH_OVERLAP_DAYS` | `2` | Re-fetch overlap for incremental sync |
| `TEST_CHANNEL_LIMIT` | `None` | Limit channels for testing |
| `TEST_SPECIFIC_SHOPS` | `None` | Filter to specific shop names |

---

## Section 14: Resilience & Error Handling

### Deadlock/Lock Retry (Database)

- **Decorator:** `@retry_on_deadlock` — catches MySQL error 1213 (deadlock) and 1205 (lock wait timeout)
- **Retries:** 3 attempts with exponential backoff (0.5s base: 0.5s, 1s, 2s)
- **Applied to:** `delete_old_daily_sales`, `update_shopee_comp_rolling_sales`, `sync_shop_daily_sales_atomic`

### API Retry (Shopee)

- **Function:** `shopee_get_json_with_retry`
- **Retries:** 3 attempts with exponential backoff (1s base: 1s, 2s, 4s)
- **Scope:** Transient `requests.RequestException` and `requests.Timeout`
- **Exception:** HTTP 403 (`ShopeeForbiddenError`) is NOT retried — triggers token refresh instead

### Hard Timeout (Shopee)

- **Function:** `shopee_get_json` wraps API calls in a background `threading.Thread`
- **Timeout:** 120s (`HARD_REQUEST_TIMEOUT_SECONDS`)
- **Purpose:** Catches cases where socket-level timeout fails (TCP keepalives, slow trickle)

### Token Auto-Refresh

- **Function:** `run_with_token_retry`
- **Trigger:** HTTP 403 from Shopee API
- **Flow:** Calls `refresh_shop_token_via_api` which POSTs to `/api/v2/auth/access_token/get` with `refresh_token`
- **Fallback:** If API refresh fails, reloads all tokens from `requestDatabase.ShopeeTokens`
- **Retries:** 1 retry after token refresh

### Atomic Transactions (Phase 3)

- **Function:** `sync_shop_daily_sales_atomic`
- **Wraps:** DELETE + INSERT + sync state update in a single transaction
- **On failure:** `connection.rollback()` — no partial/corrupted state

### Batch Processing (Phase 3 rolling update)

- Rolling updates use ID-range batching (20,000 per batch) with per-batch commits
- Reduces lock scope and deadlock risk
- Old record cleanup also batched (5,000 per DELETE) with per-batch commits and 0.05s delay

### SiteGiant Retry (Phase 4)

- **Batch level:** 3 retries with 2s exponential backoff per order detail batch
- **DB level:** 6 retries with 2s exponential backoff + random jitter for lock contention
- **Advisory lock:** `GET_LOCK('MyProductsSiteGiantSalesUpdate', 10)` prevents concurrent script instances
- **Tracking:** `BatchProcessingStats` class tracks success/failure/retry counts per batch

---

## Section 15: Local File I/O

| File | Location | Purpose |
|------|----------|---------|
| `ca_my_products_sales_sync.log` | Script directory | Logging output (FileHandler, UTF-8) |
| `sitegiant_daily_cache.json` | Script directory | SiteGiant incremental sync cache — stores daily sales buckets by date by SKU. Pruned to 90 days. Atomic write via `.tmp` + `os.replace()` |
| `sitegiant_sales_report.txt` | Script directory | Detailed SKU-centric sales report — **appended** each run (not overwritten). Includes matched/unmatched SKUs, batch reliability stats, per-SKU sales breakdown |

**Cache format (`sitegiant_daily_cache.json`):**

```json
{
  "version": 1,
  "last_fetch_date": "2026-03-10",
  "saved_at": "2026-03-10 00:45:12",
  "daily_sales": {
    "2026-03-10": {
      "SKU123": { "units": 5, "names": ["Product Name"] }
    }
  }
}
```

---

## Section 16: Data Flow Diagram

```
PHASE 1: URL Normalization
  Shopee_My_Products.our_link (READ)
    -> parse URL format (i-dot / product path / cleanup)
    -> Shopee_My_Products.our_link (UPDATE)

PHASE 2: Model Resolution
  Shopee_My_Products (READ rows missing IDs)
    -> parse our_link -> extract shop_id, item_id
    -> requestDatabase.ShopeeTokens (READ access tokens)
    -> Shopee API: GET /api/v2/product/get_model_list
    -> match our_variation to model tier_variation text
    -> Shopee_My_Products (UPDATE our_shop_id, our_item_id, our_model_id, sku)

PHASE 3: Daily Sales Sync
  Step 3.1: Shopee_VariationSalesDaily (DELETE old records >90 days)
  Step 3.2: Shopee_My_Products.our_shop_id (READ distinct shops)
  Step 3.3: requestDatabase.ShopeeTokens (READ access tokens)
  Step 3.4: For each shop:
    -> Shopee_VariationSalesSyncState (READ last_time_to)
    -> Shopee API: GET /api/v2/order/get_order_list (paginated, 15-day chunks)
    -> Shopee API: GET /api/v2/order/get_order_detail (batches of 50)
    -> Aggregate by (shop_id, item_id, model_id, sale_date)
    -> ATOMIC TRANSACTION:
        Shopee_VariationSalesDaily (DELETE in window + INSERT aggregates)
        Shopee_VariationSalesSyncState (UPSERT last_time_to)
  Step 3.5: Shopee_VariationSalesDaily (READ aggregate via LEFT JOIN)
    -> Shopee_My_Products (UPDATE 11 rolling sales/value/timestamp columns)

PHASE 4: SiteGiant Sales Sync
  -> SiteGiant API: GET /channels (list channels)
  -> For each channel:
      SiteGiant API: GET /orders (paginated, 30-day chunks)
      SiteGiant API: POST /order/multipleOrder (batch details)
      -> Aggregate daily sales by SKU
  -> Merge with local cache (sitegiant_daily_cache.json)
  -> Build SKUSalesData (7d/14d/30d/60d/90d period aggregation)
  -> Shopee_My_Products (READ sku -> row ID mapping)
  -> tmp_sku_sales (CREATE TEMP + INSERT)
  -> Shopee_My_Products (UPDATE 5 our_sales_* columns via JOIN)
  -> Shopee_VariationSalesSyncState (UPSERT shop_id=0 sentinel)
  -> Save cache + report
```

---

## Section 17: Exit Codes

Exit code is driven **primarily by Phase 3**. Phase 4 does not return an exit code.

| Code | Source | Meaning |
|------|--------|---------|
| `0` | `daily_sales_sync_main()` returns 0 | Phase 3: all shops synced successfully |
| `1` | `daily_sales_sync_main()` returns 1, OR `main()` catches uncaught exception from any phase | Phase 3: all shops failed, OR fatal error in any phase (including Phase 1, 2, or 4) |
| `2` | `daily_sales_sync_main()` returns 2 | Phase 3: partial failure (some shops OK, some failed) |

- Phase 4 (`sitegiant_sales_main`) has **no return value** — its internal errors are logged but do not affect the exit code unless they propagate as uncaught exceptions
- The script calls `sys.exit(exit_code)` at the end of `main()`

---

## Section 18: Phase-Specific Workflows

### Phase 1 Workflow — URL Normalization

1. SELECT all rows from `Shopee_My_Products` where `our_link` is non-empty
2. For each row, classify the URL:
   - If already canonical `/product/<shop>/<item>` format: check if needs cleanup (query params, fragments, trailing slash)
   - If i-dot format (`<slug>-i.<shop>.<item>`): convert to canonical
   - If neither: log as bad link
3. Batch UPDATE normalized URLs (batch size: 50,000)
4. Commit or rollback based on `DRY_RUN`

### Phase 2 Workflow — Model Resolution

1. SELECT candidate rows where any of `our_shop_id/our_item_id/our_model_id/sku` is missing
2. Group rows by `(shop_id, item_id)` parsed from canonical `our_link`
3. For each group, call Shopee API `get_model_list` once
4. Match each row to a model using the 3-tier priority (variation text, SKU, single-model fallback)
5. UPDATE matched rows. Commit per group, rollback on failure

### Phase 3 Workflow — Daily Sales Sync

1. **Cleanup:** Delete old records from `Shopee_VariationSalesDaily` beyond 90-day retention
2. **Shop discovery:** Get distinct `our_shop_id` values from `Shopee_My_Products`
3. **Token loading:** Load all access tokens from `requestDatabase.ShopeeTokens`
4. **Per-shop sync:**
   - Read sync state (`last_time_to`) to determine window
   - Fetch orders across 4 statuses in 15-day API chunks
   - Fetch order details in batches of 50
   - Aggregate by `(shop_id, item_id, model_id, sale_date)` into units_sold, orders_count, revenue
   - Atomic transaction: delete + insert + update sync state
5. **Rolling update:** LEFT JOIN aggregate from `Shopee_VariationSalesDaily` into UPDATE 11 columns in `Shopee_My_Products` (batched by ID range, 20,000 per batch)

### Phase 4 Workflow — SiteGiant Sales Sync

1. **Channel discovery:** GET `/channels`, filter out `OpenApi` types
2. **Determine fetch window:** Check local JSON cache and DB sync timestamp
   - If cache fresh: incremental fetch (days since last + 2-day overlap)
   - If cache missing/stale: full 90-day backfill
3. **Per-channel fetch:** Collect order IDs in 30-day chunks, then parallel batch detail retrieval (3 workers)
4. **Merge:** Overwrite fetched dates into cached daily data, prune >90 days
5. **Aggregate:** Convert daily cache into `SKUSalesData` with period buckets (7d/14d/30d/60d/90d)
6. **Report:** Generate and append to `sitegiant_sales_report.txt`
7. **DB update:** Load SKU to row mapping, stage into temp table, single JOIN UPDATE
8. **Persist:** Save cache file + upsert sync timestamp (shop_id=0 sentinel)

---

## Section 19: Logging

**Setup:** Dual output — `StreamHandler(stdout)` + `FileHandler("ca_my_products_sales_sync.log", utf-8)`

**Format:** `%(asctime)s | %(levelname)s | %(message)s`

**Example:** `2026-03-11 00:15:32 | INFO | message`

**Level:** `INFO`

**Key logged metrics:**

- Per-phase start/end banners
- Candidate row counts and grouping stats
- API call progress (every 20 batches at INFO level)
- Deadlock/retry warnings
- Token refresh events
- Per-shop sync success/failure with timing
- Rolling update batch progress
- SiteGiant batch processing stats (success rate, retries, data loss)
- Final summary with exit code

---

## Section 20: Key Observations for Future Analysis

1. **This is the longest and most complex script in the pipeline** — 4 phases doing fundamentally different tasks in a single sequential execution
2. **Phase 3 is the only phase that determines the exit code** — Phase 4 failures are silently absorbed unless they throw uncaught exceptions
3. **Two independent sales data sources update different columns** — Shopee API writes `shopee_var_sales_*` + `shopee_var_value_*` (variation-level via model_id), SiteGiant writes `our_sales_*` (SKU-level via exact match)
4. **SiteGiant incremental sync relies on a local JSON cache file** — if the file is lost/corrupted, it triggers a full 90-day backfill. The DB sentinel (shop_id=0) is a secondary indicator only
5. **The 90-day retention window is enforced at two levels** — daily record cleanup in `Shopee_VariationSalesDaily` AND cache pruning in `sitegiant_daily_cache.json`
6. **Lock contention is a known concern** — the script uses multiple strategies: ID-range batching, sorted insert ordering, advisory locks, session-level lock wait timeouts, and deadlock retry decorators
7. **Token refresh is self-healing** — HTTP 403 triggers automatic OAuth refresh via Shopee API, with fallback to DB reload. The refreshed token is persisted back to `requestDatabase.ShopeeTokens`
