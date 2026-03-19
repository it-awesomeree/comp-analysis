# VM TT Database Analysis (4 Core Tables + Script-Related Dependencies)

## Purpose

This document is the **authoritative database reference** for the VM TT midnight pipeline (`ca_product_info_daily`) with scope limited to:

1. `AllBots.Shopee_My_Products`
2. `AllBots.Shopee_Comp_Data`
3. `AllBots.Shopee_Variation_Match`
4. `AllBots.Shopee_Variation_Match_Runs`

And only additional tables directly used by the same 3 scripts:

- `AllBots.Shopee_VariationSalesDaily`
- `AllBots.Shopee_VariationSalesSyncState`
- `requestDatabase.ShopeeTokens`
- `requestDatabase.SitegiantToken`

---

## Verification Baseline

- VM: `VM TT`
- Scheduler task: `ca_product_info_daily`
- Trigger: daily `00:00:00 +08:00`
- Pipeline scripts (in order):
  1. `ca_product_info.py`
  2. `ca_my_products_sales_sync.py`
  3. `ca_comp_variation_matcher.py --live`
- Live DB snapshot timestamp (SGT): `2026-03-19T16:50:45`
- DB server clock at snapshot: `2026-03-19 08:50:46` (`SYSTEM`)

---

## 1. Pipeline-to-Table Ownership Map

| Script | Core table interactions | Related table interactions |
|---|---|---|
| `ca_product_info.py` | Reads `Shopee_Comp_Data`; updates/upserts `Shopee_My_Products`; uses `Shopee_Comp_Data` as authoritative mapping for normalized links/sheet/region | Reads/writes `requestDatabase.ShopeeTokens` for Shopee token refresh |
| `ca_my_products_sales_sync.py` | Updates `Shopee_My_Products` (IDs/SKU, rolling sales/value metrics, SiteGiant sales columns) | Reads/writes `requestDatabase.ShopeeTokens`; reads/writes `requestDatabase.SitegiantToken`; writes `Shopee_VariationSalesDaily`; writes `Shopee_VariationSalesSyncState` |
| `ca_comp_variation_matcher.py` | Reads `Shopee_My_Products` + latest `Shopee_Comp_Data`; writes run lifecycle to `Shopee_Variation_Match_Runs`; writes/deactivates/upserts `Shopee_Variation_Match` | None outside the 4 core tables |

---

## 2. Core Table Snapshot (Live)

| Table | Exact rows | Active rows | Freshness / latest timestamp | Notes |
|---|---:|---:|---|---|
| `AllBots.Shopee_My_Products` | 2,839 | 2,839 | `updated_at` max: `2026-03-18 16:11:19` | Distinct `our_link`: 133, distinct `our_shop_id`: 4 |
| `AllBots.Shopee_Comp_Data` | 8,076 | 8,076 | `date_taken` max: `2026-03-16` | Distinct `our_link`: 163, distinct `comp_link`: 441 |
| `AllBots.Shopee_Variation_Match_Runs` | 1,527 | N/A | `updated_at` max: `2026-03-18 19:02:47` | Statuses: DONE=1,527; FAILED/RUNNING/QUEUED=0 |
| `AllBots.Shopee_Variation_Match` | 96,185 | 33,801 | `created_at` max: `2026-03-18 19:02:47` | Inactive history retained: 62,384 |

Decision split on active `Shopee_Variation_Match` rows:

- `NO_MATCH`: 16,993 (50.27%)
- `UNSURE`: 10,702 (31.66%)
- `MATCH`: 6,106 (18.06%)

---

## 3. Core Table Schema/Key Anchors

### 3.1 `AllBots.Shopee_My_Products`

- Columns: 59
- PK: `id`
- Script-critical unique key: `uk_link_variation(region, our_link, our_variation)`
- High-impact write groups:
  - Product identity/content: `our_shop_id`, `our_item_id`, `our_model_id`, `sku`, `product_name`, `our_variation`, etc.
  - Shopee rolling metrics: `shopee_var_sales_*`, `shopee_var_value_*`, `shopee_var_sales_last_synced`
  - SiteGiant rolling metrics: `our_sales_7d/14d/30d/60d/90d`
  - Classification metadata: `sheet_name`, `region`, `status`

Current health checks:

- Active rows missing any core IDs (`our_shop_id`/`our_item_id`/`our_model_id`): `0`
- Blank/null `sheet_name` in table-level snapshot: `0`

### 3.2 `AllBots.Shopee_Comp_Data`

- Columns: 25
- PK: `id`
- Script-critical unique key: `uk_comp_snapshot(region, our_link, comp_link, comp_variation, date_taken)`
- `sheet_name` column exists and is used by `ca_product_info.py` for authoritative mapping.
- `our_item_id_extracted` is generated and used in broader app joins (not directly written by this pipeline).

Current health checks:

- Active rows with blank `comp_link`: `0`
- Blank/null `sheet_name` in table-level snapshot: `0`

### 3.3 `AllBots.Shopee_Variation_Match_Runs`

- Columns: 11
- PK: `id`
- Run lifecycle written by script: `QUEUED -> RUNNING -> DONE/FAILED`
- Additional run diagnostics: `error_message`, `skip_reason`, `data_staleness_warning`

Current health checks:

- Latest run IDs: `1527..1518`, all `DONE`
- DONE runs with zero rows in match table: `0`

### 3.4 `AllBots.Shopee_Variation_Match`

- Columns: 16
- PK: `id`
- Script-critical unique key: `uk_run_variation_pair(run_id, our_link, our_variation, comp_link, comp_variation)`
- Active/inactive model:
  - Old auto rows are deactivated (`status='inactive'`)
  - New rows inserted/upserted as `status='active'`

Current health checks:

- Rows with `run_id` missing reference in runs table: `0`
- Active rows tied to non-DONE/missing runs: `0`
- `is_manual_override=1` rows currently: `0`

---

## 4. Related Tables (Direct Dependencies Only)

### 4.1 `AllBots.Shopee_VariationSalesDaily`

- Purpose: daily fact table for per-variation sales/revenue windows.
- Used by `ca_my_products_sales_sync.py`:
  - Deletes overlapping window before upsert (idempotency)
  - Deletes records older than 90-day retention
  - Upserts daily aggregates with `ON DUPLICATE KEY UPDATE`
  - Source for rolling joins back to `Shopee_My_Products`
- Live snapshot: 73,216 rows; latest `sale_date`: `2026-03-18`; latest `last_synced_at`: `2026-03-18 16:05:29`

### 4.2 `AllBots.Shopee_VariationSalesSyncState`

- Purpose: resume pointer per Shopee shop (`last_time_to`, `last_synced_at`).
- Also stores SiteGiant sentinel state (`shop_id = 0`) in script flow.
- Live snapshot: 29 rows; latest `last_synced_at`: `2026-03-18 16:11:20`; max `last_time_to`: `1773849600`

### 4.3 `requestDatabase.ShopeeTokens`

- Purpose: Shopee API token store used by scripts #1 and #2.
- Live snapshot: 34 rows; 34 rows have non-empty `access_token`; max `updated_at`: `2026-03-19 08:40:20`

### 4.4 `requestDatabase.SitegiantToken`

- Purpose: SiteGiant bearer token used by script #2.
- Live snapshot: one row (`id=1`), `updated_at=2026-03-19 06:35:58`

---

## 5. Cross-Table Integrity Results

All checks below were `0` bad rows at the live snapshot time:

1. Active `Shopee_Variation_Match` rows linked to non-DONE/missing run rows.
2. DONE `Shopee_Variation_Match_Runs` rows with no child rows in `Shopee_Variation_Match`.
3. Active comp `(region, our_link)` pairs missing in active `Shopee_My_Products`.
4. Active my-product `(region, our_link)` pairs missing in active `Shopee_Comp_Data`.
5. `Shopee_Variation_Match.run_id` orphan references.

Interpretation: as of the snapshot, the 4-core-table pipeline state is structurally consistent.

---

## 6. Midnight Run Context (Latest Verified)

From the 2026-03-19 midnight run:

- `product_info`: completed in ~150s
- `shopee_sales_sync`: completed in ~480s
- `comp_variation_matcher`: completed in ~10,216s (~2h50m)
- Pipeline overall: `ALL PASSED`

Notable script signals from same run:

- Product info discovered 133 unique items across 4 shops; upserted 5,676 rows.
- Matcher processed total 400 listing pairs.

---

## 7. Operational Caveat (Time Zone)

The scheduler is SGT (`+08:00`) while DB server clock is `SYSTEM` time (UTC-like at capture).  
Date filters using `CURDATE()` or `DATE(...)` should always be interpreted with this offset in mind during midnight troubleshooting.

---

## 8. Maintenance Rule for This Doc

When refreshing this reference, update at minimum:

1. Snapshot timestamp (SGT + DB server clock)
2. Core table counts/freshness
3. Run status distribution
4. Integrity check results
5. Any SQL-path or ownership changes in the 3 scripts
