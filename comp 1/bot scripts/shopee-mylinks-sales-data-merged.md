# `Shopee-mylinks-sales-data-merged.py` — Complete Reference Documentation

**Location:** `VM3: C:\Users\Admin\Desktop\Shopee Comp My links Api\Shopee-mylinks-sales-data-merged.py`
**Size:** 100,552 bytes | 2,449 lines | Python 3.13 | **Last modified:** 2026-03-19 12:36:43
**Target Table:** `AllBots.Shopee_Comp`
**Database Host:** `34.142.159.230` (Google Cloud MySQL)
**Dependencies:** `mysql-connector-python`, `requests` (all others are stdlib)

---

## Live Verification (2026-03-19)

- Verified against VM3 live file: `C:\Users\Admin\Desktop\Shopee Comp My links Api\Shopee-mylinks-sales-data-merged.py`
- Scheduler context unchanged: Job #5 in the VM3 daily 4:00 PM chain
- Critical live-script corrections:
  - SiteGiant token handling is now dynamic, with refresh flow and DB persistence in `requestDatabase.SitegiantToken` (row id `1`)
  - Added DB resilience helpers (`safe_db_connect`, `ensure_db_connection`) with session timeout hardening and periodic reconnect controls
  - SiteGiant retry logic now includes adaptive recursive batch splitting (`SG_ENABLE_ADAPTIVE_BATCH_SPLIT`, `SG_MIN_BATCH_SIZE`) for transient failures
  - SKU update filtering now includes a backfill path for recent rows with missing `sitegiant_sales_value_*` fields (`SG_BACKFILL_MISSING_DAYS = 30`)
  - `PRIORITY_DAYS_LOOKBACK` in live constants is `10` days (header comments still mention `7`, which is stale)

## 1. Purpose & Overview

This script is the **Job 5** of the current 5-script daily Shopee Competitive Analysis pipeline orchestrated by `scheduler.py` on VM3. The chain starts from the daily 4:00 PM Task Scheduler trigger. It is the core data enrichment pipeline for the `AllBots.Shopee_Comp` competitive analysis table. It pulls product data and sales metrics from two separate external systems and writes them into a single database table, giving the Awesomeree team a unified view of their own Shopee MY product catalog enriched with live pricing, stock, and sales performance.

**Why it exists:** The `Shopee_Comp` table stores competitor and own-product links scraped from various sources (sheets categorized as VVIP, VIP, NEW_ITEMS, LINKS_INPUT, NONE). Those rows initially only contain a product URL and variation name. This script fills in everything else — product names, descriptions, prices, stock levels, SKUs, ratings, images, and time-bucketed sales counts — by calling the Shopee Partner API and SiteGiant order management API.

**The two phases:**
- **Phase 1 (Shopee Partner API):** Enriches each row with product metadata — name, description, price, stock, SKU, rating, images, and lifetime total sales (`our_sales`)
- **Phase 2 (SiteGiant API):** Enriches each row with time-bucketed sales counts from actual order data — `our_sales_7d`, `our_sales_14d`, `our_sales_30d`, `our_sales_60d`, `our_sales_90d`

**Key design principle:** Phase 1 runs first to populate SKU values from the Shopee API. Phase 2 then uses those SKUs to match against SiteGiant order line items. Without Phase 1's SKU population, Phase 2 cannot match sales to database rows.

---

## 2. Pipeline Position & Data Dependencies

```
scheduler.py (orchestrator, runs daily at 4:00 PM via Windows Task Scheduler)
│
├─ Job 1: ca_shopee_listing_to_db.py          (SEED — imports listings)
├─ Job 2: our_variation_preprocessing.py       (backfills our_variation)
├─ Job 3: ca_ai_variation_match.py             (AI variation matching)
├─ Job 4: shopee_comp_shopee_sales.py          (Shopee order sales sync)
├─ Job 5: Shopee-mylinks-sales-data-merged.py  ← THIS SCRIPT
└─ Jobs 6-7: scheduled out of the current chain (ads metrics and similarity)
```

- **Depends on:** Jobs 1–3 must have populated `our_link`, `our_variation`, `our_shop_id`, `our_item_id` for this script to find candidate rows and resolve models.
- **Scheduling:** Triggered daily at 4:00 PM by Windows Task Scheduler via `scheduler.py`. Has a 6-hour safety timeout. On failure, retried up to 2 times (3 total attempts) with token refresh between retries.

---

## 3. Configuration Constants (Lines 64–155)

### API Credentials & Endpoints

| Constant | Value | Purpose |
|---|---|---|
| `PARTNER_ID` | `2012161` | Shopee Partner API app ID |
| `PARTNER_KEY` | `shpk6e6b...` | HMAC-SHA256 signing key for Shopee API |
| `HOST` | `https://partner.shopeemobile.com` | Shopee API base URL |
| `PATH_BASE_INFO` | `/api/v2/product/get_item_base_info` | Product name, description, images, price |
| `PATH_EXTRA_INFO` | `/api/v2/product/get_item_extra_info` | Rating, total sales |
| `PATH_MODEL_LIST` | `/api/v2/product/get_model_list` | SKU, variation price, stock, variation images |
| `SITEGIANT_CONFIG.base_url` | `https://opensgapi.sitegiant.co/api/v1` | SiteGiant API base |
| `SITEGIANT_CONFIG.token` | `276118aa...` | SiteGiant static bearer token |

### Database Connections

| Config | Host | Database | Purpose |
|---|---|---|---|
| `DB_CFG_ALLBOTS` | `34.142.159.230` | `AllBots` | Main table (`Shopee_Comp`) reads and writes |
| `DB_CFG_TOKENS` | `34.142.159.230` | `requestDatabase` | Shopee OAuth token storage (`ShopeeTokens`) |

Password sourced from environment variable `DB_PASSWORD`.

### Processing Controls

| Constant | Value | Purpose |
|---|---|---|
| `DRY_RUN` | `False` | Live mode — writes to DB |
| `ONLY_UPDATE_WHEN_EMPTY` | `True` | Only fetch rows where at least 1 key field is NULL/empty |
| `BATCH_LIMIT` | `50000` | Max filtered rows for Phase 1 |
| `SLEEP_BETWEEN_CALLS_SECONDS` | `0.15` | Delay between Shopee API calls |

### Sheet/Date Filtering

| Constant | Value | Purpose |
|---|---|---|
| `PRIORITY_SHEET_NAMES` | `['VVIP', 'VIP', 'NEW_ITEMS']` | High-priority sheets |
| `PRIORITY_DAYS_LOOKBACK` | `10` | Lookback for priority sheets (**Note: module docstring at line 36 says 7 days — stale, code uses 10**) |
| `LINKS_INPUT_SHEET_NAME` | `'LINKS_INPUT'` | Standard input sheet |
| `LINKS_INPUT_WEEKS_LOOKBACK` | `4` | Lookback for LINKS_INPUT |
| `NONE_SHEET_NAME` | `'NONE'` | Direct-from-Shopee rows |
| `ENABLE_NONE_PROCESSING` | `True` | Whether to process NONE rows |
| `NONE_BATCH_LIMIT` | `10000` | Max NONE rows per run |
| `NONE_SKIP_DAYS` | `5` | Skip NONE rows updated within this many days |

### URL Normalization

| Constant | Value | Purpose |
|---|---|---|
| `ENABLE_URL_NORMALIZATION_WRITEBACK` | `True` | Write cleaned URLs back to DB |
| `URL_UPDATE_BATCH_SIZE` | `2000` | Batch size for URL update commits |

### SiteGiant Settings

| Constant | Value | Purpose |
|---|---|---|
| `ENABLE_SITEGIANT_SALES` | `True` | Whether Phase 2 runs |
| `DAYS_BACK` | `90` | How far back to fetch orders |
| `TIME_PERIODS` | `[7, 14, 30, 60, 90]` | Sales aggregation time buckets |
| `PARALLEL_WORKERS` | `3` | Concurrent workers for order detail fetching |
| `SG_BATCH_SIZE` | `100` | Orders per SiteGiant API page |
| `SG_BATCH_DELAY` | `0.15` | Delay between parallel batch submissions |
| `SG_REQUEST_TIMEOUT` | `90` | API request timeout (seconds) |
| `SG_MAX_RETRIES` | `3` | Retries per failed batch |
| `SG_RETRY_BASE_DELAY` | `2` | Exponential backoff base (seconds) |

### Database Reliability

| Constant | Value | Purpose |
|---|---|---|
| `DB_MAX_RETRIES` | `6` | Retries for MySQL 1205/1213 errors |
| `DB_RETRY_BASE_DELAY` | `2.0` | Exponential backoff base |
| `DB_LOCK_WAIT_TIMEOUT` | `120` | Session-level InnoDB lock wait (seconds) |
| `DB_ADVISORY_LOCK_NAME` | `ShopeeCompSalesUpdate` | Prevents concurrent script runs |
| `TMP_INSERT_CHUNK_SIZE` | `1000` | Batch size for temp table inserts |

### Testing Knobs

| Constant | Value | Purpose |
|---|---|---|
| `TEST_CHANNEL_LIMIT` | `None` | Set to `1` to test with first channel only |
| `TEST_SPECIFIC_SHOPS` | `None` | Set to e.g. `["ValueSnap"]` to filter channels |
| `REPORT_FILE` | `"Jan_report.txt"` | Hardcoded filename (stale — named after January, used year-round) |

### Order Status Filters

| List | Values | Used In |
|---|---|---|
| `VALID_STATUSES` | Paid, Completed, Shipped, Processed, Processing, Ready to Ship | `should_count_order()` — include |
| `SKIP_STATUSES` | Unpaid, Cancelled, Refunded, Failed | `should_skip_order()` — exclude |

---

## 4. Main Execution Flow

### `main()` (Line 1970)

```
1. setup_logging()  →  stdout, INFO level, timestamped
2. Log config: table name, DRY_RUN, ENABLE_SITEGIANT_SALES, filter description
3. run_shopee_api_updates()              ← Phase 1
4. if ENABLE_SITEGIANT_SALES:
     run_sitegiant_sales_updates()       ← Phase 2
5. Log "All updates complete!"
6. If DRY_RUN: print ALTER TABLE hints for sales columns
```

The DRY_RUN hint at the end prints:
```sql
ALTER TABLE Shopee_Comp ADD COLUMN our_sales_7d INT DEFAULT 0;
-- (same for 14d, 30d, 60d, 90d)
```
This tells us the `our_sales_*d` columns were added after the initial table design.

---

## 5. Phase 1: Shopee Partner API Updates

### `run_shopee_api_updates()` (Line 1605)

### 4a. Fetch Candidate Rows — `fetch_candidate_rows()` (Line 1097)

Two separate SQL queries, combined into one result set:

**Query 1 — Filtered rows (VVIP/VIP/NEW_ITEMS/LINKS_INPUT):**
```sql
SELECT id, sku, our_link, our_variation, sheet_name, date_taken
FROM AllBots.Shopee_Comp
WHERE (our_link IS NOT NULL AND TRIM(our_link) <> '')
  AND (our_variation IS NOT NULL AND TRIM(our_variation) <> '')
  AND (
    (sheet_name IN ('VVIP','VIP','NEW_ITEMS') AND date_taken >= DATE_SUB(NOW(), INTERVAL 10 DAY))
    OR
    (sheet_name = 'LINKS_INPUT' AND date_taken >= DATE_SUB(NOW(), INTERVAL 4 WEEK))
  )
  AND (product_name IS NULL OR product_name = ''
       OR our_price IS NULL
       OR our_stock IS NULL
       OR shopee_stock IS NULL
       OR sku IS NULL OR sku = '' OR sku = '-')
ORDER BY id DESC
LIMIT 50000
```

The `ONLY_UPDATE_WHEN_EMPTY=True` flag adds the last AND condition — rows are only fetched if at least one key field is still missing.

**Query 2 — NONE sheet rows** (only if `ENABLE_NONE_PROCESSING=True`):
```sql
SELECT id, sku, our_link, our_variation, sheet_name, date_taken
FROM AllBots.Shopee_Comp
WHERE (our_link IS NOT NULL AND TRIM(our_link) <> '')
  AND (our_variation IS NOT NULL AND TRIM(our_variation) <> '')
  AND sheet_name = 'NONE'
  AND (date_taken IS NULL OR date_taken < DATE_SUB(NOW(), INTERVAL 5 DAY))
ORDER BY id DESC
LIMIT 10000
```

NONE rows have **no** `ONLY_UPDATE_WHEN_EMPTY` filter — they're always eligible if they haven't been updated in 5 days.

**Deduplication (line 1159):** When combining results, NONE rows are checked against `filtered_ids` to avoid duplicates: `if int(r["id"]) not in filtered_ids`.

**Returns:** `(combined_rows, none_row_ids_set)` — the set is used later to determine overwrite vs COALESCE behavior.

### 4b. URL Normalization (Lines 1628–1651)

For each row's `our_link`, the script calls `canonicalize_shopee_link()` (line 414) which tries two normalizers in order:

**1. `normalize_product_link()` (line 376)** — Handles canonical-format URLs:
- Regex: `^/product/(\d+)/(\d+)(?:$|[/?#])`
- Strips query strings, trailing slashes, fragments
- Example: `https://shopee.com.my/product/123/456?from=ads` → `https://shopee.com.my/product/123/456`

**2. `to_product_link_from_i_dot()` (line 393)** — Handles slug-format URLs:
- Regex: `i\.(\d+)\.(\d+)` (searches anywhere in path)
- Only works for `shopee.com.my` domain
- Example: `https://shopee.com.my/Some-Product-Name-i.123.456` → `https://shopee.com.my/product/123/456`

**3. If both fail:** Returns `None`, row counted as `invalid_url_rows` and skipped entirely.

**Writeback via `apply_our_link_updates()` (line 1169):** Uses `executemany()` for batch efficiency. Checks **both** `DRY_RUN` and `ENABLE_URL_NORMALIZATION_WRITEBACK` — skips if either is blocking. URL updates are flushed in batches of 2000 with a commit after each flush:
```sql
UPDATE AllBots.Shopee_Comp SET our_link=%s WHERE id=%s
```

### 4c. Group by (shop_id, item_id) (Line 1644)

Each canonical URL is parsed via `parse_shop_item_from_canonical()` (line 429) using regex `shopee\.com\.my/product/(\d+)/(\d+)` to extract `(shop_id, item_id)`. Rows are grouped because multiple variations (different `our_variation` values) of the same product share one set of API calls.

### 4d. Load Access Tokens — `fetch_tokens_by_shop_id()` (Line 238)

```sql
SELECT shop_id, access_token FROM requestDatabase.ShopeeTokens
```
Returns `{int(shop_id): token_string}` for all shops with non-empty tokens.

### 4e. Process Each Group (Lines 1665–1797)

Phase 1 uses a plain `with requests.Session() as session:` context manager (line 1665) — no custom adapters or pooling (unlike Phase 2's SiteGiant client).

For each `(shop_id, item_id)` group, row IDs are split:
```python
none_ids_in_group = [rid for rid in row_ids if rid in none_row_ids]
normal_ids_in_group = [rid for rid in row_ids if rid not in none_row_ids]
```

A `work(access_token)` closure (line 1678) is defined with `nonlocal product_updates, model_updates` and passed to `run_with_token_retry()`.

**API Call 1 — `fetch_item_base_info()` (Line 444)**
- Endpoint: `GET /api/v2/product/get_item_base_info`
- Params: `item_id_list` (comma-separated), `need_tax_info=false`, `need_complaint_policy=false`
- Returns per item: `item_name`, `description`, `description_info` (extended), `image.image_url_list`, `price_info`, `current_price`
- Batched via `chunk_list()`: up to 50 item_ids per call, 0.15s sleep between

**API Call 2 — `fetch_item_extra_info()` (Line 464)**
- Endpoint: `GET /api/v2/product/get_item_extra_info`
- Params: `item_id_list` (comma-separated)
- Returns per item: `rating_star`, `sale` (lifetime total sales count)
- Batched via `chunk_list()`: up to 50 item_ids per call

**API Call 3 — `fetch_model_list()` (Line 484)**
- Endpoint: `GET /api/v2/product/get_model_list`
- Params: `item_id`
- Returns: `model[]` or `model_list[]` (script tries both keys — line 1695), `tier_variation[]`
- **One call per item_id** (not batched)

All three calls use HMAC-SHA256 signed URLs via `build_signed_url()` (line 217). Signing string: `{PARTNER_ID}{path}{timestamp}{access_token}{shop_id}`.

**Model list key fallback (line 1695):**
```python
models = model_list_response.get("model") or model_list_response.get("model_list") or []
```
Defensive pattern for Shopee API response version differences. If neither key exists, `models` is `[]` and no model-level updates happen for that group.

### 4f. Field Extraction Functions

| Function | Line | Input | Output | Notes |
|---|---|---|---|---|
| `extract_description()` | 547 | `base_item` | `str` | Tries plain `description` first. Falls back to `description_info.extended_description.field_list`, filtering `field_type="text"` entries and stripping URLs with regex. Joins with `\n`. |
| `extract_main_images()` | 570 | `base_item` | `List[str]` | From `image.image_url_list` |
| `extract_item_price_text()` | 576 | `base_item` | `Optional[str]` | Tries `current_price` then `price_info[0].current_price`. Returns formatted `"5.90"`. |
| `extract_model_price()` | 604 | `model` | `Optional[float]` | Tries `price_info[0].current_price` then `current_price` (**reversed order** from item-level). Returns raw float. |
| `extract_product_price_from_models()` | 618 | `List[model]` | `Optional[str]` | Collects all model prices → `"5.90"` if uniform, `"5.90 - 12.90"` if range. Used as fallback when item-level price unavailable. |
| `parse_int_maybe()` | 498 | `Any` | `Optional[int]` | Handles bool→None, int→pass, float≥0→int, digit string→int. Used for `sale` count. |
| `build_option_image_lookup()` | 632 | `model_list_response` | `Dict[(tier_idx, option_idx), url]` | Maps variation option positions to image URLs from `tier_variation[].option_list[].image.image_url`. |
| `resolve_model_option_image_url()` | 645 | `model, lookup` | `Optional[str]` | Iterates model's `tier_index`, returns **first** matching image. Typically the Color tier (tier 0). |
| `sum_seller_stock()` | 657 | `model` | `Optional[int]` | Sum of `stock_info_v2.seller_stock[].stock` where `if_saleable` is absent or `True`. **Skips** entries with `if_saleable` present but not `True`. Only handles `int` stock values. |
| `sum_shopee_stock()` | 676 | `model` | `Optional[int]` | Sum of `stock_info_v2.shopee_stock[].stock`. **No saleable filter.** Handles both `int` and string digit values. |
| `chunk_list()` | 436 | `list, size` | `List[List]` | General-purpose batch splitter. Used by base_info (50), extra_info (50), temp table inserts (1000). |

### 4g. Product-Level DB Update — `update_product_level_fields()` (Line 1180)

Applies to ALL rows in the group sharing the same `(shop_id, item_id)`.

**For normal rows** (COALESCE — only fills NULLs):
```sql
UPDATE AllBots.Shopee_Comp SET
  product_name = COALESCE(%s, product_name),
  product_description = COALESCE(%s, product_description),
  our_sales = COALESCE(%s, our_sales),
  our_rating = COALESCE(%s, our_rating),
  our_product_price = COALESCE(%s, our_product_price),
  our_main_images = COALESCE(%s, our_main_images)
WHERE id IN (...)
```

**For NONE rows** (overwrite — replaces everything):
```sql
UPDATE AllBots.Shopee_Comp SET
  product_name = %s,
  product_description = %s,
  our_sales = %s,
  our_rating = %s,
  our_product_price = %s,
  our_main_images = %s
WHERE id IN (...)
```

`our_main_images` stored as JSON array: `json.dumps(main_images, ensure_ascii=False)` — e.g., `["url1","url2","url3"]`.

### 4h. Variation-to-Model Matching (Lines 1745–1780)

For each individual row in the group:

**Path 1 — Row already has a SKU** (not NULL/empty/"-"):
- Iterates all models looking for exact `model_sku` match
- If found → uses that model for price/stock/image extraction
- If NOT found → row is **skipped** with a debug log (no model-level update)

**Path 2 — Row has no SKU** (uses variation matching):
- Row's `our_variation` is normalized via `normalize_variation()` (line 517): lowercased, commas/hyphens/underscores replaced with spaces, whitespace collapsed
- Compared against model variations built from `extract_variation_from_model()` (line 523) which reads `tier_variation` option names via `tier_index`
- Takes the **first** matching candidate from `models_by_variation` dict
- If no match → row is **skipped** (no model-level update)

`extract_variation_from_model()` details (line 523):
- Reads `model.tier_index` (list of option indices like `[2, 0]`)
- For each position, looks up `tier_variation[pos]` and gets option at that index
- Tries keys: `option_list` then `options` for the options array
- Each option can be a dict (tries keys: `option`, `value`, `name`) or raw string
- Concatenated with spaces, then normalized

### 4i. Model-Level DB Update — `update_model_level_fields_by_variation()` (Line 1227)

**For normal rows (COALESCE):**
```sql
UPDATE AllBots.Shopee_Comp SET
  sku = CASE
    WHEN (sku IS NULL OR sku = '' OR sku = '-') THEN COALESCE(%s, sku)
    ELSE sku
  END,
  our_price = COALESCE(%s, our_price),
  our_stock = COALESCE(%s, our_stock),
  shopee_stock = COALESCE(%s, shopee_stock),
  our_variation_images = COALESCE(%s, our_variation_images)
WHERE id = %s
```

**For NONE rows (overwrite):**
```sql
UPDATE AllBots.Shopee_Comp SET
  sku = CASE
    WHEN (sku IS NULL OR sku = '' OR sku = '-') THEN COALESCE(%s, sku)
    ELSE sku
  END,
  our_price = %s,
  our_stock = %s,
  shopee_stock = %s,
  our_variation_images = %s
WHERE id = %s
```

**Critical:** The SKU CASE statement is **identical in both modes**. SKU is **NEVER overwritten** once it has a value, regardless of the overwrite flag.

`our_variation_images` stored as single-element JSON array: `json.dumps([variation_image_url])` — e.g., `["https://..."]`.

### 4j. Commit, Error Handling & NONE Date Stamp

- Each `(shop_id, item_id)` group is **committed independently** (line 1789)
- On exception: `failed_groups += 1`, rollback, continue to next group
- On success: `processed_none_ids.extend(none_ids_in_group)` — **ALL** NONE row IDs in the group are tracked, including rows where variation matching found no model. This means a NONE row that got product-level data but no model-level match still gets its `date_taken` stamped.

After all groups:
```sql
UPDATE AllBots.Shopee_Comp SET date_taken = NOW() WHERE id IN (...)
```
Applied to all `processed_none_ids`. This prevents re-processing for `NONE_SKIP_DAYS` (5 days).

### 4k. Connection Management

Phase 1 opens **ONE** `mysql.connector` connection at line 1619 and keeps it open for the entire Phase 1 run. The same cursor is reused across all groups, URL updates, and date stamps.

---

## 6. Phase 2: SiteGiant Sales Updates

### `run_sitegiant_sales_updates()` (Line 1816)

### 5a. Initialize SiteGiant Client — `SiteGiantAPI` class (Line 700)

```python
session = requests.Session()
session.headers = {"Accept": "application/json", "Content-Type": "application/json", "Access-Token": token}
adapter = HTTPAdapter(pool_connections=20, pool_maxsize=20)
session.mount('https://', adapter)
```

Connection pooling with 20 concurrent connections, 90s timeout. This is more aggressive than Phase 1's plain session because Phase 2 uses parallel workers.

**`_get()` error handling (line 712):** Wraps everything in `try/except` — returns `{}` on ANY failure (exception, non-200 HTTP, non-200 response code). **Silent failure — no exception raised.** Only logs a warning: `"SiteGiant API error: ..."`.

**`_post()` error handling (line 723):** Does NOT catch exceptions — network/timeout errors propagate for retry logic. BUT: returns `{}` for non-200 response codes without raising. A SiteGiant logical error (HTTP 200 but `code: 400`) returns `{}` silently.

**`get_order_details()` docstring (line 758) says "raises exception on failure for retry handling"** — this is partially misleading. Only network/timeout errors raise. Logical API errors return empty `[]`.

Both `_get` and `_post` check `data.get("code") in [200, "200"]` — handles SiteGiant returning the code as either int or string.

### 5b. Get Channels — `api.get_channels()` (Line 732)

- Endpoint: `GET /channels`
- Returns: `channel_list[]` with `id`, `name`, `type`
- Filters out `type = "OpenApi"` channels (incompatible response format — returns list instead of dict)
- Optional further filtering by `TEST_SPECIFIC_SHOPS` list or `TEST_CHANNEL_LIMIT` count

### 5c. Load SKU to Row ID Mapping — `get_db_skus_for_sales()` (Line 1283)

```sql
SELECT id, sku
FROM AllBots.Shopee_Comp
WHERE sku IS NOT NULL AND sku != '' AND sku != '-'
  AND (
    (sheet_name IN ('VVIP','VIP','NEW_ITEMS') AND date_taken >= DATE_SUB(NOW(), INTERVAL 10 DAY))
    OR
    (sheet_name = 'LINKS_INPUT' AND date_taken >= DATE_SUB(NOW(), INTERVAL 4 WEEK))
  )
```

Returns `{sku_string: [row_id, row_id, ...]}`.

**Important:** This query does **NOT** include NONE rows. But the Phase 2 UPDATE JOIN later **does** include NONE in its WHERE clause — see Section 5g.

### 5d. Process Each Channel — `process_sg_channel_sku()` (Line 999)

**Order Collection Phase (Lines 1010–1048):**

Divides the 90-day window into 30-day chunks (`CHUNK_DAYS=30`, **hardcoded — not a config constant**):
- `num_chunks = ceil(90/30) = 3`
- Chunk 0: days 0–30 (most recent), Chunk 1: days 30–60, Chunk 2: days 60–90

For each chunk, paginates through orders:
- Endpoint: `GET /orders?channel_id=X&date_from=Y&date_to=Z&page=N&limit=100`
- `get_orders_page()` (line 740) handles both list and dict response formats
- For each order: if `should_skip_order(status)` is False → collect `order_id`
- Each page's collected order_ids become one batch (batch sizes vary — a page of 100 orders may yield fewer non-skipped IDs)
- **Two break conditions:**
  - `if not orders: break` — if page returns empty list (includes silent `_get()` failures)
  - `if chunk_orders >= chunk_total: break` — uses `total` from first page response
- If first page silently fails → `total=0` → immediate exit → **entire 30-day chunk missed silently**

**Order Detail Fetching Phase (Lines 1049–1080):**

Uses `ThreadPoolExecutor(max_workers=3)` for parallel processing:
```python
for i, batch in enumerate(all_order_batches):
    future = executor.submit(fetch_and_process_sg_batch_sku_with_retry, api, batch, now, i)
    time.sleep(SG_BATCH_DELAY)  # 0.15s between submissions
```

Each batch calls `fetch_and_process_sg_batch_sku_with_retry()` (line 897):

1. Calls `api.get_order_details(order_ids)`:
   - Endpoint: `POST /order/multipleOrder` with `{"order_id": [...]}`
   - Returns `order_list[]`

2. For each order, checks `should_count_order(status)` — skip if not valid

3. Parses order date via `parse_order_date()` (line 840):
   - Tries fields in order: `order_time`, `created_at`, `order_date`, `paid_time`, `create_time`
   - Supports: Unix timestamps (int/float), strings in formats `%Y-%m-%d %H:%M:%S`, `%Y-%m-%d`, `%Y/%m/%d %H:%M:%S`, `%Y/%m/%d`
   - **Fallback: `now - 60 days`** if no valid date found (lands in 60d + 90d buckets only)

4. Extracts products via `extract_order_products()` (line 859):
   - Products list key: tries `products` → `items` → `item_list` → `line_items`
   - Name key: tries `product_name` → `name` → `item_name` → `title`
   - SKU key: tries `sku` → `product_sku` → `isku` → `item_sku` → `variant_sku`
   - Quantity key: tries `quantity_purchased` → `quantity` → `qty`
   - **Allows items without product name** (as long as `qty > 0`)
   - Items without SKU are dropped by `add_sale()` later

5. Each `(sku, product_name, qty, days_ago)` → `SKUSalesData.add_sale()`

**Retry logic (lines 897–955):** 3 retries with exponential backoff (2s → 4s → 8s). Failed batches log order IDs. `BatchProcessingStats` (line 957) tracks success/failure/retry counts.

### 5e. Order Status Handling — Double Check with Gap

Orders are status-checked **twice** with **different** functions:

**First check — during order listing (line 1033):**
`should_skip_order(status)` (line 830) → if True, `order_id` is NOT collected.

**Second check — during order detail processing (line 907):**
`should_count_order(status)` (line 835) → if False, order is silently dropped.

**Gap:** An order with status `"On Hold"` or `"Unknown"` passes the first check (not in SKIP_STATUSES) but fails the second (not in VALID_STATUSES). Its order_id gets collected and detail-fetched (consuming an API call) but contributes zero sales data.

**Both use substring matching** (not exact):
```python
any(s.lower() in status_lower for s in VALID_STATUSES)
```
- `"Partially Shipped"` → matches `"shipped"` → counted as valid
- `"Cancelled by Buyer"` → matches `"cancelled"` → skipped

### 5f. SKU Sales Aggregation — `SKUSalesData` class (Line 769)

```python
def add_sale(self, sku, product_name, qty, days_ago):
    if not sku:
        return  # items without SKU silently dropped
    for period in TIME_PERIODS:  # [7, 14, 30, 60, 90]
        if days_ago <= period:
            self.sku_totals[sku][f"{period}d"] += qty
```

**Buckets are cumulative:** A sale from 5 days ago gets counted in 7d, 14d, 30d, 60d, AND 90d. This means `our_sales_90d >= our_sales_60d >= our_sales_30d >= our_sales_14d >= our_sales_7d` always. The values are running totals, not per-window exclusive counts.

Also tracks product names per SKU in `sku_product_names` (for reporting only).

`merge()` (line 795) combines data from multiple channels additively.

### 5g. Report Generation — `save_sku_sales_report()` (Line 1492)

Written **BEFORE** the database update (so there's always a record even if DB fails).

- Appends to `Jan_report.txt` (never overwrites). Adds `NEW RUN` separator if file exists.
- Sections: batch reliability stats, per-SKU breakdown sorted by 90d desc (with matched/unmatched icons), summary totals, unmatched SKU list (top 50)
- `match_details` is computed independently in `run_sitegiant_sales_updates()` (lines ~1940–1965) for the report, then **recomputed** inside `update_sales_by_sku()` for the DB write. The `_` return from the DB function discards the redundant copy.

### 5h. Bulk Database Update — `update_sales_by_sku()` (Line 1311)

```sql
-- 1. Set session lock timeout
SET SESSION innodb_lock_wait_timeout = 120

-- 2. Acquire advisory lock (prevents concurrent script runs)
SELECT GET_LOCK('ShopeeCompSalesUpdate', 10)
-- Raises RuntimeError if lock not acquired within 10 seconds

-- 3. Create temp table (in-memory)
CREATE TEMPORARY TABLE IF NOT EXISTS tmp_sku_sales (
  sku VARCHAR(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL PRIMARY KEY,
  s7 INT NOT NULL, s14 INT NOT NULL, s30 INT NOT NULL, s60 INT NOT NULL, s90 INT NOT NULL
) ENGINE=MEMORY

-- 4. Truncate (clean state for retries)
TRUNCATE TABLE tmp_sku_sales

-- 5. Insert matched SKU sales via chunk_list() in groups of 1000
INSERT INTO tmp_sku_sales (sku, s7, s14, s30, s60, s90)
VALUES (%s, %s, %s, %s, %s, %s)
ON DUPLICATE KEY UPDATE s7=VALUES(s7), s14=VALUES(s14), s30=VALUES(s30), s60=VALUES(s60), s90=VALUES(s90)

-- 6. Bulk UPDATE JOIN
UPDATE AllBots.Shopee_Comp c
JOIN tmp_sku_sales t ON c.sku = t.sku
SET c.our_sales_7d  = t.s7,
    c.our_sales_14d = t.s14,
    c.our_sales_30d = t.s30,
    c.our_sales_60d = t.s60,
    c.our_sales_90d = t.s90
WHERE 1=1
  AND (
    (c.sheet_name IN ('VVIP','VIP','NEW_ITEMS') AND c.date_taken >= DATE_SUB(NOW(), INTERVAL 10 DAY))
    OR (c.sheet_name = 'LINKS_INPUT' AND c.date_taken >= DATE_SUB(NOW(), INTERVAL 4 WEEK))
    OR c.sheet_name = 'NONE'
  )

-- 7. Release advisory lock (also in finally block as best-effort)
SELECT RELEASE_LOCK('ShopeeCompSalesUpdate')
```

**NONE row discrepancy:** `get_db_skus_for_sales()` does NOT include NONE rows in the SELECT, but the UPDATE JOIN **does** include `c.sheet_name = 'NONE'`. A NONE row can receive sales data only if its SKU was also found via a non-NONE row during the SKU lookup.

**TEMP table uses `ENGINE=MEMORY`:** Stored entirely in RAM, bounded by MySQL's `max_heap_table_size`. Could fail with "table is full" error if there are very many unique matched SKUs.

**Connection management:** Opens a NEW connection for each retry attempt (fresh session state after deadlock/rollback). This is different from Phase 1's single long-lived connection.

**Deadlock retry:** On MySQL errors 1205 (lock wait timeout) or 1213 (deadlock):
- Up to 6 retries
- Exponential backoff with jitter: `2.0 * 2^attempt + random(0, 0.5)` seconds
- Rollback before retry

---

## 7. Token Refresh Mechanism (Lines 250–370)

### Trigger
Shopee API returns HTTP 403 → `ShopeeForbiddenError` raised by `shopee_get_json()` (line 230) → caught by `run_with_token_retry()` (line 327).

### Flow
1. `refresh_shop_token_via_api(shop_id)` (line 250):
   - Reads `refresh_token` from `requestDatabase.ShopeeTokens WHERE shop_id = %s`
   - Signs request with: `{PARTNER_ID}{path}{timestamp}` (**no** access_token or shop_id in signing — different from normal API calls)
   - POSTs to `https://partner.shopeemobile.com/api/v2/auth/access_token/get`
   - Payload: `{shop_id, refresh_token, partner_id}`
   - On success: writes new tokens back to DB:
     ```sql
     UPDATE requestDatabase.ShopeeTokens
     SET access_token = %s, refresh_token = %s, expires_at = %s, updated_at = NOW()
     WHERE shop_id = %s
     ```
   - Default `expire_in`: 14400 seconds (4 hours)

2. If API refresh fails: falls back to full DB token reload via `fetch_tokens_by_shop_id()` (in case another process already refreshed)

3. Retries the original `work_fn(access_token)` with the new token (1 retry by default)

---

## 8. Complete Database Schema

### Tables READ

| Database.Table | Function | Query | Columns Read | Purpose |
|---|---|---|---|---|
| `requestDatabase.ShopeeTokens` | `fetch_tokens_by_shop_id()` | `SELECT shop_id, access_token` | `shop_id`, `access_token` | Load all Shopee API tokens |
| `requestDatabase.ShopeeTokens` | `refresh_shop_token_via_api()` | `SELECT refresh_token WHERE shop_id=%s` | `refresh_token` | Get refresh token for expired token |
| `AllBots.Shopee_Comp` | `fetch_candidate_rows()` | Filtered + NONE queries | `id`, `sku`, `our_link`, `our_variation`, `sheet_name`, `date_taken` | Phase 1 candidate rows |
| `AllBots.Shopee_Comp` | `get_db_skus_for_sales()` | SKU lookup with filter | `id`, `sku` | Phase 2 SKU→row_id mapping |

### Tables WRITTEN

| Database.Table | Function | Columns Written | Trigger | Mode |
|---|---|---|---|---|
| `requestDatabase.ShopeeTokens` | `refresh_shop_token_via_api()` | `access_token`, `refresh_token`, `expires_at`, `updated_at` | Token refresh on 403 | Direct UPDATE |
| `AllBots.Shopee_Comp` | `apply_our_link_updates()` | `our_link` | URL normalization | `executemany` batched (2000) |
| `AllBots.Shopee_Comp` | `update_product_level_fields()` | `product_name`, `product_description`, `our_sales`, `our_rating`, `our_product_price`, `our_main_images` | Phase 1 product-level | COALESCE (normal) / Overwrite (NONE) |
| `AllBots.Shopee_Comp` | `update_model_level_fields_by_variation()` | `sku`, `our_price`, `our_stock`, `shopee_stock`, `our_variation_images` | Phase 1 model-level | SKU: NEVER overwritten. Others: COALESCE (normal) / Overwrite (NONE) |
| `AllBots.Shopee_Comp` | `update_date_taken_to_now()` | `date_taken` | NONE rows after successful group processing | SET NOW() |
| `AllBots.Shopee_Comp` | `update_sales_by_sku()` | `our_sales_7d`, `our_sales_14d`, `our_sales_30d`, `our_sales_60d`, `our_sales_90d` | Phase 2 sales | UPDATE JOIN via MEMORY temp table |

---

## 9. Row Filtering Logic

### Sheet Name / Date Taken Rules

| Sheet Name | Lookback Window | Phase 1 Behavior | Phase 2 SKU SELECT | Phase 2 UPDATE JOIN |
|---|---|---|---|---|
| `VVIP` | 10 days | COALESCE (fill NULLs only) | Yes | Yes |
| `VIP` | 10 days | COALESCE (fill NULLs only) | Yes | Yes |
| `NEW_ITEMS` | 10 days | COALESCE (fill NULLs only) | Yes | Yes |
| `LINKS_INPUT` | 4 weeks | COALESCE (fill NULLs only) | Yes | Yes |
| `NONE` | Separate: skip if updated within 5 days | Overwrite all fields | **No** | **Yes** |
| All others | **Skipped** | Not processed | Not processed | Not processed |

### Additional Phase 1 Filters
- `our_link` must be non-empty
- `our_variation` must be non-empty
- When `ONLY_UPDATE_WHEN_EMPTY=True`: at least one of `product_name`, `our_price`, `our_stock`, `shopee_stock`, or `sku` must be NULL/empty (does NOT apply to NONE rows)

---

## 10. All External API Calls Summary

### Shopee Partner API

| Endpoint | Method | Function | Batching | Data |
|---|---|---|---|---|
| `/api/v2/product/get_item_base_info` | GET | `fetch_item_base_info()` | 50 items/call via `chunk_list()` | name, description, images, price |
| `/api/v2/product/get_item_extra_info` | GET | `fetch_item_extra_info()` | 50 items/call via `chunk_list()` | rating, total sales |
| `/api/v2/product/get_model_list` | GET | `fetch_model_list()` | 1 item/call | SKUs, variation prices, stock, variation images |
| `/api/v2/auth/access_token/get` | POST | `refresh_shop_token_via_api()` | 1 shop/call | new access_token, refresh_token |

### SiteGiant API

| Endpoint | Method | Function | Data |
|---|---|---|---|
| `/channels` | GET | `get_channels()` | Channel list (id, name, type) |
| `/orders` | GET | `get_orders_page()` | Paginated order list (order_id, status, dates) |
| `/order/multipleOrder` | POST | `get_order_details()` | Full order details (products, SKU, quantity) |

---

## 11. Data Flow Diagram

```
PHASE 1:
  AllBots.Shopee_Comp (rows with our_link + our_variation)
    |
  URL Normalization -> canonical form -> executemany writeback to DB
    |
  Group by (shop_id, item_id)
    |
  requestDatabase.ShopeeTokens -> access tokens (refresh on 403)
    |
  Shopee API: base_info + extra_info + model_list (tries "model" then "model_list" key)
    |
  Extract: name, description, rating, sales, price, images, SKU, stock
    |
  Match row -> model (by existing SKU or by normalized variation name)
    |
  DB UPDATE: product-level fields (all rows in group)
  DB UPDATE: model-level fields (per matched row)
    |
  Commit per group | Rollback on failure
    |
  DB UPDATE: date_taken = NOW() (all NONE rows in successfully committed groups)

PHASE 2:
  SiteGiant API: /channels -> channel list (filter out OpenApi)
    |
  For each channel:
    SiteGiant API: /orders (paginated, 3x30-day chunks over 90 days)
      |
    Collect order_ids (skip bad statuses via should_skip_order)
      |
    SiteGiant API: /order/multipleOrder (parallel 3 workers, retry 3x)
      |
    Filter by should_count_order -> extract (SKU, qty, date) -> SKUSalesData
      |
    Aggregate by SKU into cumulative time buckets (7d/14d/30d/60d/90d)
    |
  Merge all channels
    |
  AllBots.Shopee_Comp: SELECT id, sku -> build SKU->row_id mapping
    |
  Save report (Jan_report.txt) BEFORE DB write
    |
  DB: Advisory lock -> CREATE TEMP TABLE (MEMORY) -> INSERT sales
      -> UPDATE JOIN with sheet/date filter -> Release lock
    |
  AllBots.Shopee_Comp: our_sales_7d/14d/30d/60d/90d updated
```

---

## 12. Debugging & Operational Notes

### 11a. `our_sales` vs `our_sales_Xd`
Completely independent data sources. `our_sales` = Shopee API lifetime total (from `extra_info.sale`). `our_sales_7d/14d/30d/60d/90d` = SiteGiant order aggregation. They can diverge significantly.

### 11b. SKU Never Overwritten
The CASE statement in `update_model_level_fields_by_variation()` (line 1227) is identical for both normal and NONE rows. Once a SKU has a value, this script cannot change it.

### 11c. Silent SiteGiant `_get()` Failures
If SiteGiant is down, rate-limiting, or returning errors during order listing, `_get()` returns `{}` with only a warning log. The script sees zero orders for that channel/chunk and moves on. Check logs for `"SiteGiant API error:"` warnings.

### 11d. Silent `_post()` Logical Failures
If `_post()` gets HTTP 200 but SiteGiant body has `code != 200`, it returns `{}` without raising. The retry logic won't trigger because no exception is raised. `get_order_details()` returns `[]` — zero orders for that batch, no retry. The docstring claiming "raises exception on failure" is misleading for this case.

### 11e. Pagination Early Exit — Two Vectors
1. `if not orders: break` — empty page from silent `_get()` failure exits immediately
2. `if chunk_orders >= chunk_total: break` — if first page fails silently, `total=0`, exits immediately. **Entire 30-day chunk missed silently.**

### 11f. Date Fallback to 60 Days
If `parse_order_date()` can't parse any date field, it defaults to `now - 60 days`. These orders count in 60d and 90d buckets only, inflating those numbers relative to shorter periods.

### 11g. NONE Row Phase 2 Discrepancy
`get_db_skus_for_sales()` excludes NONE from the SKU lookup SELECT, but the UPDATE JOIN includes NONE in its WHERE clause. A NONE row only gets sales data if its SKU also exists in a VVIP/VIP/NEW_ITEMS/LINKS_INPUT row.

### 11h. NONE Rows Date-Stamped Even Without Model Match
`processed_none_ids.extend(none_ids_in_group)` adds ALL NONE IDs from a successfully committed group — even rows that had no variation/SKU match and received no model-level update. These rows get `date_taken = NOW()` and won't be retried for 5 days.

### 11i. `is_missing_sku()` Unused
Defined at line 512 but never called. SKU emptiness is checked via inline SQL and the CASE statement.

### 11j. Docstring vs Config: 7 Days vs 10 Days
Module docstring (line 36) says "past 7 days" for priority sheets. Actual `PRIORITY_DAYS_LOOKBACK = 10` (line 106). **Code uses 10.**

### 11k. Connection Patterns
Phase 1: single long-lived `mysql.connector` connection for entire run (line 1619). Phase 2 `update_sales_by_sku()`: new connection per retry attempt.

Phase 1 HTTP: plain `requests.Session()` context manager. Phase 2 HTTP: persistent session with `HTTPAdapter(pool_connections=20, pool_maxsize=20)`.

### 11l. MEMORY Engine Temp Table
`tmp_sku_sales` uses `ENGINE=MEMORY` — limited by `max_heap_table_size`. Could fail on very large SKU sets with "table is full" error.

### 11m. Report File
`Jan_report.txt` — hardcoded stale filename. Always appended, never overwritten. Written BEFORE the Phase 2 DB update as a safety net.

### 11n. Advisory Lock
`GET_LOCK('ShopeeCompSalesUpdate', 10)` — if another instance is already running Phase 2 DB update, waits up to 10 seconds then raises `RuntimeError`. Lock is released in `finally` block (best-effort). Only protects Phase 2, not Phase 1.

### 11o. Sales Buckets Are Cumulative
`our_sales_90d >= our_sales_60d >= our_sales_30d >= our_sales_14d >= our_sales_7d` always. A sale from day 5 counts in all five buckets. These are running totals, not per-window exclusive counts.

### 11p. Order Status Gap
An order with a status like "On Hold" passes `should_skip_order()` (collected) but fails `should_count_order()` (dropped). It consumes an API call for detail fetching but contributes zero sales.

### 11q. Substring Status Matching
Status matching is substring-based: "Partially Shipped" matches "Shipped", "Cancelled by Buyer" matches "Cancelled".

---

## 13. Common Failure Modes

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
| Model-level fields (price/stock) not matching | Variation normalization mismatch | Compare variation names; this script only replaces `, - _` with spaces (does NOT replace `.`, `/`, `&`, `(`, `)`) |

---

## 14. External Dependencies

| Dependency | Impact if Down |
|------------|----------------|
| MySQL at `34.142.159.230` (Kaushal SQL Machine) | Complete failure — no reads or writes possible |
| Shopee Partner API v2 | Phase 1 fails — product info not updated |
| SiteGiant API (`opensgapi.sitegiant.co`) | Phase 2 fails — `our_sales_*` columns not updated |
| `employee.awesomeree.com.my` | Pre-job token refresh by scheduler fails; Phase 1 may fail on 403 |

---

## 15. Complete Function Index

| # | Function/Class | Line | Category |
|---|---|---|---|
| 1 | `setup_logging()` | 165 | Infrastructure |
| 2 | `build_sheet_date_filter_clause()` | 178 | Filtering |
| 3 | `get_filter_description()` | 198 | Filtering |
| 4 | `class ShopeeForbiddenError` | 208 | Error handling |
| 5 | `hmac_sha256_hex()` | 213 | Shopee auth |
| 6 | `build_signed_url()` | 217 | Shopee auth |
| 7 | `shopee_get_json()` | 230 | Shopee API |
| 8 | `fetch_tokens_by_shop_id()` | 238 | DB read |
| 9 | `refresh_shop_token_via_api()` | 250 | Shopee auth + DB write |
| 10 | `run_with_token_retry()` | 327 | Error handling |
| 11 | `normalize_product_link()` | 376 | URL normalization |
| 12 | `to_product_link_from_i_dot()` | 393 | URL normalization |
| 13 | `canonicalize_shopee_link()` | 414 | URL normalization |
| 14 | `parse_shop_item_from_canonical()` | 429 | URL parsing |
| 15 | `chunk_list()` | 436 | Utility (batching) |
| 16 | `fetch_item_base_info()` | 444 | Shopee API |
| 17 | `fetch_item_extra_info()` | 464 | Shopee API |
| 18 | `fetch_model_list()` | 484 | Shopee API |
| 19 | `parse_int_maybe()` | 498 | Field extraction |
| 20 | `is_missing_sku()` | 512 | **UNUSED** |
| 21 | `normalize_variation()` | 517 | Variation matching |
| 22 | `extract_variation_from_model()` | 523 | Variation matching |
| 23 | `extract_description()` | 547 | Field extraction |
| 24 | `extract_main_images()` | 570 | Field extraction |
| 25 | `extract_item_price_text()` | 576 | Field extraction |
| 26 | `extract_model_price()` | 604 | Field extraction |
| 27 | `extract_product_price_from_models()` | 618 | Field extraction |
| 28 | `build_option_image_lookup()` | 632 | Image resolution |
| 29 | `resolve_model_option_image_url()` | 645 | Image resolution |
| 30 | `sum_seller_stock()` | 657 | Stock calculation |
| 31 | `sum_shopee_stock()` | 676 | Stock calculation |
| 32 | `class SiteGiantAPI` | 700 | SiteGiant client |
| 33 | `SiteGiantAPI._get()` | 712 | SiteGiant API (silent errors) |
| 34 | `SiteGiantAPI._post()` | 723 | SiteGiant API (raises on network error) |
| 35 | `SiteGiantAPI.get_channels()` | 732 | SiteGiant API |
| 36 | `SiteGiantAPI.get_orders_page()` | 740 | SiteGiant API |
| 37 | `SiteGiantAPI.get_order_details()` | 757 | SiteGiant API |
| 38 | `class SKUSalesData` | 769 | Sales aggregation |
| 39 | `should_skip_order()` | 830 | Order filtering |
| 40 | `should_count_order()` | 835 | Order filtering |
| 41 | `parse_order_date()` | 840 | Date parsing |
| 42 | `extract_order_products()` | 859 | Order extraction |
| 43 | `fetch_and_process_sg_batch_sku_with_retry()` | 897 | SiteGiant batch + retry |
| 44 | `class BatchProcessingStats` | 957 | Statistics tracking |
| 45 | `process_sg_channel_sku()` | 999 | Channel processing orchestrator |
| 46 | `fetch_candidate_rows()` | 1097 | DB read |
| 47 | `apply_our_link_updates()` | 1169 | DB write (URL) |
| 48 | `update_product_level_fields()` | 1180 | DB write (product) |
| 49 | `update_model_level_fields_by_variation()` | 1227 | DB write (model) |
| 50 | `update_date_taken_to_now()` | 1274 | DB write (date stamp) |
| 51 | `get_db_skus_for_sales()` | 1283 | DB read |
| 52 | `update_sales_by_sku()` | 1311 | DB write (sales bulk) |
| 53 | `save_sku_sales_report()` | 1492 | Report generation |
| 54 | `run_shopee_api_updates()` | 1605 | Phase 1 orchestrator |
| 55 | `work()` (closure) | 1678 | Phase 1 per-group logic |
| 56 | `run_sitegiant_sales_updates()` | 1816 | Phase 2 orchestrator |
| 57 | `main()` | 1970 | Entry point |
