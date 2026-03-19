# our_variation_preprocessing.py — Comprehensive Documentation

> **Script Location:** `VM3:C:\Users\Admin\Desktop\Shopee Comp My links Api\our_variation_preprocessing.py`
> **Language:** Python 3.13 | **Lines:** 1,713 | **Bytes:** 63,053 | **VM:** VM3 (current 5-script chain)
> **Last modified (live):** 2026-03-13 12:04:49 | **Last verified:** 2026-03-19

## Live Verification (2026-03-19)

- Verified against VM3 live file: `C:\Users\Admin\Desktop\Shopee Comp My links Api\our_variation_preprocessing.py`
- Scheduler context unchanged: Job #2 in the 4:00 PM VM3 pipeline
- Important behavior corrections from live script:
  - Live mode now runs a post-run blank audit with retry passes (`FINAL_AUDIT_MAX_PASSES = 2`)
  - If in-scope blank `our_variation` rows remain after max passes, the script raises a fatal error and exits non-zero
  - Candidate windows remain:
    - `VVIP` / `VIP` / `NEW_ITEMS`: 30 days
    - `LINKS_INPUT`: 28 days
  - Matching thresholds remain permissive (`CONFIDENCE_THRESHOLD = 0`) with additional fuzzy lookup fallback (`LOOKUP_FUZZY_THRESHOLD = 0.88`)
- Security note from live source: OpenAI key and DB passwords are currently hardcoded in this VM script and should be rotated/moved to environment variables in production hardening work

---

## 1. Purpose & Business Context

This script is a **preprocessing step** in the Shopee Competitive Analysis (CA) pipeline. Its sole job is to **backfill the `our_variation` column** in the `AllBots.Shopee_Comp` table for rows where it is missing.

### Why it exists

The downstream script (`ca_ai_variation_match.py`) needs `our_variation` populated to resolve `our_model_id` and complete price-comparison logic. Without `our_variation`, the CA pipeline cannot determine which specific product variant (size, color, etc.) of **our** product corresponds to a competitor's listed variation, which makes price comparison impossible at the SKU level.

### What it solves

Competitor data rows arrive with:
- `comp_variation` — the competitor's variation name (e.g., "Blk XL", "Red Medium", or `-` for single-model products)
- `our_link` — URL to our Shopee product listing

But `our_variation` (which of our product's variants matches that competitor row) is initially empty. This script fills that gap by:
1. Calling the **Shopee Partner API** to fetch our product's available variations
2. Using **OpenAI (o4-mini)** to fuzzy-match competitor variation names to our variation names
3. Writing the matched `our_variation` value back to the database

---

## 2. Pipeline Position

This script runs as **Job #2 of 5** in the `scheduler.py` sequential pipeline on VM3, launched by the daily 4:00 PM Task Scheduler trigger.

| Job # | Script | Purpose |
|-------|--------|---------|
| 1 | `ca_shopee_listing_to_db.py` | Scrape competitor listings into DB |
| **2** | **`our_variation_preprocessing.py`** | **Backfill `our_variation` (this script)** |
| 3 | `ca_ai_variation_match.py` | Resolve `our_model_id` using `our_variation` |
| 4 | `shopee_comp_shopee_sales.py` | Fetch sales data |
| 5 | `Shopee-mylinks-sales-data-merged.py` | Product info + SiteGiant sales |

Jobs 6-7 (`ca_shopee_ads_metrics.py`, `ca_similarity_check.py`) are scheduled out of the current daily chain.


### Trigger chain
```text
Windows Task Scheduler
  -> 4:00 PM daily trigger
    -> chain_today_pipeline.ps1 (waits for ResumePipeline2)
    -> scheduler.py (sequential job runner with retry logic)
      -> Job #2: our_variation_preprocessing.py (via preprocess_wrapper.py)
```

### Scheduler behavior for this job
- **Retries:** 2 attempts with 30-second gap between retries
- **Timeout:** 6 hours maximum
- **Lock file:** Prevents concurrent pipeline runs
- **Checkpoint/resume:** Pipeline can resume from the last failed job

---

## 3. Entry Point & Execution Flow

```
main()
  |-- setup_logging()                      -> Configure file + console logging
  |-- process_dash_comp_variations()       -> Phase 0: comp_variation = '-' rows
  +-- process()                            -> Phase 1: normal variation matching
```

### main() (lines 1393-1422)
- Calls `setup_logging()` to create timestamped log file
- Runs **Phase 0** first (`process_dash_comp_variations()`)
- Then runs **Phase 1** (`process()`)
- Catches any fatal exception from either phase, logs it, and exits with code 1
- On success, exits with code 0

### Execution wrappers
- **`preprocess_wrapper.py`** — Crash wrapper using `exec(open(...).read())` to catch `BaseException` including `SystemExit`
- **`run_variation_preprocessing.bat`** — Manual runner: `python "our_variation_preprocessing.py" > "manual_run_stdout.log" 2>&1`
- **`run_preprocess.bat`** — Scheduled task runner with separate stdout/stderr logs
- **`wait_and_run_variation.bat`** — Waits for a specific PID before running (dependency on prior job)

---

## 4. Phase 0: comp_variation = '-' Rows

**Function:** `process_dash_comp_variations()` (lines 834-988)

Phase 0 handles rows where `comp_variation = '-'`, meaning the competitor product is a **single-model product** with no variations. Since there is no variation text to match against, this phase uses the **competitor's product name** (`comp_product` column) to determine which of our variations it corresponds to.

### Flow
1. Fetch rows via `fetch_dash_comp_variation_rows()` — WHERE `comp_variation = '-'` AND `our_variation` is empty
2. Group rows by `(shop_id, item_id)` parsed from `our_link`
3. For each group:
   - Call Shopee API `get_model_list` to get our product's variations
   - If our product has **1 variation** -> set `our_variation = '-'` (both sides are single-model)
   - If our product has **multiple variations** -> call `call_openai_dash()` which uses `PROMPT_TEMPLATE_DASH` to match based on product name keywords (color, size, model, etc.)
4. Commit per group (transaction-per-group pattern)

### OpenAI prompt (Phase 0)
Uses `PROMPT_TEMPLATE_DASH` which instructs the AI to:
- Extract keywords from competitor product names (color, size, model, capacity, weight, material, type)
- Match those keywords to the closest variation from our list
- Return JSON array with `confidence`, `comp_product_name`, and `match` fields

---

## 5. Phase 1: Normal Variation Matching

**Function:** `process()` (lines 994-1389)

Phase 1 handles the main case: rows where `comp_variation` is a real variation string (not `-` and not empty).

### Flow
1. Fetch rows via `fetch_candidate_rows()` — WHERE `comp_variation` is present (not `-`) AND `our_variation` is empty
2. Group rows by `(shop_id, item_id)` parsed from `our_link`
3. For each group:
   - Call Shopee API `get_model_list` to get our product's variations
   - Build variation lookup via `build_variation_lookup()`
   - **No-variation product** (single model, no tiers): set `our_variation = '-'`
   - **Single variation**: set `our_variation = '-'`
   - **Multiple variations**: deduplicate `comp_variation` values, then call `call_openai()` in chunks of `MAX_VARIATIONS_PER_PROMPT` (20)
4. For each AI result:
   - Check confidence against `CONFIDENCE_THRESHOLD` (currently 0 — accepts all matches)
   - Verify matched variation exists in our lookup (with fuzzy fallback via `resolve_variation_key()`)
   - Apply the matched `our_variation` to ALL rows sharing the same `comp_variation`
5. Commit per group

### OpenAI prompt (Phase 1)
Uses `PROMPT_TEMPLATE` which instructs the AI to:
- Handle abbreviations, typos, misspellings, reordered tokens, different separators
- Match each competitor variation to the single closest semantic match from our variations
- Return JSON array with `confidence` (0-100), `original`, and `match` fields
- Always provide a match even at low confidence

---

## 6. Database Operations

### Reads

| Query | Function | Database.Table | Purpose |
|-------|----------|---------------|---------|
| Fetch candidate rows (Phase 1) | `fetch_candidate_rows()` | `AllBots.Shopee_Comp` | Rows where `our_variation` is empty, `comp_variation` is present and not `-` |
| Fetch dash rows (Phase 0) | `fetch_dash_comp_variation_rows()` | `AllBots.Shopee_Comp` | Rows where `comp_variation = '-'` and `our_variation` is empty |
| Load access tokens | `fetch_tokens_by_shop_id()` | `requestDatabase.ShopeeTokens` | All shop access tokens for Shopee API calls |
| Load refresh token | `refresh_shop_token_via_api()` | `requestDatabase.ShopeeTokens` | Single shop's refresh token for OAuth refresh |

### Writes

| Query | Function | Database.Table | Purpose |
|-------|----------|---------------|---------|
| Update our_variation | `update_our_variation()` | `AllBots.Shopee_Comp` | SET `our_variation` WHERE `id = ?` AND `our_variation` is still empty |
| Update tokens | `refresh_shop_token_via_api()` | `requestDatabase.ShopeeTokens` | SET `access_token`, `refresh_token`, `expires_at`, `updated_at` after OAuth refresh |

### Filtering window (both phases)

```sql
WHERE date_taken IS NOT NULL
  AND (
    (sheet_name IN ('VVIP', 'VIP', 'NEW_ITEMS') AND date_taken >= NOW() - INTERVAL 30 DAY)
    OR
    (sheet_name = 'LINKS_INPUT' AND date_taken >= NOW() - INTERVAL 28 DAY)
  )
ORDER BY date_taken DESC, id DESC
LIMIT 50000
```

- VVIP/VIP/NEW_ITEMS: 30-day window
- LINKS_INPUT: 28-day window
- Ordered newest-first so recent data is prioritized within the 50K limit

### Transaction model
- `autocommit = False`
- **Commit per product group** (each `(shop_id, item_id)` group is one transaction)
- On exception -> `rollback()` for that group only, then continue to next group
- Idempotent UPDATE guard: `WHERE our_variation IS NULL OR TRIM(our_variation) = ''` prevents overwriting if another process fills it concurrently

---

## 7. External API Calls

### Shopee Partner API

| Endpoint | Purpose | Auth |
|----------|---------|------|
| `/api/v2/product/get_model_list` | Retrieve all variations (models/tiers) for our product | HMAC-SHA256 signed URL |
| `/api/v2/auth/access_token/get` | Refresh expired OAuth access token | HMAC-SHA256 signed + POST body |

**Rate limiting:** `SLEEP_BETWEEN_SHOPEE_CALLS = 0.15s` between API calls

**Token retry logic (`run_with_token_retry`):**
1. Try the API call with current access token
2. On HTTP 403 -> attempt Shopee OAuth refresh (`refresh_shop_token_via_api`)
3. If OAuth refresh fails -> fall back to reloading all tokens from DB (`fetch_tokens_by_shop_id`)
4. Retry the API call once (total 2 attempts)

### OpenAI API

| Endpoint | Model | Purpose |
|----------|-------|---------|
| `POST /v1/chat/completions` | `o4-mini` | Fuzzy-match variations (Phase 1) or product-name-to-variation (Phase 0) |

**Rate limiting:** `SLEEP_BETWEEN_OPENAI_CALLS = 0.5s` between calls

**Response parsing:**
- Strips markdown code fences if present
- Replaces doubled braces `{{ }}` -> `{ }` (prompt template artifact)
- Validates JSON is a list of dicts with required fields
- Falls back to empty list on any parse error

---

## 8. Variation Lookup Builder

**Function:** `build_variation_lookup()` (lines 543-587)

Converts the Shopee API `get_model_list` response into a dictionary mapping human-readable variation strings to model objects.

### Shopee's tier structure
```
tier_variation: [
  { "name": "Color", "option_list": [{"option": "Black"}, {"option": "White"}] },
  { "name": "Size",  "option_list": [{"option": "S"}, {"option": "M"}, {"option": "L"}] }
]

model: [
  { "tier_index": [0, 1], ... }   -> "Black M"
  { "tier_index": [1, 2], ... }   -> "White L"
]
```

Each model's `tier_index` array maps to positions in `tier_variation[pos].option_list[idx]`. The function joins the resolved option values with spaces to produce the variation string.

### resolve_variation_key() (lines 590-599)
Fallback matcher: if OpenAI returns a variation string that doesn't exactly match a lookup key, this function tries case-insensitive, whitespace-normalized comparison.

---

## 9. URL Parsing

Three Shopee URL formats are supported, handled by `canonicalize_shopee_link()`:

| Format | Example | Regex |
|--------|---------|-------|
| **Path format** | `https://shopee.com.my/product/123/456` | `^/product/(\d+)/(\d+)` |
| **i.dot format** | `https://shopee.com.my/item-name-i.123.456` | `i\.(\d+)\.(\d+)` |
| **Canonical** | `shopee.com.my/product/123/456` | `shopee\.com\.my/product/(\d+)/(\d+)` |

All formats are normalized to `https://shopee.com.my/product/{shop_id}/{item_id}`, then `parse_shop_item_from_canonical()` extracts the integer `(shop_id, item_id)` tuple.

---

## 10. Logging & Output

### Log file naming
- **DRY_RUN mode:** `variation_preprocess_dryrun.log` (append mode)
- **LIVE mode:** `variation_preprocess_{YYYYMMDD_HHMMSS}_live.log` (write mode, new file each run)

### Log directory
`variation_preprocessing_logs/` (relative to script directory)

### Log format
```
2026-03-06 05:10:40 | INFO | Phase 0: Loaded 48 rows with comp_variation='-'
```

### Dual output
All log messages go to both **console (stdout)** and **log file** simultaneously.

### Dry run output
When `DRY_RUN = True`:
- Bordered section headers with DRY RUN label
- Individual `[BACKFILL]` entries showing what would change
- Summary tables with all counters

---

## 11. Phase Differences Summary

| Aspect | Phase 0 (comp_variation = '-') | Phase 1 (normal) |
|--------|-------------------------------|-------------------|
| Fetch function | `fetch_dash_comp_variation_rows()` | `fetch_candidate_rows()` |
| Input signal | `comp_product` (product name) | `comp_variation` (variation text) |
| OpenAI function | `call_openai_dash()` | `call_openai()` |
| Prompt template | `PROMPT_TEMPLATE_DASH` | `PROMPT_TEMPLATE` |
| AI response fields | `comp_product_name`, `match`, `confidence` | `original`, `match`, `confidence` |
| Deduplication key | Unique `comp_product` names | Unique `comp_variation` strings |
| Single-model handling | Set `our_variation = '-'` | Set `our_variation = '-'` |
| Confidence threshold | Not applied (all accepted) | Applied (`CONFIDENCE_THRESHOLD = 0`) |

---

## 12. Counter Semantics

### Phase 0 counters
| Counter | Meaning |
|---------|---------|
| `candidate_rows` | Total rows fetched from DB |
| `product_groups` | Unique `(shop_id, item_id)` groups |
| `single_model_fills` | Rows set to `-` because our product has only 1 variation |
| `ai_matched` | Rows matched via OpenAI |
| `skipped_no_variations` | Rows skipped because our product returned no variation strings from API |
| `failed_groups` | Groups that threw an exception (rolled back) |

### Phase 1 counters
| Counter | Meaning |
|---------|---------|
| `candidate_rows` | Total rows fetched from DB |
| `product_groups` | Unique `(shop_id, item_id)` groups |
| `skipped_bad_url` | Rows with unparseable `our_link` URLs |
| `skipped_no_api_variations` | Rows skipped because API returned no variation data |
| `openai_calls` | Number of OpenAI API requests made |
| `matched_above_threshold` | Variations successfully matched above confidence threshold |
| `below_threshold_skipped` | Matches rejected for low confidence |
| `no_match_in_lookup` | AI returned a variation string not found in our lookup |
| `updated_rows` | Actual DB rows updated |
| `failed_groups` | Groups that threw an exception (rolled back) |

---

## 13. Runtime Behavior

### Typical production run (from logs)
- **Mar 5:** Phase 0: 48 rows, Phase 1: 6,999 rows, 409 OpenAI calls, 6,840 updated, 1 failed group
- **Mar 6:** Phase 0: 0 rows, Phase 1: 41 rows, 3 groups, 21 updated, 0 failed

### Performance characteristics
- Shopee API calls: 1 per product group (to get model list)
- OpenAI calls: ~1 per 20 unique comp_variations per group (chunked by `MAX_VARIATIONS_PER_PROMPT`)
- Sleep overhead: 0.15s per Shopee call + 0.5s per OpenAI call
- Total runtime varies: seconds (few rows) to hours (thousands of rows with many groups)

### Memory
- Loads up to 50,000 rows into memory at once (`BATCH_SIZE`)
- Groups are processed sequentially; only one group's data is active at a time in API/AI calls

---

## 14. Configuration Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `DRY_RUN` | `False` | Live mode (commits to DB) |
| `OPENAI_MODEL` | `o4-mini` | OpenAI model for fuzzy matching |
| `BATCH_SIZE` | `50000` | Max rows to fetch per phase |
| `CONFIDENCE_THRESHOLD` | `0` | Accept all matches (Phase 1 only) |
| `MAX_VARIATIONS_PER_PROMPT` | `20` | Max variations per OpenAI API call |
| `SLEEP_BETWEEN_SHOPEE_CALLS` | `0.15s` | Rate limit for Shopee API |
| `SLEEP_BETWEEN_OPENAI_CALLS` | `0.5s` | Rate limit for OpenAI API |
| `LOCAL_TZ` | `UTC+8` | Malaysia timezone |
| `TABLE_NAME` | `Shopee_Comp` | Target table in AllBots database |

---

## 15. Error Recovery

### Token expiry (HTTP 403)
`run_with_token_retry()` handles this automatically:
1. Catch `ShopeeForbiddenError`
2. Call Shopee OAuth refresh endpoint to get new access token
3. Update `requestDatabase.ShopeeTokens` with new token
4. If OAuth refresh fails, reload all tokens from DB
5. Retry the original API call once

### Group-level failure
- Each `(shop_id, item_id)` group is an independent transaction
- On any exception: rollback that group, log the error, continue to next group
- Failed groups are counted but do not stop the pipeline

### OpenAI failures
- Network errors, JSON parse errors, and empty responses all return `[]`
- The group continues processing (other chunks may still succeed)
- No retry logic for OpenAI calls

### Script-level crash
- `preprocess_wrapper.py` catches `BaseException` including `SystemExit`
- `scheduler.py` retries the job up to 2 times with 30-second gaps
- Pipeline checkpoint allows resuming from this job on next run

---

## 16. File Ecosystem

| File | Purpose |
|------|---------|
| `our_variation_preprocessing.py` | Main script (this documentation) |
| `preprocess_wrapper.py` | Crash wrapper (18 lines) |
| `scheduler.py` | Pipeline orchestrator — runs this as Job #2 |
| `chain_today_pipeline.ps1` | Waits for ResumePipeline2, launches scheduler |
| `run_variation_preprocessing.bat` | Manual execution runner |
| `run_preprocess.bat` | Scheduled task runner |
| `wait_and_run_variation.bat` | Dependency-aware runner (waits for PID) |
| `variation_preprocessing_logs/` | Log output directory |

---

## 17. Key Design Notes for Debugging

**1. Phase execution order is fixed.**
Phase 0 (`comp_variation = '-'`) always runs before Phase 1 (normal matching). If Phase 0 crashes fatally, Phase 1 will not execute — the `main()` function wraps both in a single try/except block and exits with code 1.

**2. The UPDATE has an idempotent guard.**
`update_our_variation()` includes `WHERE our_variation IS NULL OR TRIM(our_variation) = ''`. If another process or a prior run already filled the value, the UPDATE silently affects 0 rows. This prevents overwriting and makes reruns safe.

**3. CONFIDENCE_THRESHOLD is currently 0.**
Every match from OpenAI is accepted regardless of confidence score. The threshold check still exists in code (Phase 1 only) but is effectively disabled. If false matches become a problem, raise this value.

**4. Phase 0 does NOT apply the confidence threshold.**
Unlike Phase 1, Phase 0's `process_dash_comp_variations()` has no confidence check. All AI results are accepted unconditionally.

**5. Single-model products always get `our_variation = '-'`.**
In both phases, if our product has only 1 variation (or no tiers at all), the script sets `our_variation = '-'` with confidence 100. This is not an AI decision — it's a deterministic shortcut.

**6. The 50,000 row BATCH_SIZE is per phase, not total.**
Phase 0 fetches up to 50K rows AND Phase 1 fetches up to 50K rows independently. On a day with heavy data, up to 100K rows could be processed in a single run.

**7. Grouped transactions mean partial commits are possible.**
If the script processes 100 groups and group #50 fails, groups 1-49 are committed and groups 51-100 continue processing. Only group #50's changes are rolled back. The final counters reflect this mixed state.

**8. OpenAI chunks are 20 variations at a time.**
`MAX_VARIATIONS_PER_PROMPT = 20` means a group with 60 unique comp_variations triggers 3 OpenAI API calls. If one chunk fails, the other chunks still proceed.

**9. `resolve_variation_key()` is the fuzzy safety net.**
If OpenAI returns a variation string that doesn't exactly match our lookup (e.g., different casing or extra spaces), this function attempts case-insensitive whitespace-normalized matching before giving up.

**10. Three URL formats are handled, but only `.com.my` domain.**
The URL parsers are hardcoded for `shopee.com.my`. URLs for other Shopee domains (SG, TH, etc.) will be rejected and counted as `skipped_bad_url`.

**11. Token refresh is two-tier.**
On HTTP 403: first tries Shopee OAuth refresh (fast, targeted), then falls back to reloading all tokens from DB (slower but comprehensive). Only 1 retry is allowed per group.

**12. The script does NOT create rows — it only updates.**
No INSERT statements exist. It reads existing rows and updates the `our_variation` column. If the upstream scraper hasn't run, there's nothing to process.

**13. `comp_product` column is only used in Phase 0.**
Phase 0 reads `comp_product` (the competitor's product name) for AI matching context. Phase 1 ignores this column entirely and works only with `comp_variation`.

**14. Dry run logs to a single appended file.**
In `DRY_RUN = True` mode, all runs append to `variation_preprocess_dryrun.log`. In live mode, each run creates a new timestamped file. Check the correct log file when debugging.

**15. The sheet_name filter creates different time windows.**
VVIP/VIP/NEW_ITEMS get a 30-day window; LINKS_INPUT gets 28 days. If rows are older than these windows, they are silently excluded from processing.

**16. OpenAI response cleanup handles template artifacts.**
The double-brace replacement in both `call_openai()` and `call_openai_dash()` fixes artifacts from Python's `.format()` on the prompt template strings.

**17. The `_parsed_shop_id` and `_parsed_item_id` fields are ephemeral.**
These are injected into the row dicts during processing (prefixed with `_`) for internal grouping. They are not database columns and don't persist.

**18. Phase 0 deduplicates by `comp_product` name.**
Multiple rows with the same competitor product name in the same group share a single AI match. The AI result is applied to all rows with that name, reducing API calls.

**19. Phase 1 deduplicates by `comp_variation` string.**
Similarly, multiple rows with the same `comp_variation` value in the same group share one AI match result. This is the primary optimization for reducing OpenAI costs.

**20. Exit code 0 means both phases completed without fatal errors.**
Individual group failures (logged and counted in `failed_groups`) do NOT cause a non-zero exit code. Only an unhandled exception that escapes the try/except in `main()` triggers exit code 1. Check `failed_groups` in logs even when exit code is 0.
