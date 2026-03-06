# Shopee Comp Analysis — Database Documentation

> **Pipeline**: Shopee Competitive Analysis (7-script pipeline on VM3)
> **Last updated**: 2026-03-06
> **Database**: `AllBots` (primary), `requestDatabase` (auth tokens)

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [Pipeline Architecture](#2-pipeline-architecture)
3. [Tables Overview](#3-tables-overview)
4. [Table #1: AllBots.Shopee_Comp (Main Table)](#4-table-1-allbotsshopee_comp-main-table)
   - [High-Level Statistics](#4a-high-level-statistics)
   - [Marketplace Distribution](#4b-marketplace-distribution)
   - [Sheet Distribution](#4c-sheet-distribution)
   - [Competitive Averages](#4d-competitive-averages)
   - [Full Column Breakdown (91 columns)](#4e-full-column-breakdown-91-columns)
   - [Indexes](#4f-indexes-15-indexes)
5. [Table #2: requestDatabase.ShopeeTokens](#5-table-2-requestdaborashopeetokens)
6. [Table #3: AllBots.Shopee_VariationSalesDaily](#6-table-3-allbotsshopee_variationsalesdaily)
7. [Table #4: AllBots.Shopee_VariationSalesSyncState](#7-table-4-allbotsshopee_variationsalessyncstate)
8. [Table #5: AllBots.Shopee_Comp_Similarity_Exclusions](#8-table-5-allbotsshopee_comp_similarity_exclusions)
9. [Temporary Table: tmp_sku_sales](#9-temporary-table-tmp_sku_sales)
10. [External APIs](#10-external-apis-used-by-the-pipeline)
11. [Event Schedulers](#11-event-schedulers)
12. [Triggers, Stored Procedures, Views & Foreign Keys](#12-triggers-stored-procedures-views--foreign-keys)
13. [Data Flow Diagram](#13-data-flow-diagram)
14. [Per-Script Summary](#14-per-script-summary)
15. [Key Observations](#15-key-observations)

---

## 1. Purpose

This is the **Shopee Competitive Analysis** pipeline — a 7-script automated system running nightly on VM3. It maintains side-by-side comparisons of **your products ("our")** vs **competitor products ("comp")** on Shopee marketplace. Each row in the main table represents a **variation-level pairing**: one of your product variations matched against a competitor's product/variation.

The pipeline populates and enriches the data through sequential stages: seeding products, resolving variations, AI matching, syncing sales from multiple sources, scraping ads metrics, and scoring similarity.

---

## 2. Pipeline Architecture

```
VM3 (DESKTOP-6C5NRUO) — Runs daily at 4:00 PM MYT
Scheduler: Smart Scheduler v2.1 (sequential chain, 5-min gap between jobs)

┌──────────────────────────────────────────────────────────────────────┐
│  Script #1: ca_shopee_listing_to_db.py                               │
│  SEED — Creates new product rows from Shopee listings                │
│  Tables: Shopee_Comp (W), ShopeeTokens (R/W)                        │
│  API: Shopee Partner API                                             │
├──────────────────────────────────────────────────────────────────────┤
│  Script #2: our_variation_preprocessing.py                           │
│  Backfills our_variation column before AI matching                   │
│  Tables: Shopee_Comp (R/W), ShopeeTokens (R/W)                      │
│  API: Shopee Partner API                                             │
├──────────────────────────────────────────────────────────────────────┤
│  Script #3: ca_ai_variation_match.py                                 │
│  Resolves IDs, cleans data, AI variation matching                    │
│  Tables: Shopee_Comp (R/W), ShopeeTokens (R/W)                      │
│  API: Shopee Partner API                                             │
├──────────────────────────────────────────────────────────────────────┤
│  Script #4: shopee_comp_shopee_sales.py                              │
│  Shopee order sales sync (needs resolved IDs from #3)                │
│  Tables: Shopee_Comp (R/W), ShopeeTokens (R/W),                     │
│          Shopee_VariationSalesDaily (R/W),                           │
│          Shopee_VariationSalesSyncState (R/W)                        │
│  API: Shopee Partner API                                             │
├──────────────────────────────────────────────────────────────────────┤
│  Script #5: Shopee-mylinks-sales-data-merged.py                      │
│  Product info + SiteGiant sales data                                 │
│  Tables: Shopee_Comp (R/W), ShopeeTokens (R/W), tmp_sku_sales (temp)│
│  API: Shopee Partner API, SiteGiant API                              │
├──────────────────────────────────────────────────────────────────────┤
│  Script #6: ca_shopee_ads_metrics.py                                 │
│  Ads metrics scraping via Chrome (needs shop_id/item_id from #3/#5)  │
│  Tables: Shopee_Comp (R/W)                                           │
│  API: Shopee Seller Centre (via Chrome/Selenium)                     │
├──────────────────────────────────────────────────────────────────────┤
│  Script #7: ca_similarity_check.py                                   │
│  AI similarity check (needs images/descriptions from #5)             │
│  Tables: Shopee_Comp (R/W), Shopee_Comp_Similarity_Exclusions (R)    │
│  API: Gemini AI API                                                  │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 3. Tables Overview

The pipeline touches exactly **5 database tables** across 2 databases, plus **1 temporary table**:

| # | Table | Database | Rows | Scripts | Purpose |
|---|---|---|---|---|---|
| 1 | `Shopee_Comp` | AllBots | 56,983 | All 7 | Main competitive analysis table |
| 2 | `ShopeeTokens` | requestDatabase | 34 | #1–#5 | Shopee API OAuth tokens |
| 3 | `Shopee_VariationSalesDaily` | AllBots | 73,996 | #4 | Daily variation-level sales snapshots |
| 4 | `Shopee_VariationSalesSyncState` | AllBots | 28 | #4 | Per-shop sync cursor/bookmark |
| 5 | `Shopee_Comp_Similarity_Exclusions` | AllBots | 0 | #7 | AI similarity exclusion overrides |
| — | `tmp_sku_sales` | AllBots | temp | #5 | Ephemeral table for batch SKU sales JOINs |

---

## 4. Table #1: `AllBots.Shopee_Comp` (Main Table)

### 4a. High-Level Statistics

| Metric | Value |
|---|---|
| **Total rows** | **56,983** |
| **Total columns** | **91** |
| **Date range** | Feb 2, 2026 → Mar 6, 2026 (~1 month of data) |
| **Distinct shops (ours)** | 28 |
| **Distinct SKUs** | 15,149 |
| **Distinct categories** | 1 |
| **Distinct sheets** | 5 |
| **All rows status** | 100% `active` |
| **Rows with our product data** | 56,482 (99.1%) |
| **Rows with competitor data** | 23,238 (40.8%) |
| **Rows with AI similarity scores** | 20,624 (36.2%) |

### 4b. Marketplace Distribution

| Marketplace | Rows | % |
|---|---|---|
| Shopee MY (Malaysia) | 56,509 | 99.2% |
| Shopee SG (Singapore) | 474 | 0.8% |

### 4c. Sheet Distribution

| Sheet | Rows | Purpose |
|---|---|---|
| NONE | 33,758 | Default/auto-discovered products |
| VVIP | 8,378 | Highest-priority products |
| NEW_ITEMS | 8,341 | Recently listed products |
| LINKS_INPUT | 3,489 | Manually inputted competitor links |
| VIP | 3,017 | Priority products |

### 4d. Competitive Averages

Based on rows with our product data populated:

| Metric | Ours | Competitor | Gap |
|---|---|---|---|
| Avg Price | RM 163.83 | RM 115.89 | Competitors 29% cheaper |
| Avg Sales | 1,343 | 5,432 | Competitors 4x more sales |
| Avg Rating | 4.64 | 4.86 | Competitors rated 0.22 higher |

### 4e. Full Column Breakdown (91 columns)

#### Group A: Identity & Metadata (8 columns)

| # | Column | Type | Nullable | Key | Default | Written By | Description |
|---|---|---|---|---|---|---|---|
| 1 | `id` | int unsigned | NO | PRI (auto_increment) | — | #1 | Primary key |
| 2 | `sku` | varchar(100) | YES | MUL | NULL | #5 | Product SKU identifier |
| 3 | `category` | varchar(100) | YES | — | NULL | #1 | Product category |
| 4 | `date_taken` | datetime | YES | MUL | NULL | #5 | When data was last refreshed |
| 5 | `sheet_name` | text | YES | MUL | NULL | #1 | Source sheet: NONE, VVIP, VIP, NEW_ITEMS, LINKS_INPUT |
| 6 | `status` | varchar(20) | YES | MUL | "active" | #1 | Row lifecycle status |
| 7 | `last_page` | varchar(20) | YES | — | "Shopee MY" | #1 | Current marketplace (MY or SG) |
| 8 | `original_page` | varchar(20) | YES | — | NULL | #1 | Original marketplace assignment |

#### Group B: Our Product Data (16 columns)

| # | Column | Type | Nullable | Key | Default | Written By | Description |
|---|---|---|---|---|---|---|---|
| 9 | `product_name` | varchar(255) | YES | MUL | NULL | #1, #5 | Our product title |
| 10 | `product_description` | text | YES | — | NULL | #5 | Our product description |
| 11 | `our_link` | text | YES | — | NULL | #1 | URL to our Shopee listing |
| 12 | `our_variation` | text | YES | — | NULL | #2, #3 | Our specific variation name |
| 13 | `shop_name` | text | YES | — | NULL | #1 | Our Shopee shop name |
| 14 | `our_shop_id` | bigint unsigned | YES | MUL | NULL | #3, #5 | Shopee shop ID |
| 15 | `our_item_id` | bigint unsigned | YES | — | NULL | #3, #5 | Shopee item ID |
| 16 | `our_model_id` | bigint unsigned | YES | — | NULL | #3, #5 | Shopee model (variation) ID |
| 17 | `our_price` | decimal(10,2) | YES | — | NULL | #5 | Our variation price |
| 18 | `our_product_price` | varchar(255) | YES | — | NULL | #5 | Product-level price (may be a range string) |
| 19 | `our_sales` | int | YES | — | NULL | #5 | Our total historical sales count |
| 20 | `our_stock` | int | YES | — | NULL | #5 | Our stock level |
| 21 | `shopee_stock` | int | YES | — | NULL | #5 | Shopee-reported stock (may differ from our_stock) |
| 22 | `our_rating` | decimal(3,2) | YES | — | NULL | #5 | Our product rating (0.00–5.00) |
| 23 | `our_main_images` | JSON | YES | — | NULL | #5 | Array of our main product image URLs |
| 24 | `our_variation_images` | text | YES | — | NULL | #5 | Our variation-specific images |

#### Group C: Competitor Data (13 columns)

| # | Column | Type | Nullable | Key | Default | Written By | Description |
|---|---|---|---|---|---|---|---|
| 25 | `comp_product` | text | YES | — | NULL | External | Competitor product title |
| 26 | `comp_product_description` | text | YES | — | NULL | External | Competitor product description |
| 27 | `comp_link` | text | YES | — | NULL | External | URL to competitor listing |
| 28 | `comp_variation` | text | YES | — | NULL | External | Competitor variation name |
| 29 | `comp_shop` | text | YES | — | NULL | External | Competitor shop name |
| 30 | `comp_price` | decimal(10,2) | YES | — | NULL | External | Competitor variation price |
| 31 | `comp_product_price` | varchar(255) | YES | — | NULL | External | Competitor product-level price (range string) |
| 32 | `comp_sales` | int | YES | — | NULL | External | Competitor total historical sales |
| 33 | `comp_monthly_sales` | text | YES | — | NULL | External | Competitor monthly sales breakdown |
| 34 | `comp_rating` | decimal(3,2) | YES | — | NULL | External | Competitor rating (0.00–5.00) |
| 35 | `comp_stock` | int | YES | — | NULL | External | Competitor stock level |
| 36 | `comp_main_images` | JSON | YES | — | NULL | External | Competitor product image URLs |
| 37 | `comp_variation_images` | text | YES | — | NULL | External | Competitor variation images |

> **Note:** Competitor columns are populated by an external process (separate scraper / `Shopee_Comp_Data` staging merge), not by the 7-script pipeline directly.

#### Group D: Analysis & Remarks (5 columns)

| # | Column | Type | Nullable | Default | Written By | Description |
|---|---|---|---|---|---|---|
| 38 | `our_advantage` | text | YES | NULL | External | What we do better (generated analysis) |
| 39 | `our_issue` | text | YES | NULL | External | Where we're weaker (generated analysis) |
| 40 | `automated_remarks` | text | YES | NULL | External | Bot-generated notes/flags |
| 41 | `user_remarks` | text | YES | NULL | External (webapp) | Human-added notes via dashboard |
| 42 | `Delete_Remark` | varchar(255) | YES | NULL | External (webapp) | Reason if row was soft-deleted |

#### Group E: SiteGiant Sales Data — Multi-period (10 columns)

Written by **Script #5** via SiteGiant API. Our sales aggregated by SKU across 5 time windows:

| # | Column | Type | Nullable | Default | Description |
|---|---|---|---|---|---|
| 43 | `our_sales_7d` | int | YES | 0 | Our unit sales, last 7 days |
| 44 | `sitegiant_sales_value_7days` | decimal(12,2) | YES | NULL | Our sales value (RM), last 7 days |
| 45 | `our_sales_14d` | int | YES | 0 | Our unit sales, last 14 days |
| 46 | `sitegiant_sales_value_14days` | decimal(12,2) | YES | NULL | Our sales value (RM), last 14 days |
| 47 | `our_sales_30d` | int | YES | 0 | Our unit sales, last 30 days |
| 48 | `sitegiant_sales_value_30days` | decimal(12,2) | YES | NULL | Our sales value (RM), last 30 days |
| 49 | `our_sales_60d` | int | YES | 0 | Our unit sales, last 60 days |
| 50 | `sitegiant_sales_value_60days` | decimal(12,2) | YES | NULL | Our sales value (RM), last 60 days |
| 51 | `our_sales_90d` | int | YES | 0 | Our unit sales, last 90 days |
| 52 | `sitegiant_sales_value_90days` | decimal(12,2) | YES | NULL | Our sales value (RM), last 90 days |

#### Group F: Shopee Variation-Level Sales (11 columns)

Written by **Script #4** via Shopee Partner API orders. Sales at the variation (model) level:

| # | Column | Type | Nullable | Default | Description |
|---|---|---|---|---|---|
| 53 | `shopee_var_sales_7d` | int | NO | 0 | Variation unit sales, last 7 days |
| 54 | `shopee_var_sales_14d` | int | NO | 0 | Variation unit sales, last 14 days |
| 55 | `shopee_var_sales_30d` | int | NO | 0 | Variation unit sales, last 30 days |
| 56 | `shopee_var_sales_60d` | int | NO | 0 | Variation unit sales, last 60 days |
| 57 | `shopee_var_sales_90d` | int | NO | 0 | Variation unit sales, last 90 days |
| 58 | `shopee_var_value_7d` | decimal(12,2) | NO | 0.00 | Variation sales value (RM), last 7 days |
| 59 | `shopee_var_value_14d` | decimal(12,2) | NO | 0.00 | Variation sales value (RM), last 14 days |
| 60 | `shopee_var_value_30d` | decimal(12,2) | NO | 0.00 | Variation sales value (RM), last 30 days |
| 61 | `shopee_var_value_60d` | decimal(12,2) | NO | 0.00 | Variation sales value (RM), last 60 days |
| 62 | `shopee_var_value_90d` | decimal(12,2) | NO | 0.00 | Variation sales value (RM), last 90 days |
| 63 | `shopee_var_sales_last_synced` | datetime | YES | NULL | Last time Shopee variation sales were synced |

#### Group G: Ads Performance Metrics (21 columns)

Written by **Script #6** via Shopee Seller Centre (Chrome/Selenium scraping):

| # | Column | Type | Nullable | Default | Description |
|---|---|---|---|---|---|
| 64 | `ads_spend_7d` | decimal(12,2) | NO | 0.00 | Ad spend (RM), last 7 days |
| 65 | `ads_spend_14d` | decimal(12,2) | NO | 0.00 | Ad spend (RM), last 14 days |
| 66 | `ads_spend_30d` | decimal(12,2) | NO | 0.00 | Ad spend (RM), last 30 days |
| 67 | `ads_spend_60d` | decimal(12,2) | NO | 0.00 | Ad spend (RM), last 60 days |
| 68 | `ads_spend_90d` | decimal(12,2) | NO | 0.00 | Ad spend (RM), last 90 days |
| 69 | `ads_visitors_7d` | int | NO | 0 | Visitors from ads, last 7 days |
| 70 | `ads_visitors_14d` | int | NO | 0 | Visitors from ads, last 14 days |
| 71 | `ads_visitors_30d` | int | NO | 0 | Visitors from ads, last 30 days |
| 72 | `ads_visitors_60d` | int | NO | 0 | Visitors from ads, last 60 days |
| 73 | `ads_visitors_90d` | int | NO | 0 | Visitors from ads, last 90 days |
| 74 | `ads_conv_rate_7d` | decimal(8,4) | NO | 0.0000 | Ad conversion rate, last 7 days |
| 75 | `ads_conv_rate_14d` | decimal(8,4) | NO | 0.0000 | Ad conversion rate, last 14 days |
| 76 | `ads_conv_rate_30d` | decimal(8,4) | NO | 0.0000 | Ad conversion rate, last 30 days |
| 77 | `ads_conv_rate_60d` | decimal(8,4) | NO | 0.0000 | Ad conversion rate, last 60 days |
| 78 | `ads_conv_rate_90d` | decimal(8,4) | NO | 0.0000 | Ad conversion rate, last 90 days |
| 79 | `roas_7d` | decimal(12,4) | NO | 0.0000 | Return on ad spend, last 7 days |
| 80 | `roas_14d` | decimal(12,4) | NO | 0.0000 | Return on ad spend, last 14 days |
| 81 | `roas_30d` | decimal(12,4) | NO | 0.0000 | Return on ad spend, last 30 days |
| 82 | `roas_60d` | decimal(12,4) | NO | 0.0000 | Return on ad spend, last 60 days |
| 83 | `roas_90d` | decimal(12,4) | NO | 0.0000 | Return on ad spend, last 90 days |
| 84 | `ads_metrics_last_synced` | datetime | YES | NULL | Last time ads metrics were synced |

#### Group H: AI Similarity Scoring (10 columns)

Written by **Script #7** via Gemini AI API:

| # | Column | Type | Nullable | Default | Description |
|---|---|---|---|---|---|
| 85 | `product_similarity_score` | int | YES | NULL | How closely comp matches our product (0–100) |
| 86 | `product_similarity_reason` | text | YES | NULL | AI-generated explanation of the score |
| 87 | `product_similarity_datetime` | datetime | YES | NULL | When the product similarity was scored |
| 88 | `variation_similarity_score` | int | YES | NULL | Variation-level match confidence (0–100) |
| 89 | `variation_similarity_reason` | text | YES | NULL | AI-generated explanation of variation match |
| 90 | `variation_similarity_datetime` | datetime | YES | NULL | When the variation similarity was scored |
| 91 | `product_similarity_excluded` | tinyint(1) | NO | 0 | Boolean — exclude from product matching |
| 92 | `variation_similarity_excluded` | tinyint(1) | NO | 0 | Boolean — exclude from variation matching |
| 93 | `has_product_exclusions` | tinyint(1) | YES | 0 | Boolean — has product-level exclusion rules |
| 94 | `has_variation_exclusions` | tinyint(1) | YES | 0 | Boolean — has variation-level exclusion rules |

#### Group I: Pricing, Promotions & Corrections (3 columns)

| # | Column | Type | Nullable | Default | Written By | Description |
|---|---|---|---|---|---|---|
| 95 | `discounted_price` | decimal(10,2) | YES | NULL | #5 | Our active promotional/discounted price |
| 96 | `voucher_name` | varchar(255) | YES | NULL | #5 | Active voucher name applied |
| 97 | `corrected_link` | text | YES | NULL | External (webapp) | Manually corrected competitor link (override) |

### 4f. Indexes (15 indexes)

| Index Name | Columns | Type | Purpose |
|---|---|---|---|
| `PRIMARY` | `id` | Unique | Primary key |
| `ft_shopee_comp_search` | `product_name`, `comp_product`, `sku`, `our_link` | Fulltext | Dashboard search functionality |
| `idx_sku` | `sku` | B-tree | SKU lookups (#5 SiteGiant matching) |
| `idx_shopee_comp_date_taken` | `date_taken` | B-tree | Date range queries |
| `idx_shopee_comp_sheet_name` | `sheet_name` | B-tree | Sheet filtering (pipeline skip logic) |
| `idx_our_shop_item_model` | `our_shop_id`, `our_item_id`, `our_model_id` | Composite B-tree | Shopee API ID lookups (#3, #4, #5, #6) |
| `idx_status_product_name` | `status`, `product_name` | Composite B-tree | Active product filtering |
| `idx_status_product_name_id` | `status`, `product_name`, `id` | Composite B-tree | Covering index for pagination |
| `idx_status_sku` | `status`, `sku` | Composite B-tree | Active SKU lookups |
| `idx_status_date_taken` | `status`, `date_taken` | Composite B-tree | Active rows by date (pipeline freshness check) |
| `idx_status_shop_name` | `status`, `shop_name` | Composite B-tree | Active rows by shop |
| `idx_status_comp_shop` | `status`, `comp_shop` | Composite B-tree | Active rows by competitor shop |
| `idx_status_comp_product_variation` | `status`, `comp_product`, `comp_variation` | Composite B-tree | Competitor product+variation lookups |
| `idx_has_product_exclusions` | `has_product_exclusions` | B-tree | Exclusion flag filtering (#7) |
| `idx_has_variation_exclusions` | `has_variation_exclusions` | B-tree | Exclusion flag filtering (#7) |

---

## 5. Table #2: `requestDatabase.ShopeeTokens`

### Purpose

Stores Shopee Open Platform OAuth credentials for all shops. Scripts #1–#5 read tokens for API authentication and write back refreshed tokens when they expire.

### Schema (9 columns)

| # | Column | Type | Nullable | Key | Default | Description |
|---|---|---|---|---|---|---|
| 1 | `id` | int | NO | PRI | — | Auto-increment ID |
| 2 | `shop_id` | varchar(50) | NO | UNI | — | Shopee shop ID (unique per shop) |
| 3 | `shop_name` | varchar(255) | NO | MUL | — | Human-readable shop name |
| 4 | `access_token` | text | NO | — | — | Current OAuth access token |
| 5 | `refresh_token` | text | NO | — | — | OAuth refresh token (used to get new access tokens) |
| 6 | `expires_at` | bigint | NO | MUL | — | Token expiry timestamp (Unix epoch) |
| 7 | `created_at` | timestamp | YES | — | CURRENT_TIMESTAMP | Row creation time |
| 8 | `updated_at` | timestamp | YES | — | CURRENT_TIMESTAMP | Last token refresh time |
| 9 | `region` | varchar(10) | YES | — | NULL | Shop region (MY, SG, etc.) |

### Statistics

| Metric | Value |
|---|---|
| **Rows** | 34 |
| **Used by scripts** | #1, #2, #3, #4, #5 (5 of 7 scripts) |
| **Access pattern** | READ tokens for API auth → WRITE back refreshed tokens |

---

## 6. Table #3: `AllBots.Shopee_VariationSalesDaily`

### Purpose

Stores **daily sales snapshots** at the variation (model) level. Script #4 fetches order data from the Shopee Partner API, decomposes it into daily per-variation records, and writes them here. It then reads them back to compute rolling aggregates (7d, 14d, 30d, 60d, 90d) that get written to `Shopee_Comp`.

### Schema (9 columns)

| # | Column | Type | Nullable | Key | Default | Description |
|---|---|---|---|---|---|---|
| 1 | `shop_id` | bigint unsigned | NO | PRI | — | Shopee shop ID |
| 2 | `item_id` | bigint unsigned | NO | MUL | — | Shopee item ID |
| 3 | `model_id` | bigint unsigned | NO | PRI | — | Shopee variation/model ID |
| 4 | `sku` | varchar(100) | YES | — | NULL | Product SKU |
| 5 | `sale_date` | date | NO | PRI | — | The specific day of sales |
| 6 | `units_sold` | int | NO | — | 0 | Units sold on that day |
| 7 | `orders_count` | int | NO | — | 0 | Number of orders on that day |
| 8 | `revenue` | decimal(12,2) | NO | — | 0.00 | Revenue (RM) on that day |
| 9 | `last_synced_at` | datetime | YES | — | NULL | When this row was last updated |

### Statistics

| Metric | Value |
|---|---|
| **Rows** | 73,996 |
| **Primary key** | Composite: (`shop_id`, `model_id`, `sale_date`) |
| **Used by scripts** | #4 only |
| **Access pattern** | WRITE daily snapshots from Shopee orders → READ back to compute rolling sums → write aggregates to `Shopee_Comp.shopee_var_sales_*` and `shopee_var_value_*` columns |

### Relationship to Shopee_Comp

```
Shopee_VariationSalesDaily.shop_id  → Shopee_Comp.our_shop_id
Shopee_VariationSalesDaily.item_id  → Shopee_Comp.our_item_id
Shopee_VariationSalesDaily.model_id → Shopee_Comp.our_model_id
```

---

## 7. Table #4: `AllBots.Shopee_VariationSalesSyncState`

### Purpose

Tracks the **sync cursor** for each shop — where the last Shopee order sync left off. Script #4 reads this to know the starting point for fetching new orders (incremental sync), then updates it after completing the sync run. This avoids re-fetching all orders from scratch every night.

### Schema (3 columns)

| # | Column | Type | Nullable | Key | Default | Description |
|---|---|---|---|---|---|---|
| 1 | `shop_id` | bigint unsigned | NO | PRI | — | Shopee shop ID (one row per shop) |
| 2 | `last_time_to` | int unsigned | NO | — | — | Unix timestamp of the last order fetched |
| 3 | `last_synced_at` | datetime | NO | — | — | When the last sync completed |

### Statistics

| Metric | Value |
|---|---|
| **Rows** | 28 (one per shop, matches 28 distinct shops in Shopee_Comp) |
| **Used by scripts** | #4 only |
| **Access pattern** | READ cursor at start → fetch orders from Shopee API after that point → WRITE updated cursor |

---

## 8. Table #5: `AllBots.Shopee_Comp_Similarity_Exclusions`

### Purpose

Stores **user-defined overrides** for the AI similarity scoring engine. When a user disagrees with the AI's product/variation match through the dashboard, they can record which specific differences should be excluded so the similarity engine stops penalizing them. Script #7 reads this table before scoring to apply exclusion rules.

### Schema (9 columns)

| # | Column | Type | Nullable | Key | Default | Description |
|---|---|---|---|---|---|---|
| 1 | `id` | int | NO | PRI (auto_increment) | — | Primary key |
| 2 | `our_link` | varchar(500) | NO | MUL | — | Our product link |
| 3 | `comp_link` | varchar(500) | NO | MUL | — | Competitor product link |
| 4 | `sku` | varchar(255) | NO | MUL | — | Product SKU |
| 5 | `exclusion_type` | enum('product','variation') | NO | MUL | — | Whether this excludes at product or variation level |
| 6 | `excluded_differences` | JSON | NO | — | — | JSON array of specific differences to ignore |
| 7 | `created_at` | timestamp | YES | — | CURRENT_TIMESTAMP | When exclusion was created |
| 8 | `updated_at` | timestamp | YES | — | CURRENT_TIMESTAMP | Last update time |
| 9 | `created_by` | varchar(100) | YES | — | "system" | Who created the exclusion |

### Statistics

| Metric | Value |
|---|---|
| **Rows** | 0 (empty — feature is new / not yet adopted) |
| **Used by scripts** | #7 only |
| **Access pattern** | READ-only by pipeline; WRITE by webapp dashboard |

### Relationship to Shopee_Comp

Feeds into `Shopee_Comp.has_product_exclusions` and `has_variation_exclusions` boolean flags. Script #7 checks this table to determine if certain differences should be ignored when computing `product_similarity_score` and `variation_similarity_score`.

---

## 9. Temporary Table: `tmp_sku_sales`

### Purpose

Ephemeral table created and dropped within **Script #5**'s execution. Used to batch-load SiteGiant sales aggregates by SKU, then JOIN-update them into `Shopee_Comp` in a single efficient operation rather than row-by-row updates.

### Lifecycle

```
Script #5 starts
  → CREATE TABLE tmp_sku_sales (sku, sales_7d, sales_14d, ..., value_7d, value_14d, ...)
  → INSERT INTO tmp_sku_sales (bulk load from SiteGiant API results)
  → UPDATE Shopee_Comp JOIN tmp_sku_sales ON sku = sku SET our_sales_7d = ..., sitegiant_sales_value_7days = ...
  → DROP TABLE tmp_sku_sales
Script #5 ends
```

---

## 10. External APIs Used by the Pipeline

| # | API | Used By | Auth Method | Purpose |
|---|---|---|---|---|
| 1 | **Shopee Partner API** | #1, #2, #3, #4, #5 | OAuth tokens from `ShopeeTokens` | Product info, prices, stock, ratings, images, order data, variation details |
| 2 | **SiteGiant API** (`opensgapi.sitegiant.co/api/v1`) | #5 | Static API token (hardcoded) | Order sales data by channel/SKU for 7d–90d breakdowns |
| 3 | **Shopee Seller Centre** | #6 | Chrome/Selenium browser automation | Ads metrics (spend, visitors, conversion rate, ROAS) — no API available, scraped from web UI |
| 4 | **Gemini AI API** | #7 | API key | Product and variation similarity scoring with natural language reasoning |

---

## 11. Event Schedulers

### `ev_cleanup_shopee_comp` — Monthly purge

| Property | Value |
|---|---|
| **Database** | AllBots |
| **Schedule** | Every **1 month** |
| **Last executed** | Feb 28, 2026 |
| **Status** | ENABLED |
| **SQL** | `DELETE FROM Shopee_Comp WHERE date_taken < DATE_FORMAT(CURDATE() - INTERVAL 1 MONTH, '%Y-%m-01')` |

This is why the table only holds ~1 month of data. Rows with `date_taken` older than the 1st of the previous month are purged.

**No other events** target any of the 5 pipeline tables.

---

## 12. Triggers, Stored Procedures, Views & Foreign Keys

| Type | Count |
|---|---|
| Triggers on any pipeline table | 0 |
| Stored procedures referencing pipeline tables | 0 |
| Views referencing pipeline tables | 0 |
| Foreign key constraints | 0 |

All relationships are enforced at the application level (Python scripts). All business logic lives in the VM3 bot scripts.

---

## 13. Data Flow Diagram

```
                     External APIs
    ┌──────────────────┬──────────────────┬─────────────────┐
    │ Shopee Partner   │ SiteGiant API    │ Shopee Seller   │  Gemini AI
    │ API              │                  │ Centre (Chrome) │  API
    └───────┬──────────┴────────┬─────────┴───────┬─────────┘────┬────
            │                   │                  │              │
   Scripts #1-5            Script #5          Script #6      Script #7
            │                   │                  │              │
            ▼                   ▼                  ▼              ▼
    ┌───────────────────────────────────────────────────────────────────┐
    │                                                                   │
    │                    AllBots.Shopee_Comp                            │
    │                    56,983 rows × 91 columns                      │
    │                                                                   │
    │  ┌─ Groups A,B: Identity + Our Product (#1,#2,#3,#5)            │
    │  ├─ Group C: Competitor Data (external process)                   │
    │  ├─ Group D: Analysis/Remarks (external/webapp)                   │
    │  ├─ Group E: SiteGiant Sales (#5 via SiteGiant API)              │
    │  ├─ Group F: Shopee Var Sales (#4 via Shopee API)                │
    │  ├─ Group G: Ads Metrics (#6 via Chrome scraping)                │
    │  ├─ Group H: AI Similarity (#7 via Gemini AI)                    │
    │  └─ Group I: Pricing/Promotions (#5)                             │
    │                                                                   │
    └──────────┬────────────────────────────────────────────────────────┘
               │
    ┌──────────┴─────────────────────────────────────────┐
    │                                                     │
    ▼                        ▼                            ▼
requestDatabase          AllBots                      AllBots
.ShopeeTokens            .Shopee_Variation            .Shopee_Comp_
 34 rows                  SalesDaily                   Similarity_
 OAuth tokens             73,996 rows                  Exclusions
 (R/W by #1-5)            (R/W by #4)                  0 rows
                                                       (R by #7)
                          AllBots
                          .Shopee_Variation
                          SalesSyncState
                          28 rows
                          (R/W by #4)

    ┌──────────────────────────────────────────────────────────────────┐
    │  ev_cleanup_shopee_comp                                          │
    │  Monthly: DELETE rows older than 1st of previous month           │
    │  Last run: Feb 28, 2026                                          │
    └──────────────────────────────────────────────────────────────────┘
```

---

## 14. Per-Script Summary

| # | Script | Tables (R/W) | APIs | Columns Written to Shopee_Comp |
|---|---|---|---|---|
| 1 | `ca_shopee_listing_to_db.py` | `Shopee_Comp` (W), `ShopeeTokens` (R/W) | Shopee Partner | `id`, `product_name`, `our_link`, `shop_name`, `our_shop_id`, `sheet_name`, `status`, `last_page`, `original_page`, `category` |
| 2 | `our_variation_preprocessing.py` | `Shopee_Comp` (R/W), `ShopeeTokens` (R/W) | Shopee Partner | `our_variation` |
| 3 | `ca_ai_variation_match.py` | `Shopee_Comp` (R/W), `ShopeeTokens` (R/W) | Shopee Partner | `our_variation`, `our_shop_id`, `our_item_id`, `our_model_id` |
| 4 | `shopee_comp_shopee_sales.py` | `Shopee_Comp` (R/W), `ShopeeTokens` (R/W), `Shopee_VariationSalesDaily` (R/W), `Shopee_VariationSalesSyncState` (R/W) | Shopee Partner | `shopee_var_sales_7d..90d`, `shopee_var_value_7d..90d`, `shopee_var_sales_last_synced` |
| 5 | `Shopee-mylinks-sales-data-merged.py` | `Shopee_Comp` (R/W), `ShopeeTokens` (R/W), `tmp_sku_sales` (temp) | Shopee Partner, SiteGiant | `product_name`, `product_description`, `sku`, `our_price`, `our_product_price`, `our_sales`, `our_stock`, `shopee_stock`, `our_rating`, `our_main_images`, `our_variation_images`, `our_shop_id`, `our_item_id`, `our_model_id`, `date_taken`, `our_sales_7d..90d`, `sitegiant_sales_value_7..90days`, `discounted_price`, `voucher_name` |
| 6 | `ca_shopee_ads_metrics.py` | `Shopee_Comp` (R/W) | Shopee Seller Centre (Selenium) | `ads_spend_7d..90d`, `ads_visitors_7d..90d`, `ads_conv_rate_7d..90d`, `roas_7d..90d`, `ads_metrics_last_synced` |
| 7 | `ca_similarity_check.py` | `Shopee_Comp` (R/W), `Shopee_Comp_Similarity_Exclusions` (R) | Gemini AI | `product_similarity_score`, `product_similarity_reason`, `product_similarity_datetime`, `variation_similarity_score`, `variation_similarity_reason`, `variation_similarity_datetime`, `product_similarity_excluded`, `variation_similarity_excluded`, `has_product_exclusions`, `has_variation_exclusions` |

---

## 15. Key Observations

1. **`Shopee_Comp` is the single source of truth** — All 7 scripts read from and write to it. It's the central hub that aggregates data from 4 different external sources (Shopee API, SiteGiant API, Shopee Seller Centre, Gemini AI).

2. **Three independent sales data sources** — SiteGiant (SKU-level, Script #5), Shopee API orders (variation-level, Script #4), and Shopee's own historical `our_sales` (Script #5). This triangulation gives more accurate sales visibility.

3. **~59% of rows have no competitor match** — Only 23,238/56,983 rows have competitor data. The competitor columns are populated by an **external process** not part of the 7-script pipeline — likely a separate scraper that writes to `Shopee_Comp_Data` (staging) which then merges into `Shopee_Comp`.

4. **Competitors are cheaper and sell more** — Average price gap of 29% (RM 115 vs RM 164) with 4x more sales volume. This is the core competitive intelligence the pipeline exists to surface.

5. **Script #4 maintains its own daily fact table** — `Shopee_VariationSalesDaily` (74K rows) stores granular daily sales to compute rolling windows. `Shopee_VariationSalesSyncState` (28 rows, one per shop) enables incremental syncing.

6. **Script #6 uses browser automation** — Ads metrics aren't available via Shopee's Partner API, so it scrapes them via Chrome/Selenium. This is the most fragile step in the pipeline.

7. **Script #7 uses Gemini AI** — Google's Gemini for similarity scoring. The exclusion table (`Shopee_Comp_Similarity_Exclusions`) is empty, suggesting this human-in-the-loop feedback feature is newly shipped.

8. **`ShopeeTokens` is the most shared dependency** — 5 of 7 scripts need it. Token refresh failures cascade across the pipeline.

9. **No foreign keys, no triggers** — The entire pipeline relies on application-level integrity. This trades data safety for write performance and deployment flexibility.

10. **Data retention is ~1 month** — The `ev_cleanup_shopee_comp` event purges monthly. The `Shopee_VariationSalesDaily` table has no cleanup event, so it will grow indefinitely (currently 74K rows).
