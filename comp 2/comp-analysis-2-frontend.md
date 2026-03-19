# Comp Analysis 2 — Frontend Documentation (VVIP & SG)

**Purpose**: Comprehensive documentation of the frontend architecture, file structure, data flow, UI components, and business logic of the Shopee MY VVIP and Shopee SG Competitive Analysis pages.

**Relationship to Comp 1**: Both VVIP and SG pages are derived from the Shopee MY page (documented in `comp 1/comp-analysis-frontend.md`). They share the same UI components (`grouped-rows.tsx`, `product-row.tsx`, `table-header.tsx`), type definitions (`lib/type.ts`), and enrichment pipeline (`shopee-products-enrichment.ts`). This document focuses on **what is different** and the **new infrastructure** added to support these pages.

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture Comparison — MY vs VVIP vs SG](#architecture-comparison--my-vs-vvip-vs-sg)
3. [File Structure](#file-structure)
4. [Page Architecture — Shopee MY VVIP](#page-architecture--shopee-my-vvip)
5. [Page Architecture — Shopee SG](#page-architecture--shopee-sg)
6. [API Layer — Client Side](#api-layer--client-side)
7. [API Layer — Server Side (Route Handlers)](#api-layer--server-side-route-handlers)
8. [Shared GET Handler Factory](#shared-get-handler-factory)
9. [Repository Layer — VVIP](#repository-layer--vvip)
10. [Repository Layer — SG](#repository-layer--sg)
11. [New Service: In-Memory Stale Cache](#new-service-in-memory-stale-cache)
12. [New Service: Normalized Group Summary](#new-service-normalized-group-summary)
13. [Enrichment Pipeline](#enrichment-pipeline)
14. [Data Flow](#data-flow)
15. [Performance Optimizations](#performance-optimizations)
16. [Database Tables](#database-tables)
17. [Caching Strategy](#caching-strategy)
18. [Known Limitations & TODOs](#known-limitations--todos)

---

## Overview

The Comp Analysis 2 feature extends the competitive analysis capability to two additional views:

- **Shopee MY VVIP** (`/analytics/table/shopee-my-vvip`) — Focuses on VVIP/VIP category products in the MY region, using the `Shopee_My_Products` table instead of `Shopee_Comp`
- **Shopee SG** (`/analytics/table/shopee-sg`) — Same architecture as VVIP but filtered to `region = 'SG'`, for Singapore market analysis

Both pages share the same 3-level hierarchical table UI as Shopee MY (documented in comp 1) but have separate:
- API routes (`/api/shopee-my-vvip/products`, `/api/shopee-sg/products`)
- Repository files (separate SQL query layers)
- Pre-computed summary tables (`Shopee_VVIP_Group_Summary`, `Shopee_SG_Group_Summary`)
- Client-side API service layers

---

## Architecture Comparison — MY vs VVIP vs SG

| Aspect | Shopee MY (Comp 1) | Shopee MY VVIP | Shopee SG |
|--------|-------------------|----------------|-----------|
| **URL** | `/analytics/table/shopee-my` | `/analytics/table/shopee-my-vvip` | `/analytics/table/shopee-sg` |
| **API Route** | `/api/products` | `/api/shopee-my-vvip/products` | `/api/shopee-sg/products` |
| **DB Table (Our Products)** | `Shopee_Comp` | `Shopee_My_Products` | `Shopee_My_Products` |
| **DB Table (Competitors)** | `Shopee_Comp` (same table) | `Shopee_Comp_Data` (JOIN) | `Shopee_Comp_Data` (JOIN) |
| **Region Filter** | N/A (MY only) | `region = 'MY'` | `region = 'SG'` |
| **Summary Table** | `Shopee_Comp_Group_Summary` | `Shopee_VVIP_Group_Summary` | `Shopee_SG_Group_Summary` |
| **Repository** | `shopee-products-repository.ts` | `shopee-vvip-products-repository.ts` | `shopee-sg-products-repository.ts` |
| **Tabs Shown** | 4 (Shopee MY, Pending, Fixed/New, Deleted) | 1 (Shopee MY only) | 1 (Shopee SG only) |
| **Shop Filter Options** | 27 hardcoded shops | 27 hardcoded shops (same) | Empty (TODO: populate when bot runs) |
| **JOIN Strategy** | Direct (same table) | `cd.our_item_id_extracted = mp.our_item_id` | `cd.our_item_id_extracted = mp.our_item_id` |
| **localStorage Prefix** | `shopeeMY:` | `shopeeMY:` | `shopeeSG:` |
| **Audit Log Model** | `shopee_products` | `shopee_products_vvip` | `shopee_products_sg` |
| **GET Handler** | Direct implementation | Shared factory (`createShopeeProductsGetHandler`) | Shared factory |

---

## File Structure

```
AWESOMEREE-WEB-APP/
├── app/
│   ├── analytics/table/
│   │   ├── shopee-my-vvip/
│   │   │   ├── page.tsx                          # VVIP page (~3,271 lines)
│   │   │   └── lib/
│   │   │       └── api.ts                        # VVIP client API (~893 lines)
│   │   └── shopee-sg/
│   │       ├── page.tsx                          # SG page (~3,242 lines)
│   │       └── lib/
│   │           └── api.ts                        # SG client API (~893 lines)
│   └── api/
│       ├── shopee-my-vvip/
│       │   └── products/
│       │       └── route.ts                      # VVIP API route (~297 lines)
│       ├── shopee-sg/
│       │   └── products/
│       │       └── route.ts                      # SG API route (~296 lines)
│       └── shopee-products/
│           └── shared-products-route.ts          # Shared GET handler factory
├── lib/services/
│   ├── shopee-vvip-products-repository.ts        # VVIP data access (~1,244 lines)
│   ├── shopee-sg-products-repository.ts          # SG data access (~1,242 lines)
│   ├── shopee-normalized-group-summary.ts        # Pre-aggregated summaries (NEW, ~281 lines)
│   ├── shopee-products-enrichment.ts             # Shared enrichment pipeline (~710 lines)
│   └── in-memory-stale-cache.ts                  # Generic stale-while-revalidate cache (NEW, ~95 lines)
└── components/
    └── Shopee-MY-History/
        ├── grouped-rows.tsx                      # Shared — same component as Shopee MY
        ├── product-row.tsx                       # Shared — same component as Shopee MY
        └── table-header.tsx                      # Shared — same component as Shopee MY
```

---

## Page Architecture — Shopee MY VVIP

### File: `app/analytics/table/shopee-my-vvip/page.tsx` (~3,271 lines)

A near-complete copy of `shopee-my/page.tsx` adapted for VVIP products.

### Key Differences from Shopee MY

| Aspect | Shopee MY | VVIP |
|--------|-----------|------|
| **Visible tabs** | 4 tabs (Shopee MY, Pending, Fixed/New, Deleted) | 1 tab only ("Shopee MY") — Pending/Fixed/Deleted code exists but UI hidden |
| **API base** | `/api/products` | `/api/shopee-my-vvip/products` |
| **Download scope** | `["filtered", "selected"]` | `["filtered"]` only |
| **GroupedRows props** | Standard | `backfillCompProducts` prop added |
| **Tab handlers** | Full 4-tab navigation | All handler code present but only active tab rendered |

### State Variables (~40 total)

Same as Shopee MY (documented in comp 1) with these differences:

| State | Default | Notes |
|-------|---------|-------|
| `activeTab` | `"Shopee MY"` | Only one tab visible in UI |
| `sort` | `{ key: "category", dir: "desc" }` | Same default |
| `salesRangePreset` | `30` | Same |
| `rowsPerPage` | `20` | Same |

### Race Condition Protection

Uses request ID refs to prevent stale responses:

```
productsRequestId (ref) — incremented on each load
pendingRequestId (ref)  — same pattern per tab
fixedRequestId (ref)
deletedRequestId (ref)

loadProducts():
  myId = ++productsRequestId.current
  ... fetch ...
  if (myId !== productsRequestId.current) return  // stale, discard
```

Additional protection:
- `loadedTabsRef` / `loadingTabsRef` — prevents concurrent loads of the same tab
- `pendingForcedReloadRef` — queues forced reloads if a load is already in progress
- `initialLoadStartedRef` / `initialLoadDoneRef` — ensures mount loads exactly once

### Hardcoded Shop Options (27 shops)

```
Adelmar, Atlusrack, BahEmas, BainsJewelry, CloudBear, Degaia,
Dollfayce, EasyBuyMY, FluxClothing, Get Any, Inflatable, KariFM,
Lil Beanie, Luluna, Nartistry, NewGem, Petit Toute, Posh Cutlery,
Primesgold, Queenswood, Reno, Runshogun, SkyBall, Taftees,
Tuffsteel, VetPlanet, Vistella
```

---

## Page Architecture — Shopee SG

### File: `app/analytics/table/shopee-sg/page.tsx` (~3,242 lines)

Nearly identical to the VVIP page with systematic substitutions.

### Key Differences from VVIP

| Aspect | VVIP | SG |
|--------|------|-----|
| **Default tab name** | `"Shopee MY"` | `"Shopee SG"` |
| **API base** | `/api/shopee-my-vvip/products` | `/api/shopee-sg/products` |
| **localStorage prefix** | `shopeeMY:` | `shopeeSG:` |
| **Shop options** | 27 shops hardcoded | Empty array (TODO: populate when bot runs) |
| **Export filename** | `shopee_Shopee-MY_...` | `shopee_Shopee-SG_...` |
| **Restore target** | `"Shopee MY"` | `"Shopee SG"` |

### Single-Tab UI

Both VVIP and SG render only one tab badge instead of the full `<TabNavigation>` component:

```
// Single tab -- Pending/Fixed/Deleted tabs omitted
// (no status/bulk-status/permanent-delete routes)
```

The handler code for all four tabs still exists in the component — only the UI rendering is limited to the main tab.

---

## API Layer — Client Side

### Files: `shopee-my-vvip/lib/api.ts` and `shopee-sg/lib/api.ts` (~893 lines each)

Identical logic, only the endpoint base path differs.

### Functions

| Function | VVIP Endpoint | SG Endpoint |
|----------|--------------|-------------|
| `fetchTabCounts()` | GET `/api/shopee-my-vvip/products/counts` | GET `/api/shopee-sg/products/counts` |
| `fetchProducts(filters)` | GET `/api/shopee-my-vvip/products` | GET `/api/shopee-sg/products` |
| `fetchProductDetails(ids)` | POST `/api/shopee-my-vvip/products/details` | POST `/api/shopee-sg/products/details` |
| `fetchAllProducts(filters)` | Paginated wrapper | Paginated wrapper |
| `updateProductStatus()` | PUT `.../status` | PUT `.../status` |
| `bulkUpdateProductStatus()` | PUT `.../bulk-status` | PUT `.../bulk-status` |
| `permanentDeleteProducts()` | DELETE `.../permanent-delete` | DELETE `.../permanent-delete` |
| `createProduct()` | POST `.../products` | POST `.../products` |
| `updateProduct()` | PUT `.../products` | PUT `.../products` |
| `updateProductCategory()` | PATCH `.../products` | PATCH `.../products` |
| `deleteProduct()` | DELETE `.../products?id=` | DELETE `.../products?id=` |

### Retry Logic

```
MAX_FETCH_ATTEMPTS: 3
TRANSIENT_HTTP_STATUSES: {500, 502, 503, 504}
Backoff: 250ms × attempt number (linear)
```

### Request Deduplication

`fetchTabCounts()` uses a module-level `_countsInFlight` promise sentinel:
- If a fetch is already in-flight, concurrent callers share the same promise
- Sentinel cleared on settle (success or error) for retry
- Prevents 503s from duplicate requests during React re-mounts

### Batch Inventory Lookup

`fetchInventoryRowsBySkuBatch(skus)`:
- Chunks into batches of 500 SKUs max
- 10s AbortController timeout per batch
- Sequential chunk processing

---

## API Layer — Server Side (Route Handlers)

### Files: `shopee-my-vvip/products/route.ts` (~297 lines) and `shopee-sg/products/route.ts` (~296 lines)

### Shared Factory Pattern

Both use `createShopeeProductsGetHandler` for the GET endpoint:

```typescript
// VVIP
const productsRoute = createShopeeProductsGetHandler({
  routeLabel: "shopee-my-vvip",
  region: "MY",
  parseFilters: parseFilterParams,
  fetchProducts: fetchVvipProducts,
});

// SG
const productsRoute = createShopeeProductsGetHandler({
  routeLabel: "shopee-sg",
  region: "SG",
  parseFilters: parseFilterParams,
  fetchProducts: fetchSgProducts,
});
```

### HTTP Methods

| Method | Purpose | VVIP Repository | SG Repository |
|--------|---------|-----------------|---------------|
| GET | Fetch products (shared handler) | `fetchVvipProducts` | `fetchSgProducts` |
| POST | Create product | `insertVvipProduct` | `insertSgProduct` |
| PUT | Update product | `updateVvipProduct` | `updateSgProduct` |
| PATCH | Update category | `updateVvipProductCategory` | `updateSgProductCategory` |
| DELETE | Soft delete | `deleteVvipProduct` | `deleteSgProduct` |

### Query Parameters

Parsed by `parseFilterParams()` (identical in both routes):

| Category | Parameters |
|----------|-----------|
| **Pagination** | `limit` (default 50, max 500), `offset` |
| **Status** | `status` (default "active"), `statusGroup` ("fixed-new") |
| **Category** | `sheetName`, `sheetNames[]`, `uncategorized` |
| **Search** | `search`, `customLink` |
| **Shop** | `shopName`, `shopId`, `shopSelection` ("unknown") |
| **Date** | `dateFrom`, `dateTo` |
| **Sort** | `sortKey` (~30 allowed keys), `sortDir` (asc/desc) |
| **Sales** | `salesWindow`, `shopeeSalesWindow` (7/14/30/60/90) |
| **Grouping** | `groupBy` ("product"), `includeCount`, `detailLevel`, `debug` |
| **Remarks** | `remarks[]`, `insight[]` |

### Allowed Sort Keys

```
my, comp, sku, date, id, category, shop, productId,
priceMy, profitMy, profitMarginMy, isrPercent,
salesMy, sgSalesValue, sgVariationPct, salesMyValue,
spSalesValue, spVariationPct, stockMy,
visitors, conversionRate, roas, adsSpend,
similarity, similarityReason, dateCompared,
priceComp, salesComp, compMonthlySales, stockComp,
remarks, mkRemark
```

### Cache Invalidation

Every mutation (POST/PUT/PATCH/DELETE) calls `productsRoute.clearCache()` to bust the shared GET handler's in-memory cache.

---

## Shared GET Handler Factory

### File: `app/api/shopee-products/shared-products-route.ts`

Creates a standardized GET handler with built-in:

- **In-memory stale-while-revalidate cache** (30s fresh TTL, 5min stale TTL, max 200 entries)
- **DB retry logic** (3 attempts, 250ms base delay, retries on ETIMEDOUT/ECONNRESET/etc.)
- **Product enrichment pipeline** (comp product grouping, variation matching)
- **Cache key generation** from request URL + query params

---

## Repository Layer — VVIP

### File: `lib/services/shopee-vvip-products-repository.ts` (~1,244 lines)

### Key Functions

| Function | Purpose |
|----------|---------|
| `fetchVvipProducts(filters, region?)` | Main query — grouped and ungrouped modes |
| `fetchVvipProductCounts(region?)` | Tab counts (active/pending/fixed+new/deleted) |
| `fetchVvipProductDetails(ids, region?)` | Detail view (descriptions, images) |
| `insertVvipProduct()` | Create |
| `updateVvipProduct()` | Update |
| `updateVvipProductCategory()` | Category change |
| `deleteVvipProduct()` | Soft delete |

### SQL Architecture

**Base tables:**
- `Shopee_My_Products mp` — our products
- `Shopee_Comp_Data cd` — competitor data (LEFT JOIN)

**JOIN strategy:**
```sql
LEFT JOIN Shopee_Comp_Data cd
  ON cd.our_item_id_extracted = mp.our_item_id
  AND cd.region = 'MY'
```

Uses numeric `our_item_id_extracted` = `our_item_id` join (avoids slow string manipulation on URLs).

### Two-Phase Grouped Query

When `groupByName: true`:

**Phase 1**: Get sorted product names for the current page
```sql
-- If summary table eligible (simple filters):
SELECT product_name, row_count FROM Shopee_VVIP_Group_Summary
WHERE status = 'active' [+ category filters]
ORDER BY [sort column] LIMIT X OFFSET Y

-- Otherwise: live GROUP BY
SELECT mp.product_name FROM Shopee_My_Products mp
[LEFT JOIN Shopee_Comp_Data cd ...]
WHERE [filters]
GROUP BY mp.product_name
ORDER BY [sort] LIMIT X OFFSET Y
```

**Phase 2**: Fetch all rows for those product names
```sql
SELECT [full column list] FROM Shopee_My_Products mp
LEFT JOIN Shopee_Comp_Data cd ...
WHERE mp.product_name IN (...phase1 names...)
ORDER BY FIELD(mp.product_name, ...phase1 order...)
```

### Sales Window Column Maps

| Window | Our Sales Col | Shopee Sales Col | Shopee Value Col |
|--------|--------------|-----------------|-----------------|
| 7d | `our_sales_7d` | `shopee_var_sales_7d` | `shopee_var_value_7d` |
| 14d | `our_sales_14d` | `shopee_var_sales_14d` | `shopee_var_value_14d` |
| 30d | `our_sales_30d` | `shopee_var_sales_30d` | `shopee_var_value_30d` |
| 60d | `our_sales_60d` | `shopee_var_sales_60d` | `shopee_var_value_60d` |
| 90d | `our_sales_90d` | `shopee_var_sales_90d` | `shopee_var_value_90d` |

> **Note:** The DB also has `sitegiant_sales_value_*` columns, but the code uses `our_sales_*` columns instead (with `COALESCE(mp.our_sales_Xd, mp.our_sales)` as fallback).

### Count Cache

LRU Map with max 100 entries, 120s TTL, keyed by JSON-serialized filter combination.

---

## Repository Layer — SG

### File: `lib/services/shopee-sg-products-repository.ts` (~1,242 lines)

Structurally identical to the VVIP repository with these differences:

| Aspect | VVIP | SG |
|--------|------|-----|
| **Region filter** | `cd.region = 'MY'` | `cd.region = 'SG'` |
| **Summary table** | `Shopee_VVIP_Group_Summary` | `Shopee_SG_Group_Summary` |
| **Summary refresh** | `refreshNormalizedGroupSummaryIfStale("vvip")` | `refreshNormalizedGroupSummaryIfStale("sg")` |
| **Function prefix** | `fetchVvipProducts`, `insertVvipProduct`, etc. | `fetchSgProducts`, `insertSgProduct`, etc. |
| **Default region fallback** | `"MY"` | `"SG"` |

### JOIN Optimization

`buildMpOnlyWhere()` generates a simplified WHERE clause that only touches the `mp` table — used when the sort/filter doesn't reference `cd.*` columns, avoiding the expensive JOIN entirely for count and name queries.

`referencesCompTable()` checks if SQL fragments reference `cd.` to decide if the JOIN is needed.

---

## New Service: In-Memory Stale Cache

### File: `lib/services/in-memory-stale-cache.ts` (~95 lines)

Generic typed cache with fresh/stale semantics and thundering-herd prevention.

### Class: `InMemoryStaleCache<T>`

```typescript
new InMemoryStaleCache<T>({
  freshTtlMs: 30_000,    // 30s fresh window
  staleTtlMs: 300_000,   // 5min stale fallback
  maxEntries: 200         // LRU eviction limit
})
```

### Methods

| Method | Purpose |
|--------|---------|
| `getFresh(key)` | Returns value only if within fresh TTL |
| `getStale(key)` | Returns value even after fresh expiry (stale fallback) |
| `set(key, value)` | Stores value with current timestamp |
| `getInFlight(key)` | Returns in-progress promise if one exists (request coalescing) |
| `setInFlight(key, promise)` | Registers an in-progress fetch |
| `clearInFlight(key)` | Removes in-flight tracking |
| `clear(key?)` | Clears one key or entire cache |

### Thundering Herd Prevention

```
Request A arrives → cache miss → starts fetch → setInFlight(key, promise)
Request B arrives → cache miss → getInFlight(key) returns promise → awaits same fetch
Request C arrives → same as B
Fetch completes → set(key, result) → clearInFlight(key)
All 3 requests get the same result from 1 DB query
```

### LRU Eviction

Uses `Map` insertion-order trick: `touch()` deletes and re-inserts a key to move it to the end. `pruneIfNeeded()` evicts from the front (oldest entries first).

---

## New Service: Normalized Group Summary

### File: `lib/services/shopee-normalized-group-summary.ts` (~281 lines)

Maintains pre-aggregated summary tables that store per-product-group rollups, avoiding expensive `GROUP BY` + `SUM` queries on every paginated request.

### Key Functions

| Function | Purpose |
|----------|---------|
| `refreshNormalizedGroupSummaryIfStale(target)` | Refreshes if last refresh > 10 minutes ago |
| `normalizedSummarySortColumn(sortKey, salesWindow, shopeeSalesWindow)` | Maps sort key to pre-computed column name |
| `supportsNormalizedSummarySort(sortKey)` | Returns true if summary table supports this sort |
| `buildRefreshSql(config)` | Generates the `REPLACE INTO` statement |

### Summary Table Schema

Pre-computed columns per `(status, product_name)` group:

| Column Category | Columns |
|----------------|---------|
| **Identity** | `status`, `product_name`, `row_count` |
| **Category** | `category_rank` (highest priority in group) |
| **Our Sales** (per window) | `our_sales_total_7d`, `our_sales_total_14d`, `our_sales_total_30d`, `our_sales_total_60d`, `our_sales_total_90d` |
| **Shopee Sales** (per window) | `sp_sales_total_7d` ... `sp_sales_total_90d` |
| **Sales Values** (per window) | `our_value_total_7d` ... `our_value_total_90d`, `sp_value_total_7d` ... `sp_value_total_90d` |
| **Competitor** | `max_date_taken`, `min_comp_product`, `min_comp_price`, `max_comp_sales`, etc. |
| **Metadata** | `updated_at` |

### Refresh Logic

```
refreshNormalizedGroupSummaryIfStale("sg"):
  1. Check lastRefresh timestamp → skip if < 10 min ago
  2. Check inFlight map → await existing refresh if running
  3. Execute REPLACE INTO Shopee_SG_Group_Summary (
       SELECT ... FROM Shopee_My_Products mp
       LEFT JOIN Shopee_Comp_Data cd ...
       GROUP BY mp.status, mp.product_name
     )
  4. Update lastRefresh timestamp
```

### spSalesValue Calculation

For `sp_value_total_*` columns:
```sql
SUM(CASE WHEN shopee_var_value_30d > 0
    THEN shopee_var_value_30d
    ELSE shopee_var_sales_30d * our_price END)
```

Uses real revenue (`shopee_var_value`) when available, falls back to `sales × price`.

### Region Differentiation

| Target | Table Written | Comp JOIN Condition |
|--------|--------------|-------------------|
| `"sg"` | `Shopee_SG_Group_Summary` | `cd.region = 'SG'` |
| `"vvip"` | `Shopee_VVIP_Group_Summary` | `cd.region = 'MY'` |

---

## Enrichment Pipeline

### File: `lib/services/shopee-products-enrichment.ts` (~710 lines)

Shared across all three pages (MY, VVIP, SG). The same `enrichShopeeProductRows()` function is called regardless of region.

### Pipeline Steps

```
Raw DB rows
  │
  ├─ 1. deduplicateCompetitorRows()
  │     Keep latest per (our_link, comp_link, comp_variation, our_variation)
  │
  ├─ 2. Parallel data fetch (Promise.all):
  │     ├─ fetchCostingTotalsBySku(uniqueSkus)
  │     ├─ getInventoryRowsBySkuBatch(uniqueSkus)
  │     └─ fetchActiveShopVouchers()
  │
  ├─ 3. Row mapping → Product objects:
  │     ├─ Our product data (price, sales, stock, images)
  │     ├─ Competitor product data
  │     ├─ Profit/margin calculations
  │     ├─ Inventory metrics (ISR%, stockout, lost value)
  │     └─ Voucher-discounted prices
  │
  ├─ 4. Group metrics:
  │     ├─ Per-group: aggregated profit, ISR%, sales totals, category priority
  │     ├─ Per-variation: variation-level profit, sales %, value displays
  │     └─ Parent sort values (~19 sort keys)
  │
  └─ 5. Attach metrics to each Product:
        groupMetrics, parentSortValues, variationMetrics
```

### VVIP/SG-Specific Enrichment Options

| Option | MY Value | VVIP/SG Value | Purpose |
|--------|----------|---------------|---------|
| `backfillTarget` | `undefined` (writes to `Shopee_Comp`) | `"vvip-sg"` (writes to `Shopee_My_Products`) | Controls which table gets voucher backfill |
| `skipBackfill` | `false` | `true` | Prevents heavy backfill-on-read for VVIP/SG |

---

## Competitor Product Display Logic

### How It Looks to the User (3-Level Hierarchy)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ LEVEL 1: Product Group Row                                    BG: #FAF9F5  │
│                                                                             │
│ [Category] [Shop] [ID] "MY Product Name ABC"         [MY ▼]  [COMP ▼]     │
│                         (aggregated metrics)          toggle   toggle       │
└─────────────────────────────────────────────────────────────────────────────┘
       │                                              │
       │ Click [MY ▼]                                 │ Click [COMP ▼]
       ▼                                              ▼
┌──────────────────────────────────┐    ┌──────────────────────────────────────┐
│ LEVEL 2a: OUR Variations         │    │ LEVEL 2b: COMP Products              │
│ BG: #F4F3EE                      │    │ BG: #F0ECE5                          │
│ Border: 3px orange (#D97757)     │    │ Border: 3px blue (#6A9BCC)           │
│                                  │    │                                      │
│ ┌─ Variation "Color - Red" ────┐ │    │ ┌─ Comp Product "Rival X" ────────┐ │
│ │  SKU: ABC-RED                │ │    │ │  Shop: CompetitorShop            │ │
│ │  Price: RM29.90              │ │    │ │  Price: RM25.90                  │ │
│ │  [▶ expand] ← only if has   │ │    │ │  Monthly Sales: 1,234            │ │
│ │     matching comp variations │ │    │ │  [▶ expand] ← shows ALL comp    │ │
│ └──────────────────────────────┘ │    │ │     variations for this product  │ │
│                                  │    │ └──────────────────────────────────┘ │
│ ┌─ Variation "Color - Blue" ───┐ │    │                                      │
│ │  SKU: ABC-BLU                │ │    │ ┌─ Comp Product "Brand Y" ────────┐ │
│ │  (no expand icon — no comps) │ │    │ │  Monthly Sales: 890              │ │
│ └──────────────────────────────┘ │    │ │  [▶ expand]                      │ │
└──────────────────────────────────┘    │ └──────────────────────────────────┘ │
                                        │                                      │
       │ Click [▶] on "Color - Red"     │ ┌─ Comp Product "Store Z" ────────┐ │
       ▼                                │ │  (max 3 comp products shown)     │ │
┌──────────────────────────────────┐    │ └──────────────────────────────────┘ │
│ LEVEL 3: Matched Comp Variations │    └──────────────────────────────────────┘
│ (matched to OUR "Color - Red")   │           │ Click [▶] on "Rival X"
│ BG: alternating #FAF9F5/#F4F3EE  │           ▼
│                                  │    ┌──────────────────────────────────────┐
│ These are comp variations that   │    │ LEVEL 3: ALL Comp Variations         │
│ the BOT matched to our "Red"     │    │ (for this comp product)              │
│ variation via the                │    │ BG: #EAE5DC                          │
│ Shopee_Variation_Match table     │    │ Border: 3px gray (#B0AEA5)           │
│                                  │    │                                      │
│ ┌─ Comp Var "Size S - Red" ───┐ │    │ ┌─ Comp Var "Size S" ──────────────┐ │
│ │  Price: RM22.90              │ │    │ │  Price: RM22.90                  │ │
│ │  Stock: 45                   │ │    │ │  Stock: 45                       │ │
│ └──────────────────────────────┘ │    │ └──────────────────────────────────┘ │
│                                  │    │                                      │
│ ┌─ Comp Var "Size M - Red" ───┐ │    │ ┌─ Comp Var "Size M" ──────────────┐ │
│ │  Price: RM22.90              │ │    │ │  Price: RM22.90                  │ │
│ │  Stock: 30                   │ │    │ │  Stock: 30                       │ │
│ └──────────────────────────────┘ │    │ └──────────────────────────────────┘ │
└──────────────────────────────────┘    └──────────────────────────────────────┘
```

### Key Rules

| Rule | Behavior |
|------|----------|
| **MY and COMP are mutually exclusive** | Opening MY closes COMP (and vice versa) for the same product group |
| **Level 2a expand icon** | Only shown if that our-variation has matching comp variations in the DB |
| **Level 2a → Level 3 (MY path)** | Shows only comp variations that the **bot matched** to that specific our-variation |
| **Level 2b expand icon** | Always shown — opens ALL variations for that comp product |
| **Level 2b → Level 3 (COMP path)** | Shows ALL variations for that comp product (not filtered by our-variation) |
| **Max 3 comp products** | At Level 2b, only top 3 comp products shown (sorted by max monthly sales, ties included) |
| **`"(no comp name)"` filtered out** | Comp products without a name are never shown |
| **Closing a parent closes children** | Closing Level 2a/2b also closes all Level 3 rows underneath |

### Variation Matching — Done by Bot, Not Frontend

The mapping between "which comp variation belongs to which of our variations" is **NOT done by the frontend**. It is pre-computed by a bot and stored in the `Shopee_Variation_Match` table in the database.

```
Bot (offline process):
  → Analyzes our variations vs comp variations
  → Writes matches to Shopee_Variation_Match table

Server (shared-products-route.ts):
  → Reads Shopee_Variation_Match to assign comp variations to our variations
  → Attaches mapping as compProductsForDisplay metadata on Product rows

Frontend (grouped-rows.tsx):
  → Just reads the pre-computed mapping
  → Renders comp variations under the correct our-variation
  → Does NOT do any matching logic itself
```

If no bot match exists, the server falls back to assigning all comp variations to the first our-variation.

---

### Step 1: Database — Fetch All Comp Rows

The SQL returns ALL competitor rows via LEFT JOIN — no filtering at the SQL level:

```sql
LEFT JOIN Shopee_Comp_Data cd
  ON cd.our_item_id_extracted = mp.our_item_id
  AND cd.region = '<MY|SG>'
```

**Comp columns selected:**
- `cd.comp_product` (name), `cd.comp_link` (URL), `cd.comp_variation`, `cd.comp_shop`
- `cd.comp_price`, `cd.comp_product_price`, `cd.comp_sales`, `cd.comp_monthly_sales`
- `cd.comp_rating`, `cd.comp_stock`, `cd.date_taken`

No date/null filtering in SQL — all comp rows for each product are returned. Filtering and dedup happen server-side.

### Step 2: Enrichment — Map to Product Objects

Each raw DB row becomes one `Product` with competitor fields:

| DB Column | Product Field |
|-----------|--------------|
| `comp_product` | `competitorProduct.name` |
| `comp_shop` | `competitorProduct.shopName` |
| `comp_variation` | `competitorProduct.variation` |
| `comp_price` | `competitorProduct.price` (parsed to number) |
| `comp_sales` | `competitorProduct.sales` |
| `comp_monthly_sales` | `competitorProduct.monthlySales` |
| `comp_stock` | `competitorProduct.stock` |
| `comp_link` | `competitorProduct.url` |
| `comp_main_images` | `competitorProduct.mainImages` |
| `comp_variation_images` | `competitorProduct.variationImages` |

### Step 3: Server-Side Dedup & Top-3 Selection

**Function:** `limitAndDedupCompetitors()` in `shared-products-route.ts`

```
For each our-product group (keyed by ourProduct.url):
  │
  ├─ 1. Build compSalesMap:
  │     Key = "shopName\0compName"
  │     Value = max monthlySales among all rows for that comp product
  │
  ├─ 2. Top 3 + ties selection:
  │     Sort comp products by monthlySales DESC
  │     If ≤ 3 products → keep all
  │     If > 3 → find 3rd-place sales value (cutoffSales)
  │            → keep all products with sales ≥ cutoffSales
  │
  ├─ 3. Build compProductsForDisplay metadata:
  │     Per surviving comp product:
  │       - name, shopName, url, sales, monthlySales, rating
  │       - All unique variations (deduped by name, case-insensitive)
  │         with price, stock, dateScraped, variationImages
  │
  ├─ 4. Assign comp variations to our variations:
  │     Uses Shopee_Variation_Match table (bot-managed matches)
  │     Fallback: all comp variations → first our-variation
  │
  └─ 5. Attach compProductsForDisplay to every Product row in the group
```

### Step 4: UI Rendering — grouped-rows.tsx

#### Building Competitor Groups (Level 2b)

Two paths depending on whether server-side metadata exists:

```
Path A (preferred): compProductsForDisplay metadata exists
  → buildCompGroupsFromDisplayMeta() reconstructs groups from server metadata
  → Preserves server's top-3 selection and complete variation lists

Path B (fallback): No metadata
  → getQualifiedCompRows() runs client-side top-3 algorithm
```

#### `getQualifiedCompRows()` Client-Side Algorithm

```
For each our-variation:
  │
  ├─ 1. Filter out rows where comp name is empty or "(no comp name)"
  │
  ├─ 2. Group valid rows by comp product (key = "shopName\0compName")
  │     Track maxSales per product group
  │
  ├─ 3. Top 3 + ties:
  │     Rank products by maxSales DESC
  │     Keep top 3 + any tied at 3rd place:
  │       thirdSales = ranked[2].maxSales
  │       keep all where index < 3 OR maxSales ≥ thirdSales
  │
  ├─ 4. Backfill: If a comp product survives top-3 on ANY our-variation,
  │     include ALL its variations from ALL our-variations
  │
  └─ 5. Dedup by (shopName, compName, compVariation)
        Keep row with highest monthlySales
        Sort by monthlySales DESC
```

#### Filtering & Sorting for Display

```
visibleCompGroups = compGroups
  .filter(cg => cg.name !== "(no comp name)")     // exclude unnamed
  .filter(cg => !excludedCompKeys.has(key))        // exclude user-excluded
  .sort((a, b) => maxMonthlySales(b) - maxMonthlySales(a))  // sort DESC
  .slice(0, 3)                                     // hard cap at 3 visible
```

#### Level 2b — COMP Product Row Rendering

| Element | Source | Display |
|---------|--------|---------|
| Shop name | `competitorProduct.shopName` | Clickable link (if URL available) |
| Product ID | Extracted from `comp_link` | Numeric string |
| Product name | `competitorProduct.name` | Text + external link + image preview + description |
| Price | `competitorProduct.priceText` | Text (not formatted as RM) |
| Sales | `competitorProduct.sales` | Integer (lifetime) |
| Monthly sales | `competitorProduct.monthlySalesDisplay` | Formatted string with date context |
| Date scraped | `dateScraped` | Date string |
| Expand button | (UI state) | Eye/EyeOff toggle → opens Level 3 variations |

#### Level 3 — COMP Variation Row Rendering

| Element | Source | Display |
|---------|--------|---------|
| Variation name | `competitorProduct.variation` | Badge component |
| Image | `competitorProduct.variationImages` | Side-by-side (MY variation vs COMP variation) |
| Price | `competitorProduct.price` | `RM{value.toFixed(2)}` |
| Stock | `competitorProduct.stock` | Integer |
| Date scraped | `dateScraped` | Date string |
| Sales columns | — | Intentionally blank (sales are product-level, not variation-level) |

#### Expand/Collapse — Mutual Exclusion

```
State:
  openGroups  (Set) — MY variations expanded (Level 2a)
  openComp    (Set) — COMP products expanded (Level 2b)
  openCompVar (Set) — COMP variations expanded (Level 3)

handleToggleMy(gkey):
  Open MY → closes COMP + all COMP children for this group
  Close MY → closes all MY children

handleToggleComp(gkey):
  Open COMP → closes MY + all MY children for this group
  Close COMP → closes all COMP children

Key format for children: "${gkey}|||${variationKey}"
```

### Difference from Shopee MY (Comp 1)

| Aspect | Shopee MY | VVIP / SG |
|--------|-----------|-----------|
| **Comp data source** | Same `Shopee_Comp` table (flat) | `Shopee_Comp_Data` via LEFT JOIN |
| **JOIN key** | N/A (same table) | `cd.our_item_id_extracted = mp.our_item_id` (numeric, fast) |
| **Region filter** | N/A | `cd.region = 'MY'` or `cd.region = 'SG'` on JOIN |
| **Server-side top-3** | Same `limitAndDedupCompetitors()` | Same function |
| **Client-side fallback** | Same `getQualifiedCompRows()` | Same function |
| **UI rendering** | Same `grouped-rows.tsx` | Same component |
| **Variation matching** | `Shopee_Variation_Match` table | Same table |

The display logic is **identical** — only the data source differs.

---

## Data Flow

### Complete Request Lifecycle

```
Browser (page.tsx)
  │
  ├─ 1. buildFilters() → constructs FetchFilters from UI state
  │
  ├─ 2. fetchProducts(filters) → lib/api.ts
  │     ├─ Retry: up to 3 attempts on 500/502/503/504
  │     └─ Backoff: 250ms × attempt
  │
  ├─ 3. HTTP GET → /api/shopee-[sg|my-vvip]/products?...
  │
  ├─ 4. route.ts → createShopeeProductsGetHandler
  │     ├─ Check in-memory cache (fresh hit → return immediately)
  │     ├─ Check stale cache (stale hit → return stale, revalidate in background)
  │     └─ Cache miss → proceed to DB
  │
  ├─ 5. Repository (shopee-[sg|vvip]-products-repository.ts)
  │     ├─ Phase 1: Get sorted product names
  │     │   ├─ Try summary table (if simple filters)
  │     │   └─ Fallback: live GROUP BY
  │     ├─ Phase 2: Fetch all rows for those names
  │     └─ DB retry: 3 attempts on ETIMEDOUT/ECONNRESET
  │
  ├─ 6. enrichShopeeProductRows() → shared enrichment pipeline
  │     ├─ Parallel: costing + inventory + vouchers
  │     └─ Compute group/variation metrics
  │
  ├─ 7. Response → page.tsx
  │     ├─ deduplicateCompetitorRows()
  │     └─ Set products state
  │
  └─ 8. GroupedRows component renders 3-level hierarchy
        (Same component as Shopee MY — documented in comp 1)
```

---

## Performance Optimizations

### Problem: HTTP 500 Errors from Heavy Load

The VVIP and SG pages were returning intermittent HTTP 500 errors due to:

1. **Heavy analytics queries** overwhelming the database
2. **Costly match lookups** — variation matching scans repeated unnecessarily
3. **Duplicate tab reload requests** — multiple redundant API calls on tab switches
4. **No transient error handling** — temporary failures crashed the page
5. **First-load race conditions** — async calls silently failing
6. **No caching layer** — every page load hit the database directly

### Solutions Implemented

| Optimization | Layer | Impact |
|-------------|-------|--------|
| **In-memory stale cache** (30s fresh, 5min stale) | Server GET handler | Eliminates repeated DB queries for same data |
| **Thundering herd prevention** | Cache + Summary refresh | Concurrent requests share one DB query |
| **Pre-computed summary tables** (10min refresh) | Repository | Avoids expensive GROUP BY + SUM on every request |
| **Two-phase grouped query** | Repository | Phase 1 touches summary table; Phase 2 fetches only needed rows |
| **JOIN avoidance** | Repository | `buildMpOnlyWhere()` skips comp table JOIN when not needed |
| **`limit + 1` pattern** | Repository | Detects `hasMore` without separate COUNT query |
| **Request coalescing** | Client API | `fetchTabCounts()` deduplicates concurrent calls |
| **Retry with backoff** | Client API + Server DB | 3 attempts, 250ms × attempt on transient errors |
| **Stale response discard** | Page (request ID refs) | Prevents stale data from overwriting fresh data |
| **Parallel enrichment** | Enrichment pipeline | `Promise.all` for costing + inventory + vouchers |
| **Fire-and-forget backfill** | Enrichment | Voucher backfill doesn't block page load |
| **LRU count cache** (120s TTL) | Repository | Avoids re-running expensive COUNT queries |

---

## Database Tables

| Table | Database | Purpose |
|-------|----------|---------|
| `Shopee_My_Products` | `webapp_test` | Our products (both MY and SG, differentiated by `region` column) |
| `Shopee_Comp_Data` | `AllBots` | Competitor data (joined via `our_item_id_extracted`) |
| `Shopee_VVIP_Group_Summary` | `webapp_test` | Pre-computed VVIP group rollups (10min refresh) |
| `Shopee_SG_Group_Summary` | `webapp_test` | Pre-computed SG group rollups (10min refresh) |
| `shopee_sales` | `webapp_test` | Sales data for enrichment |

### Region Column

`Shopee_My_Products.region` defaults to `'MY'`. SG products have `region = 'SG'`. Both share the same table — the region column is the sole differentiator in WHERE and JOIN clauses.

---

## Caching Strategy

| Cache | Location | TTL | Max Size | Purpose |
|-------|----------|-----|----------|---------|
| **GET response cache** | Server (InMemoryStaleCache) | 30s fresh / 5min stale | 200 entries | Stale-while-revalidate for GET requests |
| **Summary tables** | DB (materialized) | 10 min refresh | N/A | Pre-computed group rollups |
| **Count cache** | Server (LRU Map) | 120s | 100 entries | Tab count queries |
| **Tab counts dedup** | Client (module-level promise) | Per-request | 1 | Prevents duplicate count fetches |
| **Product details** | Client (detailsCacheRef) | Per-session | Unlimited | Lazy-loaded images/descriptions |
| **Column visibility** | Client (localStorage) | Persistent | N/A | `shopeeMY:` or `shopeeSG:` prefix |
| **Column widths** | Client (localStorage) | Persistent | N/A | Same prefix |
| **Density** | Client (localStorage) | Persistent | N/A | Same prefix |

---

## Known Limitations & TODOs

| Item | Status | Notes |
|------|--------|-------|
| **SG shop filter options** | TODO | Empty array — waiting for bot to populate SG data |
| **Single-tab UI** | Intentional | Pending/Fixed/Deleted tabs coded but hidden — no status/bulk-status/permanent-delete routes exist yet |
| **spSalesValue formula** | Known issue | Uses `shopee_var_value` when > 0, falls back to `sales × price` — may differ from Shopee MY calculation |
| **SG data in DB** | Waiting | SG page shows empty until bot adds rows with `region = 'SG'` |
| **VVIP sheet name expansion** | Note | `"VVIP"` expands to `["VVIP", "VVIP_SG"]` in filter queries |
