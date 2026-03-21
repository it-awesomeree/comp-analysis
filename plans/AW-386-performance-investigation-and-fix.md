# AW-386 Comp Analysis SG & VVIP — Performance Investigation + Fix Plan

**Date**: 2026-03-21
**Ticket**: AW-386 (Comp. Analysis SG epic — AW-333)
**Assignee**: Melinda
**Status**: PR #689 merged to production — 500 errors fixed, but pages still slow (20-30s load)

---

## Context

Melinda's PR #689 fixed the intermittent HTTP 500 errors on the Shopee SG and VVIP Competitive Analysis pages. The fix added:
- In-memory stale cache with request coalescing
- Frontend retry logic for transient 500/503 errors
- First-load race condition fix
- Tab reload deduplication
- Shared route extraction
- Summary table for grouped sort optimization
- Parallelized enrichment + variation match fetches

**Result**: Pages no longer crash — but they take 20-30+ seconds to load data. The 500 errors are gone, but the performance is unacceptable.

---

## The Problem (Plain English)

Imagine you ask a library to find you the **3 best-selling books** by a specific author. Instead of looking up the 3 best sellers directly, the library:

1. Pulls out **every book ever written** by that author — including every edition, every reprint, and every translation from the last 3 months
2. Stacks all 120,000 books on your desk
3. Then sorts through them to hand you the 3 you actually want

That's what's happening on the Comp Analysis SG and VVIP pages right now:

- You ask for **20 products** with their **top 3 competitors**
- The database fetches **120,000 rows** (every competitor, every variation, every historical date)
- The server throws away 99% and gives you the ~1,000 rows you actually need
- This takes **20-30 seconds**

---

## Root Cause Analysis (Verified)

### Why the database fetches too much

Two factors multiply together:

**1. Old data included (3x multiplier)**
Competitor data is scraped every ~7 days. There are 3 dates of historical data (Mar 6, 13, 20). The page only shows the latest date, but the database fetches ALL 3 dates — tripling the data.

**2. Every combination calculated (cartesian product)**
The database JOIN matches at the product level (item_id), not the variation level. So a product with 84 colour/size options and 540 competitor entries generates 84 × 540 = 45,360 rows for just ONE product. The server then picks the best 3 competitors and throws away the rest.

### Measured Data

| Metric | SG | VVIP |
|---|---|---|
| Our products (active) | 1,010 | 1,829 |
| Comp data rows | 5,527 | similar scale |
| Rows fetched for 20-product page | **119,432** | similar scale |
| Rows actually needed | ~1,000 | ~1,000 |
| Historical dates (no filter) | 3 dates | 3 dates |
| Worst single product | 84 variations × 540 comp = 45,360 rows | 54 variations × 211 comp = 11,394 rows |

### Where in the code

| File | Function | What it does |
|---|---|---|
| `lib/services/shopee-sg-products-repository.ts` | `fetchSgProducts()` grouped mode, Step 2 | The main data query — LEFT JOINs all comp data with no date filter |
| `lib/services/shopee-vvip-products-repository.ts` | `fetchVvipProducts()` grouped mode, Step 2 | Same as SG |
| `app/api/shopee-products/shared-products-route.ts` | `limitAndDedupCompetitors()` | Receives 120K rows, filters down to top 3 competitors per product |
| `lib/services/shopee-products-enrichment.ts` | `enrichShopeeProductRows()` | Processes ALL 120K rows before dedup happens |

### The JOIN that causes the explosion

```sql
LEFT JOIN Shopee_Comp_Data cd 
  ON cd.our_item_id_extracted = mp.our_item_id 
  AND cd.region = 'SG'
-- No date_taken filter — ALL historical dates included
-- No per-product row limit — ALL comp rows for every variation
```

---

## Fix Plan — Two Phases

### Phase 1: Only fetch the latest date (Quick Win)

**What changes:** Before fetching competitor data, ask the database "what's the most recent scrape date?" Then only fetch competitor data from that date.

**Impact:** 120,000 rows → ~40,000 rows. Page load drops from ~25s to ~8-10s.

**Risk:** Very low — we're filtering out old data the page was already ignoring. 2-line change per file.

**Files changed:** 2 files
- `lib/services/shopee-sg-products-repository.ts` — add date pre-query + WHERE filter in `fetchSgProducts()`
- `lib/services/shopee-vvip-products-repository.ts` — add date pre-query + WHERE filter in `fetchVvipProducts()`

**How it works:**
1. New pre-query: `SELECT MAX(date_taken) FROM Shopee_Comp_Data WHERE region = ?` (~16ms)
2. Add to WHERE clause: `AND (cd.date_taken IS NULL OR cd.date_taken = ?)`
3. `IS NULL` preserves products that have no competitor data yet

**Safety:** Only applies when user has NOT set a custom date range. If user filters by date, existing behavior is preserved. The enrichment layer's `deduplicateCompetitorRows()` already keeps only the latest date — this just does the same filtering earlier at the database level.

### Phase 2: Fetch products and competitors separately (Full Fix)

**What changes:** Instead of asking the database to combine products + competitors in one giant query, fetch them in two smaller queries and combine in the server.

**Impact:** 40,000 rows → ~1,200 rows total. Page load drops to ~1-2s.

**Risk:** Medium — requires restructuring how data flows through 4 files. Should be a separate ticket.

**Files changed:** 4 files
- `lib/services/shopee-sg-products-repository.ts` — split Step 2 into two queries
- `lib/services/shopee-vvip-products-repository.ts` — same
- `app/api/shopee-products/shared-products-route.ts` — modify `limitAndDedupCompetitors()` to work with separate data
- `lib/services/shopee-products-enrichment.ts` — simplify dedup since date is pre-filtered

**How it works:**
1. Query A: Fetch our products only (no comp JOIN) — ~572 rows
2. Query B: Fetch comp data separately for those item_ids, latest date only — ~616 rows
3. Merge in server code using a lookup map instead of SQL cartesian product

---

## Staging Test Plan

1. Apply Phase 1 fix to a feature branch
2. PR to `test` → auto-deploys to staging
3. Agnes loads both SG and VVIP pages on staging and checks:
   - Does the data still look correct? (same products, same competitors, same prices)
   - Is it noticeably faster?
   - Does tab switching still work?
4. If good → proceed to production
5. If bad → close the PR, staging reverts to previous code, nothing changed

---

## Recommendation

Start with **Phase 1 only**. It's safe, small, and gives immediate relief. If 8-10 seconds is still too slow, Phase 2 can follow as a separate ticket.

---

## Database Reference

### Table sizes (as of 2026-03-21)

| Table | Rows | Data MB | Index MB |
|---|---|---|---|
| Shopee_Variation_Match | 135,563 | 77.61 | 232.41 |
| Shopee_Comp_Data | 6,770 | 38.58 | 25.47 |
| Shopee_My_Products | 2,070 | 12.52 | 3.63 |
| Shopee_SG_Group_Summary | 48 | 0.06 | 0.02 |
| Shopee_VVIP_Group_Summary | 0 | 0.02 | 0.02 |

### Relevant indexes

**Shopee_My_Products:**
- `idx_region_status_product_name` (region, status, product_name) — used by grouped names query
- `idx_region_status_item` (region, status, our_item_id) — used by data fetch

**Shopee_Comp_Data:**
- `idx_region_item_date` (region, our_item_id_extracted, date_taken) — used by JOIN

### Comp data date distribution (SG)

| Date | Description |
|---|---|
| 2026-03-06 | Oldest scrape |
| 2026-03-13 | Middle scrape |
| 2026-03-20 | Latest scrape (only one needed) |
