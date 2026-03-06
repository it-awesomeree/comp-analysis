# Comp Analysis — Frontend Documentation

**Branch**: `main` (merged from `fix/dup-var-and-date-scraped-main` via PR #494)
**Date**: March 2026
**Purpose**: Comprehensive documentation of the frontend architecture, file structure, data flow, UI components, and business logic of the Shopee MY Competitive Analysis feature.

---

## Table of Contents

1. [Overview](#overview)
2. [File Structure](#file-structure)
3. [Page Architecture — Shopee MY Table](#page-architecture--shopee-my-table)
4. [Page Architecture — AI Analysis Dashboard](#page-architecture--ai-analysis-dashboard)
5. [Component Deep Dive — grouped-rows.tsx](#component-deep-dive--grouped-rowstsx)
6. [Component Deep Dive — product-row.tsx](#component-deep-dive--product-rowtsx)
7. [Component Deep Dive — table-header.tsx](#component-deep-dive--table-headertsx)
8. [Component Deep Dive — Shared Components](#component-deep-dive--shared-components)
9. [API Layer](#api-layer)
10. [Data Flow](#data-flow)
11. [Key Logic & Calculations](#key-logic--calculations)
12. [Type Definitions](#type-definitions)
13. [UI Features & Interactions](#ui-features--interactions)
14. [Product Lifecycle & State Transitions](#product-lifecycle--state-transitions)
15. [Export Functionality](#export-functionality)
16. [Database Tables](#database-tables)
17. [Caching Strategy](#caching-strategy)

---

## Overview

The Comp Analysis feature allows users to compare our Shopee products against competitor products. It displays our products grouped by product name, with variations nested underneath, and competitor data attached to each variation.

There are two main entry points:
- **Shopee MY Table** (`/analytics/table/shopee-my`) — The primary comp analysis table with full filtering, sorting, pagination, and 3-level grouped display
- **Comp Analysis Dashboard** (`/comp-analysis`) — AI-powered analysis using Google Gemini 2.0 Flash to compare products with images and structured data

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
│       ├── CompAnalysisDashboard.tsx          # AI analysis dashboard (~1428 lines)
│       ├── SharedComponents.tsx               # Reusable UI: ExpandableText, ImageCell, LinkCell, etc.
│       ├── TestDataDashboard.tsx              # Test data UI
│       └── TestDataCompDashboard.tsx          # Test data comp UI
├── lib/
│   ├── type.ts                               # Core Product interface + SimilaritySnapshot (~194 lines)
│   ├── shopee-ids.ts                         # URL ID extraction (shop_id, product_id)
│   ├── services/
│   │   ├── shopee-products-repository.ts     # Primary data access layer (~86KB)
│   │   ├── shopee-products-repository-mysql.ts # MySQL-specific queries
│   │   └── shopee-products-enrichment.ts     # Server-side enrichment (sales, profit, velocity)
│   └── shared/
│       ├── shopee-history-calculations.ts    # Grouping, sales resolution, metrics (~673 lines)
│       └── shopee-sales-values.ts            # Sales value utilities
```

---

## Page Architecture — Shopee MY Table

### File: `app/analytics/table/shopee-my/page.tsx` (~138KB)

This is the main page — a large client-side React component managing the entire comp analysis experience.

### State Variables

| State Variable | Type | Default | Purpose |
|---|---|---|---|
| `activeTab` | `TabType` | `"Shopee MY"` | Current tab: Shopee MY / Pending / Deleted / Fixed/New |
| `searchQuery` | `string` | `""` | Search input for product/SKU |
| `selectedShopId` | `string` | `"all"` | Shop filter selection |
| `selectedRemarks` | `string[]` | `[]` | Remarks filter |
| `selectedCategories` | `ShopeeMyCategoryFilter[]` | `[]` | Category filters (VVIP/VIP/LINKS_INPUT/NONE) |
| `dateRange` | `DateRange` | `{ from: undefined, to: undefined }` | Date filter |
| `salesRangePreset` | `number \| null` | `30` | SiteGiant sales window (7/14/30/60/90) |
| `shopeeSalesRangePreset` | `number \| null` | `30` | Shopee sales window |
| `metricsRangePreset` | `number \| null` | `30` | Metrics time window (7/30/90) |
| `competitorFilter` | `CompetitorFilter` | `"all"` | "all" / "with-compare" / "no-compare" |
| `customLinkInput` | `string` | `""` | Custom link filter input |
| `products` | `Product[]` | `[]` | Active "Shopee MY" tab products |
| `pendingProducts` | `Product[]` | `[]` | Pending tab products |
| `deletedProducts` | `DeletedProduct[]` | `[]` | Deleted tab products |
| `fixedProducts` | `FixedProduct[]` | `[]` | Fixed/New tab products |
| `selectedProducts` | `Set<string>` | `new Set()` | Selected product IDs for bulk ops |
| `currentPage` | `number` | `1` | Pagination current page |
| `rowsPerPage` | `number` | `20` | Rows per page (20/50/100) |
| `sort` | `SortState \| null` | `{ key: "category", dir: "desc" }` | Sort column and direction |
| `visibleCols` | `ColVisibility` | from localStorage | Column visibility toggle state |
| `colWidths` | `ColWidths` | from localStorage | Column width overrides |
| `density` | `Density` | `"normal"` from localStorage | "compact" / "normal" / "comfortable" |
| `loading` | `boolean` | `false` | Loading state |
| `error` | `string \| null` | `null` | Error message |
| `editingProduct` | `Product \| null` | `null` | Product being edited in modal |
| `showAddNewModal` | `boolean` | `false` | Add new product modal |
| `competitorSelectionProduct` | `Product \| null` | `null` | Competitor selection modal |
| `showDeleteConfirmation` | `boolean` | `false` | Delete confirmation modal |
| `showPermanentDeleteConfirmation` | `boolean` | `false` | Permanent delete modal |
| `filterReloadToken` | `number` | `0` | Triggers data reload on filter change |

### useEffect Hooks

| Purpose | Dependencies | Action |
|---------|-------------|--------|
| Reset to page 1 on sort change | `[sort]` | `setCurrentPage(1)` |
| Persist column visibility | `[visibleCols]` | Save to localStorage `"shopeeMY:columnVisibility"` |
| Persist column widths | `[colWidths]` | Save to localStorage `"shopeeMY:colWidths"` |
| Persist density | `[density]` | Save to localStorage `"shopeeMY:density"` |
| Initial load on mount | `[]` (once) | Load tab counts + Shopee MY data |
| Load tab data on change | `[activeTab, currentPage, rowsPerPage, filterReloadToken]` | Fetch data for active tab |
| Re-fetch on server sort | `[sort, activeTab]` | Skip if client-only sort |
| Validate page within range | `[activeTabTotal, rowsPerPage, currentPage]` | Reset page if beyond max |
| Reset pagination on filter | Multiple filter deps | `setCurrentPage(1)`, increment `filterReloadToken` |
| Clear selection on page change | `[currentPage, rowsPerPage, activeTab]` | `setSelectedProducts(new Set())` |

### Filter Bar UI Elements

| Filter | Icon | Options | Behavior |
|--------|------|---------|----------|
| **Search** | (input) | Free text | Searches product name, SKU, URL |
| **Sales Window** | CalendarClock | 7, 14, 30, 60, 90 days | Applies to SG + Shopee sales simultaneously |
| **Metrics Window** | BarChart3 | 7, 30, 90 days | Applies to advertising metrics |
| **Category** | Layers | VVIP, VIP, Links Input, Uncategorized | Multi-select toggle + optional custom link input |
| **Competitor** | Users | All, Yes (with-compare), No (no-compare) | Filter by competitor presence |
| **Shop** | Store | 27 hardcoded shop names | Single select dropdown |
| **Remarks** | (icon) | Dynamic from data | Multi-select |
| **Date Range** | (calendar) | From/To date picker | Filters by `date_taken` |
| **Column Manager** | (icon) | Toggle column visibility | Persisted in localStorage |
| **Download** | (icon) | CSV / XLSX | Export filtered or selected rows |

### Key Functions

| Function | Purpose |
|----------|---------|
| `handleExcludeComparison()` | POST to `/api/products/similarity-exclusion`, then `refreshAllTabs()` |
| `refreshAllTabs()` | Clears loaded tabs tracking, re-fetches counts + active tab data |
| `buildFilters()` | Constructs `FetchFilters` from all current state for API calls |
| `loadProducts()` | Fetches active Shopee MY products with `groupByName: true` |
| `loadPendingProducts()` | Fetches pending products |
| `loadFixedProducts()` | Fetches fixed/new products, transforms to `FixedProduct` |
| `loadDeletedProducts()` | Fetches deleted products |
| `handleBulkDelete()` | Soft-delete selected products with undo support |
| `handleBulkSendToFix()` | Move selected to Pending tab |
| `handleBulkRestore()` | Restore to original tab using `originalPage` |
| `handleBulkPermanentDelete()` | Hard delete (VIP/VVIP only, excludes LINKS_INPUT) |
| `handleProductCategoryUpdate()` | Updates `sourceType` across all tab arrays |
| `deduplicateCompetitorRows()` | Dedupes by (our_link, comp_link, comp_variation, our_variation), keeps latest |

---

## Page Architecture — AI Analysis Dashboard

### File: `components/comp-analysis/CompAnalysisDashboard.tsx` (~1428 lines)

AI-powered analysis using Google Gemini 2.0 Flash for similarity scoring and competitive insights.

### 4 Analysis Types

| Type | Output Format | Purpose |
|------|--------------|---------|
| `product_similarity` | JSON | 90% match confidence scoring between our product & competitor product |
| `variation_similarity` | JSON | 90% match confidence scoring between our variation & competitor variation |
| `product_competitive` | Markdown | Product-level positioning, metrics, recommendations |
| `variation_competitive` | Markdown | Variation-level gaps, mismatches, opportunities |

### How It Works

1. User selects analysis type from dropdown
2. Selects 1+ products from the paginated product table (fetched from `/api/comp-analysis/products`)
3. Clicks "Start Analysis"
4. **Similarity modes**: Batch parallel `Promise.allSettled` for all selected products. Returns JSON with `similarity_score` (0-100), `reason`, `differences_exclude` array
5. **Competitive modes**: Single product POST. Returns markdown analysis
6. Results displayed in table (similarity) or rendered markdown (competitive)
7. **Retry with exclusions**: On retry, previous "reason" fields are merged into exclusion list for the next run

### UI Layout

```
Header (title + Analysis Type dropdown + View Test Data button)
  |
Analysis Type Description (color-coded badge)
  |
Error Display (if present)
  |
Prompt Editor (textarea + Reset + char count + Retry/Start buttons)
  |
Result Panel (Similarity table OR Competitive markdown/JSON)
  |
Products Table (searchable, sortable, paginated with checkboxes)
  |
Selected Items Badge
```

---

## Component Deep Dive — grouped-rows.tsx

### File: `components/Shopee-MY-History/grouped-rows.tsx` (~2400 lines)

The core display component rendering the 3-level hierarchical table.

### Props

```typescript
{
  rows: Product[]
  visibleCols: ColVisibility
  onRowSelect: (id: string, selected: boolean) => void
  selectedIds: Set<string>
  onDelete / onRestore / onEdit / onSendToFixed / onQuickSendToFix
  onSelectCompetitors / onSaveRemarks / onSaveSku / onCategoryChange
  onRequestDetails?: (ids: string[]) => void
  isDeletedMode? / isPendingMode? / isFixedMode?
  page? / pageSize?
  parentSort?: { key, dir } | null
  onTotalGroups?: (n: number) => void
  isShopeeMy?: boolean
  colWidths: ColWidths
  showMySalesValue?: boolean
  onExcludeComparison?: (comparisonId, exclusionData) => Promise<any>
}
```

### State & Memoized Values

| Variable | Type | Purpose |
|----------|------|---------|
| `openGroups` | `Set<string>` | Level 1 MY expand state |
| `openVars` | `Set<string>` | Level 2 MY variation expand state |
| `openComp` | `Set<string>` | Level 1 COMP expand state |
| `openCompVar` | `Set<string>` | Level 2 COMP variation expand state |
| `excludeDialogOpen` | `boolean` | Exclusion dialog open state |
| `excludePending` | `boolean` | Exclusion API call in progress |
| `excludeTarget` | `object \| null` | Current exclusion target data |
| `groupedAll` | `useMemo` | All products grouped via `groupProducts()` |
| `sortedGroupedAll` | `useMemo` | Sorted groups (by parent sort) |
| `grouped` | `useMemo` | Paginated slice of sorted groups |
| `variationMeta` | `useMemo` | Pre-computed hasCompetitors map + sorted variations |

### 3-Level Hierarchy Rendering

#### Level 1: Product Group Row
- **Background**: `#FAF9F5` (off-white)
- **Columns**: Category (select dropdown), Shop (link), Product ID, Select/Expand toggle, MY Product name + images, SKU placeholder
- **Metrics (aggregated)**: Price, Profit, Profit Margin, ISR%, Stockout Date, Lost Value, SG Sales, SP Sales, Stock, Visitors, Conversion Rate, ROAS, Ads Spend
- **COMP summary**: Count badge + eye icon toggle
- **Expand behavior**: Click MY toggle opens variations (closes COMP), click COMP toggle opens competitor groups (closes MY). **Mutually exclusive** per group key.
- **Category**: Determined by `getHighestPriorityCategory()` — highest priority from all rows in group

#### Level 2a: MY Variations (when MY expanded)
- **Background**: `#F4F3EE`, **left border**: `3px solid #D97757` (orange)
- **Columns**: Category (select), Image preview, Variation badge, SKU
- **Metrics (per-variation)**: From `variationMetrics` — profit, margin, ISR%, stockout, sales, stock, ads
- **Expand behavior**: Only if `hasCompetitors` is true. Opens Level 3 competitors.
- **Sorting**: Variations with competitors appear first (pre-computed in `variationMeta`)

#### Level 2b: COMP Products (when COMP expanded)
- **Background**: `#F0ECE5`, **left border**: `3px solid #6A9BCC` (blue)
- **Columns**: Category (text label, not editable), Shop (link), COMP product name + images + description
- **Similarity display**: Score with icon (green CheckCircle2 >= 90%, red XCircle < 90%)
- **Exclusion**: Orange Filter icon if `hasProductExclusions`. Clickable XCircle opens AlertDialog.
- **Filtering**: Excludes `"(no comp name)"` entries. Top 3 competitor groups by max monthly sales.

#### Level 3: COMP Variations (under COMP product)
- **Background**: `#EAE5DC`, **left border**: `3px solid #B0AEA5` (gray)
- **Columns**: COMP variation image (side-by-side with MY), COMP price, COMP stock
- **Top 3 + ties**: Sorts by monthly sales desc, includes all tied with 3rd place, caps at 10

### Expand/Collapse Logic

```
handleToggleMy(gkey):
  Opening MY -> closes COMP + all COMP children for this group
  Closing MY -> closes all MY children for this group

handleToggleComp(gkey):
  Opening COMP -> closes MY + all MY children for this group
  Closing COMP -> closes all COMP children for this group

Children keys: "${gkey}|||${variationKey}" — cleared by prefix match
```

### variationMeta useMemo

Pre-computes per group:
1. `hasCompMap`: Map of variation key -> boolean (has competitors?)
2. `sorted`: Variations re-ordered with competitors-first priority

Dependency: `[grouped]` — only recomputes when grouped data changes.

---

## Component Deep Dive — product-row.tsx

### File: `components/Shopee-MY-History/product-row.tsx` (~1000 lines)

Individual row component used for Level 3 leaf rows and sometimes standalone rows.

### Props

```typescript
{
  product: Product
  isSelected: boolean
  onSelect: (selected: boolean) => void
  onDelete / onRestore / onEdit / onSendToFixed / onQuickSendToFix
  onSelectCompetitors / onSaveRemarks / onSaveMkRemark / onSaveSku
  onExcludeComparison?: (comparisonId, exclusionData) => Promise<void>
  onCategoryChange?: (productId, newCategory) => Promise<void>
  onRequestDetails?: (ids: string[]) => void
  visibleCols?: ColVisibility
  colWidths: ColWidths
  isShopeeMy?: boolean
  isDeletedMode? / isPendingMode? / isFixedMode? / isAmendmentMode?
  hideMyName? / hideMyLink? / hideMyVariation? / hideProductId?
  hideRowSelect? / hideMyShop? / hideCompShop? / hideSkuContent?
  suppressMyMetrics?: boolean
  showMySalesValue?: boolean
  rowBgColor?: string           // Solid background for sticky cells
}
```

### Two Layout Modes

**Shopee MY Layout** (`isShopeeMy = true`):
Split columns — MY metrics on left, COMP data on right. ~30 individual columns.

**Default Layout** (`isShopeeMy = false`):
Combined columns — MY|COMP side-by-side in grid cells. Fewer but wider columns.

### Similarity Score Display

| Condition | Display |
|-----------|---------|
| `score === null` | "-" (gray dash) |
| `score >= 90` | Green CheckCircle2 (not clickable) |
| `score < 90` + reason exists + `canExcludeComparison` | Red XCircle with hover-to-Ban (clickable, opens AlertDialog) |
| `score < 90` + no reason or no callback | Red XCircle (not clickable) |

**canExcludeComparison** requires: `onExcludeComparison` provided AND `isShopeeMy` AND score not null AND score < 90 AND reason !== "-"

### Exclusion AlertDialog

**Title**: "Add Product-Level Exclusion?" or "Add Variation-Level Exclusion?"

**Content**:
1. Info box: "What happens next" — 3-step explanation (mark excluded, next bot run ignores it, score may improve)
2. Current issue box: Shows the similarity reason in italic
3. Details: Exclusion type, Comp product, Comp variation, Comp shop

**On confirm**:
- Constructs `{ our_link, comp_link, sku, exclusion_type, reason }`
- Calls `onExcludeComparison(product.id, exclusionData)`
- Shows toast on success/error

### Exclusion Indicator

Orange Filter icon (h-3 w-3) with tooltip "Product exclusions active" / "Variation exclusions active". Shows when `hasProductExclusions` or `hasVariationExclusions` is true.

### Automated Remarks Tokens

Generated from product data comparisons:

| Token | Color | Condition |
|-------|-------|-----------|
| `OSH` (Our Sales Higher) | Green | our sales > comp sales |
| `OSLTC` (Our Sales Lower) | Red | our sales < comp sales |
| `WHMS` (We Have More Stock) | Green | our stock > comp stock |
| `OSLC` (Our Stock Lower) | Red | our stock < comp stock |
| `ORBTC` (Our Rating Better) | Green | our rating > comp rating |
| `ORLTC` (Our Rating Lower) | Red | our rating < comp rating |
| `IP` (Increase Price) | Green | our price < comp price |
| `DP` (Decrease Price) | Red | our price > comp price |
| `COOF` (Comp Out of Stock) | Green | comp stock = 0 |
| `OVOOS` (Our Variation OOS) | Red | our stock = 0 |

### Actions by Mode

| Mode | Available Actions |
|------|-------------------|
| **Normal** | Copy JSON, Send to Fix, Select Competitors, Delete |
| **Pending** | Restore, Send to Fixed, Edit, Delete |
| **Deleted** | Restore, Delete (permanent) |
| **Fixed** | Restore, Delete |

### Inline Editing

- **Remarks**: Click to edit, blur/Enter to save. Calls `onSaveRemarks(id, text)`
- **MK Remark**: Same pattern. Calls `onSaveMkRemark(id, text)`
- **SKU**: Click to edit, blur/Enter to save, Escape to cancel. Calls `onSaveSku(id, text)`

### Sticky Cells

```typescript
const cellStyle = (id: ColumnId) => {
  if (rowBgColor && STICKY_COLUMNS.includes(id)) {
    return { width: W(id), ...getStickyStyle(id, colWidths, false, rowBgColor) }
  }
  return { width: W(id) }
}
```

`rowBgColor` is passed from GroupedRows for alternating backgrounds on Level 3 rows.

---

## Component Deep Dive — table-header.tsx

### File: `components/Shopee-MY-History/table-header.tsx`

Defines the table structure, column definitions, sticky behavior, and column group configuration.

### STICKY_COLUMNS

```typescript
["category", "shop", "productId", "select", "my", "sku"]
```

First 6 columns frozen during horizontal scroll.

### DEFAULT_COL_WIDTHS (key columns)

| Column | Width (px) | Column | Width (px) |
|--------|-----------|--------|-----------|
| select | 32 | my | 208 |
| comp | 280 | sku | 224 |
| category | 160 | shop | 250 |
| productId | 112 | priceMy | 112 |
| profitMy | 112 | profitMarginMy | 150 |
| isrPercent | 112 | projectedStockoutDate | 200 |
| lostValue | 140 | salesMy | 112 |
| sgSalesValue | 140 | sgVariationPct | 165 |
| visitors | 120 | conversionRate | 160 |
| roas | 120 | adsSpend | 140 |
| similarity | 180 | similarityReason | 240 |
| priceComp | 112 | salesComp | 120 |
| compMonthlySales | 160 | stockComp | 96 |
| remarks | 240 | mkRemark | 240 |

### COLUMN_GROUPS (7 groups)

| Group ID | Label | Color (Hex) | Columns |
|----------|-------|-------------|---------|
| `productIdentity` | Product Identity | `#6B6560` Warm Charcoal | category, shop, productId, select, my, sku |
| `pricingProfit` | Pricing & Profit | `#788C5D` Claude Green | priceMy, cofundVoucher, shopeeDiscountedPrice, profitMy, profitMarginMy, isrPercent |
| `inventory` | Inventory | `#C17A3C` Warm Amber | projectedStockoutDate, lostValue, salesMy |
| `salesPerformance` | Sales & Shopee | `#6A9BCC` Claude Blue | sgSalesValue, sgVariationPct, salesMyValue, spSalesValue, spVariationPct, stockMy |
| `advertising` | Advertising | `#D97757` Claude Orange | visitors, conversionRate, roas, adsSpend |
| `competitors` | Competitors | `#9A7D6E` Warm Brown | similarity, similarityReason, dateCompared, comp, priceComp, salesComp, compMonthlySales, stockComp, date |
| `actionsNotes` | Notes & Actions | `#908E85` Mid-Gray | remarks, mkRemark, actions |

### getStickyStyle() Function

Calculates CSS `position: sticky` with accumulated `left` offsets:

```typescript
getStickyStyle(columnId, colWidths, isHeader = true, bgColor?)
```

- Loops through preceding sticky columns, sums widths + gaps (20px each)
- Returns: `position: sticky`, `left: Xpx`, `zIndex: 100 (header) / 90 (row)`
- `boxShadow` covers the gap between columns
- `clipPath: inset(0 -20px 0 0)` hides scroll-through content

### ResizableHeaderCell

Drag handle on right edge for column resizing. Min 72px, max 640px. Calls `onResize(id, width)`.

### Sales Window Dropdowns

Options: [7, 14, 30, 60, 90] days + "Clear filter". Applied to SiteGiant and Shopee sales independently.

---

## Component Deep Dive — Shared Components

### File: `components/comp-analysis/SharedComponents.tsx` (~272 lines)

| Component | Props | Purpose |
|-----------|-------|---------|
| `ExpandableText` | `{ text, limit? }` | Collapses long text (>100 chars) with Show more/less toggle |
| `ImageCell` | `{ images: string[] }` | Toggle gallery with Eye/EyeOff icon, clickable thumbnails with full-size modal |
| `LinkCell` | `{ url, label? }` | External link (target=_blank), truncated max-w-100px with tooltip |
| `SortableHeader` | `{ label, column, currentSortBy, currentSortOrder, onSort }` | Clickable `<th>` with ArrowUp/Down/UpDown indicators |
| `SearchBar` | `{ value, onChange, onSearch, placeholder?, isLoading? }` | Input + Search button + Clear button, Enter triggers search |

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
| `updateProductCategory(id, sourceType)` | PATCH | `/api/products` | Change category |
| `deleteProduct(id)` | DELETE | `/api/products?id=X` | Soft delete |
| `permanentDeleteProducts(ids)` | DELETE | `/api/products/permanent-delete` | Hard delete |
| `fetchInventoryRowsBySkuBatch(skus)` | POST | `/api/sales_inventory/batch` | Batch inventory lookup (chunked, 500 max per batch, 10s timeout) |

### `fetchTabCounts()` Deduplication

```typescript
let _countsInFlight: Promise<TabCounts> | null = null

// If a fetch is already in-flight, share the same promise
// Prevents 503s from duplicate requests during React re-mounts
// Sentinel cleared on settle (success or error) for retry
```

### Key Query Parameters for `/api/products`

| Param | Type | Description |
|-------|------|-------------|
| `limit`, `offset` | number | Pagination (default page size: 500) |
| `status` | string | active / pending / fixed / new / deleted |
| `sheetName` / `sheetNames` | string | Category filter (VVIP/VIP/LINKS_INPUT/NEW_ITEMS) |
| `uncategorized` | "1" | Uncategorized only filter |
| `customLink` | string | Filter by specific product link |
| `search` | string | Search product name, SKU, URL |
| `shopName` / `shopId` | string | Filter by shop |
| `shopSelection` | "unknown" | Products with unknown shop |
| `dateFrom`, `dateTo` | string | Date range filter (YYYY-MM-DD) |
| `remarks` / `insight` | string[] | Remark filters |
| `salesWindow` | 7/14/30/60/90 | SiteGiant sales period |
| `shopeeSalesWindow` | 7/14/30/60/90 | Shopee sales period |
| `metricsWindow` | 7/30/90 | Advertising metrics period |
| `sortKey`, `sortDir` | string | Sort column and direction |
| `groupBy` | "product" | Group by product name |
| `includeCount` | "1" | Include total count in response |
| `detailLevel` | "summary"/"full" | Response detail level |
| `debug` | "1" | Include debug info in response |

---

## Data Flow

### Main Table Data Flow

```
1. User interacts with filters/pagination/sort
       |
2. page.tsx useEffect triggers -> buildFilters() constructs FetchFilters
       |
3. fetchProducts(filters) -> GET /api/products?...
       |
4. API route (route.ts):
   - shopee-products-repository.ts builds SQL query
   - LEFT JOIN on Shopee_Comp_Similarity_Exclusions for exclusion data
   - Returns raw rows from Shopee_Comp table
       |
5. enrichShopeeProductRows() (server-side in API route):
   - Adds sales values from shopee_sales table
   - Calculates profit and profit margins
   - Computes sales velocity metrics
   - Adds group/variation metrics (groupMetrics, variationMetrics)
       |
6. Response -> page.tsx
       |
7. deduplicateCompetitorRows() removes duplicate comp rows
       |
8. groupProducts(rows) from shopee-history-calculations.ts:
   - Groups rows by product name
   - Nests variations by SKU (fallback to variation name)
   - Deduplicates variations with same SKU
   - Uses latest date_taken for product name
       |
9. GroupedRows component receives grouped data -> renders 3-level hierarchy
```

### Similarity Exclusion Flow

```
1. User clicks X/Ban icon on similarity score < 90% (in ProductRow or GroupedRows)
       |
2. AlertDialog opens with exclusion confirmation
   - Auto-filled with similarity reason
   - Shows exclusion type (product/variation)
   - Shows comp product details
       |
3. User confirms -> onExcludeComparison(comparisonId, {
     our_link, comp_link, sku, exclusion_type, reason
   })
       |
4. page.tsx handleExcludeComparison():
   - POST /api/products/similarity-exclusion
   - Validates inputs (length limits, sanitization)
   - Calls addSimilarityExclusion() in repository
       |
5. Repository:
   - INSERT/UPDATE on Shopee_Comp_Similarity_Exclusions
   - SET has_*_exclusions = 1 on matching Shopee_Comp rows
   - Invalidate count cache
       |
6. refreshAllTabs() reloads UI with updated exclusion indicators
```

### AI Analysis Flow (Comp Analysis Dashboard)

```
1. User selects analysis type + products from table
       |
2. POST /api/comp-analysis/analyze for each selected product
   Body: { product_name, product_description, sku, our_link,
           our_variation, shop_name, user_prompt, comp_link, date_taken }
       |
3. API route:
   - Queries comp_ai_test table for matching records
   - Extracts image URLs from our_main_images, our_variation_images, etc.
   - Calls Gemini 2.0 Flash API with prompt + inline images
     (max 3 main + 1 variation per product, 18MB total, 3MB per image)
       |
4. Response -> Similarity JSON or Competitive markdown
       |
5. Display: SimilarityResult table (JSON mode) or ReactMarkdown (competitive mode)
```

---

## Key Logic & Calculations

### File: `lib/shared/shopee-history-calculations.ts` (~673 lines)

### groupProducts(products)

1. Iterates all products
2. Groups by `product_name` (trimmed, or "(no name)")
3. Within each group, variations keyed by:
   - **SKU** (normalized: lowercase, trimmed, non-alphanumeric removed) when available
   - **Variation name** (normalized: lowercase, trimmed, collapsed whitespace) as fallback
4. Tracks `latestName` and `latestDate` per group — product name comes from most recently scraped row
5. Returns `ProductGroup[]` with `name`, `productId`, `first`, `variations[]`

### resolveSalesNumber(rows, picker)

- Finds the row with the latest `date_taken` (fallback: `dateScraped`)
- Applies `picker` function to extract the target value
- Returns the numeric sales value from that latest row
- **NOT max, NOT sum** — always latest only

### getHighestPriorityCategory(rows)

Priority: VVIP (4) > VIP (3) > LINKS_INPUT (2) > NEW_ITEMS (1)

If all rows are "NONE", returns "NONE". Otherwise defaults to "VIP" if no category found.

### getQualifiedCompRows(variations)

Complex multi-step algorithm:
1. Per-variation: get top 3 competitors + ties by monthly sales
2. Backfill: include all variations of surviving competitor products
3. Dedup by (shop, name, variation), keep highest monthlySales
4. Sort descending by sales
5. Cap at 10 results per variation

### computeGroupIsrPercent(variations)

ISR% = (sum monthly sales / sum current stock) * 100, deduplicated by normalized SKU.

### computeVariationAggregates(variations)

Reduces all variations to totals: sgValueTotal, spValueTotal, sgSalesTotal, spSalesTotal, adsSpendTotal, visitorsTotal, convRateSum/Count/Weighted, roasSum/Count.

### computeParentSortValue(group, key)

Computes sort value for parent group row based on sort key type. Handles: category, shop, price, profit, margin, ISR%, stockout, sales, similarity score, etc.

### Top 3 + Ties (in grouped-rows.tsx)

```
Competitor Groups (Level 2b):
  Sort by max monthly sales per group (descending)
  -> Take first 3 groups

Competitor Variations (Level 3):
  Sort by monthly sales descending
  -> Find 3rd place value
  -> Include ALL with value >= 3rd place
  -> Cap at 10 maximum
```

### Value Formatting Functions

| Function | Format | Example |
|----------|--------|---------|
| `formatPriceValue(n)` | `RM{n.toFixed(2)}` | "RM12.50" |
| `formatVisitorsValue(n)` | `n.toLocaleString()` | "1,234" |
| `formatConversionRateValue(n)` | `n.toFixed(4)` | "0.0523" |
| `formatRoasValue(n)` | `n.toFixed(4)` | "3.2100" |
| `formatAdsSpendValue(n)` | `n.toFixed(2)` | "150.00" |
| `formatProfitValue(n)` | `n.toFixed(2)` | "25.50" |
| `formatProfitMargin(n)` | `n.toFixed(2)%` | "15.30%" |
| `formatIsrPercentValue(n)` | `n.toLocaleString(...)%` | "8.50%" |
| `formatProjectedStockoutDate(n)` | `DD/MM/YY (Asia/KL)` | "15/03/26" |
| `formatLostValue(n)` | `RM n.toLocaleString(...)` | "RM 1,234.56" |

---

## Type Definitions

### File: `lib/type.ts` (~194 lines)

### Core Product Interface

```typescript
interface Product {
  id: string
  sku?: string

  ourProduct: {
    url: string
    name: string
    variation: string
    price: number
    sales: number
    rating: number
    description?: string
    priceText?: string
    mainImages?: string[]
    variationImages?: string[]
  }

  competitorProduct: {
    url: string
    name: string
    variation: string
    price: number
    sales: number
    monthlySales?: number | null
    monthlySalesDisplay?: string
    rating: number
    shopName: string
    stock?: number
    description?: string
    priceText?: string
    mainImages?: string[]
    variationImages?: string[]
  }

  ourShop: { name: string; stock: number; advantage: string; issue: string }
  soldCount: { ours: number; competitor: number }

  // Sales & metrics
  salesTier?: number
  shopeeSalesTier?: number
  shopeeValueRaw?: number | null
  adsVisitors?: number | null
  adsConversionRate?: number | null
  adsSpend?: number | null
  roas?: number | null
  isrPercent?: number | null
  stockoutProjectedDateMs?: number | null
  stockoutLostValue?: number | null
  profit?: number | null
  profitMarginPct?: number | null

  // Display
  salesValueDisplay?: string
  shopeeSalesValueDisplay?: string
  cofundVoucherLabel?: string
  shopeeDiscountedPrice?: number | null

  // Metadata
  dateTaken?: string | null
  dateScraped: string
  remarks: string
  automatedRemarks?: string[]
  status: "active" | "pending" | "fixed" | "new"
  lastPage: TabType
  originalPage?: TabType
  sourceType?: string               // VVIP, VIP, LINKS_INPUT, NEW_ITEMS
  additionalCompetitors?: CompetitorProduct[]

  // Similarity & exclusions
  productSimilarity?: SimilaritySnapshot
  variationSimilarity?: SimilaritySnapshot
  hasProductExclusions?: boolean
  hasVariationExclusions?: boolean
  productExclusionsJson?: string
  variationExclusionsJson?: string

  // Computed metrics (server-side enrichment)
  groupMetrics?: ShopeeGroupMetrics
  variationMetrics?: ShopeeVariationMetrics
  parentSortValues?: ShopeeParentSortValues
}
```

### SimilaritySnapshot

```typescript
interface SimilaritySnapshot {
  score: number | null    // 0-100
  reason: string          // Description of why products don't match
  datetime: string        // ISO 8601 timestamp
}
```

### ShopeeGroupMetrics (aggregated at product group level)

```typescript
{
  category?: string
  profit?: number | null
  profitMargin?: number | null
  isrPercent?: number | null
  stockoutProjectedDateMs?: number | null
  stockoutLostValue?: number | null
  sgSalesTotal?: number
  spSalesTotal?: number
  sgSalesValueTotal?: number
  spSalesValueTotal?: number
  adsSpendTotal?: number
  visitorsTotal?: number
  convRateAvg?: number | null
  roasAvg?: number | null
}
```

### ShopeeVariationMetrics (per-variation)

Similar to group metrics but singular values, plus display strings (`sgVariationPctDisplay`, `spVariationPctDisplay`, `salesValueDisplay`, `shopeeSalesValueDisplay`).

### Other Types

- `DeletedProduct extends Product` — adds `deletedAt`, `deletedBy?`
- `FixedProduct` — minimal: id, ourProduct URL/variation, competitorProduct URL/variation, status
- `TabType` — `"Shopee MY" | "Deleted" | "Pending" | "Fixed/New"`
- `ProductGroup` — `{ name, productId, first, variations[] }`

### shopee-ids.ts

```typescript
type ShopeeIds = { shop_id: string; product_id: string; canonical_url: string }

function extractShopeeIds(urlString: string | null): ShopeeIds | null
// Regex: /product/(\d+)/(\d+) or i.(\d+).(\d+)
// Returns canonical: https://shopee.com.my/product/{shop_id}/{product_id}
```

---

## UI Features & Interactions

### Frozen/Sticky Columns

First 6 columns frozen: Category, Shop, Product ID, Select, MY Product, SKU.

- CSS `position: sticky` with cumulative `left` offsets
- Z-index: Header (100) > Row cells (90) > Regular content (1)
- Background masking via `boxShadow` + `clipPath` to prevent scroll-through bleed
- Each level has its own background color passed to `getStickyStyle()`

### Grouped Column Headers

Two-row header: Group names (colored bar) on row 1, individual column names on row 2.

### Density Toggle

3 modes persisted in localStorage (`shopeeMY:density`):

| Mode | Padding | Font Size |
|------|---------|-----------|
| Compact (S) | 4px | 14px (0.875rem) |
| Normal (M) | 6px | 15px (0.9375rem) |
| Comfortable (L) | 10px | 16px (1rem) |

Applied via CSS custom property `--row-py`.

### Category Badge Colors

| Category | Border | Text | Background |
|----------|--------|------|------------|
| VVIP | `#D97757` | `#C15F3C` | `#FDF5F0` |
| VIP | `#6A9BCC` | `#4A7BA8` | `#F0F5FA` |
| LINKS_INPUT | `#788C5D` | `#5E7045` | `#F2F5EE` |
| NEW_ITEMS | `#B0AEA5` | `#6B6560` | `#F5F4F0` |
| NONE/Uncategorized | `#D0CEC6` | `#6B6560` | `#FAF9F5` |

### Row Color Scheme

| Level | Background | Left Border |
|-------|------------|-------------|
| Level 1 (Product Group) | `#FAF9F5` | None |
| Level 2a (MY Variation) | `#F4F3EE` | `#D97757` orange (3px) |
| Level 2b (COMP Product) | `#F0ECE5` | `#6A9BCC` blue (3px) |
| Level 3 (COMP Variation) | `#EAE5DC` | `#B0AEA5` gray (3px) |
| ProductRow (alternating) | `#FAF9F5` / `#F4F3EE` | None |

### Price Comparison Icons (Default Layout)

| Condition | Icon | Color |
|-----------|------|-------|
| MY price < COMP price | TrendingDown | Green |
| MY price > COMP price | TrendingUp | Red |
| MY price = COMP price | Minus | Gray |

### Dialogs Z-Index

All Dialog and AlertDialog overlays set to `z-[200]` to appear above sticky columns (z-100).

---

## Product Lifecycle & State Transitions

```
Shopee MY (active)
  |-> Pending (pending)         [Send to Fix]
  |-> Fixed/New (fixed)         [Mark Fixed]
  |-> Deleted (deleted)         [Soft Delete — supports undo]

Pending (pending)
  |-> Fixed/New (fixed)         [Send to Fixed]
  |-> Shopee MY (active)        [Restore]
  |-> Deleted (deleted)

Fixed/New (fixed | new)
  |-> Shopee MY (active)        [Restore]
  |-> Deleted (deleted)

Deleted (deleted)
  |-> Original tab              [Restore via originalPage field]
  |-> Permanent delete          [Irreversible — VIP/VVIP only]
```

**Undo**: Single/bulk delete offers 1-time undo via `undoAction` callback stored in state.

---

## Export Functionality

### Two Export Modes

**Simple Mode**: Respects column visibility. Headers match visible columns in left-to-right order.

**Arranged Mode**: Deterministic DB-style column order. Auto-generates "Auto Remark" column from stock/sales/rating/price comparisons. Prunes empty columns. Exports user remarks + automated remarks separately.

### Export Scope

- `"filtered"`: All rows matching current filters (ignores pagination — fetches all pages)
- `"selected"`: Only checked rows

### File Formats

- **CSV**: Proper quote escaping via `csvQuote()`
- **XLSX**: Limited to 50,000 rows (falls back to CSV if exceeded)
- **Filename**: `shopee_[TAB]_[SCOPE]_[DATE].csv|xlsx`

---

## Database Tables

| Table | Purpose |
|-------|---------|
| `Shopee_Comp` | Main table — our products + competitor data, similarity scores, status, exclusion flags (`has_product_exclusions`, `has_variation_exclusions`) |
| `Shopee_Comp_Similarity_Exclusions` | Excluded comparisons. Composite key: (our_link + comp_link + sku + exclusion_type). Stores `excluded_differences` JSON array. Max 50 reasons per record. |
| `Shopee_Comp_Group_Summary` | Cached analytics per product/variation (10-min refresh TTL) |
| `shopee_sales` | Sales data used for enrichment (SiteGiant + Shopee sales) |
| `comp_ai_test` | Used by Comp Analysis Dashboard for Gemini analysis |

---

## Caching Strategy

| Cache | Location | TTL | Purpose |
|-------|----------|-----|---------|
| Count cache | Server (repository) | 2 min | Total row counts per tab, invalidated on mutations |
| Summary refresh | Server (repository) | 10 min | Pre-computed group metrics in `Shopee_Comp_Group_Summary` |
| Details cache | Client (page.tsx ref) | Per-session | Product descriptions/images fetched on-demand, cached in `detailsCacheRef` |
| In-flight dedup | Client (api.ts) | Per-request | Prevents duplicate `fetchTabCounts` calls during re-mounts |
| Column visibility | Client (localStorage) | Persistent | `shopeeMY:columnVisibility` |
| Column widths | Client (localStorage) | Persistent | `shopeeMY:colWidths` |
| Density setting | Client (localStorage) | Persistent | `shopeeMY:density` |
