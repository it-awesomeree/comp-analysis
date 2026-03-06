# ca_shopee_ads_metrics.py -- Comprehensive Documentation

**Location:** `VM3 > C:\Users\Admin\Desktop\Shopee Comp My links Api\ca_shopee_ads_metrics.py`
**Size:** 2,331 lines / 106 KB
**Last Modified:** 2026-03-05 17:20:04
**Language:** Python 3.13
**Pipeline Position:** Job #6 of 7 in the Shopee Comp Analysis overnight pipeline (`scheduler.py`)

---

## 1. Purpose

This script is a **Selenium-based web scraper** that automates the collection of **advertising metrics** and **business insights** (visitors/traffic data) from the **Shopee Malaysia Seller Center** (`seller.shopee.com.my`).

It logs into the Seller Center, navigates to each shop's Ads and Business Insights pages, downloads exported Excel/CSV reports across multiple time periods (7d, 30d, 90d), parses them, and writes the extracted metrics back to the MySQL database.

**In short:** It answers the question _"How much are we spending on ads, what is our ROAS, how many visitors/clicks do our products get, and what is the conversion rate?"_ for every active product across all our Shopee MY shops, over 7-day, 30-day, and 90-day windows.

---

## 2. High-Level Workflow

```
main()
  |-> Get shop IDs (from DB or config)
  |-> Open Chrome browser (with saved Shopee session)
  |-> FOR EACH shop_id:
  |     |-> Navigate to shop list -> search by shop_id -> click "Details"
  |     |
  |     |-> [IF RUN_ADS_PAGE=True] SHOPEE ADS PAGE SCRAPE:
  |     |     |-> Navigate: Shopee Ads menu -> close popups -> product tab
  |     |     |-> Download 7-day ads CSV/Excel  -> parse -> store in shop_data.ads_7d
  |     |     |-> Download 30-day ads CSV/Excel -> parse -> store in shop_data.ads_30d
  |     |     |-> Download 90-day ads CSV/Excel -> parse -> store in shop_data.ads_90d
  |     |
  |     |-> [IF RUN_BUSINESS_INSIGHTS=True] BUSINESS INSIGHTS SCRAPE:
  |     |     |-> Navigate: Business Insights menu -> password prompt -> popups
  |     |     |     -> products tab -> traffic tab
  |     |     |-> Download 7-day business CSV  -> parse -> store in shop_data.business_7d
  |     |     |-> Download 30-day business CSV -> parse -> store in shop_data.business_30d
  |     |     |-> Download 3 individual monthly files (descending: most recent first)
  |     |     |     -> If ALL 3 succeed: aggregate into shop_data.business_90d
  |     |     |     -> If ANY fails: abort, mark business_90d_success=False
  |     |
  |     |-> UPDATE DATABASE:
  |     |     |-> update_ads_metrics_in_db()      -> writes ads spend + ROAS columns
  |     |     |-> update_business_metrics_in_db() -> writes visitor + conversion columns
  |     |
  |-> Cleanup downloaded files (past 2 hours)
  |-> Close browser
```

---

## 3. Database Operations

### 3.1 Database Connection

| Property | Value |
|---|---|
| **Host** | `34.142.159.230` (Google Cloud SQL) |
| **Database** | `AllBots` |
| **Table** | `Shopee_Comp` |
| **Auth** | `root` / `DB_PASSWORD` environment variable |

### 3.2 Database Reads

| Function | Query | Purpose |
|---|---|---|
| `get_unique_shop_ids()` | `SELECT DISTINCT our_shop_id FROM Shopee_Comp WHERE our_shop_id IS NOT NULL AND status = 'active' ORDER BY our_shop_id` | Gets all unique active shop IDs to process |
| `get_product_ids_for_shop(shop_id)` | `SELECT DISTINCT our_item_id FROM Shopee_Comp WHERE our_shop_id = %s AND our_item_id IS NOT NULL AND status = 'active'` | Gets product IDs for a given shop (used for reference) |

### 3.3 Database Writes

**Function: `update_ads_metrics_in_db()`**

```sql
UPDATE Shopee_Comp
SET ads_spend_7d  = %s,
    ads_spend_30d = %s,
    ads_spend_90d = %s,
    roas_7d       = %s,
    roas_30d      = %s,
    roas_90d      = %s,
    ads_metrics_last_synced = %s
WHERE our_shop_id = %s AND our_item_id = %s
```

- Writes for **every product_id found in the downloaded ads files**
- Updates 7 columns per product row

**Function: `update_business_metrics_in_db()`**

If 90-day data succeeded (all 3 months downloaded):

```sql
UPDATE Shopee_Comp
SET ads_visitors_7d   = %s,
    ads_visitors_30d  = %s,
    ads_visitors_90d  = %s,
    ads_conv_rate_7d  = %s,
    ads_conv_rate_30d = %s,
    ads_conv_rate_90d = %s
WHERE our_shop_id = %s AND our_item_id = %s
```

If 90-day data failed (fallback -- only 7d and 30d):

```sql
UPDATE Shopee_Comp
SET ads_visitors_7d   = %s,
    ads_visitors_30d  = %s,
    ads_conv_rate_7d  = %s,
    ads_conv_rate_30d = %s
WHERE our_shop_id = %s AND our_item_id = %s
```

### 3.4 Shopee_Comp Columns Written By This Script

| Column | Source | Description |
|---|---|---|
| `ads_spend_7d` | Ads Page "Expense" col | Ad spend last 7 days (RM) |
| `ads_spend_30d` | Ads Page "Expense" col | Ad spend last 30 days (RM) |
| `ads_spend_90d` | Ads Page "Expense" col | Ad spend last 90 days (RM) |
| `roas_7d` | Ads Page "ROAS" col | Return on ad spend -- 7 days |
| `roas_30d` | Ads Page "ROAS" col | Return on ad spend -- 30 days |
| `roas_90d` | Ads Page "ROAS" col | Return on ad spend -- 90 days |
| `ads_metrics_last_synced` | `datetime.now()` | Timestamp of last successful sync |
| `ads_visitors_7d` | Business Insights "Unique Product Clicks" | Visitors/clicks -- 7 days |
| `ads_visitors_30d` | Business Insights "Unique Product Clicks" | Visitors/clicks -- 30 days |
| `ads_visitors_90d` | Aggregated from 3 months | Visitors/clicks -- 90 days (sum) |
| `ads_conv_rate_7d` | orders / clicks | Conversion rate -- 7 days |
| `ads_conv_rate_30d` | orders / clicks | Conversion rate -- 30 days |
| `ads_conv_rate_90d` | Aggregated orders / clicks | Conversion rate -- 90 days |

---

## 4. File Downloads and Parsing

### 4.1 Ads Page Files

| Property | Value |
|---|---|
| **Filename prefix** | `Product-Ads-Overall-Data-` |
| **Format** | CSV or Excel |
| **Header row** | Row 9 (0-indexed: 8) |
| **Downloads per shop** | 3 files (7d, 30d, 90d date ranges) |

**Key columns parsed:**

| Column Header | Excel Position | Maps To |
|---|---|---|
| `Product ID` | Column D | Matched to `our_item_id` in DB |
| `Expense` | Column V | `ads_spend` (RM) |
| `ROAS` | Column W | `roas` (return on ad spend) |

### 4.2 Business Insights Files

| Property | Value |
|---|---|
| **Filename prefix** | `producttraffic_Product_Card` |
| **Format** | CSV or Excel |
| **Header row** | Row 1 (0-indexed: 0) |
| **Downloads per shop** | 5 files (7d, 30d, + 3 individual months) |

**Key columns parsed:**

| Column Header | Excel Position | Maps To |
|---|---|---|
| `Item ID` | Column A | Matched to `our_item_id` in DB |
| `Orders` | Column H | Order count |
| `Unique Product Clicks` | Column O | Visitor/click count (stored as `ads_visitors_*d`) |

**90-day aggregation formula:**
```
conversion_rate = total_orders_across_3_months / total_clicks_across_3_months
```

---

## 5. Authentication and Session Management

### 5.1 Shopee Login

- Uses a **persistent Chrome profile** at `C:\Users\Admin\AppData\Local\Google\Chrome\FreshProfile` (already logged in)
- **Session expiry detection:** Checks current URL for patterns: `accounts.shopee`, `account/signin`, `account/login`
- **Auto re-login flow** (if session expires):
  1. Clicks "Main/Sub Account" login option
  2. Enters credentials (`SHOPEE_LOGIN_ID` / `SHOPEE_PASSWORD` from env vars)
  3. Clicks "Send to Email" to trigger OTP delivery
  4. Polls **Zoho Mail via IMAP** for OTP email (checks INBOX, Newsletter, Notification, Spam folders)
  5. Extracts 6-digit OTP from email body (looks for `<b>XXXXXX</b>` pattern first, then any 6-digit number)
  6. Enters OTP and clicks Verify
  7. Saves debug screenshots at each stage to `C:\Users\Admin\Downloads\otp_*.png`

### 5.2 Business Insights Password

- Business Insights requires a separate password: `U2vwYfzh7hDr6XC`
- Auto-detected and entered when the password prompt appears after navigating to the page

### 5.3 Zoho Mail Config (OTP Retrieval)

| Property | Value |
|---|---|
| **IMAP Server** | `imappro.zoho.com` |
| **IMAP Port** | `993` (SSL) |
| **Credentials** | `ZOHO_EMAIL` / `ZOHO_PASSWORD` env vars |
| **OTP Timeout** | 120 seconds max |
| **Poll Interval** | 5 seconds between inbox checks |
| **Folders Searched** | INBOX, Newsletter, Notification, Spam |

---

## 6. Environment Variables Required

| Variable | Required | Purpose |
|---|---|---|
| `DB_PASSWORD` | **Yes** (crashes at import if missing) | MySQL database password |
| `ZOHO_EMAIL` | For OTP re-login | Zoho Mail email address |
| `ZOHO_PASSWORD` | For OTP re-login | Zoho Mail password |
| `SHOPEE_LOGIN_ID` | For OTP re-login | Shopee Seller Center login identifier |
| `SHOPEE_PASSWORD` | For OTP re-login | Shopee Seller Center password |

---

## 7. Run Configuration Flags

| Flag | Default | Purpose |
|---|---|---|
| `RUN_ADS_PAGE` | `True` | Enable/disable Shopee Ads page scraping |
| `RUN_BUSINESS_INSIGHTS` | `True` | Enable/disable Business Insights scraping |
| `RUN_SPECIFIC_STORES` | `False` | If `True`, only process `SPECIFIC_STORE_IDS` instead of all active shops from DB |
| `SPECIFIC_STORE_IDS` | `[252419950]` | Shop IDs to process when `RUN_SPECIFIC_STORES=True` |
| `CLICK_RETRY_ATTEMPTS` | `3` | Number of retries for failed UI clicks (total = 1 + retries) |
| `CLICK_RETRY_DELAY` | `2.0` | Seconds between click retries |

---

## 8. Error Handling and Resilience

### Click Retry Logic
- Every UI click is attempted up to **4 times** (1 initial + 3 retries) with 2-second delay between attempts
- If a click is intercepted by an overlay, automatically tries to close popups and remove modal DOM elements via JavaScript

### Stale Chrome Cleanup
- Before opening the browser, `kill_stale_chrome()` finds and kills any Chrome processes using the automation profile via `wmic`
- Removes lock files: `SingletonLock`, `SingletonSocket`, `SingletonCookie`

### Download Monitoring
- Polls the download directory for `.crdownload` (partial) files
- Waits up to 60 seconds for download completion
- Filters by expected filename prefix to avoid picking up wrong files

### 90-Day Fail-Fast Strategy
- If **ANY** of the 3 monthly downloads fails, immediately aborts remaining months
- Marks `business_90d_success = False` -- the 90-day columns will NOT be updated in DB
- 7-day and 30-day data still updates normally

### Per-Shop Error Isolation
- If one shop fails with an exception, the script logs the error and continues to the next shop
- Does not abort the entire run

### Modal/Popup Handling
- `close_all_popups()` -- tries multiple CSS and XPath selectors for close buttons
- `dismiss_modal_overlays()` -- last resort: removes `.eds-modal__*` elements from DOM via JavaScript

---

## 9. File Cleanup

- `cleanup_downloaded_files(hours=2)` runs at the end of `main()`
- Deletes files matching these prefixes from the Downloads folder that were modified in the last 2 hours:
  - `producttraffic_Product_Card.`
  - `Product-Ads-Overall-Data-`

---

## 10. Scheduling and Invocation

| Property | Value |
|---|---|
| **Scheduler** | `scheduler.py` on VM3 |
| **Pipeline position** | Job #6 of 7 |
| **Dependencies** | Needs `our_shop_id` and `our_item_id` to already exist in `Shopee_Comp` (populated by earlier pipeline jobs #1-5) |
| **Bot monitoring** | Uses `bot_runner.py` SDK at `C:\bot-sdk\` for run tracking (heartbeats, success/failure) |
| **Log file** | `comp_analysis_ads_data.log` (overwritten each run) |
| **Stderr/stdout logs** | `ads_metrics_task_stderr.log`, `ads_metrics_task_stdout.log` |

### Pipeline Order Context

```
1. ca_shopee_listing_to_db.py            (scrapes competitor listings)
2. our_variation_preprocessing.py        (preprocesses our product variations)
3. Shopee-mylinks-sales-data-merged.py   (product info + SiteGiant sales)
4. shopee_comp_shopee_sales.py           (Shopee order sales sync)
5. Shopee-mylinks-sales-data-merged.py   (product info + SiteGiant sales)
6. ca_shopee_ads_metrics.py              <-- THIS SCRIPT
7. ca_similarity_check.py               (AI similarity matching)
```

---

## 11. Data Models (Dataclasses)

### AdsMetrics
```python
@dataclass
class AdsMetrics:
    product_id: int
    ads_spend: float = 0.0   # From "Expense" column
    roas: float = 0.0        # From "ROAS" column
```

### BusinessMetrics
```python
@dataclass
class BusinessMetrics:
    product_id: int
    orders: float = 0.0
    unique_product_clicks: int = 0      # Stored as "visitors" in DB
    conversion_rate: float = 0.0        # Calculated: orders / unique_product_clicks
```

### ShopData
```python
@dataclass
class ShopData:
    shop_id: int
    shop_name: str = ""

    # Ads metrics by time period
    ads_7d:  Dict[int, AdsMetrics]       # product_id -> metrics
    ads_30d: Dict[int, AdsMetrics]
    ads_90d: Dict[int, AdsMetrics]

    # Business metrics by time period
    business_7d:  Dict[int, BusinessMetrics]
    business_30d: Dict[int, BusinessMetrics]
    business_90d: Dict[int, BusinessMetrics]   # Aggregated from 3 months

    business_90d_success: bool = False          # True only if all 3 months succeeded

    # Raw monthly data (for debugging)
    business_month1: Dict[int, BusinessMetrics]
    business_month2: Dict[int, BusinessMetrics]
    business_month3: Dict[int, BusinessMetrics]
```

---

## 12. Key Debugging Points

### Script does not start
- **Check:** `DB_PASSWORD` env var is set. The script crashes immediately at import time if missing (`os.environ["DB_PASSWORD"]` -- not `.get()`)

### No shops processed
- **Check:** `Shopee_Comp` table has rows with `status = 'active'` and non-null `our_shop_id`
- **Check:** If `RUN_SPECIFIC_STORES = True`, verify `SPECIFIC_STORE_IDS` is not empty

### Session expired / login loop
- **Check:** Zoho Mail credentials in env vars (`ZOHO_EMAIL`, `ZOHO_PASSWORD`)
- **Check:** Screenshots in `C:\Users\Admin\Downloads\otp_*.png` for visual debugging of each OTP login stage
- **Check:** Shopee may have changed the login UI -- verify selectors in `handle_shopee_otp_login()`

### Click failures / navigation broken
- **Likely cause:** Shopee UI updated, CSS/XPath selectors are stale
- **Check:** The long CSS/XPath selector configs at the top of the file (lines ~130-280)
- **Check:** `comp_analysis_ads_data.log` for which specific selector failed
- **Tip:** The script takes screenshots on failures -- check Downloads folder

### 90-day business metrics not updating
- **Check:** Logs for which month failed. The 3-month download is fail-fast
- **Note:** Year picker navigation is PERSISTENT -- clicking prev year only happens once. If the first month crosses a year boundary, the `navigate_to_prev_year` flag handles it

### Downloaded files not found
- **Check:** `C:\Users\Admin\Downloads` for files with expected prefixes
- **Check:** `wait_for_download()` timeout is 60s -- if Shopee is slow, downloads may timeout
- **Tip:** Multiple duplicate CSVs with `(1)`, `(2)` suffixes indicate repeated downloads from retries

### Business Insights password rejected
- **Check:** If Shopee changed the BI password, update constant `BUSINESS_INSIGHTS_PASSWORD` (currently: `U2vwYfzh7hDr6XC`)

### Chrome profile lock / browser does not open
- **Cause:** Stale Chrome processes from a previous crashed run
- **Auto-fix:** `kill_stale_chrome()` handles this automatically
- **Manual fix:** Delete lock files at `C:\Users\Admin\AppData\Local\Google\Chrome\FreshProfile\SingletonLock`

### Data not matching expected products
- **Note:** The script writes metrics for products found in the **downloaded export files**, not products queried from the DB. If a product has no ads running, it will not appear in the Ads export and will not receive an update

### Wait times between downloads
- Ads page: 15 seconds between downloads
- Business Insights: **60 seconds** between downloads (longer due to Shopee rate limiting)
- Download button: Waits 15 seconds before clicking the download button after requesting export

---

## 13. Key Functions Reference

| Function | Purpose |
|---|---|
| `main()` | Entry point -- orchestrates full pipeline |
| `process_single_shop()` | Handles one shop end-to-end |
| `scrape_shopee_ads_page()` | Downloads 3 ads files (7d, 30d, 90d) |
| `scrape_business_insights()` | Downloads 5 business files (7d, 30d, 3 months) |
| `process_ads_file()` | Parses ads CSV/Excel into `AdsMetrics` dict |
| `process_business_file()` | Parses business CSV/Excel into `BusinessMetrics` dict |
| `aggregate_three_months_business_data()` | Sums 3 months into 90-day totals |
| `update_ads_metrics_in_db()` | Writes ads spend + ROAS to `Shopee_Comp` |
| `update_business_metrics_in_db()` | Writes visitors + conversion rate to `Shopee_Comp` |
| `handle_shopee_otp_login()` | Full OTP re-login flow |
| `fetch_otp_from_zoho()` | Polls Zoho IMAP for OTP email |
| `ensure_logged_in()` | Session check + auto re-login trigger |
| `execute_click_sequence()` | Clicks a series of UI elements with retry |
| `click_element_with_retry()` | Single element click with 4-attempt retry |
| `close_all_popups()` | Dismisses Shopee modal popups |
| `dismiss_modal_overlays()` | JS-based DOM removal of modals |
| `cleanup_downloaded_files()` | Post-run file cleanup |
| `kill_stale_chrome()` | Pre-run Chrome process cleanup |
