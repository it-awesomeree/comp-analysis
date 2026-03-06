# Comp Analysis — Frontend Documentation

**Branch**: `main` (merged from `fix/dup-var-and-date-scraped-main` via PR #494)
**Date**: March 2026
**Purpose**: Document the frontend architecture, file structure, data flow, and key logic of the Shopee MY Competitive Analysis feature.

---

## Table of Contents

1. [Overview](#overview)
2. [File Structure](#file-structure)
3. [Page Architecture](#page-architecture)
4. [Component Hierarchy](#component-hierarchy)
5. [API Layer](#api-layer)
6. [Data Flow](#data-flow)
7. [Key Logic & Calculations](#key-logic--calculations)
8. [Type Definitions](#type-definitions)
9. [UI Features](#ui-features)
10. [Database Tables](#database-tables)
11. [Caching Strategy](#caching-strategy)

---

## Overview

The Comp Analysis feature allows users to compare our Shopee products against competitor products. It displays our products grouped by product name, with variations nested underneath, and competitor data attached to each variation.

There are two main entry points:
- **Shopee MY Table** (`/analytics/table/shopee-my`) — The primary comp analysis table with full filtering, sorting, pagination, and grouped display
- **Comp Analysis Dashboard** (`/comp-analysis`) — AI-powered analysis using Google Gemini to compare products with images and structured data

---

## File Structure

```
AWESOMEREE-WEB-APP/
├── app/
│   ├── analytics/table/shopee-my/
│   │   ├── page.tsx                          # Main Shopee MY page (~138KB) — state, filters, data fetching
│   │   └── lib/
│   │       ├── api.ts                        # Frontend API client — fetchProducts(), fetchTabCounts(), etc.
│   │       ├── shopee-types.ts               # Page-specific type aliases
│   │       └── utils.ts                      # cn() utility (clsx + tailwind-merge)
│   ├── comp-analysis/
│   │   ├── page.tsx                          # Comp Analysis AI Dashboard entry
│   │   ├── test-data/page.tsx                # Test data viewer
│   │   └── test-data-comp/page.tsx           # Test data comp viewer
│   └── api/
│       ├── products/
│       │   ├── route.ts                      # Main products API (GET/POST/PUT/PATCH/DELETE)
│       │   ├── similarity-exclusion/route.ts # Similarity exclusion CRUD (POST/GET/DELETE)
│       │   ├── counts/route.ts               # Tab count API
│       │   ├── bulk-status/route.ts          # Bulk status update
│       │   └── permanent-delete/route.ts     # Permanent delete
│       └── comp-analysis/
│           ├── analyze/route.ts              # Gemini AI analysis endpoint
│           ├── products/route.ts             # Comp analysis product list
│           ├── test-data/route.ts            # Test data API
│           └── test-data-comp/route.ts       # Test data comp API
├── components/
│   ├── Shopee-MY-History/
│   │   ├── grouped-rows.tsx                  # Hierarchical grouped view (~2400 lines)
│   │   ├── product-row.tsx                   # Individual product row (~1000 lines)
│   │   └── table-header.tsx                  # Column definitions, sticky config, column groups
│   └── comp-analysis/
│       ├── CompAnalysisDashboard.tsx          # AI analysis dashboard (~69KB)
│       ├── SharedComponents.tsx               # Reusable UI: ExpandableText, ImageCell, LinkCell, etc.
│       ├── TestDataDashboard.tsx              # Test data UI
│       └── TestDataCompDashboard.tsx          # Test data comp UI
├── lib/
│   ├── type.ts                               # Core Product interface + SimilaritySnapshot
│   ├── shopee-ids.ts                         # URL ID extraction (shop_id, product_id)
│   ├── services/
│   │   ├── shopee-products-repository.ts     # Primary data access layer (~86KB)
│   │   ├── shopee-products-repository-mysql.ts # MySQL-specific queries
│   │   └── shopee-products-enrichment.ts     # Server-side enrichment (sales, profit, velocity)
│   └── shared/
│       ├── shopee-history-calculations.ts    # Grouping, sales resolution, metrics (~24KB)
│       └── shopee-sales-values.ts            # Sales value utilities
```

---

## Page Architecture

### Shopee MY Table (`page.tsx` — ~138KB)

This is the main page. It's a large client-side component that manages:

**State Management (React useState/useEffect):**
- Product data rows + pagination (limit, offset, total)
- Active tab: `active` | `pending` | `deleted` | `fixed-new`
- Filters: search, category (VVIP/VIP/LINKS_INPUT/NEW_ITEMS), shop, date range, remarks, sales window
- Sort: single column sort with `sortKey` + `sortDir`
- UI state: density mode, column visibility, open/expanded groups, selected rows
- Modals: edit product, add product, competitor selection

**Data Fetching Flow:**
```
page.tsx useEffect (on filter/tab/sort change)
  -> fetchProducts() from lib/api.ts
    -> GET /api/products?limit=X&offset=Y&status=...&search=...
      -> shopee-products-repository.ts SQL query
      -> enrichShopeeProductRows() adds sales/profit/velocity
    -> Returns { rows, total, groupTotal }
  -> groupProducts(rows) from shopee-history-calculations.ts
  -> setState with grouped data
  -> Render GroupedRows component
```

**Key Functions in page.tsx:**
- `handleExcludeComparison()` — POST to `/api/products/similarity-exclusion` to exclude a comparison
- `refreshAllTabs()` — Refetch data after mutations
- `enrichShopeeProductRows()` — Called server-side in the API route, enriches raw DB rows with calculated fields

---

## Component Hierarchy

```
page.tsx (Shopee MY)
|-- Filter Bar (search, category, shop, date range, sales window, density toggle)
|-- Tab Bar (Active | Pending | Deleted | Fixed/New) with counts
|-- Pagination Controls (top)
|-- Table Header (table-header.tsx)
|   |-- Column Group Headers (color-coded: Product Identity, Pricing, Inventory, etc.)
|   |-- Individual Column Headers (sortable)
|-- GroupedRows (grouped-rows.tsx)
|   |-- For each ProductGroup:
|       |-- Level 1: Product Group Row (product name, aggregated metrics)
|           |-- For each Variation:
|               |-- Level 2: Variation Row (SKU, our price, our sales, etc.)
|                   |-- For each Competitor (top 3 + ties):
|                       |-- Level 3: ProductRow (product-row.tsx)
|                           |-- Competitor details (name, price, sales, similarity score)
|-- Pagination Controls (bottom)
```

### grouped-rows.tsx (~2400 lines)

This is the core display component. Key responsibilities:

- **3-level hierarchy**: Product Group -> Variation -> Competitor
- **Expand/collapse** at each level (tracked by `openGroups` and `openVars` Sets)
- **Competitor grouping**: Groups competitor rows by comp product name, then shows variations under each
- **Top 3 + ties**: Sorts competitors by monthly sales descending, shows top 3 but includes ties at the 3rd position
- **Sticky columns**: First 6 columns (Category, Shop, Product ID, Select, MY Product, SKU) are frozen during horizontal scroll
- **Similarity indicators**: Shows score with color coding (green >= 90%, red < 90%)
- **Exclusion UI**: Orange filter icon when a comparison has been excluded
- **Category badge**: Displays product category (VVIP/VIP/LINKS_INPUT/NEW_ITEMS) using server-side priority

### product-row.tsx (~1000 lines)

Individual Level 3 competitor row. Key responsibilities:

- Displays competitor product name, price, sales, monthly sales, stock, rating
- Similarity score visualization (checkmark/X icons)
- Exclusion dialog: AlertDialog that confirms excluding a comparison with auto-filled reason
- Row selection (checkbox)
- Edit/delete actions
- Sticky cell positioning matching grouped-rows

### table-header.tsx

Defines the table structure:

- `COLUMNS` array — all column definitions with IDs, labels, widths
- `COLUMN_GROUPS` — 7 color-coded groups (Product Identity, Pricing & Profit, Inventory, Sales & Shopee, Advertising, Competitors, Actions & Notes)
- `STICKY_COLUMNS` — first 6 columns that stay frozen
- `getStickyStyle()` — calculates CSS `position: sticky` with correct `left` offsets
- `DEFAULT_COL_WIDTHS` — default pixel widths per column

---

## API Layer

### Frontend Client (`lib/api.ts`)

| Function | Method | Endpoint | Purpose |
|----------|--------|----------|---------|
| `fetchProducts(filters)` | GET | `/api/products` | Main data fetch with all filters |
| `fetchTabCounts()` | GET | `/api/products/counts` | Tab badge counts (deduplicated in-flight) |
| `fetchAllProducts(filters)` | GET | `/api/products` (paginated) | Fetch all pages sequentially |
| `updateProductStatus(id, status)` | PUT | `/api/products/status` | Change product status |
| `bulkUpdateProductStatus(ids, status)` | PUT | `/api/products/bulk-status` | Bulk status change |
| `createProduct(data)` | POST | `/api/products` | Create new product |
| `updateProduct(data)` | PUT | `/api/products` | Update product |
| `deleteProduct(id)` | DELETE | `/api/products?id=X` | Soft delete |
| `permanentDeleteProducts(ids)` | DELETE | `/api/products/permanent-delete` | Hard delete |
| `fetchInventoryRowsBySkuBatch(skus)` | POST | `/api/sales_inventory/batch` | Batch inventory lookup |

### Key Query Parameters for `/api/products`

| Param | Type | Description |
|-------|------|-------------|
| `limit`, `offset` | number | Pagination |
| `status` | string | active/pending/fixed/new/deleted |
| `sheetName` / `sheetNames` | string | Category filter (VVIP/VIP/LINKS_INPUT/NEW_ITEMS) |
| `search` | string | Search product name, SKU, URL |
| `shopName` | string | Filter by shop |
| `dateFrom`, `dateTo` | string | Date range filter |
| `salesWindow` | 7/14/30/60/90 | Sales period window |
| `sortKey`, `sortDir` | string | Sort column and direction |
| `groupBy` | "product" | Group by product name |
| `includeCount` | "1" | Include total count |
| `detailLevel` | "summary"/"full" | Response detail level |

---

## Data Flow

### Main Table Data Flow

```
1. User interacts with filters/pagination
       |
2. page.tsx triggers fetchProducts() with filter params
       |
3. GET /api/products -> shopee-products-repository.ts
   - Builds SQL query with JOINs and WHERE clauses
   - LEFT JOIN on Shopee_Comp_Similarity_Exclusions for exclusion data
   - Returns raw rows from Shopee_Comp table
       |
4. enrichShopeeProductRows() (server-side)
   - Adds sales values from shopee_sales table
   - Calculates profit and profit margins
   - Computes sales velocity metrics
   - Adds group/variation metrics
       |
5. Response -> page.tsx
       |
6. groupProducts() from shopee-history-calculations.ts
   - Groups rows by product name
   - Nests variations by SKU (fallback to variation name)
   - Deduplicates variations with same SKU
   - Uses latest date_taken for product name
       |
7. GroupedRows component renders 3-level hierarchy
```

### Similarity Exclusion Flow

```
1. User clicks X icon on a similarity score < 90% in ProductRow
       |
2. AlertDialog shows exclusion confirmation
   - Auto-filled with similarity reason
   - Shows exclusion type (product/variation)
       |
3. onExcludeComparison(our_link, comp_link, sku, type, reason)
       |
4. POST /api/products/similarity-exclusion
   - Validates inputs (length limits, sanitization)
   - Calls addSimilarityExclusion() in repository
       |
5. Repository:
   - INSERT/UPDATE on Shopee_Comp_Similarity_Exclusions
   - SET has_*_exclusions = 1 on matching Shopee_Comp rows
   - Invalidate count cache
       |
6. refreshAllTabs() reloads UI with updated indicators
```

---

## Key Logic & Calculations

### Product Grouping (`shopee-history-calculations.ts`)

**`groupProducts(products)`**
- Groups by `product_name` (not product ID)
- Within each group, variations are keyed by **SKU** when available, falling back to normalized variation name — this prevents duplicate variations when SKU is the same
- Tracks `latestName` and `latestDate` — the displayed product name comes from the most recently scraped row
- Returns `ProductGroup[]` with nested `variations[]`, each containing `rows[]`

**`resolveSalesNumber(rows, picker)`**
- Finds the row with the latest `date_taken` (fallback: `dateScraped`)
- Returns the sales value from that row (NOT max, NOT sum — latest only)
- Used for all sales-related columns

**`getHighestPriorityCategory(rows)`**
- Priority: VVIP (4) > VIP (3) > LINKS_INPUT (2) > NEW_ITEMS (1)
- Used for the category badge display when groups span multiple pages

### Top 3 Competitors + Ties (`grouped-rows.tsx`)

```
Sort competitors by COMP Monthly Sales (descending)
-> Find value of 3rd-place competitor
-> Include ALL competitors with value >= 3rd place
-> This means 4+ competitors can show if they tie with 3rd
```

### Competitor Filtering

- Competitors with name `"(no comp name)"` are filtered out from display
- Competitors with monthly sales = 0 ARE shown (previously were hidden)
- Only competitors from the **latest scrape date** per product are shown

### Variation Sorting in UI

Variations within a group are sorted so that **variations with competitors appear first**, followed by variations without competitors. This is computed via `variationMeta` useMemo for performance.

---

## Type Definitions

### Core Product Interface (`lib/type.ts`)

Key fields relevant to comp analysis:

```typescript
interface Product {
  // Our product
  ourProduct: {
    name: string
    url: string
    variation: string
    price: number
    sales: number
    stock: number
    rating: number
  }

  // Competitor product
  competitorProduct: {
    name: string
    url: string
    variation: string
    price: number
    sales: number
    monthlySales: number
    stock: number
    rating: number
    shopName: string
  }

  // Similarity scoring (filled by bot)
  productSimilarity?: SimilaritySnapshot
  variationSimilarity?: SimilaritySnapshot

  // Exclusion flags (stored in DB)
  hasProductExclusions?: boolean
  hasVariationExclusions?: boolean
  productExclusionsJson?: string
  variationExclusionsJson?: string

  // Metadata
  sku: string
  status: string
  sourceType: string          // VVIP, VIP, LINKS_INPUT, NEW_ITEMS
  date_taken: string
  dateScraped: string
  automatedRemarks: string
  userRemarks: string
}

interface SimilaritySnapshot {
  score: number | null         // 0-100
  reason: string               // Why they don't match
  datetime: string             // ISO timestamp
}
```

### ProductGroup (from groupProducts)

```typescript
interface ProductGroup {
  name: string                 // Product name (latest scraped)
  productId: string | null     // Shopee product ID from URL
  first: Product               // First row in group
  variations: {
    variation: string          // Variation name or SKU
    rows: Product[]            // All rows for this variation (incl. competitors)
  }[]
}
```

---

## UI Features

### Frozen/Sticky Columns
First 6 columns stay visible during horizontal scroll: Category, Shop, Product ID, Select, MY Product, SKU. Uses CSS `position: sticky` with calculated `left` offsets and z-index layering (header: 100, rows: 90).

### Grouped Column Headers
7 color-coded groups with Muji-inspired warm palette:

| Group | Color | Columns |
|-------|-------|---------|
| Product Identity | Warm Charcoal (#6B6560) | Category, Shop, Product ID, Select, MY Product, SKU |
| Pricing & Profit | Claude Green (#788C5D) | MY Price, Profit, Profit Margin %, ISR % |
| Inventory | Warm Amber (#C17A3C) | Projected Stockout, Lost Value, SiteGiant Sales |
| Sales & Shopee | Claude Blue (#6A9BCC) | SG Sales Value, SG Variation %, Shopee Sales, SP Sales Value, SP Variation %, MY Stock |
| Advertising | Claude Orange (#D97757) | Visitors, Conversion Rate, ROAS, Ads Spend |
| Competitors | Warm Brown (#9A7D6E) | Similarity, Reason, Date Compared, COMP Product/Price/Sales/Stock, Date Scraped |
| Actions & Notes | Mid-Gray (#908E85) | Bot Remarks, MK Remark, Actions |

### Density Toggle
3 modes persisted in localStorage:
- **Compact (S)**: 4px padding, 14px font
- **Normal (M)**: 6px padding, 15px font (default)
- **Comfortable (L)**: 10px padding, 16px font

### Similarity Scoring Display
- Score >= 90%: Green checkmark
- Score < 90%: Red X icon (clickable to exclude)
- Orange filter icon: Comparison has been excluded
- Clicking exclusion opens AlertDialog with reason pre-filled

### Tab System
4 tabs with live counts: Active, Pending, Deleted, Fixed/New. Counts fetched via single optimized API call (`fetchTabCounts`) with in-flight deduplication.

### Pagination
Server-side pagination with configurable page size (default 500). Category badge fix ensures groups split across pages show the correct badge via SQL window function `group_category_priority`.

---

## Database Tables

| Table | Purpose |
|-------|---------|
| `Shopee_Comp` | Main table — our products + competitor data, similarity scores, status |
| `Shopee_Comp_Similarity_Exclusions` | Excluded comparisons (our_link + comp_link + sku + type = composite key) |
| `Shopee_Comp_Group_Summary` | Cached analytics per product/variation (10-min refresh TTL) |
| `shopee_sales` | Sales data used for enrichment |

---

## Caching Strategy

| Cache | TTL | Purpose |
|-------|-----|---------|
| Count cache (server) | 2 min | Total row counts per tab |
| Summary refresh (server) | 10 min | Pre-computed group metrics |
| In-flight dedup (client) | Per-request | Prevents duplicate fetchTabCounts calls |
| Density setting (client) | Persistent | localStorage `shopeeMY:density` |
| Column visibility (client) | Persistent | localStorage column toggle state |
