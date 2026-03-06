# `ca_shopee_listing_to_db.py` — Complete Script Documentation

**Location:** `VM3 → C:\Users\Admin\Desktop\Shopee Comp My links Api\ca_shopee_listing_to_db.py`
**Size:** 813 lines, 34,616 bytes | **Last modified:** 2026-03-02
**Python:** 3.13 (`C:\Users\Admin\AppData\Local\Programs\Python\Python313\python.exe`)
**Dependencies:** `mysql-connector-python`, `requests`

---

## 1. Purpose

This script is **Job 1 (the SEED job)** of a 7-script nightly Shopee Competitive Analysis pipeline orchestrated by `scheduler.py` on VM3. Its sole purpose is to sync all **live product listings** from **28 Awesomeree Shopee stores** into the `AllBots.Shopee_Comp` database table by calling the Shopee Partner API.

It populates the **"our product"** side of the competitive analysis table (columns prefixed `our_*`). Every other script in the pipeline depends on the seed rows this script creates. Without it, downstream scripts have nothing to enrich — no variation matching, no sales data, no ads metrics, no competitor similarity scoring.

**In simple terms:** This script answers the question *"What products do we currently have live across all our Shopee stores?"* and writes that answer to the database every night, adding only products it hasn't seen before.

---

## 2. Pipeline Position & Data Dependencies

```
scheduler.py (orchestrator, runs nightly via Windows Task Scheduler)
│
├─ Job 1: ca_shopee_listing_to_db.py          ← THIS SCRIPT (SEED)
├─ Job 2: our_variation_preprocessing.py       (backfills our_variation column)
├─ Job 3: ca_ai_variation_match.py             (AI variation matching)
├─ Job 4: shopee_comp_shopee_sales.py          (Shopee order sales sync)
├─ Job 5: Shopee-mylinks-sales-data-merged.py  (product info + SiteGiant sales)
├─ Job 6: ca_shopee_ads_metrics.py             (ads metrics)
└─ Job 7: ca_similarity_check.py               (AI similarity scoring)
```

- **Scheduling:** Triggered nightly by Windows Task Scheduler via `scheduler.py`. Has a 6-hour safety timeout. On failure, retried up to 2 times (3 total attempts) with token refresh between retries.
- **Boot recovery:** If VM3 reboots mid-pipeline, `boot_resume_pipeline.ps1` detects a checkpoint file and resumes from the last incomplete job.
- **Chain recovery:** `chain_today_pipeline.ps1` waits for any resume to finish, then starts a fresh daily pipeline cycle.
- **All 7 jobs run sequentially** with 5-minute gaps between them.

---

## 3. Execution Flow (Step by Step)

```
1. setup_logging()
   → Logs to stdout only (INFO level)
   → Format: "2026-03-02 10:30:15 | INFO | message"

2. fetch_tokens_by_shop_id()
   → READ: requestDatabase.ShopeeTokens → all (shop_id, access_token) pairs
   → Opens and closes its own short-lived DB connection
   → Returns Dict[int, str] mapping shop_id → access_token

3. Filter stores: match 28 hardcoded shop_ids against available tokens
   → Logs which stores have tokens and which will be skipped

4. Connect to AllBots database (single connection, shared across all stores)

5. load_existing_product_keys(conn)
   → READ: AllBots.Shopee_Comp → SELECT DISTINCT our_shop_id, our_item_id, our_model_id
   → Loads into Python set() for O(1) dedup lookups
   → This is the KEY mechanism that makes the script incremental

6. For each store (up to 28):
   a. fetch_all_live_item_ids()
      → API: /api/v2/product/get_item_list (item_status=NORMAL)
      → Paginated: 100 items per page, follows has_next_page/next_offset
      → Handles both dict and int item formats from API
      → Returns sorted, deduplicated list of item_ids

   b. For each batch of 50 items:
      → API: /api/v2/product/get_item_base_info (batched, 50 ids per call)
             - Excludes tax_info and complaint_policy to reduce payload
      → API: /api/v2/product/get_item_extra_info (batched, 50 ids per call)

   c. For each individual item:
      → Extract product-level fields (name, description, rating, sales, images, price)
      → API: /api/v2/product/get_model_list (1 call per item — NOT batched)
      → If product has no models → creates synthetic model with model_id=0

   d. For each model/variation:
      → Check: is (shop_id, item_id, model_id) in existing_keys set?
        → YES: skip (increment skipped_existing counter)
        → NO: build row tuple, add to buffer, add key to set
      → Extract variation-level fields (SKU, price, stock, variation text, variation image)

   e. When buffer reaches 2,000 rows:
      → WRITE: INSERT INTO AllBots.Shopee_Comp (executemany, all-or-nothing)
      → Flush buffer

   f. Final flush of remaining buffered rows for this store

   g. On any exception: log error, continue to next store (no crash)

7. Close allbots_conn

8. Log summary stats: stores processed/failed, items, models, new inserts, skipped
```

---

## 4. Database Reads & Writes (Complete)

### Reads

| # | Database.Table | Query | When | Connection |
|---|---------------|-------|------|------------|
| 1 | `requestDatabase.ShopeeTokens` | `SELECT shop_id, access_token FROM ShopeeTokens` | Startup | Own short-lived connection |
| 2 | `requestDatabase.ShopeeTokens` | `SELECT refresh_token WHERE shop_id = %s` | On HTTP 403 | Own short-lived connection |
| 3 | `AllBots.Shopee_Comp` | `SELECT DISTINCT our_shop_id, our_item_id, our_model_id WHERE all NOT NULL` | Startup (once) | Shared `allbots_conn` |

### Writes

| # | Database.Table | Operation | When | Connection |
|---|---------------|-----------|------|------------|
| 1 | `requestDatabase.ShopeeTokens` | `UPDATE SET access_token, refresh_token, expires_at, updated_at` | On token refresh (403 recovery) | Own short-lived connection |
| 2 | `AllBots.Shopee_Comp` | `INSERT INTO ... (21 columns) VALUES ... (executemany)` | Every 2,000 rows or end-of-store | Shared `allbots_conn` |

**Critical behavior:** This script is **INSERT-only**. It **never updates** existing rows (no price/stock/sales updates). It **never deletes** rows. If a product's price or stock changes, this script will NOT reflect that — downstream scripts (Jobs 4-6) handle updates to existing rows.

---

## 5. The 21 Columns Inserted

| Column | Data Source | Extraction Logic |
|--------|-----------|-----------------|
| `product_name` | `get_item_base_info` → `item_name` | Falls back to `"item_id={id}"` if empty |
| `product_description` | `get_item_base_info` → `description` or `extended_description.field_list[type=text]` | URLs stripped via regex; extended description joins text fields with newlines |
| `sku` | `get_model_list` → `model_sku` | Per-variation SKU, NULL if empty |
| `our_link` | Constructed | Always `https://shopee.com.my/product/{shop_id}/{item_id}` |
| `our_variation` | `get_model_list` → `tier_variation` + `tier_index` mapping | Resolves tier indices to option labels, e.g. "Black Large". Empty string if no variation. |
| `shop_name` | Hardcoded `STORE_NAMES` dict | e.g. "Karlmobel", "Murahya" |
| `our_price` | `get_model_list` → `price_info[0].current_price` | Float, per-variation price. Falls back to direct `current_price` field. |
| `our_product_price` | `get_item_base_info` → `current_price`, or min-max from models | String like `"29.90"` or `"29.90 - 59.90"`. Falls back to computing min/max across all model prices. |
| `our_sales` | `get_item_extra_info` → `sale` | Integer total sales count |
| `our_stock` | `get_model_list` → `stock_info_v2.seller_stock` | Sum of seller stock entries (only saleable entries counted) |
| `shopee_stock` | `get_model_list` → `stock_info_v2.shopee_stock` | Sum of Shopee warehouse stock (all entries, handles string values) |
| `our_rating` | `get_item_extra_info` → `rating_star` | Float star rating |
| `our_main_images` | `get_item_base_info` → `image.image_url_list` | JSON array of image URL strings |
| `our_variation_images` | `get_model_list` → tier option images | JSON array with single variation image URL, or NULL |
| `our_shop_id` | Config | Integer shop_id |
| `our_item_id` | API | Integer item_id |
| `our_model_id` | API | Integer model_id (0 if no variations) |
| `status` | Hardcoded | Always `"active"` |
| `last_page` | Hardcoded | Always `"Shopee MY"` |
| `date_taken` | Runtime | `date.today()` — the date the row was first inserted |
| `sheet_name` | Hardcoded | Always `"NONE"` |

---

## 6. Shopee Partner API Usage

| Endpoint | Granularity | Batching | Rate Limit |
|----------|------------|----------|------------|
| `/api/v2/product/get_item_list` | 1 call per page per store | 100 items/page, paginated | 150ms sleep |
| `/api/v2/product/get_item_base_info` | 1 call per batch | 50 item_ids per call, excludes tax_info & complaint_policy | 150ms sleep |
| `/api/v2/product/get_item_extra_info` | 1 call per batch | 50 item_ids per call | 150ms sleep |
| `/api/v2/product/get_model_list` | **1 call per item** | NOT batched | 150ms sleep |
| `/api/v2/auth/access_token/get` | 1 call per store (on demand) | N/A | Only on 403 |

**API authentication:** HMAC-SHA256 signing. Regular API calls sign: `partner_id + path + timestamp + access_token + shop_id`. Token refresh signs differently: `partner_id + path + timestamp` (no access_token, no shop_id). Partner ID: `2012161`.

**Bottleneck:** The `get_model_list` call is per-item (not batched), so a store with 500 products makes 500 individual model API calls. With 150ms sleep each, that's ~75 seconds just for model fetches per store, plus base/extra info calls.

---

## 7. Token Management & 403 Recovery

```
On HTTP 403 (ShopeeForbiddenError):
  1. Call Shopee OAuth refresh API with stored refresh_token
     → POST /api/v2/auth/access_token/get
     → Payload: { shop_id, refresh_token, partner_id }
     → Uses requests.post() directly (not shared session — no connection pooling)
     → Opens its own short-lived DB connection to read refresh_token and write new tokens
  2. If refresh succeeds:
     → UPDATE requestDatabase.ShopeeTokens with new access_token + refresh_token + expires_at
     → expires_at stored as Unix timestamp: current_time + expire_in (default 14400 = 4 hours)
     → Retry the failed API call with new token
  3. If refresh fails:
     → Fall back: reload ALL tokens from DB (in case another process refreshed them)
     → Retry with reloaded token
  4. Max 1 retry per API call (retries=1 in run_with_token_retry)
```

**Note:** The scheduler also does a pre-flight token refresh via the web app endpoint (`employee.awesomeree.com.my/api/shopee/refresh-tokens`) before starting this script, AND a per-job refresh before each pipeline job. So tokens are typically fresh when this script starts.

---

## 8. Deduplication Strategy

The unique key is the tuple `(our_shop_id, our_item_id, our_model_id)`.

1. At startup, ALL existing keys are loaded from DB into a Python `set` via `load_existing_product_keys()`
2. For each model encountered, the key is checked against this set — O(1) lookup
3. If the key exists → skip (counter incremented, no DB write)
4. If the key is new → row is buffered AND the key is immediately added to the set (prevents intra-run duplicates if the same product somehow appears twice)

**Implication:** If you delete rows from `Shopee_Comp` and re-run this script, it will re-insert them. If you want to "refresh" a product's data, you must delete its row first, then re-run.

---

## 9. The 28 Stores

| Shop ID | Store Name | Shop ID | Store Name |
|---------|-----------|---------|-----------|
| 323123914 | Encikplastic | 1262266197 | Lovento |
| 322597841 | das.nature | 741350502 | luxbinT |
| 321742675 | MasonGym | 1628391442 | Lenza Eyewear |
| 321734518 | Stationarylah | 1156025778 | Solaree |
| 289957447 | Kedai-Garden | 826077906 | Adelmar |
| 275072345 | Bidarimu | 741362022 | Wilmermobel |
| 429078278 | Chairsy | 1156006182 | JagerHelmet |
| 252419950 | Karlmobel | 738554401 | Flashree |
| 214632596 | FynnKoffer | 648555578 | das.nature2 |
| 176242540 | Emas Gift | 631161167 | Hiranai |
| 139072319 | Murahya | 422263284 | ValueSnap |
| 126025674 | BahEmas | 503721641 | dutchgaming |
| 428943241 | Secret Racer | 373454740 | Gameniture |
| 336726327 | Atlusrack | 1686573083 | Kinata SG |

**Note:** All links are generated as `shopee.com.my` URLs, including Kinata SG (shop_id 1686573083).

---

## 10. Configuration Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `DRY_RUN` | `False` | If `True`, logs inserts without writing to DB. Token refreshes and API calls still execute normally. `new_inserted` counter will show 0. |
| `SLEEP_BETWEEN_CALLS_SECONDS` | `0.15` | Rate limit between Shopee API calls |
| `ITEM_LIST_PAGE_SIZE` | `100` | Products per page in item_list API |
| `BASE_EXTRA_BATCH_SIZE` | `50` | Items per batch for base_info/extra_info API calls |
| `INSERT_BATCH_SIZE` | `2000` | Rows buffered before DB flush |
| `DEFAULT_MODEL_ID` | `0` | Synthetic model_id for products without variations |
| `DEFAULT_VARIATION_LABEL` | `""` | Empty string for products without variations |
| `TARGET_TABLE` | `Shopee_Comp` | Target DB table |
| `PARTNER_ID` | `2012161` | Shopee Partner API app ID |

---

## 11. Environment Requirements

| Requirement | Value |
|-------------|-------|
| `DB_PASSWORD` env var | **Required** — `os.environ["DB_PASSWORD"]` (crashes with `KeyError` at module load if missing) |
| DB Host | `34.142.159.230` (Kaushal SQL Machine, Google Cloud) |
| DB User | `root` |
| Databases used | `AllBots` (table: `Shopee_Comp`), `requestDatabase` (table: `ShopeeTokens`) |
| Shopee API Host | `partner.shopeemobile.com` |

---

## 12. Failure Modes & Debugging Guide

| Symptom | Likely Cause | Where to Look |
|---------|-------------|---------------|
| `KeyError: 'DB_PASSWORD'` at startup | Environment variable not set | Check scheduled task's environment or the bot_runner that launches this |
| `RuntimeError: Missing access token for shop_id=X` | Token missing from `ShopeeTokens` table for that shop | Query `requestDatabase.ShopeeTokens` for that shop_id |
| `ShopeeForbiddenError (HTTP 403)` repeated | Token refresh failing — refresh_token may be expired/revoked | Check `ShopeeTokens.refresh_token`, may need manual re-auth |
| `RuntimeError: get_item_list failed` | Shopee API error (rate limit, maintenance, invalid params) | Check Shopee error code in the message |
| Script produces 0 new inserts but runs fine | All products already exist in DB (normal for daily incremental runs) | Check `skipped_existing` count in logs |
| Script takes very long | Many products across 28 stores + 150ms per API call + model_list is per-item | Normal — estimate ~1hr+ for full run |
| One store fails, others succeed | Per-store exception handling catches and continues | Check logs for `"Error processing store"` |
| Many stores fail in sequence after a certain point | Likely dropped `allbots_conn` (single shared DB connection, no reconnect logic) | Check if early stores succeeded but later ones all have MySQL connection errors |
| DB connection timeout/lost | Kaushal SQL Machine down or network issue | Check DB server status |
| Entire batch of 2,000 rows fails to insert | One bad row in the batch (e.g., value too long, encoding error) — `executemany` is all-or-nothing | All buffered rows for that store are lost; check for data anomalies in that store's products |
| HTTP 429 or 500 from Shopee kills a store | Only 403 is retried with token refresh. All other HTTP errors bubble up and skip the store | Shopee API logs; the error message includes status code |
| `our_stock` is unexpectedly lower than `shopee_stock` | `sum_seller_stock` excludes non-saleable inventory and only handles int types; `sum_shopee_stock` includes all entries and handles string stock values | Compare raw API response for that model |
| `product_description` is NULL | Both simple `description` and `extended_description.field_list` were empty for that product | Normal for some Shopee listings |

### Log Format Example

```
2026-03-02 10:30:15 | INFO | Starting multi-store Shopee sync -> AllBots.Shopee_Comp (DRY_RUN=False)
2026-03-02 10:30:15 | INFO | Total stores to process: 28
2026-03-02 10:30:16 | INFO | Loaded 45230 existing product keys from Shopee_Comp
2026-03-02 10:30:16 | INFO | ============================================================
2026-03-02 10:30:16 | INFO | Processing store: Encikplastic (323123914)
2026-03-02 10:30:17 | INFO | Store Encikplastic (323123914): Found 142 live items
2026-03-02 10:30:45 | INFO | Store Encikplastic: Inserted batch of 312 rows (total new: 312)
2026-03-02 10:30:45 | INFO | Store Encikplastic complete: items=142, models=312, new=45, skipped=267
...
2026-03-02 11:45:00 | INFO | SYNC COMPLETE
2026-03-02 11:45:00 | INFO |   Stores processed: 27
2026-03-02 11:45:00 | INFO |   Stores failed: 1
2026-03-02 11:45:00 | INFO |   Total items fetched: 3200
2026-03-02 11:45:00 | INFO |   Total models fetched: 8500
2026-03-02 11:45:00 | INFO |   New records inserted: 150
2026-03-02 11:45:00 | INFO |   Existing records skipped: 8350
```

---

## 13. Extraction Helper Details

### `extract_description` — Dual Format Handling

Handles two Shopee description formats:

1. **Simple:** `item.description` (plain text string) — used if present and non-empty
2. **Extended:** `item.description_info.extended_description.field_list` — falls back to this if #1 is empty
   - Iterates `field_list`, keeps only `field_type == "text"` entries (skips images, etc.)
   - Strips URLs from text using regex `(?i)\b(?:https?://|www\.)\S+\b`
   - Joins remaining text segments with newlines

### `extract_item_price_text` — Product-Level Price

Priority chain:
1. `item.current_price` (direct field, handles int/float/string)
2. `item.price_info[0].current_price` (array format)
3. Returns None if neither works

### `extract_model_price` — Variation-Level Price

Priority chain:
1. `model.price_info[0].current_price` (handles int/float)
2. `model.current_price` (direct field fallback)
3. Returns None if neither works

### `our_product_price` Fallback Chain

1. First tries `extract_item_price_text(base_item)` — item-level price
2. If None → `extract_product_price_from_models(models)` — computes min/max across all model prices
   - Single price: `"29.90"`
   - Range: `"29.90 - 59.90"` (when min != max, threshold 0.0001)

### Variation Image Resolution

Two-step process:
1. `build_option_image_lookup()` — builds `{(tier_index, option_index): image_url}` from `tier_variation` data
2. `resolve_model_option_image_url()` — for a given model, walks its `tier_index` array and returns the **first** matching image

Only the first tier with an image wins per model.

### Stock Calculation Differences

| Behavior | `sum_seller_stock` | `sum_shopee_stock` |
|----------|-------------------|-------------------|
| Checks `if_saleable` | Yes — skips entries where `if_saleable` is explicitly `False` | No — sums all entries regardless |
| Handles string stock values | No — only counts `int` type | Yes — parses digit strings like `"50"` |
| Returns | `int` total or `None` if no valid entries | `int` total or `None` if no valid entries |

### `build_variation_text_from_model` — Variation Label Construction

Maps a model's `tier_index` array (e.g., `[0, 2]`) to human-readable option labels:
- Walks each position in `tier_index`
- Looks up the corresponding tier in `tier_variation`
- Finds the option at that index from `option_list` (or `options`)
- Handles options as dicts (tries `option`, `value`, `name` keys) or plain strings
- Joins all resolved labels with spaces: e.g., `"Black Large"`

### `normalize_variation` — UNUSED

Defined but **never called** anywhere in the script. Lowercases and normalizes comma/dash/underscore to spaces. Likely a leftover from an earlier approach.

---

## 14. DB Connection Lifecycle

A critical detail for debugging long runs:

- `allbots_conn` is opened **once** in `main()` and shared across all 28 stores sequentially. It is only closed in the `finally` block of `main()`.
- There is **no reconnection logic**. If the MySQL connection drops (idle timeout, network blip, Kaushal SQL Machine restart) mid-run, all subsequent stores will fail with a MySQL connection error.
- The `requestDatabase` connections (for tokens) are different — `fetch_tokens_by_shop_id()` and `refresh_shop_token_via_api()` each open/close their **own** short-lived connections, so they're more resilient to connection drops.

---

## 15. HTTP Error Handling — Only 403 Triggers Retry

`shopee_get_json()` handles HTTP errors:

```
403 → ShopeeForbiddenError → caught by run_with_token_retry → token refresh + retry (1 retry)
All other HTTP errors (429, 500, etc.) → resp.raise_for_status() → requests.HTTPError
  → NOT caught by retry mechanism → bubbles up to process_store's try/except → store is skipped
```

There is no retry or backoff for rate limiting (429), server errors (500), or network timeouts.

---

## 16. Closure Pattern Inside `process_store`

Inside the `for item_batch in chunk_list(...)` loop, closures are defined and immediately called:

```python
def fetch_base_extra_work(token: str):
    base_map = fetch_item_base_info(session, token, shop_id, item_batch)
    ...
```

The closure captures `item_batch` by reference. This works correctly because the closure is **called immediately** via `run_with_token_retry()` within the same loop iteration. Same pattern used for `fetch_all_ids_work` and `fetch_model_work`.

---

## 17. `insert_rows` — All-or-Nothing Batch

`insert_rows()` uses `cur.executemany(sql, rows)` — if a single row in the 2,000-row batch has a data issue (value too long, encoding error), the **entire batch fails** and nothing gets inserted. The `conn.commit()` only runs if `executemany` succeeds.

This exception bubbles up to `process_store`'s outer `try/except`, which logs the error and skips to the next store — but **all buffered rows for that store are lost** (not just the problematic batch, because the exception exits the store processing loop).

---

## 18. Unused Code (Dead Code)

| Item | Type | Notes |
|------|------|-------|
| `from collections import defaultdict` | Import | Never referenced |
| `from dataclasses import dataclass` | Import | Never referenced |
| `from urllib.parse import urlparse` | Import | Never referenced |
| `normalize_variation()` | Function | Defined but never called. `build_variation_text_from_model()` is used instead. |

These won't cause errors but add noise when debugging.

---

## 19. `fetch_all_live_item_ids` — Pagination & Edge Cases

The pagination logic handles multiple API response formats:

- Tries `response.item` first, then `response.item_list` (Shopee API inconsistency across versions)
- Handles items returned as either `{"item_id": 123}` dicts or bare `123` integers
- `next_offset` fallback: if Shopee returns a non-int `next_offset`, it manually increments by `ITEM_LIST_PAGE_SIZE` (100)
- Final result is `sorted(set(all_ids))` — deduplicated and deterministic ordering
- Debug logging only (not visible at INFO level): `"get_item_list: offset=X fetched=Y total=Z has_next=True"`

---

## 20. Key Design Notes for Debugging

1. **INSERT-only, never updates** — If you need refreshed price/stock/sales on existing products, this script won't help. Downstream scripts (Jobs 4-6) handle updates to existing rows.

2. **In-memory dedup set** — The existing keys set is loaded ONCE at startup. If another process inserts rows while this script is running, duplicates are theoretically possible (though unlikely since the scheduler runs jobs sequentially with a process lock).

3. **model_list is the bottleneck** — Unlike base_info and extra_info (batched at 50), model_list requires one API call per item. A store with 500 items = 500 model calls = ~75 seconds minimum.

4. **Per-store error isolation** — The `try/except` in `process_store` catches all exceptions for a single store without crashing the entire run. Check logs for `"Error processing store"` to find which stores failed.

5. **Token refresh chain** — Three layers of token management: (a) scheduler pre-flight refresh via web app, (b) per-job refresh before this script runs, (c) in-script 403 → OAuth refresh → DB fallback.

6. **`date_taken` = insertion date** — This column records when the row was first inserted, not when the product was created on Shopee. Useful for tracking when new products were discovered.

7. **Kinata SG gets MY links** — All `our_link` values use `shopee.com.my/product/...` regardless of store region.

8. **No file-based logging** — This script only logs to stdout. When run via `scheduler.py`, the output is captured in the subprocess. There is no dedicated log file for this script alone.

9. **`DB_PASSWORD` crashes at import time** — Because `os.environ["DB_PASSWORD"]` is used at module level in `DB_CFG_ALLBOTS` and `DB_CFG_TOKENS`, a missing env var causes a `KeyError` before any function runs — no graceful error message.

10. **Token refresh uses `requests.post()` directly** — Not the shared `session` object, so it doesn't benefit from HTTP connection pooling.
