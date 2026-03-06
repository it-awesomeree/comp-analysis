# Comp Analysis — Backend Documentation

**Purpose**: Comprehensive documentation of the backend architecture — API routes, repository layer, data enrichment, database schema, AI integration, and server-side logic for the Shopee MY Competitive Analysis feature.

---

## Table of Contents

1. [Overview](#overview)
2. [File Structure](#file-structure)
3. [Database Schema](#database-schema)
4. [API Routes — Products](#api-routes--products)
5. [API Routes — Similarity Exclusion](#api-routes--similarity-exclusion)
6. [API Routes — Tab Counts](#api-routes--tab-counts)
7. [API Routes — AI Analysis](#api-routes--ai-analysis)
8. [Repository Layer — shopee-products-repository.ts](#repository-layer)
9. [Data Enrichment — shopee-products-enrichment.ts](#data-enrichment)
10. [Database Connection — db.ts](#database-connection)
11. [Query Building & SQL Patterns](#query-building--sql-patterns)
12. [Caching & Performance](#caching--performance)
13. [Security](#security)
14. [Data Pipeline — End to End](#data-pipeline--end-to-end)

---

## Overview

The backend serves the Comp Analysis frontend through Next.js API routes. The stack:

- **Framework**: Next.js 15 App Router (API routes at `app/api/*/route.ts`)
- **Database**: MySQL (Cloud SQL) via `mysql2` with connection pooling
- **AI**: Google Gemini 2.0 Flash for similarity/competitive analysis
- **Auth**: Firebase Admin SDK + middleware session validation (not in individual routes)
- **Pattern**: API Route -> Repository -> MySQL, with server-side enrichment between DB and response

---

## File Structure

```
AWESOMEREE-WEB-APP/
├── app/api/
│   ├── products/
│   │   ├── route.ts                      # Main CRUD (GET/POST/PUT/PATCH/DELETE)
│   │   ├── similarity-exclusion/route.ts # Exclusion management (POST/GET/DELETE)
│   │   ├── counts/route.ts              # Tab counts (single optimized query)
│   │   ├── status/route.ts              # Single status update
│   │   ├── bulk-status/route.ts         # Bulk status update
│   │   └── permanent-delete/route.ts    # Hard delete
│   └── comp-analysis/
│       ├── analyze/route.ts             # Gemini AI analysis
│       ├── products/route.ts            # Product list for AI dashboard
│       ├── test-data/route.ts           # Test data API
│       └── test-data-comp/route.ts      # Test data comp API
├── lib/
│   ├── db.ts                            # Connection pooling, executeWithRetry
│   ├── services/
│   │   ├── shopee-products-repository.ts    # Primary data access (~86KB, 2394 lines)
│   │   ├── shopee-products-repository-mysql.ts # MySQL-specific queries
│   │   └── shopee-products-enrichment.ts    # Server-side enrichment (~26KB)
│   ├── auth.ts                          # Firebase Admin auth
│   └── api-auth.ts                      # API route auth helpers
```

---

## Database Schema

### Primary Table: `Shopee_Comp`

The main table storing our products paired with competitor products.

#### Our Product Fields

| Column | Type | Purpose |
|--------|------|---------|
| `id` | INT AUTO_INCREMENT PK | Row identifier |
| `product_name` | VARCHAR | Our product name |
| `product_description` | TEXT | Our product description |
| `sku` | VARCHAR | SKU (nullable) |
| `our_link` | LONGTEXT | Our Shopee product URL |
| `our_variation` | VARCHAR | Our variation name |
| `shop_name` | VARCHAR | Our shop name |
| `our_price` | DECIMAL | Our selling price |
| `our_product_price` | DECIMAL | Our list price |
| `our_sales` | INT | All-time sales count |
| `our_stock` | INT | Current stock quantity |
| `shopee_stock` | INT | Shopee-reported stock |
| `our_rating` | DECIMAL | Rating (0-5) |
| `our_advantage` | VARCHAR | Bot-generated advantage text |
| `our_issue` | VARCHAR | Bot-generated issue text |

#### Competitor Fields

| Column | Type | Purpose |
|--------|------|---------|
| `comp_product` | VARCHAR | Competitor product name |
| `comp_product_description` | TEXT | Competitor description |
| `comp_link` | LONGTEXT | Competitor Shopee URL |
| `comp_variation` | VARCHAR | Competitor variation name |
| `comp_shop` | VARCHAR | Competitor shop name |
| `comp_price` | DECIMAL | Competitor price |
| `comp_product_price` | DECIMAL | Competitor list price |
| `comp_sales` | INT | Competitor lifetime sales |
| `comp_monthly_sales` | INT | Competitor monthly sales |
| `comp_rating` | DECIMAL | Competitor rating |
| `comp_stock` | INT | Competitor stock |

#### Time-Windowed Sales Columns (per window: 7d, 14d, 30d, 60d, 90d)

| Column Pattern | Type | Purpose |
|---------------|------|---------|
| `our_sales_{N}d` | INT | Our sales over N days |
| `shopee_var_sales_{N}d` | INT | Shopee variation sales |
| `shopee_var_value_{N}d` | DECIMAL | Shopee variation sales value (MYR) |
| `ads_spend_{N}d` | DECIMAL | Ad spend over N days |
| `roas_{N}d` | DECIMAL | Return on ad spend |
| `ads_visitors_{N}d` | INT | Ad visitors |
| `ads_conv_rate_{N}d` | DECIMAL | Conversion rate |

#### Similarity Scoring Fields

| Column | Type | Purpose |
|--------|------|---------|
| `product_similarity_score` | DECIMAL | Bot-generated product similarity (0-100) |
| `product_similarity_reason` | TEXT | Why products don't match |
| `product_similarity_score_datetime` | TIMESTAMP | When score was computed |
| `product_similarity_excluded` | TINYINT(1) | Legacy exclusion flag |
| `variation_similarity_score` | DECIMAL | Variation-level similarity (0-100) |
| `variation_similarity_reason` | TEXT | Variation mismatch reason |
| `variation_similarity_score_datetime` | TIMESTAMP | When variation score was computed |
| `variation_similarity_excluded` | TINYINT(1) | Legacy exclusion flag |
| `has_product_exclusions` | TINYINT(1) | New exclusion flag (signals Python bot to reprocess) |
| `has_variation_exclusions` | TINYINT(1) | New exclusion flag (signals Python bot to reprocess) |

#### Metadata Fields

| Column | Type | Purpose |
|--------|------|---------|
| `date_taken` | DATE | When data was scraped |
| `automated_remarks` | TEXT | Bot-generated remarks |
| `user_remarks` | TEXT | User-entered remarks |
| `sheet_name` | VARCHAR | Category: VVIP, VIP, LINKS_INPUT, NEW_ITEMS, NONE |
| `status` | VARCHAR | active, pending, fixed, new, deleted |
| `last_page` | VARCHAR | Last UI tab the product was on |
| `original_page` | VARCHAR | Original tab (for restore) |
| `Delete_Remark` | VARCHAR | Set to 'DELETED' when soft-deleted |

### Table: `Shopee_Comp_Similarity_Exclusions`

Stores user-created exclusions marking specific comparisons to be reprocessed.

| Column | Type | Purpose |
|--------|------|---------|
| `id` | INT AUTO_INCREMENT PK | Row identifier |
| `our_link` | LONGTEXT | Our product URL |
| `comp_link` | LONGTEXT | Competitor URL |
| `sku` | VARCHAR (nullable) | SKU (NULL means product-level) |
| `exclusion_type` | VARCHAR(20) | "product" or "variation" |
| `excluded_differences` | JSON | Array of reason strings (max 50 entries) |
| `created_at` | TIMESTAMP | Creation time |
| `updated_at` | TIMESTAMP | Last update time |
| `created_by` | VARCHAR | User who created it |

**Composite key**: (our_link, comp_link, sku, exclusion_type) — used for JOIN lookups.

### Table: `Shopee_Comp_Group_Summary`

Pre-computed aggregates refreshed every 10 minutes for performance.

| Column | Type | Purpose |
|--------|------|---------|
| `status` | VARCHAR | Product status |
| `product_name` | VARCHAR | Product name (group key) |
| `max_id` | INT | Latest row ID in group |
| `max_date_taken` | DATE | Most recent scrape date |
| `min_comp_product` | VARCHAR | First competitor name |
| `min_sku` | VARCHAR | First SKU |
| `row_count` | INT | Rows in group |
| `sg_sales_total_{N}d` | INT | SiteGiant sales total (per window) |
| `sg_value_total_{N}d` | DECIMAL | SiteGiant value total |
| `sg_var_pct_{N}d` | DECIMAL | Variation percentage |
| `sp_sales_total_{N}d` | INT | Shopee sales total |
| `sp_value_total_{N}d` | DECIMAL | Shopee value total |
| `sp_var_pct_{N}d` | DECIMAL | Shopee variation percentage |
| `updated_at` | TIMESTAMP | Last refresh time |

### Table: `Shopee_Comp_Remarks`

Normalized remarks storage (1:many with Shopee_Comp).

| Column | Type | Purpose |
|--------|------|---------|
| `comp_id` | INT (FK) | Foreign key to Shopee_Comp.id |
| `remark` | VARCHAR | Remark text |
| `source` | VARCHAR | "user" or "automated" |

### Table: `comp_ai_test`

Used by the Gemini AI analysis endpoint for competitor comparison data.

---

## API Routes — Products

### File: `app/api/products/route.ts`

#### GET — Fetch Products

```
GET /api/products?limit=50&offset=0&status=active&search=...&sheetName=VVIP&...
```

**Flow**:
1. Parse query params via `parseFilterParams()` (whitelist validated)
2. Call `fetchShopeeProducts(filters)` from repository
3. Call `enrichShopeeProductRows(rows, { metricsWindow })` for server-side enrichment
4. Return enriched rows with pagination metadata

**Response**:
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

**Allowed Sort Keys** (27 keys, whitelist-validated):
```
my, comp, sku, date, id, category, shop, productId,
priceMy, profitMy, profitMarginMy, isrPercent,
salesMy, sgSalesValue, sgVariationPct,
salesMyValue, spSalesValue, spVariationPct,
stockMy, visitors, conversionRate, roas, adsSpend,
similarity, similarityReason, dateCompared,
priceComp, salesComp, compMonthlySales, stockComp,
remarks, mkRemark
```

**Allowed Insight Filters** (10 filters):
```
sales-higher, sales-lower, stock-higher, stock-lower,
rating-higher, rating-lower, price-lower, price-higher,
comp-oos, ours-oos
```

#### POST — Create Product

```
POST /api/products
Body: { ourProduct, competitorProduct, remarks, sourceType }
```

- Calls `insertShopeeProduct(payload)`
- Default: `sheet_name="VIP"`, `status="new"`, `last_page="Fixed/New"`
- Also inserts into `Shopee_Comp_Remarks` if remarks provided
- Invalidates count cache
- Returns `{ success, id, affectedRows }`

#### PUT — Update Product

```
PUT /api/products
Body: { id, ourProduct, competitorProduct, remarks, sku }
```

- Validates `id` is positive integer
- Calls `updateShopeeProduct(payload)`
- Returns `{ success, affectedRows }`

#### PATCH — Update Category

```
PATCH /api/products
Body: { id, sheetName } or { id, sourceType }
```

- Accepts either `sheetName` or `sourceType`
- Calls `updateShopeeProductCategory(id, sheetName)`
- Returns `{ success, affectedRows }`

#### DELETE — Soft Delete

```
DELETE /api/products?id=123
```

- ID passed as query parameter
- Calls `deleteShopeeProduct(id)` (hard delete from table)
- Invalidates count cache
- Returns `{ success, affectedRows }`

**All mutations are logged** via `logApiAction()` with model_name, action, record_id, new_data.

---

## API Routes — Similarity Exclusion

### File: `app/api/products/similarity-exclusion/route.ts`

#### POST — Add Exclusion

```
POST /api/products/similarity-exclusion
Body: {
  our_link: string (max 2048),
  comp_link: string (max 2048),
  sku: string | null (max 255),
  exclusion_type: "product" | "variation",
  reason: string (max 1000)
}
```

**Validation & Sanitization**:
- Length limits enforced (2048 for links, 255 for SKU, 1000 for reason)
- Control characters removed (0x00-0x1F, 0x7F)
- `exclusion_type` must be exactly "product" or "variation" (whitelist)
- Null/empty SKU stored as `NULL` in DB (no placeholder hack)

**What Happens in Repository**:
1. `SELECT ... FOR UPDATE` lock on existing exclusion row
2. If exists: `JSON_ARRAY_APPEND()` adds reason atomically (prevents race conditions)
   - Guards: `JSON_LENGTH < 50` AND `NOT JSON_CONTAINS` (no duplicates)
3. If new: `INSERT` with reason as JSON array `["reason"]`
4. `UPDATE Shopee_Comp SET has_product_exclusions = 1` (or variation) for all matching rows
5. Invalidate count cache

**Response**: `{ exclusionId, excludedDifferences, isNew, affectedRows }`

#### GET — Fetch Exclusions

```
GET /api/products/similarity-exclusion?our_link=X&comp_link=Y&sku=Z
GET /api/products/similarity-exclusion?our_link=X&comp_link=Y&sku_is_null=true
```

- Returns all exclusion records for the comparison
- Separates into `productExclusions[]` and `variationExclusions[]`

#### DELETE — Remove Exclusion

```
DELETE /api/products/similarity-exclusion
Body: { our_link, comp_link, sku, exclusion_type, reason? }
```

- If `reason` provided: removes only that specific reason from the JSON array
- If `reason` omitted: removes entire exclusion record
- After removal, checks if any exclusions remain
- Updates `has_*_exclusions` flag accordingly (1 if remain, 0 if not)
- Invalidates count cache

**Error handling**: Safe error messages only — never exposes internal details to client.

---

## API Routes — Tab Counts

### File: `app/api/products/counts/route.ts`

Single optimized query replaces 4 separate count queries:

```sql
SELECT
  IFNULL(SUM(status = 'active'), 0) AS active_count,
  COUNT(DISTINCT CASE WHEN status = 'active' THEN product_name END) AS active_groups,
  IFNULL(SUM(status = 'pending'), 0) AS pending_count,
  IFNULL(SUM(status = 'fixed'), 0) + IFNULL(SUM(status = 'new'), 0) AS fixed_new_count,
  IFNULL(SUM(status = 'deleted'), 0) AS deleted_count
FROM Shopee_Comp
```

- Uses `idx_status_product_name` covering index
- 1 DB connection instead of 4
- Conditional aggregation (SUM with boolean) instead of GROUP BY
- Returns `{ active, activeGroups, pending, fixedNew, deleted }`
- Cache: `private, max-age=30, stale-while-revalidate=60`

---

## API Routes — AI Analysis

### File: `app/api/comp-analysis/analyze/route.ts`

Uses **Google Gemini 2.0 Flash** for AI-powered competitor analysis.

#### Request

```json
{
  "product_name": "string",
  "product_description": "string | null",
  "sku": "string | null",
  "our_link": "string | null",
  "our_variation": "string | null",
  "shop_name": "string | null",
  "user_prompt": "string (full prompt text)",
  "comp_link": "string | null",
  "date_taken": "string | null"
}
```

#### Processing Pipeline

```
1. Query comp_ai_test table for matching records
   WHERE our_link = ? AND comp_link = ? AND date_taken = ?
       |
2. Extract image URLs from DB fields
   (our_main_images, our_variation_images, comp equivalents)
   Handles: JSON arrays, malformed JSON, plain strings, Buffer
       |
3. Download images with validation
   - HEAD request first (check size, MIME type)
   - Skip if > 3MB or not image/*
   - Download with 12-second timeout per image
   - Normalize MIME (jpg->jpeg, x-png->png)
   - Base64 encode for Gemini inline payload
       |
4. Build Gemini request
   - user_prompt + structured product/competitor JSON
   - Inline images attached as Parts
       |
5. Call Gemini 2.0 Flash API
       |
6. Return analysis text + metadata
```

#### Image Limits

| Limit | Value |
|-------|-------|
| Max total inline payload | 18 MB |
| Max per image | 3 MB |
| Download timeout per image | 12 seconds |
| Our main images max | 3 |
| Our variation images max | 1 |
| Competitor rows with images | 2 (first 2 competitors) |
| Comp main images max | 3 per competitor |
| Comp variation images max | 1 per competitor |

#### Response

```json
{
  "success": true,
  "analysis": "Gemini's response text (JSON or Markdown depending on prompt)",
  "recordCount": 5,
  "imagesAttached": 8
}
```

### File: `app/api/comp-analysis/products/route.ts`

Fetches paginated product list for the AI dashboard.

```
GET /api/comp-analysis/products?search=X&sortBy=product_name&sortOrder=asc&page=1&pageSize=20
```

- Queries `comp_ai_test` table
- LIKE search on product_name, sku, shop_name
- Sort columns whitelist-validated
- Count query cached for 5 minutes
- `skipCount=true` skips expensive COUNT(*) if total already known

---

## Repository Layer

### File: `lib/services/shopee-products-repository.ts` (~86KB, 2394 lines)

The primary data access layer. All database interactions go through this file.

### fetchShopeeProducts(filters)

The main query function. Two strategies based on `groupByName`:

#### Strategy 1: Grouped Query (`groupByName = true`)

Used by the main Shopee MY tab. Two-phase pagination:

```
Phase 1: Get paginated product NAMES
  SELECT DISTINCT product_name FROM Shopee_Comp
  WHERE [filters] ORDER BY [sort] LIMIT ? OFFSET ?

Phase 2: Get ALL rows for those product names
  SELECT * FROM Shopee_Comp
  WHERE product_name IN (?) AND status IN ('active','fixed','new')
  LEFT JOIN Shopee_Comp_Similarity_Exclusions ...
```

**Why two phases?** A product group may have 50+ variation rows. Pagination must happen at the group level, not row level. Phase 1 picks which groups, Phase 2 fetches all their rows.

**Date filter behavior**: Date filter applies to Phase 1 (which products to include), but Phase 2 fetches ALL variations regardless of date (so you see the full product even if some variations were scraped on different dates).

**Summary table optimization**: If `status=active` and no filters are applied, uses pre-computed `Shopee_Comp_Group_Summary` table instead of live aggregation. Falls back to live query if summary is stale.

#### Strategy 2: Row-Level Query (`groupByName = false`)

Used by Pending/Deleted/Fixed tabs. Simple single query:

```sql
SELECT * FROM Shopee_Comp
WHERE [filters]
ORDER BY [sort]
LIMIT ? OFFSET ?
```

### SELECT Columns (51+ columns)

The query selects all core fields plus:

```sql
-- Sales display (window-aware)
COALESCE(our_sales_{N}d, our_sales) AS our_sales_display

-- Shopee sales/value (window-aware)
shopee_var_sales_{N}d AS shopee_sales_display
shopee_var_value_{N}d AS shopee_value_display

-- Exclusion data (LEFT JOIN)
has_product_exclusions,
has_variation_exclusions,
exc.excluded_differences AS product_exclusions_json,   -- from JOIN
exc2.excluded_differences AS variation_exclusions_json  -- from second JOIN
```

### Exclusion JOIN

```sql
LEFT JOIN Shopee_Comp_Similarity_Exclusions exc
  ON exc.our_link = sc.our_link
  AND exc.comp_link = sc.comp_link
  AND (exc.sku = sc.sku OR (exc.sku IS NULL AND sc.sku IS NULL))
  AND exc.exclusion_type = 'product'

LEFT JOIN Shopee_Comp_Similarity_Exclusions exc2
  ON exc2.our_link = sc.our_link
  AND exc2.comp_link = sc.comp_link
  AND (exc2.sku = sc.sku OR (exc2.sku IS NULL AND sc.sku IS NULL))
  AND exc2.exclusion_type = 'variation'
```

Two separate JOINs to get product-level and variation-level exclusions independently.

### Aggregate Sorting (for grouped queries)

When sorting by aggregate columns (sgSales, spSales, etc.), uses CTEs:

```sql
WITH var_sales AS (
  SELECT product_name, our_variation,
    MAX(shopee_var_sales_{N}d) AS mv
  FROM Shopee_Comp WHERE status='active'
  GROUP BY product_name, our_variation
),
totals AS (
  SELECT product_name, SUM(mv) AS total
  FROM var_sales GROUP BY product_name
)
SELECT DISTINCT sc.product_name
FROM Shopee_Comp sc
LEFT JOIN totals t ON t.product_name = sc.product_name
WHERE [filters]
ORDER BY t.total DESC
LIMIT ? OFFSET ?
```

### WHERE Clause Building

`buildWhere(filters, tableAlias)` dynamically constructs conditions:

| Filter | SQL Pattern |
|--------|-------------|
| `sheetName` | `sheet_name IN (?)` — VVIP expands to (VVIP, VVIP_SG) |
| `uncategorizedOnly` | `sheet_name IS NULL OR sheet_name = '' OR sheet_name = 'NONE'` |
| `customLink` | `our_link LIKE ? OR comp_link LIKE ?` |
| `search` | FULLTEXT match + LIKE fallback on product_name, comp_product, sku, our_link |
| `shopId` | `our_shop_id = ?` |
| `shopName` | `comp_shop = ? OR shop_name = ?` (case-insensitive) |
| `dateFrom/dateTo` | `date_taken >= ? AND date_taken < ?` (dateTo incremented by 1 day) |
| `remarks` | `EXISTS (SELECT 1 FROM Shopee_Comp_Remarks WHERE comp_id = sc.id AND remark IN (?))` |
| `insightFilters` | OR'd comparisons: `our_sales > comp_sales`, `our_stock = 0`, etc. |
| `status` | Default excludes 'fixed','new' unless specified |
| `statusGroup=fixed-new` | `status IN ('fixed','new')` |

### Similarity Exclusion Functions

#### addSimilarityExclusion(payload)

**Transaction flow**:
```
BEGIN TRANSACTION
  1. SELECT ... FOR UPDATE (lock row)
  2. IF EXISTS:
       UPDATE SET excluded_differences = JSON_ARRAY_APPEND(...)
       WHERE JSON_LENGTH < 50 AND NOT JSON_CONTAINS(reason)
     ELSE:
       INSERT (our_link, comp_link, sku, exclusion_type, excluded_differences)
       VALUES (?, ?, ?, ?, JSON_ARRAY(?))
  3. UPDATE Shopee_Comp SET has_{type}_exclusions = 1
     WHERE our_link = ? AND comp_link = ? AND (sku conditions)
COMMIT
```

- Uses `JSON_ARRAY_APPEND()` for atomic, race-condition-free array updates
- `JSON_LENGTH < 50` prevents unbounded growth
- `NOT JSON_CONTAINS` prevents duplicate reasons
- `FOR UPDATE` row lock prevents concurrent modifications

#### removeSimilarityExclusion(payload)

```
BEGIN TRANSACTION
  1. SELECT ... FOR UPDATE
  2. IF reason provided:
       Filter array to remove specific reason
       IF array now empty: DELETE row
       ELSE: UPDATE with filtered array
     ELSE:
       DELETE entire row
  3. Check if ANY exclusions remain for (our_link, comp_link, sku, type)
  4. UPDATE Shopee_Comp SET has_{type}_exclusions = (1 if remain, 0 if not)
COMMIT
```

### Other Repository Functions

| Function | Purpose |
|----------|---------|
| `insertShopeeProduct(payload)` | INSERT into Shopee_Comp + Shopee_Comp_Remarks |
| `updateShopeeProduct(payload)` | UPDATE product fields by ID |
| `updateShopeeProductCategory(id, sheetName)` | UPDATE sheet_name only |
| `updateProductStatus(payload)` | UPDATE status, last_page, original_page |
| `bulkUpdateProductStatus(payload)` | UPDATE multiple rows via `IN (?)` |
| `deleteShopeeProduct(id)` | DELETE single row |
| `permanentDeleteProducts(ids)` | DELETE multiple rows via `IN (?)` |
| `invalidateCountCache()` | Clear entire count cache |
| `getSimilarityExclusions(our_link, comp_link, sku)` | Fetch exclusion records |

---

## Data Enrichment

### File: `lib/services/shopee-products-enrichment.ts` (~26KB)

Transforms raw DB rows into fully-computed Product objects with metrics.

### enrichShopeeProductRows(rows, options)

**Pipeline**:

```
Raw DB rows
    |
1. Deduplication
   Remove duplicate competitor rows, keep latest date_taken per
   (our_link, comp_link, comp_variation, our_variation)
    |
2. Batch External Data (parallel)
   ├── costingMap: selling price, total cost (MYR), shipping per SKU
   ├── inventoryMap: ISR%, projected stockout date, lost value per SKU
   └── voucherMap: active shop vouchers for discounted pricing
    |
3. Row Transformation
   Each raw row -> Product object with:
   ├── ourProduct: { name, variation, price, sales, rating, images, description }
   ├── competitorProduct: { name, variation, price, sales, monthlySales, rating, shopName, stock, images }
   ├── ourShop: { name, stock, advantage, issue }
   ├── Financial: profit, profitMarginPct (from costing)
   ├── Ad metrics: visitors, convRate, roas, adsSpend (selected by metricsWindow)
   ├── Stockout: projectedDateMs, lostValue, monthlySales (from inventory)
   └── Similarity: product/variation scores + exclusion flags
    |
4. Competitor Filtering
   Keep only latest comp scrape per (product, sourceType) group
    |
5. Grouping & Aggregation
   groupProducts() -> for each ProductGroup:
   ├── groupMetrics (aggregated across all variations)
   │   ├── profit: sum
   │   ├── profitMargin: weighted average
   │   ├── isrPercent: from inventory
   │   ├── stockoutProjectedDateMs: earliest across variations
   │   ├── stockoutLostValue: sum
   │   ├── sgSalesTotal, spSalesTotal: sums
   │   ├── sgSalesValueTotal, spSalesValueTotal: sums
   │   ├── visitorsTotal, adsSpendTotal: sums
   │   ├── convRateAvg: weighted average (by visitors)
   │   └── roasAvg: simple average
   │
   ├── variationMetrics (per variation)
   │   ├── Same fields as group but per variation
   │   ├── sgVariationPctDisplay: "X.X%" (this var / group total)
   │   ├── spVariationPctDisplay: "X.X%"
   │   └── salesValueDisplay: formatted "X k" or "X.X k"
   │
   └── parentSortValues (for 20+ columns)
       Pre-computed sort values for client-side column sorting
    |
6. Metadata Backfill
   Attach groupMetrics, variationMetrics, parentSortValues to each Product
    |
7. Background Task (fire-and-forget)
   backfillAllDiscountedPrices(): Update Shopee_Comp rows with
   discounted prices from active vouchers (doesn't block response)
```

### Profit Calculation

```
profit = sellingPrice - totalCostMyr
profitMarginPct = (profit / sellingPrice) * 100
```

Where:
- `sellingPrice` from costing table (per SKU)
- `totalCostMyr` from costing table (product cost + shipping)

### Metrics Window Selection

The `metricsWindow` parameter (7, 30, or 90 days) determines which ad columns to use:

| metricsWindow | Visitors Column | Conv Rate Column | ROAS Column | Ads Spend Column |
|--------------|-----------------|------------------|-------------|------------------|
| 7 | `ads_visitors_7d` | `ads_conv_rate_7d` | `roas_7d` | `ads_spend_7d` |
| 30 (default) | `ads_visitors_30d` | `ads_conv_rate_30d` | `roas_30d` | `ads_spend_30d` |
| 90 | `ads_visitors_90d` | `ads_conv_rate_90d` | `roas_90d` | `ads_spend_90d` |

### Competitor Monthly Sales Formatting

Special handling based on date: if `date_taken >= 2026-01-21`, uses `monthlySalesDisplay` formatted string. Otherwise uses raw numeric `comp_monthly_sales`.

---

## Database Connection

### File: `lib/db.ts`

### Three Connection Pools

| Pool | Database | Connection Limit | Purpose |
|------|----------|-----------------|---------|
| Default (DatabasePool) | `webapp_test` / `webapp` | 15 | General operations |
| Shopee | ShopeeDatabase | 6 | Shopee-specific queries |
| AllBots | AllBots | 14 | Comp analysis, similarity, logging |

### Pool Configuration

```typescript
{
  waitForConnections: true,
  connectionLimit: varies,
  queueLimit: 200,           // Max pending connections
  connectTimeout: 5000,      // 5 seconds
  enableKeepAlive: true,
  multipleStatements: false,  // Prevents SQL injection via stacking
  charset: "utf8mb4_unicode_ci"
}
```

- **Socket vs Host**: Auto-detects Cloud SQL Unix socket (path starts with `/` or `\\`)
- **Lazy initialization**: Pools created on first use, persist across requests (singleton)

### executeWithRetry Pattern

```typescript
async function executeWithRetry<T>(
  operation: (connection: PoolConnection) => Promise<T>,
  maxRetries: number = 3
): Promise<T>
```

- Gets connection from pool
- Executes operation callback
- On failure: exponential backoff (1s, 2s, 3s) with retry
- Releases connection on success or final failure

---

## Query Building & SQL Patterns

### Sort Column Whitelist

All sort keys map to validated SQL expressions via `SORT_COLUMN_MAP`:

```typescript
{
  my: "product_name",
  comp: "comp_product",
  sku: "sku",
  date: "date_taken",
  category: "CASE WHEN sheet_name='VVIP' THEN 4 WHEN ... END",
  shop: "LOWER(TRIM(shop_name))",
  priceMy: "our_price",
  profitMy: "CAST(our_price - COALESCE(cost, 0) AS DECIMAL(10,2))",
  similarity: "product_similarity_score",
  // ... 27 total
}
```

This prevents SQL injection — user-supplied sort keys are looked up in the map, never interpolated directly.

### Category Sort Expression

```sql
CASE
  WHEN sheet_name IN ('VVIP','VVIP_SG') THEN 4
  WHEN sheet_name = 'VIP' OR sheet_name IS NULL THEN 3
  WHEN sheet_name = 'LINKS_INPUT' THEN 2
  WHEN sheet_name = 'NEW_ITEMS' THEN 1
  ELSE 0
END
```

### Search Implementation

Two-tier search:
1. **FULLTEXT** boolean mode on indexed columns (fast)
2. **LIKE** fallback on product_name, comp_product, sku, our_link (catches partial matches)

URL detection: If search term looks like a URL (contains `/` or `.`), prioritizes link matching.

### Insight Filter Expressions

| Filter | SQL |
|--------|-----|
| `sales-higher` | `our_sales > comp_sales` |
| `sales-lower` | `our_sales < comp_sales AND comp_sales > 0` |
| `stock-higher` | `our_stock > comp_stock` |
| `stock-lower` | `our_stock < comp_stock AND comp_stock > 0` |
| `rating-higher` | `our_rating > comp_rating` |
| `rating-lower` | `our_rating < comp_rating AND comp_rating > 0` |
| `price-lower` | `our_price < comp_price` |
| `price-higher` | `our_price > comp_price AND comp_price > 0` |
| `comp-oos` | `comp_stock = 0 OR comp_stock IS NULL` |
| `ours-oos` | `our_stock = 0 OR our_stock IS NULL` |

---

## Caching & Performance

### Count Cache (Repository)

| Property | Value |
|----------|-------|
| Type | In-memory Map |
| TTL | 120 seconds |
| Max entries | 100 (LRU eviction) |
| Key | JSON of 15 filter dimensions |
| Invalidation | On any mutation (insert, update, delete, status change) |

### Summary Table Cache

| Property | Value |
|----------|-------|
| Table | `Shopee_Comp_Group_Summary` |
| TTL | 10 minutes |
| Refresh | `REPLACE INTO` with aggregated subqueries |
| Dedup | `summaryInFlight` Promise prevents concurrent refreshes |
| Used when | `groupByName=true` AND `status=active` AND no filters |

### HTTP Caching

| Endpoint | Cache-Control |
|----------|---------------|
| GET /api/products | `private, max-age=30, stale-while-revalidate=60` |
| GET /api/products/counts | `private, max-age=30, stale-while-revalidate=60` |

### Comp Analysis Product Count Cache

| Property | Value |
|----------|-------|
| TTL | 5 minutes |
| Scope | Per search term |
| Skippable | `skipCount=true` param |

---

## Security

### SQL Injection Prevention

- **Parameterized queries**: All user input via `?` placeholders
- **Whitelist maps**: Sort columns (`SORT_COLUMN_MAP`), exclusion columns (`EXCLUSION_COLUMN_MAP`), insight filters
- **No string interpolation** of user input into SQL
- `multipleStatements: false` in pool config prevents query stacking

### Input Validation

| Field | Max Length | Additional |
|-------|-----------|-----------|
| `our_link` | 2048 chars | Control characters stripped |
| `comp_link` | 2048 chars | Control characters stripped |
| `sku` | 255 chars | NULL-safe handling |
| `reason` | 1000 chars | Control characters stripped |
| `exclusion_type` | — | Exact whitelist: "product" or "variation" |
| `sortKey` | — | Must exist in SORT_COLUMN_MAP |
| `insightFilter` | — | Must exist in allowed list |
| Remark tokens | 100 chars each | Max 20 tokens per remark |

### Transaction Safety

- `FOR UPDATE` row locks in exclusion functions
- `withTransaction()` wrapper for multi-statement atomicity
- `JSON_ARRAY_APPEND()` for atomic array modifications (no read-modify-write race)

### Error Handling

- API routes: Safe error messages only (whitelist of known messages)
- Never expose SQL errors, stack traces, or internal details to client
- Dev-only logging via `devLog()` (NODE_ENV=development)

---

## Data Pipeline — End to End

### Request: GET /api/products (Shopee MY tab)

```
Browser: fetchProducts({ status: "active", groupByName: true, salesWindow: 30, ... })
    |
API Route (route.ts):
    parseFilterParams() -> validated ProductFilters
    |
Repository (shopee-products-repository.ts):
    fetchShopeeProducts(filters)
    |
    ├── Check count cache (120s TTL)
    │   Hit? -> Use cached total/groupTotal
    │   Miss? -> COUNT(*) query -> cache result
    |
    ├── Check summary table (if no filters)
    │   Fresh? -> Use pre-computed aggregates
    │   Stale? -> refreshShopeeCompGroupSummaryIfStale()
    |
    ├── Phase 1: Paginated product names
    │   SELECT DISTINCT product_name ... LIMIT ? OFFSET ?
    |
    ├── Phase 2: All rows for those products
    │   SELECT * FROM Shopee_Comp
    │   WHERE product_name IN (?)
    │   LEFT JOIN Shopee_Comp_Similarity_Exclusions (x2: product + variation)
    |
    └── Return { rows, total, groupTotal, debug }
    |
Enrichment (shopee-products-enrichment.ts):
    enrichShopeeProductRows(rows, { metricsWindow: 30 })
    |
    ├── Deduplicate competitor rows (keep latest per comparison)
    ├── Batch fetch: costingMap, inventoryMap, voucherMap (parallel)
    ├── Transform each row -> Product object
    ├── Filter to latest comp scrape per group
    ├── Group by product name -> compute groupMetrics, variationMetrics
    ├── Attach metrics to each Product
    └── Fire-and-forget: backfillDiscountedPrices()
    |
API Route:
    Return JSON { success, data: enrichedRows, total, groupTotal, ... }
    Set Cache-Control: private, max-age=30
    |
Browser: receives Product[] -> groupProducts() -> render GroupedRows
```

### Request: POST /api/products/similarity-exclusion

```
Browser: user clicks ❌ on similarity score -> AlertDialog -> confirm
    |
API Route (similarity-exclusion/route.ts):
    Validate & sanitize inputs (length, type whitelist, control chars)
    |
Repository:
    addSimilarityExclusion(payload)
    |
    BEGIN TRANSACTION
    ├── SELECT ... FOR UPDATE (lock row)
    ├── IF exists: JSON_ARRAY_APPEND (atomic, guarded by length + duplicate checks)
    │   ELSE: INSERT new row with JSON_ARRAY(reason)
    ├── UPDATE Shopee_Comp SET has_{type}_exclusions = 1
    │   WHERE our_link = ? AND comp_link = ? AND sku conditions
    COMMIT
    |
    invalidateCountCache()
    |
API Route:
    Return { exclusionId, excludedDifferences, isNew, affectedRows }
    |
Browser: refreshAllTabs() -> orange Filter icon appears
    |
[Later] Python bot reads has_*_exclusions flags
    -> Re-runs similarity with excluded differences
    -> May produce higher score
```
