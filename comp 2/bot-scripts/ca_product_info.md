# `ca_product_info.py` — Comprehensive Script Analysis

## 1. Identity & Location

| Attribute | Value |
|---|---|
| **Script** | `ca_product_info.py` (current VM TT version; renamed from `ca_my_products_info_refresh.py`) |
| **Location (VM TT)** | `C:\Users\Admin\Desktop\ca_sg\ca_product_info.py` |
| **Local source** | `C:\Users\User\Documents\Playground\repos\melinda-workspace\scripts\comp-analysis-pipeline\ca_my_products_info_refresh.py` |
| **Lines** | ~1.2k |
| **Language** | Python 3.13 |
| **Pipeline Position** | Job #1 of 3 in `ca_sg_pipeline.py` (runs first) |
| **Pipeline Task** | `ca_product_info_daily` (Windows Task Scheduler, daily at midnight MYT) |
| **Timeout** | 2 hours (set by pipeline) |
| **Typical Runtime** | ~5-6 minutes |
| **Dependencies** | `mysql-connector-python`, `requests` |

---

## 2. Purpose

Refreshes **our own product catalog** in `AllBots.Shopee_My_Products` by:
1. **Discovering** new product links from `Shopee_Comp_Data` (VVIP sheet only) that don't yet exist in `Shopee_My_Products`
2. **Refreshing** all existing active records by fetching live data from Shopee Partner API
3. **Backfilling** missing `sheet_name` values for active rows using normalized reference maps from `Shopee_Comp_Data`, then falling back to existing `Shopee_My_Products` rows

It does **NOT** touch sales or similarity columns — those are handled by scripts #2 and #3 in the pipeline.

---

## 3. Configuration Constants (lines 62-134)

| Constant | Value | Purpose |
|---|---|---|
| `DRY_RUN` | `False` (line 101) | Controls whether DB writes actually execute |
| `SLEEP_BETWEEN_CALLS_SECONDS` | `0.15` (150ms) | Delay between Shopee API calls |
| `BASE_EXTRA_BATCH_SIZE` | `50` | Max item_ids per base/extra API call |
| `UPDATE_BATCH_SIZE` | `500` | Rows per DB commit batch |
| `DEFAULT_VARIATION_LABEL` | `""` (empty string) | For products with no variations |
| `DEFAULT_MODEL_ID` | `0` | For products with no models |
| `PARTNER_ID` | `2012161` | Shopee Partner API app ID |
| `HOST` | `https://partner.shopeemobile.com` | API base URL |

**Hardcoded store name map** (lines 69-98): 28 stores mapped by `shop_id` → display name. This is used **only for logging/`shop_name` column** — the actual shops processed are determined by DB data + token availability, not this dict. If a shop_id isn't in this map, it falls back to `"shop_{shop_id}"` (line 1097).

---

## 4. Database Connections

| Config Variable | Database | Host | Purpose |
|---|---|---|---|
| `DB_CFG_WEBAPP` (line 122) | `AllBots` | `34.142.159.230` | Read/write product data |
| `DB_CFG_TOKENS` (line 129) | `requestDatabase` | `34.142.159.230` | Read/write Shopee API tokens |

Both use `root` user with `DB_PASSWORD` environment variable.

**Connection pattern:**
- `AllBots`: Single connection opened in `main()`, kept alive with `ensure_connection()` before each store (line 1112), closed in `finally` block
- `requestDatabase`: Ephemeral connections — opened and closed per-call inside `fetch_tokens_by_shop_id()` and `refresh_shop_token_via_api()`

---

## 5. Database READS

### 5A. `requestDatabase.ShopeeTokens` — Token Lookup
- **Function:** `fetch_tokens_by_shop_id()` (line 229)
- **Query:** `SELECT shop_id, access_token FROM requestDatabase.ShopeeTokens`
- **When:** Once at startup (line 1059), and again as fallback if API token refresh fails (line 342)
- **Returns:** `{shop_id: access_token}` dict for all shops

### 5B. `requestDatabase.ShopeeTokens` — Refresh Token Lookup
- **Function:** `refresh_shop_token_via_api()` (line 240)
- **Query:** `SELECT refresh_token FROM requestDatabase.ShopeeTokens WHERE shop_id = %s`
- **When:** Only when Shopee API returns HTTP 403 (expired token)

### 5C. `AllBots.Shopee_Comp_Data` — New Link Discovery
- **Function:** `discover_new_links_from_comp_data()` (line 648)
- **Query (lines 666-674):**
  ```sql
  SELECT DISTINCT c.our_link, c.region, c.sheet_name
  FROM AllBots.Shopee_Comp_Data c
  LEFT JOIN AllBots.Shopee_My_Products m
    ON c.our_link = m.our_link AND c.region = m.region
  WHERE c.status = 'active'
    AND c.sheet_name = 'VVIP'
    AND m.id IS NULL
  ```
- **Purpose:** Finds `our_link` URLs in Comp Data that have NO matching row in My Products yet (LEFT JOIN ... IS NULL pattern)
- **Returns 3 maps:** `new_shop_items` (shop_id → {item_id: our_link}), `sheet_name_map` (our_link → sheet_name), `region_map` (our_link → region)
- **URL parsing:** Extracts `shop_id` and `item_id` from each `our_link` via regex `/product/(\d+)/(\d+)` (line 637)
- **Current VM TT note:** `normalize_our_link()` strips trailing slashes before lookup, and the current script also loads authoritative `sheet_name` / `region` reference maps from both `Shopee_Comp_Data` and existing `Shopee_My_Products` rows before processing.

### 5D. `AllBots.Shopee_My_Products` — Existing Active Records
- **Function:** `load_active_items_by_shop()` (line 725)
- **Query (lines 733-739):**
  ```sql
  SELECT our_shop_id, our_item_id, our_link, region
  FROM AllBots.Shopee_My_Products
  WHERE status = 'active'
    AND our_shop_id IS NOT NULL
    AND our_item_id IS NOT NULL
  ```
- **Returns:** `shop_items` dict (shop_id → {item_id: our_link}) + `region_map` (our_link → region)
- **Dedup:** Uses first `our_link` found per (shop_id, item_id) pair

---

## 6. Database WRITES

### 6A. `AllBots.Shopee_My_Products` — UPSERT (main target)
- **Function:** `batch_upsert_product_info()` (line 784)
- **Operation:** `INSERT ... ON DUPLICATE KEY UPDATE`
- **Unique key:** `uk_link_variation` = (`region`, `our_link`, `our_variation`)

#### INSERT columns (for new records — lines 811-819):

| Column | Source |
|---|---|
| `region` | `region_map` (Comp Data priority → existing DB → fallback "MY") |
| `our_link` | From Comp Data discovery or existing records (NOT constructed by script) |
| `our_variation` | `get_model_list` API → `build_variation_text_from_model()` |
| `our_shop_id` | Parsed from `our_link` URL |
| `our_item_id` | Parsed from `our_link` URL |
| `our_model_id` | `get_model_list` API |
| `product_name` | `get_item_base_info` API → `item_name` |
| `product_description` | `get_item_base_info` API → `description` or `extended_description` |
| `sku` | `get_model_list` API → `model_sku` |
| `our_price` | `get_model_list` API → model-level `price_info[0].current_price` |
| `our_product_price` | `get_item_base_info` API → item-level price; **fallback: computed min-max from model prices** |
| `our_stock` | `get_model_list` API → `stock_info_v2.seller_stock` (summed) |
| `shopee_stock` | `get_model_list` API → `stock_info_v2.shopee_stock` (summed) |
| `our_rating` | **`get_item_extra_info` API → `rating_star`** (line 944) |
| `our_main_images` | `get_item_base_info` API → `image.image_url_list` (stored as JSON) |
| `our_variation_images` | `get_model_list` API → option image lookup (stored as JSON array) |
| `shop_name` | From `STORE_NAMES` dict or fallback `"shop_{id}"` |
| `status` | Hardcoded `"active"` (line 871) |
| `sheet_name` | From `sheet_name_map` / post-refresh backfill, can be NULL before the final backfill pass |

> Note: `created_at` is **NOT** in the INSERT column list — it relies on MySQL column DEFAULT (`CURRENT_TIMESTAMP`).

#### ON DUPLICATE KEY UPDATE behavior (for existing records — lines 824-841):

| Column | Update Behavior |
|---|---|
| `our_shop_id` | Always overwritten |
| `our_item_id` | Always overwritten |
| `our_model_id` | Always overwritten |
| `product_name` | Always overwritten |
| `product_description` | Always overwritten |
| `sku` | **COALESCE** — preserves existing if new is NULL (protects out-of-stock SKUs) |
| `our_price` | Always overwritten |
| `our_product_price` | Always overwritten |
| `our_stock` | Always overwritten |
| `shopee_stock` | Always overwritten |
| `our_rating` | Always overwritten |
| `our_main_images` | Always overwritten |
| `our_variation_images` | Always overwritten |
| `shop_name` | Always overwritten |
| `sheet_name` | **COALESCE** — preserves existing if new is NULL |
| `updated_at` | Set to `NOW()` |
| **`status`** | **NOT updated** (only set on INSERT) |
| **`created_at`** | **NOT in query** (never touched after initial insert) |

#### Columns explicitly NOT written by this script:
- `our_sales`, `our_sales_7d/14d/30d/60d/90d` — script #2 (sales_sync)
- `shopee_var_sales_*`, `shopee_var_value_*`, `shopee_var_sales_last_synced` — script #2
- `sitegiant_sales_value_*` — script #2
- `discounted_price`, `voucher_name` — script #2
- `product_similarity_*`, `variation_similarity_*`, `*_excluded`, `has_*_exclusions` — script #3 (comp_variation_matcher)
- `automated_remarks`, `user_remarks`, `Delete_Remark`, `last_page`, `original_page` — admin/manual

### 6B. `requestDatabase.ShopeeTokens` — Token Refresh
- **Function:** `refresh_shop_token_via_api()` (lines 285-293)
- **When:** Only when Shopee API returns HTTP 403
- **Query:**
  ```sql
  UPDATE requestDatabase.ShopeeTokens
  SET access_token = %s, refresh_token = %s,
      expires_at = %s, updated_at = NOW()
  WHERE shop_id = %s
  ```

### 6C. `AllBots.Shopee_My_Products` — Sheet-Name Backfill
- **Function:** `backfill_missing_sheet_names()` (current VM TT version)
- **Operation:** `UPDATE ... JOIN` against normalized `our_link` keys and resolved sheet names from `Shopee_Comp_Data`, then a second pass against existing `Shopee_My_Products` rows as fallback
- **When:** After all store refreshes complete
- **Effect:** Fills active rows with null/blank `sheet_name` values and updates `updated_at`

---

## 7. External API Calls (Shopee Partner API)

| # | Endpoint | Function | Batch | Provides |
|---|---|---|---|---|
| 1 | `/api/v2/product/get_item_base_info` | `fetch_item_base_info()` (line 357) | Up to 50 item_ids | `item_name`, `description`/`extended_description`, `image.image_url_list`, `price_info`/`current_price` |
| 2 | `/api/v2/product/get_item_extra_info` | `fetch_item_extra_info()` (line 390) | Up to 50 item_ids | `rating_star` |
| 3 | `/api/v2/product/get_model_list` | `fetch_model_list()` (line 420) | 1 item_id per call | `model` list (model_sku, price_info, stock_info_v2, tier_index), `tier_variation` (option names/images) |
| 4 | `/api/v2/auth/access_token/get` | `refresh_shop_token_via_api()` (line 254) | N/A (POST) | New `access_token`, `refresh_token`, `expire_in` |

**All API calls are HMAC-SHA256 signed** using `PARTNER_KEY` + `PARTNER_ID` + timestamp + access_token + shop_id (lines 163-173).

**Retry behavior:**
- `shopee_get_json()` (line 176): 3 attempts with exponential backoff (1s, 2s) on 5xx/timeout/connection errors
- `run_with_token_retry()` (line 312): 1 retry on HTTP 403 — refreshes token via Shopee API, then retries the work function

---

## 8. Processing Flow (verified from `main()`, line 1052)

```
main()
  |
  |-- 1. setup_logging() -> stdout handler
  |-- 2. fetch_tokens_by_shop_id() -> READ requestDatabase.ShopeeTokens (all tokens)
  |-- 3. Connect to AllBots DB (single connection for entire run)
  |
  |-- 4. load_sheet_name_reference_maps()
  |     +-- READ normalized refs from Shopee_Comp_Data and Shopee_My_Products
  |     +-- Prefer VVIP > VIP > LINKS_INPUT > NEW_ITEMS when resolving sheet_name
  |
  |-- 5. discover_new_links_from_comp_data()
  |     +-- READ Shopee_Comp_Data LEFT JOIN Shopee_My_Products
  |        WHERE status='active' AND sheet_name='VVIP' AND m.id IS NULL
  |     +-- Parse shop_id/item_id from normalized our_link URLs
  |     +-- Returns: new_shop_items, discovered_sheet_name_map, discovered_region_map
  |
  |-- 6. load_active_items_by_shop()
  |     +-- READ Shopee_My_Products WHERE status='active'
  |     +-- Returns: shop_items (by shop_id), existing_region_map
  |
  |-- 7. Merge reference maps (discovered refs only fill gaps in the authoritative maps)
  |-- 8. Merge new discoveries into shop_items (existing items + new items combined)
  |
  |-- 9. FOR EACH shop_id in shop_items:
  |     |-- Skip if no token for this shop
  |     |-- ensure_connection() -> reconnect AllBots DB if stale
  |     |-- process_store_refresh():
  |     |   |-- Chunk item_ids into batches of 50
  |     |   |-- For each batch:
  |     |   |   |-- fetch_item_base_info() -> batched API call (with token retry)
  |     |   |   |-- fetch_item_extra_info() -> batched API call (with token retry)
  |     |   |   +-- For each item_id in batch:
  |     |   |       |-- fetch_model_list() -> per-item API call (with token retry)
  |     |   |       |-- Extract: name, description, images, rating, prices
  |     |   |       |-- For each model/variation:
  |     |   |       |   |-- Build variation text from tier options
  |     |   |       |   |-- Extract: SKU, model price, seller stock, shopee stock, variation image
  |     |   |       |   +-- Append to updates_buffer
  |     |   |       +-- Flush buffer when >= 500 rows
  |     |   +-- Final flush of remaining rows
  |     |   +-- Each flush -> batch_upsert_product_info() -> INSERT ON DUPLICATE KEY UPDATE
  |     +-- Log store results (items_processed, rows_updated)
  |
  |-- 10. backfill_missing_sheet_names()
  |     +-- UPDATE active rows with blank sheet_name using normalized our_link keys
  |     +-- Fallback to existing Shopee_My_Products references when needed
  |
  +-- 11. Log final summary + close DB connection
```

---

## 9. Table Schema — `AllBots.Shopee_My_Products` (79 columns)

### Indexes

| Index Name | Columns | Unique |
|---|---|---|
| `PRIMARY` | `id` | Yes |
| `uk_link_variation` | `region`, `our_link`, `our_variation` | Yes |
| `idx_our_shop_item_model` | `our_shop_id`, `our_item_id`, `our_model_id` | No |
| `idx_region` | `region` | No |
| `idx_sheet_name` | `sheet_name` | No |
| `idx_sku` | `sku` | No |
| `idx_status_product_name` | `status`, `product_name` | No |
| `idx_status_shop_name` | `status`, `shop_name` | No |
| `idx_status_sku` | `status`, `sku` | No |
| `ft_search` | `product_name`, `sku`, `our_link` (fulltext) | No |

### Column Ownership Map

| Column Group | Written By | Columns |
|---|---|---|
| **Identity (key)** | This script (INSERT only) | `region`, `our_link`, `our_variation` |
| **Product IDs** | This script | `our_shop_id`, `our_item_id`, `our_model_id` |
| **Product info** | This script | `product_name`, `product_description`, `sku`, `our_price`, `our_product_price`, `our_stock`, `shopee_stock`, `our_rating`, `our_main_images`, `our_variation_images`, `shop_name` |
| **Metadata** | This script (INSERT) / COALESCE (UPDATE) | `status`, `sheet_name` |
| **Timestamps** | This script (UPDATE) / MySQL DEFAULT (INSERT) | `updated_at`, `created_at` |
| **Sales (rolling)** | Script #2 — sales_sync | `our_sales`, `our_sales_7d/14d/30d/60d/90d` |
| **Shopee var sales** | Script #2 — sales_sync | `shopee_var_sales_7d/14d/30d/60d/90d`, `shopee_var_value_7d/14d/30d/60d/90d`, `shopee_var_sales_last_synced` |
| **SiteGiant sales** | Script #2 — sales_sync | `sitegiant_sales_value_7/14/30/60/90days` |
| **Pricing** | Script #2 — sales_sync | `discounted_price`, `voucher_name` |
| **Variation matching** | Script #3 — comp_variation_matcher | `product_similarity_score/reason/datetime`, `variation_similarity_score/reason/datetime`, `product_similarity_excluded`, `variation_similarity_excluded`, `has_product_exclusions`, `has_variation_exclusions` |
| **Admin/manual** | Web app / manual | `automated_remarks`, `user_remarks`, `Delete_Remark`, `last_page`, `original_page` |

---

## 10. Data Profile (as of 2026-03-11)

| Metric | Value |
|---|---|
| **Total rows in Shopee_My_Products** | 1,902 (1,653 MY + 249 SG) |
| **Shops with data** | 4: Karlmobel (649 rows), Murahya (636), ValueSnap (368), Kinata SG (249) |
| **Unique items** | 89 (81 MY + 8 SG) |
| **Source VVIP links in Comp Data** | 115 distinct (107 MY + 8 SG), 3,378 total VVIP rows |
| **Last update** | 2026-03-10 4:06 PM MYT |

---

## 11. Resilience Features

| Feature | Implementation | Lines |
|---|---|---|
| HTTP retry (transient) | 3 attempts, exponential backoff (1s, 2s) on 5xx/timeout | 176-204 |
| Token self-healing | 403 → refresh via Shopee API → retry work function | 312-346, 240-309 |
| Token fallback | If API refresh fails → reload all tokens from DB | 341-342 |
| DB reconnect | `ensure_connection()` pings DB before each store | 211-222 |
| SKU preservation | `COALESCE(VALUES(sku), sku)` — doesn't overwrite existing with NULL | 830 |
| sheet_name preservation | `COALESCE(VALUES(sheet_name), sheet_name)` | 839 |
| Delisted item handling | Items not returned by API are logged+skipped, not errored | 927-933 |
| Per-store error isolation | Each store wrapped in try/except, failure doesn't stop other stores | 1134-1139 |
| Closure fix | Default arg binding (`_batch=item_batch`, `_iid=item_id`) prevents stale references | 913, 955 |

---

## 12. Downstream Dependencies

**This script MUST run first** because scripts #2 and #3 depend on `Shopee_My_Products` rows existing:

| Downstream Script | Depends On (from this script) |
|---|---|
| #2 `ca_my_products_sales_sync.py` | `our_link`, `our_item_id`, `our_shop_id` to sync sales |
| #3 `ca_comp_variation_matcher.py` | `our_variation`, `product_name`, `sku` for AI matching against competitor data |

If this script fails, downstream scripts either process stale data or skip new products entirely.

---

## 13. Known Issues & Edge Cases

| Issue | Status | Detail |
|---|---|---|
| Null SKUs for out-of-stock variations | Expected | Shopee API returns null `model_sku` for depleted stock; COALESCE preserves existing SKU |
| GCP Cloud SQL firewall | Fixed (Mar 10) | VM TT IP `151.240.33.25` was not whitelisted; script failed from Mar 6-10 |
| `sheet_name` varchar(50) in AllBots | Low risk | Current values ("VVIP") are well under 50 char limit |
| Docstring vs code mismatch | Minor | Script docstring (line 35) still understates `sheet_name` handling; code writes `status`, `sheet_name`, and backfills missing `sheet_name` values after refresh |
| Store name map completeness | Low risk | `STORE_NAMES` has 28 stores; any shop not in the map defaults to `"shop_{id}"` for display |

---

## 14. Extraction Helper Functions

| Function | Lines | Purpose |
|---|---|---|
| `build_variation_text_from_model()` | 439-467 | Joins tier option names (e.g., "Red Large") from `tier_variation` + model's `tier_index` |
| `extract_description()` | 470-494 | Gets plain `description` or parses `extended_description.field_list` text fields (strips URLs) |
| `extract_main_images()` | 497-500 | Extracts `image.image_url_list` from base info |
| `extract_item_price_text()` | 503-530 | Gets item-level price from `current_price` or `price_info[0].current_price` |
| `extract_model_price()` | 533-546 | Gets model-level price from `price_info[0].current_price` or `current_price` |
| `extract_product_price_from_models()` | 549-560 | Computes min-max price range string from all models |
| `build_option_image_lookup()` | 563-573 | Builds `{(tier_idx, option_idx): image_url}` lookup from tier variations |
| `resolve_model_option_image_url()` | 576-588 | Resolves a model's variation image by matching its `tier_index` against the lookup |
| `sum_seller_stock()` | 591-607 | Sums `stock_info_v2.seller_stock` entries (only saleable ones) |
| `sum_shopee_stock()` | 610-627 | Sums `stock_info_v2.shopee_stock` entries |

---

*Last verified: 2026-03-18 from current VM TT source at `C:\Users\Admin\Desktop\ca_sg\ca_product_info.py` (~1.2k lines)*
