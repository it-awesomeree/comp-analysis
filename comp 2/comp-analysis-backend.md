# Comp Analysis 2 — Backend Documentation (VVIP & SG)

**Purpose**: Comprehensive documentation of the backend architecture for the Shopee MY VVIP and Shopee SG Competitive Analysis pages — API routes, repository layer, shared utilities, database schema, and how region isolation works.

---

## Table of Contents

1. [Overview](#overview)
2. [File Structure](#file-structure)
3. [Database Schema](#database-schema)
4. [API Routes — VVIP Products](#api-routes--vvip-products)
5. [API Routes — SG Products](#api-routes--sg-products)
6. [API Routes — AI Analysis (Shared)](#api-routes--ai-analysis-shared)
7. [Repository Layer — VVIP](#repository-layer--vvip)
8. [Repository Layer — SG](#repository-layer--sg)
9. [VVIP vs SG: Differences](#vvip-vs-sg-differences)
10. [Normalized Group Summary](#normalized-group-summary)
11. [Data Enrichment (Shared)](#data-enrichment-shared)
12. [Shared Calculations — shopee-history-calculations.ts](#shared-calculations)
13. [Database Connection — mysql-allbots.ts](#database-connection)
14. [Query Building & SQL Patterns](#query-building--sql-patterns)
15. [Caching & Performance](#caching--performance)
16. [Security](#security)
17. [Data Pipeline — End to End](#data-pipeline--end-to-end)

---

## Overview

Both VVIP and SG pages share the same architecture but are **fully isolated by region**. The stack:

- **Framework**: Next.js 15 App Router (API routes at `app/api/*/route.ts`)
- **Database**: MySQL (AllBots DB) via `mysql-allbots.ts` — NOT the main `db.ts` pool
- **Tables**: `Shopee_My_Products` (mp) + `Shopee_Comp_Data` (cd) — shared between VVIP and SG
- **Region Isolation**: VVIP passes `region = "MY"`, SG passes `region = "SG"` — filtering happens at SQL JOIN time (`cd.region = 'MY'` vs `cd.region = 'SG'`)
- **AI**: Shared `/api/comp-analysis/analyze/route.ts` (Gemini 2.0 Flash) — region-agnostic
- **Auth**: Firebase Admin SDK via middleware (not in individual routes)
- **Pattern**: API Route → Repository → AllBots MySQL, with shared enrichment between DB and response

### Key Architectural Decision

Unlike Comp 1 (Shopee MY normal page) which uses `Shopee_Comp` (legacy flat table) via `shopee-products-repository.ts` and the main `db.ts` pool, Comp 2 uses **normalized tables** (`Shopee_My_Products` + `Shopee_Comp_Data`) via dedicated repository files and the `mysql-allbots.ts` connection. The two systems are completely independent — Comp 2 changes cannot affect Comp 1.

---

## File Structure

```
AWESOMEREE-WEB-APP/
├── app/api/
│   ├── shopee-my-vvip/products/
│   │   ├── route.ts                    # Main GET/POST/PUT/PATCH/DELETE (region="MY")
│   │   ├── counts/route.ts            # Tab counts (region="MY")
│   │   └── details/route.ts           # Product descriptions & images (region="MY")
│   ├── shopee-sg/products/
│   │   ├── route.ts                    # Main GET/POST/PUT/PATCH/DELETE (region="SG")
│   │   ├── counts/route.ts            # Tab counts (region="SG")
│   │   └── details/route.ts           # Product descriptions & images (region="SG")
│   └── comp-analysis/
│       └── analyze/route.ts           # Gemini AI analysis (shared, region-agnostic)
├── lib/
│   ├── services/
│   │   ├── shopee-vvip-products-repository.ts    # VVIP data access (~278 lines)
│   │   ├── shopee-sg-products-repository.ts      # SG data access (~278 lines)
│   │   ├── shopee-normalized-group-summary.ts    # Pre-computed summary tables (~350 lines)
│   │   ├── shopee-products-enrichment.ts         # Server-side enrichment (shared, ~1000 lines)
│   │   └── mysql-allbots.ts                      # AllBots DB connection wrapper
│   ├── shared/
│   │   ├── shopee-history-calculations.ts        # Grouping, metrics, sorting (~616 lines)
│   │   └── shopee-sales-values.ts                # Sales number parsing utilities
│   └── type.ts                                    # Product type definitions
```

---

## Database Schema

### Database: `AllBots` (MySQL, Cloud SQL)

Both VVIP and SG pages query the same two tables. The `region` column is the **sole differentiator**.

### Table 1: `Shopee_My_Products` (alias: `mp`)

Our product variations — always-current, overwritten daily by in-house API bot.

#### Core Product Fields

| Column | Type | Purpose |
|--------|------|---------|
| `id` | INT UNSIGNED AUTO_INCREMENT PK | Row identifier |
| `product_name` | VARCHAR(255) | Our product name |
| `product_description` | TEXT | Our product description |
| `sku` | VARCHAR(100) | SKU (nullable) |
| `our_link` | TEXT | Our Shopee product URL |
| `our_item_id` | BIGINT UNSIGNED | Numeric item ID extracted from URL |
| `our_model_id` | BIGINT UNSIGNED | Model ID |
| `our_shop_id` | BIGINT UNSIGNED | Shop ID |
| `shop_name` | TEXT | Our shop name |
| `sheet_name` | TEXT | Category: VVIP, VIP, LINKS_INPUT, NEW_ITEMS, NONE |
| `our_variation` | TEXT | Our variation name |
| `region` | VARCHAR(10) | **"MY" or "SG"** — the key differentiator |

#### Pricing Fields

| Column | Type | Purpose |
|--------|------|---------|
| `our_price` | DECIMAL(10,2) | Selling price |
| `our_product_price` | VARCHAR(255) | List price |
| `discounted_price` | DECIMAL(10,2) | Discounted price |
| `voucher_name` | VARCHAR(255) | Active voucher |

#### Metrics (Current)

| Column | Type | Purpose |
|--------|------|---------|
| `our_sales` | INT | All-time sales |
| `our_stock` | INT | Current stock |
| `shopee_stock` | INT | Shopee-reported stock |
| `our_rating` | DECIMAL(3,2) | Rating (0-5) |

#### Time-Windowed Sales (per window: 7d, 14d, 30d, 60d, 90d)

| Column Pattern | Type | Purpose |
|---------------|------|---------|
| `our_sales_{N}d` | INT | SiteGiant sales over N days |
| `shopee_var_sales_{N}d` | INT | Shopee variation-level sales |
| `shopee_var_value_{N}d` | DECIMAL(12,2) | Shopee variation sales value (revenue) |

#### Ads Metrics (per window: 7d, 14d, 30d, 60d, 90d)

| Column Pattern | Type | Purpose |
|---------------|------|---------|
| `ads_spend_{N}d` | DECIMAL(12,2) | Ad spend |
| `roas_{N}d` | DECIMAL(12,4) | Return on ad spend |
| `ads_visitors_{N}d` | INT | Ad visitors |
| `ads_conv_rate_{N}d` | DECIMAL(8,4) | Conversion rate |

#### Metadata

| Column | Type | Purpose |
|--------|------|---------|
| `status` | VARCHAR(20) | active, pending, fixed, new, deleted |
| `automated_remarks` | TEXT | Bot-generated remarks |
| `user_remarks` | TEXT | User-entered remarks |
| `last_page` | VARCHAR(20) | Last UI tab |
| `original_page` | VARCHAR(20) | Original tab (for restore) |
| `Delete_Remark` | VARCHAR(255) | Set on soft-delete |

#### Key Indexes

| Index | Columns |
|-------|---------|
| `uk_link_variation` | `(our_link(500), our_variation(255))` UNIQUE |
| `idx_sku` | `(sku)` |
| `idx_our_shop_item_model` | `(our_shop_id, our_item_id, our_model_id)` |
| `idx_status_product_name` | `(status, product_name)` |
| `ft_search` | FULLTEXT on `(product_name, sku, our_link)` |

---

### Table 2: `Shopee_Comp_Data` (alias: `cd`)

Competitor data — historical, append-only. One row per competitor variation per scrape date.

#### Competitor Fields

| Column | Type | Purpose |
|--------|------|---------|
| `id` | BIGINT UNSIGNED AUTO_INCREMENT PK | Row identifier |
| `our_link` | TEXT | Maps back to our product |
| `our_item_id_extracted` | BIGINT UNSIGNED | **Pre-extracted numeric item ID** — indexed for fast JOINs |
| `comp_product` | TEXT | Competitor product name |
| `comp_product_description` | TEXT | Competitor description |
| `comp_link` | TEXT | Competitor Shopee URL |
| `comp_variation` | TEXT | Competitor variation name |
| `comp_shop` | TEXT | Competitor shop name |
| `comp_price` | DECIMAL(10,2) | Competitor price |
| `comp_product_price` | VARCHAR(255) | Competitor list price |
| `comp_sales` | INT | Competitor lifetime sales |
| `comp_monthly_sales` | TEXT | Competitor monthly sales |
| `comp_rating` | DECIMAL(3,2) | Competitor rating |
| `comp_stock` | INT | Competitor stock |
| `comp_main_images` | JSON | Competitor product images |
| `comp_variation_images` | TEXT | Competitor variation images |
| `region` | VARCHAR(10) | **"MY" or "SG"** — filtered in JOIN |

#### Scrape & Analysis Fields

| Column | Type | Purpose |
|--------|------|---------|
| `date_taken` | DATE | Scrape date |
| `corrected_link` | TEXT | Corrected URL if applicable |
| `our_advantage` | TEXT | AI-generated advantage |
| `our_issue` | TEXT | AI-generated issue |
| `status` | VARCHAR(20) | active, deleted |
| `created_at` | DATETIME | Row creation time |

---

### Table 3: `Shopee_VVIP_Group_Summary`

Pre-computed aggregates for VVIP page. Refreshed every 10 minutes.

| Column | Type | Purpose |
|--------|------|---------|
| `status` | VARCHAR | Product status |
| `product_name` | VARCHAR | Group key |
| `max_id` | INT | Latest row ID |
| `min_our_item_id` | BIGINT UNSIGNED | **Shopee product ID (tiebreaker for sort)** — auto-migrated via `ensureSummarySchema()` |
| `max_date_taken` | DATE | Most recent scrape date |
| `min_comp_product` | VARCHAR | First competitor name |
| `min_sku` | VARCHAR | First SKU |
| `row_count` | INT | Rows in group |
| `category_rank` | INT | Pre-computed category sort rank |
| `shop_name_empty` | INT | 1 if shop name is empty |
| `min_shop_name` | VARCHAR | First shop name (sorted) |
| `sg_sales_total_{N}d` | INT | SiteGiant sales total (per window) |
| `sp_sales_total_{N}d` | INT | Shopee sales total (per window) |
| `sp_value_total_{N}d` | DECIMAL | Shopee value total (per window) |
| `updated_at` | TIMESTAMP | Last refresh time |

### Table 4: `Shopee_SG_Group_Summary`

Same schema as VVIP summary, but for SG data. Uses `cd.region = 'SG'` in its refresh query.

---

## API Routes — VVIP Products

### File: `app/api/shopee-my-vvip/products/route.ts`

Uses `createShopeeProductsGetHandler()` shared handler with VVIP-specific config.

**Hardcoded region**: `REGION = "MY"`

#### GET — Fetch Products

```
GET /api/shopee-my-vvip/products?limit=50&offset=0&status=active&search=...&sheetName=VVIP&...
```

**Flow**:
1. Parse query params via `parseFilterParams()` (whitelist validated)
2. Call `fetchVvipProducts(filters, "MY")` from VVIP repository
3. Call `enrichShopeeProductRows(rows, { metricsWindow })` for server-side enrichment
4. Return enriched rows with pagination metadata

**Response**: Same shape as Comp 1
```json
{
  "success": true,
  "data": [...],
  "count": 50,
  "total": 1234,
  "groupTotal": 567,
  "message": "50 records fetched",
  "filters": { ... },
  "debug": { "limit": 50, "offset": 0, "timings": { ... } }
}
```

**Cache**: `Cache-Control: private, max-age=30, stale-while-revalidate=60`

**Allowed Sort Keys** (15 keys, whitelist-validated):
```
my, comp, sku, date, id, category, shop,
priceMy, salesMy, stockMy,
priceComp, salesComp, compMonthlySales, stockComp,
sgSales, sgSalesValue, spSales, spSalesValue,
remarks
```

**Note**: VVIP does NOT have similarity/exclusion sort keys (no `Shopee_Comp_Similarity_Exclusions` JOIN).

#### POST — Create Product

```
POST /api/shopee-my-vvip/products
Body: { ourProduct, competitorProduct, remarks, sourceType }
```

- Calls `insertVvipProduct(payload)` — transactional
- Inserts into `Shopee_My_Products` + optionally `Shopee_Comp_Data`
- Extracts `our_item_id` from URL for JOIN compatibility
- Default: `sheet_name="VIP"`, `status="new"`, `last_page="Fixed/New"`
- Invalidates VVIP count cache

#### PUT — Update Product

```
PUT /api/shopee-my-vvip/products
Body: { id, ourProduct, competitorProduct, remarks, sku }
```

- Updates `Shopee_My_Products` only (not comp data)
- Invalidates count cache

#### PATCH — Update Category

```
PATCH /api/shopee-my-vvip/products
Body: { id, sheetName }
```

- Updates `sheet_name` on `Shopee_My_Products`
- Invalidates count cache

#### DELETE — Soft Delete

```
DELETE /api/shopee-my-vvip/products?id=123
```

- Sets `status = 'deleted'`, `Delete_Remark = 'Deleted via UI'`
- Does NOT hard-delete the row
- Invalidates count cache

---

### File: `app/api/shopee-my-vvip/products/counts/route.ts`

```
GET /api/shopee-my-vvip/products/counts
```

Single optimized query:
```sql
SELECT
  IFNULL(SUM(status = 'active'), 0)    AS active_count,
  COUNT(DISTINCT CASE WHEN status = 'active' THEN product_name END) AS active_groups,
  IFNULL(SUM(status = 'pending'), 0)   AS pending_count,
  IFNULL(SUM(status = 'fixed'), 0) + IFNULL(SUM(status = 'new'), 0) AS fixed_new_count,
  IFNULL(SUM(status = 'deleted'), 0)   AS deleted_count
FROM Shopee_My_Products
WHERE region = 'MY'
```

- Returns `{ active, activeGroups, pending, fixedNew, deleted }`
- Cache: `private, max-age=30, stale-while-revalidate=60`

---

### File: `app/api/shopee-my-vvip/products/details/route.ts`

```
POST /api/shopee-my-vvip/products/details
Body: { ids: number[] }
```

- Calls `fetchVvipProductDetails(ids, "MY")`
- Returns: `product_description`, `comp_product_description`, image arrays
- Max 200 IDs per request
- JOIN uses `NORMALIZED_LINK_EQ` for MY region, `our_item_id_extracted` for others

---

## API Routes — SG Products

### File: `app/api/shopee-sg/products/route.ts`

**Hardcoded region**: `REGION = "SG"`

Identical structure and sort keys as VVIP. The only differences:
- Imports from `shopee-sg-products-repository` instead of `shopee-vvip-products-repository`
- Calls `fetchSgProducts(filters, "SG")` instead of `fetchVvipProducts(filters, "MY")`

### File: `app/api/shopee-sg/products/counts/route.ts`

Same as VVIP but with `fetchSgProductCounts("SG")` → `WHERE region = 'SG'`

### File: `app/api/shopee-sg/products/details/route.ts`

Same as VVIP but with `fetchSgProductDetails(ids, "SG")`

---

## API Routes — AI Analysis (Shared)

### File: `app/api/comp-analysis/analyze/route.ts`

Shared between Comp 1, VVIP, and SG. Region-agnostic — receives product data in request body, doesn't query by region.

Uses **Google Gemini 2.0 Flash** for AI-powered competitor analysis.

(See Comp 1 backend doc for full details — identical behavior.)

---

## Repository Layer — VVIP

### File: `lib/services/shopee-vvip-products-repository.ts` (~278 lines)

#### Table Constants

```typescript
const MP = "Shopee_My_Products";
const CD = "Shopee_Comp_Data";
const SUMMARY_TABLE = "Shopee_VVIP_Group_Summary";
```

#### JOIN Strategy

```typescript
// Primary JOIN — indexed numeric match (fast)
const JOIN_BY_ITEM_ID = `LEFT JOIN Shopee_Comp_Data cd ON cd.our_item_id_extracted = mp.our_item_id`;

// Fallback JOIN — string comparison (slow, used only for MY region details)
const NORMALIZED_LINK_EQ = "TRIM(TRAILING '/' FROM cd.our_link) = TRIM(TRAILING '/' FROM mp.our_link)";
```

**`vvipJoinForRegion(region?, useLatestPerProduct?)`**:
- No region: `JOIN_BY_ITEM_ID` (no filter)
- With region, no latest filter: `JOIN_BY_ITEM_ID AND cd.region = '<region>'`
- With region + `useLatestPerProduct = true`: **Derived table optimization** — creates a subquery `cd_max` that finds `MAX(date_taken)` per `our_item_id_extracted`, then double-JOINs to return only the latest comp data per product (~3x row reduction):

```sql
LEFT JOIN (
  SELECT our_item_id_extracted, MAX(date_taken) AS latest_date
  FROM Shopee_Comp_Data WHERE region = 'MY'
  GROUP BY our_item_id_extracted
) cd_max ON cd_max.our_item_id_extracted = mp.our_item_id
LEFT JOIN Shopee_Comp_Data cd
  ON cd.our_item_id_extracted = mp.our_item_id
  AND cd.region = 'MY'
  AND cd.date_taken = cd_max.latest_date
```

**When `useLatestPerProduct` is used**: Only when `!filters.dateFrom && !filters.dateTo` — i.e., when no date filter is active. When a date filter IS active, the derived table is skipped so all comp data is available for accurate date-based filtering.

**Key insight**: Always uses `our_item_id_extracted` (indexed integer) for the JOIN. Never falls back to string matching on the main query — only on the details endpoint for MY region.

#### SELECT Columns (35 columns)

The query selects from both `mp.*` and `cd.*` tables, aliasing them to match what the enrichment layer expects:

```sql
mp.id, mp.product_name, mp.sku, mp.our_link, mp.our_variation,
mp.shop_name, mp.our_price, mp.our_product_price, mp.discounted_price, mp.voucher_name,
mp.our_sales, mp.our_stock, mp.shopee_stock, mp.our_rating,

cd.comp_product, cd.comp_link, cd.comp_variation, cd.comp_shop,
cd.comp_price, cd.comp_product_price, cd.comp_sales, cd.comp_monthly_sales,
cd.comp_rating, cd.comp_stock,

NULL AS our_advantage, NULL AS our_issue,   -- Not available in normalized schema

cd.date_taken, mp.automated_remarks, mp.user_remarks, mp.sheet_name,
mp.status, mp.last_page, mp.original_page,

mp.our_sales_7d/14d/30d/60d/90d,
mp.shopee_var_sales_7d/14d/30d/60d/90d,
mp.shopee_var_value_7d/14d/30d/60d/90d
```

Plus dynamic window-aware columns:
```sql
COALESCE(mp.our_sales_30d, mp.our_sales) AS our_sales_display
COALESCE(mp.shopee_var_sales_30d, 0) AS shopee_sales_display
COALESCE(mp.shopee_var_value_30d, 0) AS shopee_value_display
```

**Note**: `our_advantage` and `our_issue` are aliased as NULL — these fields don't exist in the normalized schema (they were in the legacy `Shopee_Comp` table).

#### fetchVvipProducts(filters, region?)

Main query function. Two strategies based on `groupByName`:

**Strategy 1: Grouped Query** (`groupByName = true`) — used by Active tab

Two-phase pagination:
```
Phase 1: Get paginated product NAMES (with optional summary table optimization)

  Option A — Summary table (when canUseGroupedSummary = true):
    SELECT product_name FROM Shopee_VVIP_Group_Summary
    WHERE status = ? ORDER BY [sort] LIMIT ? OFFSET ?

  Option B — Live GROUP BY (when filters are active):
    SELECT mp.product_name FROM Shopee_My_Products mp
    [LEFT JOIN Shopee_Comp_Data cd ...]
    [WHERE ...] GROUP BY mp.product_name
    ORDER BY [sort] LIMIT ? OFFSET ?

Phase 2: Get ALL rows for those product names
  SELECT [columns] FROM Shopee_My_Products mp
  LEFT JOIN Shopee_Comp_Data cd ON cd.our_item_id_extracted = mp.our_item_id AND cd.region = 'MY'
  WHERE [original filters] AND mp.product_name IN (?,?,?,...)
  ORDER BY FIELD(mp.product_name, ?,?,?,...), [row sort]
```

**Smart JOIN detection**: `referencesCompTable()` checks if WHERE/ORDER clauses reference `cd.*`. If they don't, the Phase 1 names query skips the JOIN entirely (uses `buildMpOnlyWhere()` for a much faster mp-only query).

**Strategy 2: Row-Level Query** (`groupByName = false`) — used by Pending/Deleted/Fixed tabs

```sql
SELECT [columns] FROM Shopee_My_Products mp
LEFT JOIN Shopee_Comp_Data cd ON cd.our_item_id_extracted = mp.our_item_id AND cd.region = 'MY'
WHERE [filters] ORDER BY [sort] LIMIT ? OFFSET ?
```

#### Count Logic

Counts are fetched alongside the main query:
- **Cache hit** (120s TTL): use cached `total` and `groupTotal`
- **Last page** (`hasMore = false`): compute from offset + results length
- **Summary table**: `SELECT COUNT(*) AS groupTotal, SUM(row_count) AS total FROM Shopee_VVIP_Group_Summary WHERE status = ?`
- **Live count**: `SELECT COUNT(DISTINCT mp.product_name) AS groupTotal` + `SELECT COUNT(DISTINCT mp.id) AS total`

#### Other Exported Functions

| Function | Purpose |
|----------|---------|
| `fetchVvipProductCounts(region?)` | Tab counts (active, pending, fixed/new, deleted) |
| `fetchVvipProductDetails(ids, region?)` | Descriptions + images for product IDs |
| `insertVvipProduct(payload)` | Transactional INSERT into mp + cd |
| `updateVvipProduct(payload)` | UPDATE mp fields by ID |
| `updateVvipProductCategory(id, sheetName)` | UPDATE sheet_name only |
| `deleteVvipProduct(id)` | Soft delete (status='deleted') |
| `invalidateVvipCountCache()` | Clear in-memory count cache |

---

## Repository Layer — SG

### File: `lib/services/shopee-sg-products-repository.ts` (~278 lines)

**99% identical to VVIP repository.** The differences are:

| Aspect | VVIP | SG |
|--------|------|-----|
| Summary table | `Shopee_VVIP_Group_Summary` | `Shopee_SG_Group_Summary` |
| Default region | `"MY"` | `"SG"` |
| JOIN function | `vvipJoinForRegion()` | `sgJoinForRegion()` |
| Cache invalidation | `invalidateVvipCountCache()` | `invalidateSgCountCache()` |
| Refresh call | `refreshNormalizedGroupSummaryIfStale("vvip")` | `refreshNormalizedGroupSummaryIfStale("sg")` |
| Function names | `fetchVvipProducts()`, etc. | `fetchSgProducts()`, etc. |
| Date filter region | `const regionVal = region ?? "MY"` | `const regionVal = region ?? "SG"` |
| Date filter JOIN | Uses `cd.our_item_id_extracted` | Uses `SUBSTRING_INDEX(TRIM(TRAILING '/' FROM c2.our_link), '/', -1)` |

#### sgJoinForRegion(region?, useLatestPerProduct?) — Special Logic

```typescript
function sgJoinForRegion(region?: string, useLatestPerProduct?: boolean): string {
  if (!region) return JOIN_BY_ITEM_ID;
  if (useLatestPerProduct) {
    // Same derived table optimization as VVIP — MAX(date_taken) per product
    return `LEFT JOIN (...cd_max...) LEFT JOIN ${CD} cd ON ... AND cd.date_taken = cd_max.latest_date`;
  }
  if (region === "MY") return `${JOIN_BY_LINK} AND cd.region = 'MY'`;  // String match for MY
  return `${JOIN_BY_ITEM_ID} AND cd.region = '${region}'`;             // Item ID match for SG/others
}
```

**Why different for MY?** Shopee MY URLs use `shopee.com.my` domain while SG uses `shopee.sg`. The `our_item_id_extracted` column works for same-region JOINs, but cross-region (MY comp data linked to SG products) needs string-based URL matching.

**`useLatestPerProduct`**: Same behavior as VVIP — only applied when no date filter active.

---

## VVIP vs SG: Differences Summary

| Component | VVIP (MY) | SG |
|-----------|-----------|-----|
| **API base path** | `/api/shopee-my-vvip/products` | `/api/shopee-sg/products` |
| **Region constant** | `"MY"` | `"SG"` |
| **Repository file** | `shopee-vvip-products-repository.ts` | `shopee-sg-products-repository.ts` |
| **Summary table** | `Shopee_VVIP_Group_Summary` | `Shopee_SG_Group_Summary` |
| **JOIN condition** | `cd.our_item_id_extracted = mp.our_item_id AND cd.region = 'MY'` | `cd.our_item_id_extracted = mp.our_item_id AND cd.region = 'SG'` |
| **Details JOIN (own region)** | `NORMALIZED_LINK_EQ` (string) | `our_item_id_extracted` (numeric) |
| **Date filter subquery** | `cd.our_item_id_extracted` | `SUBSTRING_INDEX(...)` (string extraction) |
| **Data source** | `mp WHERE region='MY'` + `cd WHERE region='MY'` | `mp WHERE region='SG'` + `cd WHERE region='SG'` |
| **Enrichment** | Shared `enrichShopeeProductRows()` | Same |
| **Calculations** | Shared `shopee-history-calculations.ts` | Same |
| **Types** | Re-exported from `shopee-products-repository.ts` | Same |

**What's identical**: SELECT columns, sales window helpers, sort logic, filter logic, insight expressions, cache implementation, CRUD operations, count queries.

---

## Normalized Group Summary

### File: `lib/services/shopee-normalized-group-summary.ts` (~350 lines)

Pre-computes group-level aggregates so the main query can sort by aggregate columns without expensive GROUP BY operations.

#### Two Summary Tables

| Config | VVIP | SG |
|--------|------|-----|
| Table name | `Shopee_VVIP_Group_Summary` | `Shopee_SG_Group_Summary` |
| Region filter | `cd.region = 'MY'` | `cd.region = 'SG'` |
| Refresh key | `"vvip"` | `"sg"` |

#### When Summary is Used

`canUseGroupedSummary(filters)` returns `true` only when:
- `groupByName = true`
- Status is specified (not "all")
- **No active filters**: no search, customLink, shopName, shopId, date range, sheet name filter, insight filters, remarks
- Sort key is supported by summary table

#### Schema Auto-Migration

`ensureSummarySchema()` automatically adds the `min_our_item_id BIGINT UNSIGNED` column to both summary tables if it doesn't exist. Runs once per process lifetime (flag: `summarySchemaEnsured`). Catches errno 1060 (duplicate column) gracefully.

#### Staleness Check

- TTL: **10 minutes**
- `refreshNormalizedGroupSummaryIfStale(type)` checks `updated_at` timestamp
- Uses `summaryInFlight` Promise to prevent concurrent refreshes
- Falls back to live query if refresh fails

#### Summary Refresh Query

The refresh rebuilds the entire summary table using `REPLACE INTO`:

```sql
REPLACE INTO Shopee_VVIP_Group_Summary (
  status, product_name, max_id, min_our_item_id, max_date_taken,
  min_comp_product, min_sku, row_count,
  category_rank, shop_name_empty, min_shop_name,
  sg_sales_total_7d, ..., sp_value_total_90d,
  updated_at
)
SELECT
  mp.status, mp.product_name, MAX(mp.id), MAX(cd.date_taken),
  MIN(mp.our_item_id), MIN(cd.comp_product), MIN(mp.sku), COUNT(DISTINCT mp.id),
  MAX(CASE WHEN sheet_name='VVIP' THEN 4 ... END),
  MIN(CASE WHEN shop_name IS NULL THEN 1 ELSE 0 END),
  MIN(LOWER(TRIM(mp.shop_name))),
  SUM(COALESCE(mp.our_sales_7d, mp.our_sales, 0)), ...,
  SUM(CASE WHEN COALESCE(mp.shopee_var_value_90d, 0) > 0
      THEN mp.shopee_var_value_90d
      ELSE COALESCE(mp.shopee_var_sales_90d, 0) * COALESCE(mp.our_price, 0) END),
  NOW()
FROM Shopee_My_Products mp
LEFT JOIN Shopee_Comp_Data cd
  ON cd.our_item_id_extracted = mp.our_item_id AND cd.region = 'MY'
WHERE mp.region = 'MY'
GROUP BY mp.status, mp.product_name
```

#### Sales Value Formula in Summary

**Important**: The summary table uses a CASE fallback for `sp_value_total`:
```sql
CASE WHEN COALESCE(mp.shopee_var_value_{N}d, 0) > 0
  THEN mp.shopee_var_value_{N}d                          -- Real Shopee revenue
  ELSE COALESCE(mp.shopee_var_sales_{N}d, 0) * mp.our_price  -- Fallback: sales x price
END
```

This matches the `shopeeValueRaw` fallback logic in `shopee-history-calculations.ts`.

#### Supported Sort Columns

`normalizedSummarySortColumn(sortKey, salesWindow, shopeeSalesWindow)` maps sort keys to summary columns:

| Sort Key | Summary Column |
|----------|---------------|
| `sgSales` | `sg_sales_total_{window}d` |
| `sgSalesValue` | `sg_value_total_{window}d` |
| `spSales` | `sp_sales_total_{shopeeWindow}d` |
| `spSalesValue` | `sp_value_total_{shopeeWindow}d` |

Other sort keys (my, comp, sku, date, category, shop, priceMy, salesMy, etc.) use pre-computed columns: `min_comp_product`, `min_sku`, `max_date_taken`, `category_rank`, `min_our_price`, `max_our_sales`, etc.

**Default summary sort**: `min_our_item_id ASC` (not `max_id DESC`). This uses the Shopee product ID as tiebreaker for consistent grouping.

---

## Data Enrichment (Shared)

### File: `lib/services/shopee-products-enrichment.ts` (~1000 lines)

Shared by Comp 1, VVIP, and SG. Transforms raw DB rows into fully-computed Product objects.

(Same pipeline as Comp 1 — see Comp 1 backend doc for full details.)

**Key steps relevant to Comp 2**:
1. **Deduplication**: Removes duplicate competitor rows, keeps latest `date_taken` per `(our_link, comp_link, comp_variation, our_variation)`
2. **Batch external data** (parallel): costingMap (profit), inventoryMap (ISR/stockout), voucherMap (discounts)
3. **Row transformation**: Each raw row -> Product object with ourProduct, competitorProduct, financial metrics
4. **Grouping**: `groupProducts()` -> compute groupMetrics, variationMetrics, parentSortValues
5. **Background**: Fire-and-forget `backfillDiscountedPrices()`

---

## Shared Calculations

### File: `lib/shared/shopee-history-calculations.ts` (~616 lines)

Used by all three pages (Comp 1, VVIP, SG) for client-side and server-side data processing.

#### Key Exports

| Function | Purpose |
|----------|---------|
| `groupProducts(products)` | Groups by product_id (or name), then by SKU (or normalized variation) |
| `groupCompetitors(rows)` | Groups by (shop name, comp name) — separate entries per shop |
| `getHighestPriorityCategory(rows)` | Resolves: VVIP(4) > VIP(3) > LINKS_INPUT(2) > NEW_ITEMS(1) > NONE(0) |
| `resolveSalesNumber(rows, pick)` | Gets value from the row with **latest date_taken** (not max value) |
| `computeParentSortValue(group, key)` | Pre-computes sort values for 20+ columns per group |
| `computeGroupIsrPercent(variations)` | ISR % = (sum monthly sales) / (sum current stock) x 100, deduped by SKU |
| `computeGroupStockoutSummary(variations)` | Earliest projected stockout + total lost value |
| `computeVariationAggregates(variations)` | Per-group totals: sgSales, spSales, sgValue, spValue |
| `getQualifiedCompRows(variations)` | Top 3 competitor PRODUCTS + ties per variation, then union + dedup |
| `getCompMonthlySales(row)` | Extract comp monthly sales with consistent fallback paths |

#### groupProducts() — Deduplication Logic

```typescript
// Group by product_id when available, fall back to name
const key = productId ?? (p.ourProduct.name?.trim() || "(no name)")

// Within each group, variations keyed by SKU (or normalized variation name)
const variationKey = sku || normalizeVariationKey(rawVariation)
```

This prevents:
- Products with same name but different URLs from merging (product_id differs)
- Duplicate variations like "Red" vs "red" vs " Red " from creating separate entries

#### getQualifiedCompRows() — Top 3 Algorithm

```
1. Per variation: group rows by competitor PRODUCT (shopName + compName)
2. Rank products by max monthlySales DESC
3. Keep top 3 products + ties (same sales as 3rd place)
4. Include ALL rows from surviving products (so expanding shows all variations)
5. Backfill: if a comp product survives on ANY variation, include all its variations
6. Dedup by (shopName, compName, compVar) — keep highest monthlySales
7. Cap at MAX_QUALIFIED_COMP_ROWS (10) to prevent UI explosion
8. Sort final result by monthlySales DESC
```

#### Sales Value Handling

```typescript
// Primary: use shopeeValueRaw (real Shopee revenue) if available
const spValue = shopeeValueRaw != null && shopeeValueRaw > 0
  ? shopeeValueRaw
  : getSalesValueNumber(variationPrice, variationSpSalesRaw)  // Fallback: sales x price
```

---

## Database Connection

### File: `lib/services/mysql-allbots.ts`

Lightweight wrapper around the AllBots database connection.

| Export | Purpose |
|--------|---------|
| `query<T>(sql, params)` | SELECT — typed results |
| `execute(sql, params)` | INSERT/UPDATE/DELETE |
| `withTransaction(callback)` | Atomic operations with auto-commit/rollback |

**Connection pattern**: Each call creates a new connection from `getDbConnectionForAllBots()` (from `lib/db.ts`), executes, then releases. Uses the AllBots pool (14 connections).

**Important**: This is a different pool from the main `db.ts` default pool used by Comp 1. Comp 2's queries go through AllBots, which has the normalized tables.

---

## Query Building & SQL Patterns

### Sort Column Whitelist

All sort keys are mapped via `SORT_COLUMN_MAP` or `buildGroupOrderConfig()` — never interpolated directly.

#### Default Sort Order

**Changed in PR #747**: The default sort is now:

```
Category DESC → Shop ASC → Date DESC → ProductID (mp.our_item_id) ASC
```

All sort tiebreakers now use `mp.our_item_id` (Shopee product ID) instead of `mp.id` (auto-increment). This ensures products are grouped by their actual Shopee identity rather than insertion order.

#### Ungrouped Sort Map (15 keys)

```typescript
{
  my:               "mp.product_name, mp.our_variation, mp.our_item_id",
  comp:             "cd.comp_product, cd.comp_variation, mp.our_item_id",
  sku:              "mp.sku, mp.our_item_id",
  date:             "cd.date_taken, mp.our_item_id",
  id:               "mp.our_item_id",
  category:         "CASE WHEN sheet_name='VVIP' THEN 4 ... END, shop_name_empty, shop_name_sort, cd.date_taken DESC, mp.our_item_id ASC",
  shop:             "cd.comp_shop, mp.our_item_id",
  priceMy:          "mp.our_price, mp.our_item_id",
  salesMy:          "CAST(mp.our_sales AS SIGNED), mp.our_item_id",
  stockMy:          "CAST(mp.our_stock AS SIGNED), mp.our_item_id",
  priceComp:        "cd.comp_price, mp.our_item_id",
  salesComp:        "CAST(cd.comp_sales AS SIGNED), mp.our_item_id",
  compMonthlySales: "CAST(cd.comp_monthly_sales AS SIGNED), mp.our_item_id",
  stockComp:        "CAST(cd.comp_stock AS SIGNED), mp.our_item_id",
  remarks:          "mp.automated_remarks, mp.our_item_id",
}
```

#### Grouped Sort — Aggregate Keys

For sales aggregate sorts (sgSales, sgSalesValue, spSales, spSalesValue), uses SUM expressions:

```sql
-- sgSales
SUM(COALESCE(mp.our_sales_30d, mp.our_sales, 0))

-- sgSalesValue
SUM(COALESCE(mp.our_sales_30d, mp.our_sales, 0) * COALESCE(mp.our_price, 0))

-- spSales
SUM(COALESCE(mp.shopee_var_sales_30d, 0))

-- spSalesValue
SUM(CASE WHEN COALESCE(mp.shopee_var_value_30d, 0) > 0
    THEN COALESCE(mp.shopee_var_value_30d, 0)
    ELSE COALESCE(mp.shopee_var_sales_30d, 0) * COALESCE(mp.our_price, 0) END)
```

### WHERE Clause Building

`buildWhere(filters, region?)` dynamically constructs conditions:

| Filter | SQL Pattern |
|--------|-------------|
| `region` | `mp.region = ?` |
| `sheetName` | `mp.sheet_name IN (?)` — VVIP expands to ('VVIP', 'VVIP_SG') |
| `search` | `mp.product_name LIKE ? OR mp.sku LIKE ? OR mp.our_link LIKE ?` (LIKE only, no FULLTEXT) |
| `customLink` | `mp.our_link LIKE ? OR cd.comp_link LIKE ?` |
| `shopId` | `mp.our_shop_id = ?` |
| `shopName` | Complex: exact match + LIKE with space/paren suffixes on both mp and cd |
| `dateFrom/dateTo` | `cd.date_taken >= ? AND cd.date_taken < ?` (dateTo +1 day) |
| `insightFilters` | OR'd comparisons: `mp.our_sales > cd.comp_sales`, etc. |
| `status` | Default: `mp.status IN ('fixed','new')` unless specified |

#### Date Filter — Hide Pure-NONE Products

When date filter is active, an extra subquery hides products whose ALL variations have `sheet_name = 'NONE'`:

```sql
(mp.product_name IN (
    SELECT DISTINCT product_name FROM Shopee_My_Products
    WHERE region = ? AND sheet_name IN ('VVIP','VVIP_SG','VIP','LINKS_INPUT','NEW_ITEMS')
) OR NOT EXISTS (
    SELECT 1 FROM Shopee_My_Products p2
    INNER JOIN Shopee_Comp_Data c2 ON [join] AND c2.region = ?
    WHERE p2.region = ? AND p2.sheet_name IN (...) AND [date conditions]
))
```

The `NOT EXISTS` fallback ensures everything shows if no categorized products have comp data in the date range.

### Insight Filter Expressions

| Filter | SQL |
|--------|-----|
| `sales-higher` | `COALESCE(mp.our_sales, 0) > COALESCE(cd.comp_sales, 0)` |
| `sales-lower` | `COALESCE(mp.our_sales, 0) < COALESCE(cd.comp_sales, 0)` |
| `stock-higher` | `COALESCE(mp.our_stock, 0) > COALESCE(cd.comp_stock, 0)` |
| `stock-lower` | `COALESCE(mp.our_stock, 0) < COALESCE(cd.comp_stock, 0)` |
| `rating-higher` | `COALESCE(mp.our_rating, 0) > COALESCE(cd.comp_rating, 0)` |
| `rating-lower` | `COALESCE(mp.our_rating, 0) < COALESCE(cd.comp_rating, 0)` |
| `price-lower` | `COALESCE(mp.our_price, 0) < COALESCE(cd.comp_price, 0)` |
| `price-higher` | `COALESCE(mp.our_price, 0) > COALESCE(cd.comp_price, 0)` |
| `comp-oos` | `COALESCE(cd.comp_stock, 0) = 0` |
| `ours-oos` | `COALESCE(mp.our_stock, 0) = 0` |

---

## Caching & Performance

### Count Cache (per repository)

| Property | Value |
|----------|-------|
| Type | In-memory Map (separate for VVIP and SG) |
| TTL | 120 seconds |
| Max entries | 100 (LRU eviction) |
| Key | JSON of 15 filter dimensions + region |
| Invalidation | On any mutation (insert, update, delete, category change) |

### Summary Table Cache

| Property | VVIP | SG |
|----------|------|-----|
| Table | `Shopee_VVIP_Group_Summary` | `Shopee_SG_Group_Summary` |
| TTL | 10 minutes | 10 minutes |
| Refresh | `REPLACE INTO` with JOIN + GROUP BY | Same |
| Dedup | `summaryInFlight` Promise (per type) | Same |
| Used when | `groupByName=true`, `status` set, no filters active | Same |

### HTTP Caching

| Endpoint | Cache-Control |
|----------|---------------|
| All GET endpoints | `private, max-age=30, stale-while-revalidate=60` |

### Smart JOIN Skipping

Both repositories use `referencesCompTable()` to check if the WHERE/ORDER clause actually needs the comp table. When it doesn't (e.g., filtering only by mp columns), the Phase 1 names query skips the LEFT JOIN entirely — significantly faster for unfiltered paginated browsing.

### Latest-Date Derived Table (PR #747)

When no date filter is active, `useLatestPerProduct = true` triggers a derived table JOIN that pre-selects only the latest `date_taken` per product from `Shopee_Comp_Data`. This reduces the result set by ~3x compared to fetching all historical comp rows and deduplicating client-side.

---

## Security

### SQL Injection Prevention

- **Parameterized queries**: All user input via `?` placeholders
- **Whitelist maps**: Sort keys, insight filters — all validated before use
- **No string interpolation** of user input into SQL (only region constants are interpolated, and those are hardcoded in route files)
- `multipleStatements: false` in pool config prevents query stacking

### Input Validation

| Field | Validation |
|-------|-----------|
| `sortKey` | Must exist in SORT_COLUMN_MAP or GROUP_ORDER_MAP |
| `status` | Whitelist: active, pending, fixed, new, deleted, all |
| `salesWindow` | Must be 7, 14, 30, 60, or 90 |
| `sheetName` | Expanded (VVIP -> VVIP+VVIP_SG), then parameterized |
| `limit` | Clamped to max 100 (grouped), positive integer required |
| `offset` | Non-negative integer required |
| `insightFilters` | Each must map to a valid `insightExpr()` |

### Transaction Safety

- `withTransaction()` wrapper for INSERT operations (mp + cd atomically)
- Auto-commit on success, auto-rollback on failure
- Connection released in all code paths

---

## Data Pipeline — End to End

### Request: GET /api/shopee-my-vvip/products (VVIP Active tab)

```
Browser: fetchProducts({ status: "active", groupByName: true, salesWindow: 30, ... })
    |
API Route (shopee-my-vvip/products/route.ts):
    parseFilterParams() -> validated ProductFilters
    region = "MY" (hardcoded)
    |
Repository (shopee-vvip-products-repository.ts):
    fetchVvipProducts(filters, "MY")
    |
    |-- Check count cache (120s TTL)
    |   Hit? -> Use cached total/groupTotal
    |   Miss? -> will compute later
    |
    |-- Check summary table (if canUseGroupedSummary)
    |   Fresh? -> Use Shopee_VVIP_Group_Summary for Phase 1
    |   Stale? -> refreshNormalizedGroupSummaryIfStale("vvip")
    |   Fail? -> Fall back to live GROUP BY
    |
    |-- Phase 1: Paginated product names
    |   Summary path: SELECT product_name FROM Shopee_VVIP_Group_Summary ...
    |   Live path:    SELECT mp.product_name FROM Shopee_My_Products mp
    |                 [LEFT JOIN Shopee_Comp_Data cd ... AND cd.region='MY']
    |                 [WHERE mp.region='MY' AND ...] GROUP BY mp.product_name
    |                 ORDER BY [sort] LIMIT 51 OFFSET 0
    |
    |-- Determine JOIN strategy
    |   No date filter? -> useLatestPerProduct = true (derived table, ~3x fewer rows)
    |   Date filter active? -> useLatestPerProduct = false (all comp data needed)
    |
    |-- Phase 2: All rows for those products
    |   SELECT [35 columns + 3 window columns]
    |   FROM Shopee_My_Products mp
    |   [derived table cd_max if useLatestPerProduct]
    |   LEFT JOIN Shopee_Comp_Data cd
    |     ON cd.our_item_id_extracted = mp.our_item_id AND cd.region = 'MY'
    |     [AND cd.date_taken = cd_max.latest_date if useLatestPerProduct]
    |   WHERE [filters] AND mp.product_name IN ('Product A', 'Product B', ...)
    |   ORDER BY FIELD(mp.product_name, ...), mp.our_item_id ASC
    |
    \-- Return { rows, total, groupTotal, debug }
    |
Enrichment (shopee-products-enrichment.ts):
    enrichShopeeProductRows(rows, { metricsWindow: 30 })
    |
    |-- Deduplicate competitor rows (keep latest per comparison)
    |-- Batch fetch: costingMap, inventoryMap, voucherMap (parallel)
    |-- Transform each row -> Product object
    |-- Filter to latest comp scrape per group
    |-- Group by product name -> compute groupMetrics, variationMetrics
    |-- Attach metrics to each Product
    \-- Fire-and-forget: backfillDiscountedPrices()
    |
API Route:
    Return JSON { success, data: enrichedRows, total, groupTotal, ... }
    Set Cache-Control: private, max-age=30
    |
Browser: receives Product[] -> groupProducts() -> render GroupedRows
```

### Request: GET /api/shopee-sg/products (SG Active tab)

**Identical pipeline** except:
- Region = `"SG"` (hardcoded)
- Repository: `shopee-sg-products-repository.ts`
- JOIN: `cd.region = 'SG'`
- Summary: `Shopee_SG_Group_Summary`
- Refresh: `refreshNormalizedGroupSummaryIfStale("sg")`
