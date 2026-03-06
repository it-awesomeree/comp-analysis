# ca_similarity_check.py — Comprehensive Documentation

**Location:** `VM3 > C:\Users\Admin\Desktop\Shopee Comp My links Api\ca_similarity_check.py`
**Size:** 65,439 bytes / 1,708 lines
**Language:** Python 3
**Last Modified:** 2026-02-24

---

## 1. Purpose

This script is the **AI-powered product similarity checker** for the Shopee Competitive Analysis pipeline. It compares **our product listings** against **competitor product listings** and assigns similarity scores at two levels:

- **Product-level similarity** — Does ANY variation of our product match ANY variation of the competitor's product?
- **Variation-level similarity** — Does the SPECIFIC our-variation match the SPECIFIC competitor-variation?

It uses **Google's Gemini AI (gemini-3-flash-preview)** with both text and images to make these judgments. A score >= 90 means "match."

---

## 2. Pipeline Position

This is **Job #7 (last)** in the nightly scheduler pipeline on VM3:

```
1. ca_shopee_listing_to_db.py          → Seeds new product rows
2. our_variation_preprocessing.py      → Backfills our_variation column
3. ca_ai_variation_match.py            → Resolves IDs, cleans data, AI matching
4. shopee_comp_shopee_sales.py         → Shopee order sales sync
5. Shopee-mylinks-sales-data-merged.py → Product info + SiteGiant sales
6. ca_shopee_ads_metrics.py            → Ads metrics
7. ca_similarity_check.py             → AI similarity (needs images/descriptions from #5)
```

It runs last because it depends on images and descriptions populated by upstream scripts.

---

## 3. Execution Flow

```
1. Setup logging → ai_similarity_YYYYMMDD_HHMMSS.log
2. PROPAGATION PHASE → Copy existing similarity scores to matching records (avoids redundant API calls)
3. FETCH PHASE → Query DB for rows needing similarity analysis
4. DISTRIBUTE → Round-robin rows across 8 worker threads
5. PARALLEL PROCESSING → Each worker: fetch exclusions → fetch images → call Gemini → parse JSON → update DB
6. SUMMARY → Log final stats (processed, failed, skipped, rate per second, API key stats)
```

### Detailed Flow per Row (inside worker thread)

```
1. fetch_exclusions_for_row()  → Query Shopee_Comp_Similarity_Exclusions table
2. fetch_images_for_row()      → Download up to 8 images in parallel, base64 encode
3. build_prompt_context()      → Construct product/competitor JSON context
4. build_full_prompt()         → Assemble system prompt + exclusions + rules + context
5. call_gemini_api()           → Send to Gemini with rate limiting + key rotation
6. parse_similarity_response() → Extract JSON scores from AI response (4-level fallback)
7. update_row_with_retry()     → Write scores to DB with deadlock retry
```

---

## 4. Database Reads & Writes

### 4.1 Database Connection

| Setting | Value |
|---|---|
| Host | `34.142.159.230` (Google Cloud SQL) |
| Database | `AllBots` |
| User | `root` |
| Password | From `DB_PASSWORD` environment variable |

### 4.2 Tables Accessed

#### `Shopee_Comp` (PRIMARY — Read & Write)

**READ — `fetch_rows_to_process()`**

```sql
SELECT
    id, product_name, product_description, sku, our_link, our_variation,
    shop_name, our_price, our_product_price, our_sales, our_stock, our_rating,
    comp_product, comp_product_description, comp_link, comp_variation,
    comp_shop, comp_price, comp_product_price, comp_sales, comp_rating, comp_stock,
    our_main_images, our_variation_images, comp_main_images, comp_variation_images,
    date_taken, sheet_name,
    has_product_exclusions, has_variation_exclusions
FROM Shopee_Comp
WHERE <filtering conditions>
ORDER BY
    date_taken DESC,
    (has_product_exclusions + has_variation_exclusions) DESC,
    product_name ASC
LIMIT 15000
```

**Filtering modes (mutually exclusive, checked in priority order):**

| Mode | Config Flags | Filter | Bypasses Common Filters? |
|---|---|---|---|
| **Selected Records** | `SELECTED_RECORDS_ONLY=True` | `id IN (...)` from `RECORD_IDS` list | YES — skips SKIP_ALREADY_PROCESSED, REQUIRE_IMAGES, RUN_ONE_STORE |
| **VVIP** | `VVIP=True` | `sheet_name='VVIP'` AND `date_taken >= 1 week ago` | No |
| **Default** | Both False | VVIP/VIP/NEW_ITEMS within 1 week, OR LINKS_INPUT within 4 weeks | No |

**Universal filters (apply to ALL modes including SELECTED_RECORDS_ONLY):**
- `comp_link IS NOT NULL AND comp_link != ''`

**Common filters (apply to all modes EXCEPT SELECTED_RECORDS_ONLY):**
- `SKIP_ALREADY_PROCESSED`: Skips rows that already have scores, UNLESS they have exclusion flags (`has_product_exclusions = 1` OR `has_variation_exclusions = 1`)
- `REQUIRE_IMAGES`: Requires `our_main_images IS NOT NULL OR comp_main_images IS NOT NULL`
- `RUN_ONE_STORE`: Filter to `shop_name = <TEST_STORE>`

**ORDER BY prioritization:**
1. Most recent records first (`date_taken DESC`)
2. Records with exclusions get priority (`(has_product_exclusions + has_variation_exclusions) DESC`)
3. Alphabetical tiebreaker (`product_name ASC`)

**WRITE — `update_row_with_retry()`**

```sql
UPDATE Shopee_Comp SET
    product_similarity_score = %s,
    product_similarity_reason = %s,
    product_similarity_datetime = %s,
    variation_similarity_score = %s,
    variation_similarity_reason = %s,
    variation_similarity_datetime = %s,
    has_product_exclusions = 0,
    has_variation_exclusions = 0
WHERE id = %s
```

Key detail: **Clears the exclusion flags** (`has_product_exclusions=0`, `has_variation_exclusions=0`) after processing to prevent infinite reprocessing loops.

**WRITE — `propagate_similarity_data()`**

```sql
UPDATE Shopee_Comp SET
    product_similarity_score = %s,
    product_similarity_reason = %s,
    product_similarity_datetime = %s,
    variation_similarity_score = %s,
    variation_similarity_reason = %s,
    variation_similarity_datetime = %s,
    product_similarity_excluded = %s,
    variation_similarity_excluded = %s,
    has_product_exclusions = %s,
    has_variation_exclusions = %s
WHERE our_link = %s
  AND comp_link = %s
  AND (our_variation = %s OR (our_variation IS NULL AND %s IS NULL))
  AND (
      product_similarity_score IS NULL
      OR product_similarity_score = 0
      OR variation_similarity_score IS NULL
      OR variation_similarity_score = 0
  )
```

Propagation copies scores from already-scored records to new records with the same `(our_link, comp_link, our_variation)` combination, saving API calls. The source record is selected as the **most recent** valid record via `MAX(COALESCE(product_similarity_datetime, variation_similarity_datetime))`.

**Important distinction:** Propagation copies `has_product_exclusions` and `has_variation_exclusions` from source AS-IS, while the regular update always resets them to 0.

Propagation commits in batches of **100 combinations** and does `connection.rollback()` on exception.

#### `Shopee_Comp_Similarity_Exclusions` (Read Only)

**READ — `fetch_exclusions_for_row()`**

```sql
SELECT exclusion_type, excluded_differences
FROM Shopee_Comp_Similarity_Exclusions
WHERE our_link = %s
  AND comp_link = %s
  AND sku = %s
```

- Returns product-level and variation-level exclusions (differences manually marked as "not relevant" via the web app)
- Exclusions are injected into the AI prompt so Gemini ignores those specific differences
- **SKU normalization:** empty/null SKUs become `__NO_SKU__` (matches the web app's `EXCLUSION_SKU_PLACEHOLDER`)
- **Exclusion type isolation:** Product exclusions ONLY apply to product decisions; variation exclusions ONLY apply to variation decisions

### 4.3 Columns Written to `Shopee_Comp`

| Column | Type | Description |
|---|---|---|
| `product_similarity_score` | int (0-100) | Product-level similarity score |
| `product_similarity_reason` | string | Empty if >=90, else 1-20 word explanation |
| `product_similarity_datetime` | datetime | UTC timestamp of when score was set |
| `variation_similarity_score` | int (0-100) | Variation-level similarity score |
| `variation_similarity_reason` | string | Empty if >=90, else 1-20 word explanation |
| `variation_similarity_datetime` | datetime | UTC timestamp of when score was set |
| `has_product_exclusions` | bit | Reset to 0 after processing |
| `has_variation_exclusions` | bit | Reset to 0 after processing |
| `product_similarity_excluded` | (propagation only) | Copied from source record |
| `variation_similarity_excluded` | (propagation only) | Copied from source record |

---

## 5. Gemini AI Integration

### 5.1 API Configuration

| Setting | Value |
|---|---|
| Model | `gemini-3-flash-preview` |
| Endpoint | `https://generativelanguage.googleapis.com/v1beta/models/gemini-3-flash-preview:generateContent` |
| API Keys | 2 keys with automatic rotation on rate limit |
| Temperature | 0.1 (near-deterministic) |
| Max output tokens | 8192 |
| Request timeout | 120 seconds |

### 5.2 Prompt Construction

The prompt is assembled from three parts:

```
COMBINED_SIMILARITY_PROMPT_BASE     (role + goal)
+ EXCLUDED_DIFFERENCES_TEMPLATE     (only if exclusions exist)
+ SIMILARITY_RULES                  (scope, scoring, output format)
+ Product context                   (our product + competitor data as JSON)
```

When exclusions exist (even partially), both sections are always rendered:
```
PRODUCT-LEVEL EXCLUSIONS (Apply to PRODUCT decision only):
  1. Different packaging size
VARIATION-LEVEL EXCLUSIONS: None
```

This prevents ambiguity about whether exclusions were intentionally omitted.

### 5.3 What the AI Prompt Contains

**Product context sent to Gemini:**

Our product fields:
- `product_name`, `product_description`, `sku`, `our_variation`, `shop_name`, `our_link`
- `our_price`, `our_product_price`, `our_sales`, `our_stock`, `our_rating`
- `our_main_images` (URLs), `our_variation_images` (URLs)

Competitor fields:
- `comp_product`, `comp_product_description`, `comp_link`, `comp_variation`, `comp_shop`
- `comp_price`, `comp_product_price`, `comp_sales`, `comp_rating`, `comp_stock`
- `comp_main_images` (URLs), `comp_variation_images` (URLs)

**Important:** Images are delivered to Gemini in TWO ways simultaneously:
1. **As text URLs** inside the JSON context
2. **As base64 inline_data** parts (downloaded and encoded separately)

This means even if base64 encoding fails for some images, Gemini still sees the URLs in the text context (though it cannot browse them).

### 5.4 Scope Restriction

The prompt explicitly restricts Gemini to ONLY use these fields for scoring:
- Title, Description, Pictures (main), Supporting pictures
- Variation names (list) for PRODUCT decision
- Variation name (single) for VARIATION decision

Price, sales, stock, and rating are sent as contextual metadata but the AI is instructed NOT to use them for scoring.

### 5.5 AI Output Format (Expected JSON)

```json
{
  "product": {
    "similarity_score": 85,
    "reason": "Different bundle quantity",
    "date": "2026-03-06T03:45:00.000Z"
  },
  "variation": {
    "similarity_score": 90,
    "reason": "",
    "date": "2026-03-06T03:45:00.000Z"
  }
}
```

Hard constraints:
- `similarity_score`: integer 0-100 (cast to `int` even if Gemini returns float/string)
- `reason`: empty string `""` if score >= 90; 1-20 words if score < 90
- Must not mention excluded differences in reason

### 5.6 Scoring Rules

**Default exclusions (always active, baked into prompt — not configurable):**
- Ignore brand/model names as a deciding factor (often made-up/unreliable)
- Ignore any logo elements on product/packaging as a difference
- Treat minor surface pattern differences as similar IF dimensions/specs match
- Pattern changes indicating different version/function still count as different

**Scoring deductions (start at 100, then subtract):**

| Difference | Deduction |
|---|---|
| Different category/purpose | -50 |
| Different size/dimensions/capacity | -30 |
| Different bundle quantity/pack count/contents | -30 |
| Different variation meaning (color/size/flavor/pack size) | -30 |
| Different compatibility/fit/ingredients/material (major) | -25 |
| Unclear/insufficient info on key specs/variation | Cap at 70 |

**Multi-variation product decision rule:**
- Gemini evaluates ALL plausible my-variation vs competitor-variation pairings
- Final product score = **BEST (highest-scoring)** pairing found
- If at least one pairing scores >= 90, product matches
- If ALL pairings capped by insufficient info, cap final product score at 70

This explains log entries like `P=100, V=40` — the product matches via a different variation pairing, but the specific variation pair doesn't match.

**Variation decision:**
- Only evaluates the SINGLE specific pair provided (our_variation vs comp_variation)

**Self-check instruction:**
The prompt ends with a FINAL SELF-CHECK section instructing Gemini to silently verify before returning:
- Score >= 90 → reason must be empty
- Score < 90 → reason must be 1-20 words
- Output must have exactly two keys: `product`, `variation`
- Reasons must not mention excluded differences for their respective decision type

---

## 6. Image Handling

### 6.1 Image Limits

| Setting | Value |
|---|---|
| Max total image bytes | 18 MB |
| Max per-image bytes | 3 MB |
| Image fetch timeout | 12 seconds |
| Image fetch workers | 8 parallel threads per row |
| Our main images max | 3 |
| Our variation images max | 1 |
| Competitor main images max | 3 |
| Competitor variation images max | 1 |
| **Total max images per row** | **8** |

### 6.2 Image Processing Pipeline

1. **URL extraction** (`extract_urls`): Handles JSON arrays, double-encoded strings, Python-style single-quote lists, plain URLs, regex fallback
2. **Deduplication** (`unique_limit`): Removes duplicate URLs before downloading
3. **Parallel download** (`fetch_image_as_base64`): Uses thread-local HTTP sessions with connection pooling
4. **Early rejection**: Uses `stream=True` to inspect headers (content-type, content-length) before downloading the body — saves bandwidth
5. **MIME normalization**: `image/jpg` → `image/jpeg`, `image/pjpeg` → `image/jpeg`, `image/x-png` → `image/png`
6. **MIME fallback**: If content-type is missing, not `image/*`, or not in supported list → defaults to `image/jpeg`
7. **Supported types**: PNG, JPEG, WebP, HEIC, HEIF, GIF

### 6.3 Image Ordering

Images are always sent to Gemini in this fixed order:
```
OUR_MAIN_1, OUR_MAIN_2, OUR_MAIN_3, OUR_VAR_1,
COMP_MAIN_1, COMP_MAIN_2, COMP_MAIN_3, COMP_VAR_1
```

Our images first, competitor second. Main images before variation images. This is maintained regardless of download completion order (parallel download results are re-ordered back to this sequence).

### 6.4 Greedy Size Limit

After parallel download, images are collected in the fixed order above. If any image would push past the 18MB total limit, **ALL remaining images are dropped** (greedy `break`, not best-fit). This means if early images (especially `OUR_MAIN`) are unusually large, later images like `COMP_VAR` may be truncated.

---

## 7. Parallel Processing Architecture

### 7.1 Configuration

| Setting | Value |
|---|---|
| Worker threads | 8 |
| API rate limit | 8 requests/second (shared globally) |
| Batch commit size | 10 rows per DB commit |
| Max retries on error | 3 (per row, for deadlocks) |
| Batch limit | 15,000 rows max per run |

### 7.2 Threading Model

- **Thread-local HTTP sessions** with connection pooling (20 connections each) — used for image downloads only
- **Thread-local DB connections** — each worker has its own MySQL connection
- **Thread-local API key index** — workers distributed across keys at startup, switch on rate limit
- **Gemini API calls** use the module-level `requests.post` directly (not thread-local sessions)
- **Round-robin row distribution** — rows split evenly across workers

### 7.3 Rate Limiting

The `RateLimiter` uses a simple interval-based approach with a threading lock:

- `min_interval = 1/8 = 0.125s` between requests
- The lock is held WHILE sleeping, which serializes all threads through a single bottleneck
- The rate limit is **GLOBAL across ALL threads and BOTH API keys** — capped at 8 req/s total
- The dual API keys are for **failover** (when Google rate-limits one key), NOT for doubling throughput
- Actual throughput is slightly less than 8/s due to lock contention overhead

### 7.4 API Key Rotation

1. Each worker starts with a key index assigned by `worker_id % 2` (evenly distributed)
2. On rate limit (HTTP 429, 403 with quota message, or 503 with overloaded):
   - Record failure for current key
   - Try alternate key
   - If alternate succeeds: worker **permanently switches** to that key for all future requests
   - If both keys rate-limited: **skip the record** entirely (tracked as `skipped_rate_limit`)
3. Key statistics (successes, failures, rate limits) are tracked per-key and reported at end

### 7.5 Deadlock Handling

MySQL InnoDB can deadlock when multiple threads update different rows. Handled with:
- Retry up to 3 times on errno 1213
- Exponential backoff: 0.1s, 0.2s, 0.3s
- If all retries fail: log error, skip the row

### 7.6 Progress Tracking

Uses a sliding-window rate calculation (2-minute window) for stable ETA estimates:
- `completed = processed + failed + skipped_rate_limit` (all outcomes count toward progress)
- Windowed rate preferred over average rate when available
- Startup grace: uses elapsed time as window when < 2 minutes have passed
- `processed_with_exclusions` is a separate informational counter (does not affect rate/ETA)

---

## 8. Response Parsing

`parse_similarity_response` has **4 fallback levels:**

1. **Strip markdown** (` ```json ``` `) → **Extract JSON bounds** (`{` to `}`) → **Direct parse** via `json.loads()`
2. **Fix embedded newlines** in quoted string values (replace `\n`, `\r` with space) → **Retry parse**
3. **Regex extraction** — individually extract `"product": {...}` and `"variation": {...}` blocks, pull `similarity_score` and `reason` from each. Can return **partial results** (only product or only variation)
4. **Give up** — log first 200 chars of the unparseable response → return None (row gets NULL scores)

The `_extract_results` validator:
- Checks key exists in parsed dict and value is a dict
- Requires `similarity_score` field to be present
- Casts score to `int` (handles float/string from Gemini)
- Defaults to `0` if `similarity_score` is missing
- Returns None if neither product nor variation could be extracted

---

## 9. Configuration Flags (Debugging Knobs)

| Flag | Default | Purpose |
|---|---|---|
| `DRY_RUN` | `False` | Skips all DB writes; counts only |
| `SKIP_ALREADY_PROCESSED` | `True` | Skips rows that already have scores (unless they have exclusions) |
| `REQUIRE_IMAGES` | `True` | Only process rows that have at least one image |
| `RUN_ONE_STORE` | `False` | Filter to a single store (set `TEST_STORE`) |
| `TEST_STORE` | `'KarlMobel'` | Store name when `RUN_ONE_STORE=True` |
| `VVIP` | `False` | Only process VVIP sheet records from last 1 week |
| `SELECTED_RECORDS_ONLY` | `False` | Only process specific row IDs in `RECORD_IDS` |
| `RECORD_IDS` | `[]` | List of numeric IDs when `SELECTED_RECORDS_ONLY=True` |
| `PROPAGATE_SIMILARITY_DATA` | `True` | Enable/disable the propagation pre-step |
| `PARALLEL_WORKERS` | `8` | Number of worker threads |
| `API_REQUESTS_PER_SECOND` | `8` | Global rate limit across all threads and keys |
| `BATCH_COMMIT_SIZE` | `10` | Rows per DB commit in worker threads |
| `BATCH_LIMIT` | `15000` | Max rows to fetch per run |

---

## 10. Logging

- **File:** `ai_similarity_YYYYMMDD_HHMMSS.log` in the same directory as the script
- **Console:** stdout
- **Encoding:** UTF-8
- **Format:** `2026-03-06 03:45:40 | INFO | [W_0] [1/2793] Row 372495: P=100, V=100 | Key=1 | 0.22/s, ETA 210.9m`

### Log Line Format

```
[completed/total] Row <id>: P=<product_score>, V=<variation_score> | Key=<key_index> | <rate>/s, ETA <minutes>m
```

- `P=-` or `V=-` means NULL score (parse failure or API error)
- `Key=0` or `Key=1` shows which Gemini API key was used

### Final Summary Includes

- Pre-propagated count
- Processed via API count (with/without exclusions breakdown)
- Failed count
- Skipped (rate limited) count
- Elapsed time and rate
- Total API calls
- Per-key statistics (successes, failures, rate limits)

---

## 11. Error Handling & Resilience

| Scenario | Handling |
|---|---|
| **Gemini rate limit (429/403/503)** | Rotate to alternate API key; if both rate-limited, skip the record |
| **Gemini timeout (>120s)** | `RequestException` caught, treated as non-rate-limit failure |
| **Gemini response truncated (`MAX_TOKENS`)** | Warning logged, partial text still returned as "success" — parser attempts extraction |
| **Gemini safety filter block** | Warning logged, usually no content → returns None |
| **Gemini unexpected response structure** | Error logged, returns None |
| **JSON parse failure from Gemini** | 4-level fallback (see Section 8) |
| **MySQL deadlock (errno 1213)** | Retry up to 3 times with exponential backoff (0.1s, 0.2s, 0.3s) |
| **Image download failure** | Silently skip that image, continue with others |
| **Image non-image content-type** | Early rejection via header inspection (stream=True) |
| **Image oversized (>3MB per image)** | Skipped before/after download |
| **Image total >18MB** | Greedy break — all remaining images dropped |
| **MIME type unknown** | Defaults to `image/jpeg` |
| **Worker thread exception** | Log error, increment failed count, continue to next row |
| **Propagation exception** | `connection.rollback()`, re-raise |
| **SELECTED_RECORDS_ONLY with empty RECORD_IDS** | Warning logged, returns empty list (no processing) |

---

## 12. Typical Runtime

Based on production logs (2026-03-06):

- **Propagation phase:** ~6 minutes (654 combinations → 1,201 records updated)
- **API processing phase:** 2,793 rows at ~0.22 rows/sec → ~3.5 hours estimated
- **Total daily run:** ~4 hours

---

## 13. Key Debugging Checkpoints

### No rows to process?
- Check `date_taken` freshness — VVIP/VIP/NEW_ITEMS must be within 1 week, LINKS_INPUT within 4 weeks
- Check `comp_link` population (must not be NULL/empty)
- Check `our_main_images` / `comp_main_images` presence (when `REQUIRE_IMAGES=True`)
- Check if all rows already have scores and no exclusion flags set

### All scores NULL after run?
- Check Gemini API key validity (look for HTTP 401/403 errors in log)
- Check response parsing failures (`Failed to parse response:` in log)
- Check `DB_PASSWORD` environment variable is set

### Slow processing?
- Check rate limit hits in log (frequent key rotation = hitting Google's limits)
- Check image download times (large images slow down each row)
- Check for `Gemini response truncated` warnings (large prompts → slower responses)

### Scores not propagating?
- Verify `(our_link, comp_link, our_variation)` tuples match exactly between source and target records
- Source must have BOTH `product_similarity_score` and `variation_similarity_score` as non-NULL, non-zero
- Target must have at least one score as NULL or 0

### Exclusions not working?
- Check `Shopee_Comp_Similarity_Exclusions` table for matching records
- Verify SKU normalization — empty SKUs must be stored as `__NO_SKU__` in exclusions table
- Check `has_product_exclusions` / `has_variation_exclusions` flags are set on the `Shopee_Comp` rows
- Verify exclusion type matches (`product` vs `variation`)

### Infinite reprocessing?
- The script clears exclusion flags after processing (`has_product_exclusions=0`, `has_variation_exclusions=0`)
- If they keep reappearing, the web app is re-setting them
- Check if the web app's exclusion save endpoint also sets these flags

### Parse failures in log?
- `Gemini response truncated` → Response exceeded `maxOutputTokens: 8192` (image-heavy request)
- `Failed to parse response:` → Gemini returned non-JSON; 4-level fallback all failed
- Row ends up with `P=-, V=-` (NULL scores) — will be retried next run

### "Both API keys rate limited" in log?
- Both Gemini API keys exceeded Google's quota
- Affected rows are SKIPPED (not failed) — tracked separately in stats
- Check if request volume increased (more rows to process than usual)

### Deadlock warnings?
- `Deadlock on row <id>, retry N/3` — normal with 8 concurrent threads writing to same table
- If retries consistently fail (3/3), check for long-running transactions on the MySQL server

### Images missing from AI analysis?
- Check `our_main_images` / `comp_main_images` column contents (may be malformed JSON)
- Check image URLs are accessible (not expired Shopee CDN links)
- Check total image size — if early images are large, later images get dropped (greedy break at 18MB)
- Images are ordered: OUR_MAIN → OUR_VAR → COMP_MAIN → COMP_VAR; competitor variation images are most likely to be dropped

---

## 14. Dependencies

### Python Packages
- `mysql.connector` — MySQL database connectivity
- `requests` — HTTP client for Gemini API and image downloads
- `base64` — Image encoding for Gemini inline_data
- `json`, `re`, `logging`, `threading`, `queue`, `time`, `os`, `sys` — Standard library

### External Services
- **Google Gemini API** — AI model for similarity analysis
- **Google Cloud SQL** (`34.142.159.230`) — MySQL database
- **Shopee CDN** — Image URLs stored in DB columns

### Environment Variables
- `DB_PASSWORD` — MySQL root password (required, no default)

---

## 15. Complete Function Reference

| Function | Lines (approx) | Purpose |
|---|---|---|
| `build_full_prompt()` | ~207-240 | Assemble prompt from base + exclusions template + rules |
| `setup_logging()` | ~243-260 | Configure console + file logging with timestamp |
| `APIKeyManager` (class) | ~263-320 | Thread-safe API key rotation with per-key stats tracking |
| `RateLimiter` (class) | ~323-358 | Thread-safe rate limiter (interval-based, sleeps under lock) |
| `ProcessingStats` (class) | ~361-480 | Thread-safe progress tracking with 2-min sliding-window rate |
| `get_http_session()` | ~485-495 | Thread-local HTTP session with connection pooling (20 conn) |
| `get_thread_key_index()` | ~498-501 | Get current thread's preferred API key index |
| `set_thread_key_index()` | ~504-506 | Set current thread's preferred API key index |
| `extract_urls()` | ~509-585 | Parse image URLs from various DB storage formats |
| `unique_limit()` | ~588-598 | Deduplicate URLs, return first N unique |
| `normalize_mime_type()` | ~601-608 | Standardize MIME type variants |
| `fetch_image_as_base64()` | ~611-650 | Download single image, base64 encode with size/type checks |
| `fetch_images_for_row()` | ~653-712 | Parallel download all images for a row (8 workers) |
| `is_rate_limit_error()` | ~715-762 | Detect rate limit from HTTP response or exception |
| `call_gemini_api_with_key()` | ~765-840 | Single Gemini API call with specific key |
| `call_gemini_api()` | ~843-925 | Gemini API call wrapper with automatic key rotation |
| `parse_similarity_response()` | ~928-995 | 4-level fallback JSON parser for AI response |
| `_extract_results()` | ~998-1010 | Validate and extract product/variation from parsed JSON |
| `create_db_connection()` | ~1013-1015 | Create MySQL connection from DB_CFG |
| `propagate_similarity_data()` | ~1018-1195 | Copy existing scores to matching records (pre-step) |
| `fetch_rows_to_process()` | ~1198-1268 | Query rows needing similarity analysis |
| `update_row_with_retry()` | ~1271-1303 | Write scores with deadlock retry (3 attempts) |
| `normalize_exclusion_sku()` | ~1306-1309 | Empty SKU → `__NO_SKU__` placeholder |
| `fetch_exclusions_for_row()` | ~1312-1378 | Query exclusions from Shopee_Comp_Similarity_Exclusions |
| `build_prompt_context()` | ~1381-1430 | Build product/competitor JSON for prompt context |
| `process_row()` | ~1433-1478 | Orchestrate single row: exclusions → images → API → parse |
| `worker_thread()` | ~1481-1570 | Worker thread main loop with DB commit batching |
| `main()` | ~1573-1708 | Entry point — propagate → fetch → distribute → process → summary |
