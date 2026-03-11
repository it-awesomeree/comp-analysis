# Comp Variation Matcher Script (`ca_comp_variation_matcher.py`) — Full Analysis

**Script**: `ca_comp_variation_matcher.py` (1,421 lines, ~50KB)  
**Location**: `C:\Users\Admin\Desktop\ca_sg\ca_comp_variation_matcher.py`  
**VM**: VM TT (`192.168.144.135`, WinRM port `65504`)  
**Jira**: AW-299 (Melinda) / AW-266, related PR #353 (AW-332)  
**Doc Reference**: Variation Matching Bot Technical Documentation V3.1

---

## 1. Purpose

AI-powered bot that matches **competitor product variations** to **our product variations** for the Shopee MY competitive analysis system. It determines which competitor SKU/variation corresponds to which of our SKU/variations so that price/sales comparisons are meaningful at the variation level (e.g., "Black XL" matches to "Hitam XL").

---

## 2. Pipeline Context

*Source: `ca_sg_pipeline.py` — not from this script itself*

This is **Script #4 (final)** in the `ca_sg_pipeline.py` pipeline, triggered by the `ca_product_info_daily` Windows Scheduled Task at **midnight daily**.

| # | Script ID | Script File | Timeout | Purpose |
|---|-----------|-------------|---------|---------|
| 1 | `product_info` | `ca_product_info.py` | 2h | Refresh product info from Shopee API |
| 2 | `sales_sync` | `ca_my_products_sales_sync.py` | 2h | URL normalization + sales sync |
| 3 | `ads_metrics` | `ca_shopee_ads_metrics.py` | 3h | Ads metrics scraping (Selenium) |
| **4** | **`comp_variation_matcher`** | **`ca_comp_variation_matcher.py`** | **2h** | **AI variation matching (`--live`)** |

The pipeline runs scripts sequentially with **60s rest** between each. Each script gets **1 retry** with a **120s delay** before retry. Script #4 is invoked with the `--live` flag.

---

## 3. Configuration Constants

*Source: Lines 64-124 (Section 1 of the script)*

### 3.1 LLM Configuration

| Constant | Value | Line |
|----------|-------|------|
| `ANTHROPIC_API_KEY` | From env var `ANTHROPIC_API_KEY` (default `""`) | 67 |
| `ANTHROPIC_API_URL` | `https://api.anthropic.com/v1/messages` | 68 |
| `ANTHROPIC_MODEL` | `claude-sonnet-4-5-20250929` | 69 |
| `ANTHROPIC_VERSION` | `2023-06-01` | 70 |
| `OPENAI_KEY` | From env var `OPENAI_API_KEY` (default `""`) | 73 |
| `OPENAI_MODEL` | `gpt-4o-mini` | 74 |
| `OPENAI_API_URL` | `https://api.openai.com/v1/chat/completions` | 75 |

### 3.2 Database Configuration

| Constant | Value | Line |
|----------|-------|------|
| `DB_CFG["host"]` | `34.142.159.230` | 79 |
| `DB_CFG["user"]` | `root` | 80 |
| `DB_CFG["database"]` | `AllBots` | 82 |

**Note**: The script's docstring (lines 20-24) references `webapp_test.Shopee_My_Products` etc., but the actual `DB_CFG` connects to database **`AllBots`**. All queries use bare table names with no database prefix. The docstring is outdated/inaccurate.

### 3.3 Table Names

| Constant | Value | Line |
|----------|-------|------|
| `TABLE_MY_PRODUCTS` | `Shopee_My_Products` | 85 |
| `TABLE_COMP_DATA` | `Shopee_Comp_Data` | 86 |
| `TABLE_MATCH` | `Shopee_Variation_Match` | 87 |
| `TABLE_RUNS` | `Shopee_Variation_Match_Runs` | 88 |

### 3.4 Pre-Filter and Confidence Thresholds

| Constant | Value | Line |
|----------|-------|------|
| `PRE_FILTER_THRESHOLD` | `0.10` | 91 |
| `PRE_FILTER_AUTO_CONFIDENCE` | `5` | 92 |
| `CONFIDENCE_MATCH_MIN` | `80` (80-100 = MATCH) | 95 |
| `CONFIDENCE_UNSURE_MIN` | `40` (40-79 = UNSURE; 0-39 = NO_MATCH) | 96 |
| `TOP_N` | `3` | 99 |
| `STALENESS_THRESHOLD_DAYS` | `7` | 102 |

### 3.5 LLM Concurrency and Retry

| Constant | Value | Line |
|----------|-------|------|
| `MAX_CONCURRENT_LLM` | `5` | 105 |
| `LLM_TIMEOUT_SECONDS` | `60` | 106 |
| `LLM_RETRY_DELAY` | `2.0` | 107 |
| `LLM_MAX_RETRIES` | `1` | 108 |
| `LLM_SLEEP_BETWEEN_CALLS` | `0.3` | 109 |

### 3.6 DB Deadlock Retry

| Constant | Value | Line |
|----------|-------|------|
| `MAX_DEADLOCK_RETRIES` | `3` | 112 |
| `DEADLOCK_RETRY_BASE_DELAY` | `0.5` | 113 |

### 3.7 Other

| Constant | Value | Line |
|----------|-------|------|
| `DRY_RUN` | `False` | 116 |
| `MODEL_VERSION` | `v1-sonnet` | 117 |
| `LOCAL_TZ` | `UTC+8` (Asia/Kuala_Lumpur) | 120 |
| `LOG_DIR` | `comp_variation_matcher_logs` | 123 |

**Important -- `DRY_RUN` default discrepancy**: The module-level default is `DRY_RUN = False` (line 116), meaning the script runs in **LIVE mode by default**. However, the `--live` flag's argparse help text (line 1351) says `"Commit to database (default: dry-run/preview mode)"`, and the `main()` function (line 1361) only does `if args.live: DRY_RUN = False` -- which is a no-op since it is already `False`. The `--live` flag has no effect. The docstring and argparse help text are outdated relative to the code.

---

## 4. Database Reads and Writes

### 4.1 Tables READ

| # | Function | Table | Query | Lines |
|---|----------|-------|-------|-------|
| R1 | `fetch_our_variations()` | `Shopee_My_Products` | `SELECT DISTINCT our_variation, product_name, sku FROM Shopee_My_Products WHERE our_link = %s AND status = 'active'` | 810-820 |
| R2 | `fetch_comp_variations()` | `Shopee_Comp_Data` | Step 1: `SELECT MAX(date_taken) as max_date FROM Shopee_Comp_Data WHERE comp_link = %s AND status = 'active'` / Step 2: `SELECT DISTINCT comp_variation, comp_product FROM Shopee_Comp_Data WHERE comp_link = %s AND date_taken = %s AND status = 'active'` | 828-855 |
| R3 | `get_manual_overrides()` | `Shopee_Variation_Match` | `SELECT DISTINCT our_variation FROM Shopee_Variation_Match WHERE our_link = %s AND comp_link = %s AND is_manual_override = 1 AND status = 'active'` | 790-802 |
| R4 | `check_concurrent_runs()` | `Shopee_Variation_Match_Runs` | `SELECT id FROM Shopee_Variation_Match_Runs WHERE our_link = %s AND comp_link = %s AND status IN ('QUEUED', 'RUNNING') LIMIT 1` | 693-710 |
| R5 | `discover_listing_pairs()` | `Shopee_My_Products` JOIN `Shopee_Comp_Data` | `SELECT DISTINCT m.our_link, c.comp_link FROM Shopee_My_Products m INNER JOIN Shopee_Comp_Data c ON m.our_link = c.our_link WHERE m.status = 'active' AND c.status = 'active'` | 1221-1227 |
| R6 | `discover_listing_pairs()` | `Shopee_Variation_Match_Runs` | `SELECT id FROM Shopee_Variation_Match_Runs WHERE our_link = %s AND comp_link = %s AND status = 'DONE' LIMIT 1` (per pair, to filter already-done) | 1236-1241 |

### 4.2 Tables WRITTEN

| # | Function | Table | Operation | Details | Lines |
|---|----------|-------|-----------|---------|-------|
| W1 | `create_run()` | `Shopee_Variation_Match_Runs` | **INSERT** | Columns: `our_link, comp_link, status='QUEUED', model_version, parameters` (JSON), `data_staleness_warning`. Returns auto-increment `id`. | 718-740 |
| W2 | `update_run_status()` | `Shopee_Variation_Match_Runs` | **UPDATE** | Sets `status` (QUEUED->RUNNING->DONE/FAILED). Optionally sets `error_message` (truncated to 2000 chars) or `skip_reason` (truncated to 255 chars). Matched by `WHERE id = %s`. | 750-782 |
| W3 | `deactivate_old_matches()` | `Shopee_Variation_Match` | **UPDATE** | Sets `status = 'inactive'`. Normal mode: `WHERE our_link = %s AND comp_link = %s AND is_manual_override = 0 AND status = 'active'`. With `force_rerun`: drops the `is_manual_override = 0` condition (deactivates all including manual overrides). | 875-907 |
| W4 | `store_results()` | `Shopee_Variation_Match` | **INSERT** with **ON DUPLICATE KEY UPDATE** | INSERT columns: `run_id, our_link, our_variation, comp_link, comp_variation, rank, confidence_score, decision, reason, match_method='auto', is_manual_override=0, status='active'`. The 9 parameterized values are: `run_id, our_link, our_variation, comp_link, comp_variation, rank, confidence, decision, reason` (reason truncated to 2000 chars). ON DUPLICATE KEY UPDATE updates: `rank, confidence_score, decision, reason, match_method, is_manual_override`. Note: `status` is NOT updated on conflict -- existing status is preserved. Uses `executemany` for batch insert. | 912-965 |

### 4.3 Decorated DB Functions

The following write functions use the `@retry_on_deadlock()` decorator: `create_run()`, `update_run_status()`, `deactivate_old_matches()`, `store_results()`. The decorator retries on MySQL errno **1213** (deadlock) and **1205** (lock wait timeout), up to 3 retries with exponential backoff (base 0.5s). Lines 186-199.

---

## 5. Two-Stage Matching Pipeline

### 5.1 Stage 1: Deterministic Pre-Filter (Section 4 of script, lines 200-390)

A fast code-based similarity check that eliminates obvious non-matches before LLM evaluation.

**Pre-processing** (`normalize_variation()`, lines 265-278):
1. Remove emoji (via regex covering Unicode emoji blocks)
2. Lowercase + strip
3. Collapse whitespace
4. Tokenize on `[\s\-/,_|+:;()]+`
5. Expand abbreviations via lookup dict

**Abbreviation expansions** (`ABBREVIATIONS` dict, lines 204-224):
- English color abbrevs: `blk->black`, `wht->white`, `rd->red`, `blu->blue`, `grn->green`, `ylw->yellow`, `pnk->pink`, `gry->gray`, `grey->gray`, `brn->brown`, `org->orange`, `prpl->purple`, `slvr->silver`, `gld->gold`
- Malay->English colors: `hitam->black`, `putih->white`, `merah->red`, `biru->blue`, `hijau->green`, `kuning->yellow`, `kelabu->gray`, `coklat->brown`, `oren->orange`, `jingga->orange`, `ungu->purple`, `krim->cream`, `emas->gold`, `perak->silver`
- English size abbrevs: `sml->small`, `med->medium`, `lrg->large`, `xs->extra small`, `2xl->xxl`, `3xl->xxxl`
- Malay->English sizes: `kecil->small`, `sederhana->medium`, `besar->large`
- Unit abbrevs: `pcs->pieces`, `pc->piece`, `pk->pack`

**Category sets** used for category matching:
- `COLORS`: 30 values (black, white, red, blue, green, yellow, pink, gray, brown, orange, purple, silver, gold, cream, navy, beige, maroon, teal, khaki, charcoal, ivory, coral, burgundy, olive, tan, rose, lavender, turquoise, transparent, clear)
- `SIZES`: 13 values (s, m, l, xl, xxl, xxxl, small, medium, large, extra small, extra large, free size, one size)
- `MATERIALS`: 16 values (cotton, polyester, nylon, leather, plastic, metal, wood, steel, aluminium, aluminum, rubber, silicone, glass, ceramic, bamboo, stainless)

**Composite similarity score** (`compute_composite_similarity()`, lines 318-345):
```
composite = 0.5 * token_overlap + 0.3 * cat_score + 0.2 * edit_ratio
```

| Component | Weight | Function | Algorithm |
|-----------|--------|----------|-----------|
| Token overlap | **0.5** | `jaccard_similarity()` | Jaccard = intersection / union on token sets |
| Category match | **0.3** | `category_match_score()` | Average Jaccard across color/size/material (only categories where at least one side has data) |
| Edit distance | **0.2** | `edit_distance_ratio()` | `SequenceMatcher(None, s1, s2).ratio()` (character-level, 0-1.0) |

**Edge cases**:
- Both empty/dash -> returns `1.0` (product-level match for single-SKU items)
- One empty, one not -> returns `0.0`

**Pre-filter outcome** (`pre_filter_pairs()`, lines 348-385):
- Score >= `0.10` -> candidate passes to Stage 2 (LLM evaluation)
- Score < `0.10` -> auto `NO_MATCH` with confidence `5`
- Candidates sorted by score descending per our_variation

### 5.2 Stage 2: LLM Evaluation (Section 5 of script, lines 390-645)

AI-powered matching for candidates that passed the pre-filter.

**LLM prompt** (`LLM_PROMPT_TEMPLATE`, lines 392-422):  
Sends the LLM:
- Our product name, our variation, our SKU
- Competitor product name
- Numbered list of candidate competitor variations

The prompt instructs the LLM to:
- Focus on color, size, material, quantity, type attributes
- Handle English abbreviations and Malay-English translations
- Handle naming convention differences (e.g., "Black 39" vs "BLK-39" vs "Black / EU39")
- Handle measurement equivalences (e.g., 300x500cm = 3x5m)
- Give LOW confidence for vague labels ("Random Color", "Assorted", "-")
- Return ONLY valid JSON array: `[{"comp_variation": "...", "confidence": 0-100, "reason": "..."}]`

**LLM call chain** (`call_llm()`, line 541):
1. Try `call_claude_sonnet()` first -- if `ANTHROPIC_API_KEY` is empty string, returns `None` immediately (no API call made)
2. If Claude returns `None`, falls back to `call_openai_fallback()`
3. If both fail, `evaluate_candidates()` raises `RuntimeError`

**Claude API call details** (`call_claude_sonnet()`, lines 440-480):
- Endpoint: `https://api.anthropic.com/v1/messages`
- Headers: `x-api-key`, `anthropic-version: 2023-06-01`, `content-type: application/json`
- Payload: `model=claude-sonnet-4-5-20250929`, `max_tokens=4096`, `temperature=0`
- Retries on status 429 or 529 (rate limit / overloaded), up to 1 retry with 2s exponential delay
- Returns `data["content"][0]["text"]`

**OpenAI API call details** (`call_openai_fallback()`, lines 483-520):
- Endpoint: `https://api.openai.com/v1/chat/completions`
- Headers: `Authorization: Bearer {key}`, `Content-Type: application/json`
- Payload: `model=gpt-4o-mini`, `temperature=0`
- Same retry logic (429/529, 1 retry, 2s exponential)
- Returns `data["choices"][0]["message"]["content"]`

**Response parsing** (`parse_llm_response()`, lines 545-600):
1. Strips markdown code fences if present
2. Parses JSON
3. Validates each `comp_variation` against expected list (case-insensitive fallback match)
4. Clamps confidence to 0-100
5. Truncates reason to 500 chars
6. Unknown comp_variations are silently dropped

**Missing variation backfill** (`evaluate_candidates()`, lines 605-640):  
If the LLM did not return results for some expected comp_variations, they are backfilled with `confidence=0, reason="Not evaluated by LLM"`.

---

## 6. Confidence Calibration and Ranking (Section 6 of script, lines 645-690)

**Decision labels** (`apply_decision_label()`, lines 650-656):

| Confidence Range | Decision Label |
|-----------------|----------------|
| 80-100 | `MATCH` |
| 40-79 | `UNSURE` |
| 0-39 | `NO_MATCH` |

**Top-N selection** (`rank_and_select_top_n()`, lines 660-685):
- `N = min(TOP_N, total_comp_variations)` -> effectively min(3, total comp vars)
- Sorts all results (pre-filter auto-rejects + LLM results) by confidence descending
- Assigns rank 1, 2, 3 to the top entries
- Each result gets: `comp_variation, confidence, decision, reason, rank`

---

## 7. Orchestration Flow (Section 7 of script, lines 690-1200)

### Per listing pair (`process_listing_pair()`, lines 970-1200):

```
 1. Check concurrent runs (R4) -> skip if QUEUED/RUNNING exists
 2. Prepare params dict (no DB operation)
 3. Fetch our variations (R1) from Shopee_My_Products
 4. Fetch comp variations (R2) from Shopee_Comp_Data (latest date_taken only)
 5. Check data staleness (>7 days -> sets warning flag)
 6. Create run row in DB (W1) -- INSERT status=QUEUED
    (happens AFTER fetching variations because staleness_flag is needed)
 7. Update run -> RUNNING (W2)
 8. Edge case: zero our_vars or zero comp_vars -> DONE with skip_reason (W2)
 9. Check manual overrides (R3) -- skip manually overridden our_variations
    (unless force_rerun, which sets manual_vars to empty set)
10. If all variations have manual overrides -> DONE with skip_reason (W2)
11. Deactivate old auto-matched entries (W3)
12. Stage 1: Pre-filter all pairs (code-only, no DB ops)
13. Stage 2: LLM evaluation via ThreadPoolExecutor (max 5 concurrent)
    - Each our_variation is evaluated independently
    - Pre-filter rejects get confidence=5 auto NO_MATCH
    - LLM-eligible candidates get sent to Claude/OpenAI
    - 0.3s sleep between LLM calls within each thread
    - Results combined and ranked by top-N
14. Check failure rate -- if >50% of our_variations failed LLM -> run FAILED (W2)
15. Store results (W4) -- INSERT into Shopee_Variation_Match
16. Mark run DONE (W2)
```

**Exception handling**: If any unhandled exception occurs, the run is marked FAILED (W2) with `error_message` truncated to 2000 chars. The DB connection is always closed in `finally`.

**Summary dict returned per pair**: `{our_link, comp_link, status, our_variations, comp_variations, pre_filter_passed, pre_filter_rejected, llm_evaluated, results_stored, failed_variations}`

### Batch mode (`discover_listing_pairs()` + `run_batch()`, lines 1205-1300):

1. `discover_listing_pairs()` (R5 + R6):
   - Finds all `(our_link, comp_link)` pairs where both `Shopee_My_Products` and `Shopee_Comp_Data` have active data (via INNER JOIN on `our_link`)
   - If NOT `force_rerun`: filters out pairs that already have a DONE run in `Shopee_Variation_Match_Runs` (per-pair query)
   - If `force_rerun`: returns all pairs without filtering
2. `run_batch()`:
   - Iterates all pairs sequentially (no parallel pair processing)
   - Calls `process_listing_pair()` for each
   - Logs per-pair result and final batch summary (status counts + total stored)

---

## 8. Execution Modes (Section 9 of script, lines 1305-1421)

| Mode | How Invoked | Behavior |
|------|-------------|----------|
| **Batch** | No `--our-link`/`--comp-link` flags | Discovers all listing pairs, processes sequentially |
| **Single pair** | `--our-link` + `--comp-link` (both required) | Processes one specific listing pair |
| **Force rerun** | `--force-rerun` | Clears manual overrides, skips already-done filter, reprocesses everything |
| **Live** | `--live` | Sets `DRY_RUN = False` -- **but this is already the default, so this flag is a no-op** |

**Note on DRY_RUN gating**: When `DRY_RUN = True`, the following operations are skipped:
- `create_run()` returns 0 without INSERT
- `update_run_status()` logs but does not UPDATE
- `deactivate_old_matches()` returns 0 without UPDATE
- `store_results()` returns 0 without INSERT
- Dry run prints a match results summary table instead

Since `DRY_RUN = False` by default, all runs are live unless the module-level constant is manually changed.

---

## 9. External API Dependencies

| API | Endpoint | Auth | Purpose |
|-----|----------|------|---------|
| Anthropic (Claude) | `https://api.anthropic.com/v1/messages` | `x-api-key` header from `ANTHROPIC_API_KEY` env var | Primary LLM for variation matching |
| OpenAI | `https://api.openai.com/v1/chat/completions` | `Authorization: Bearer` from `OPENAI_API_KEY` env var | Fallback LLM when Claude unavailable |
| MySQL | `34.142.159.230` (Google Cloud SQL) | `root` user, password in script | All database reads and writes |

---

## 10. Python Dependencies (from imports, lines 45-62)

**External packages** (require pip install):
- `mysql.connector` (from `mysql-connector-python`)
- `requests`

**Standard library** (no install needed):
- `argparse`, `json`, `logging`, `os`, `re`, `sys`, `time`, `traceback`
- `collections.defaultdict`
- `concurrent.futures.ThreadPoolExecutor`, `concurrent.futures.as_completed`
- `datetime.datetime`, `datetime.timedelta`, `datetime.timezone`
- `difflib.SequenceMatcher`
- `typing` (Dict, List, Optional, Set, Tuple, Any)

**No Selenium** -- this is a pure data processing + API script (unlike Script #3 `ca_shopee_ads_metrics.py`).

---

## 11. Logging (Section 2 of script, lines 127-155)

**Log directory**: `comp_variation_matcher_logs/` (relative to script location)

**Log file naming**:
- Live mode: `matcher_YYYYMMDD_HHMMSS_live.log` (new file per run, mode `w`)
- Dry run mode: `matcher_dryrun.log` (appended, mode `a`)

**Log format**: `%(asctime)s | %(levelname)s | %(message)s` (datefmt: `%Y-%m-%d %H:%M:%S`)

**Outputs to**: Both console (stdout) and file simultaneously.

---

## 12. Safety Mechanisms

| Mechanism | Code Location | Detail |
|-----------|---------------|--------|
| **Concurrent run prevention** | `check_concurrent_runs()`, line 693 | Checks for existing QUEUED/RUNNING run for the same (our_link, comp_link) pair. Skips if found. |
| **Manual override preservation** | `get_manual_overrides()`, line 785 + orchestration line 1065 | Skips our_variations with `is_manual_override = 1` unless `--force-rerun`. |
| **Staleness warning** | `check_data_staleness()`, line 858 | If comp data's latest `date_taken` is >7 days old, sets `data_staleness_warning = 1` on the run row. |
| **Failure rate check** | Orchestration line 1167 | If >50% of our_variations fail LLM evaluation, entire run is marked FAILED. |
| **LLM fallback** | `call_llm()`, line 541 | Claude Sonnet -> OpenAI GPT-4o-mini. If Claude API key is empty, skips immediately to OpenAI. |
| **LLM response validation** | `parse_llm_response()`, line 548 | Strips markdown fences, parses JSON, validates `comp_variation` names against expected list (case-insensitive), clamps confidence 0-100, truncates reason to 500 chars. |
| **Deadlock retry** | `@retry_on_deadlock()` decorator, line 186 | Retries on MySQL errno 1213 (deadlock) and 1205 (lock wait timeout), up to 3 retries with exponential backoff (0.5s base). Applied to `create_run`, `update_run_status`, `deactivate_old_matches`, `store_results`. |
| **DRY_RUN gating** | Throughout W1-W4 functions | All DB write functions check `DRY_RUN` and skip with log message if True. |

---

## 13. Script Sections Summary

| Section | Lines | Content |
|---------|-------|---------|
| Docstring | 1-42 | Purpose, Jira refs, usage, reads/writes (note: DB name outdated) |
| Section 1: Config | 64-124 | All constants (API, DB, thresholds, retry settings) |
| Section 2: Logging | 127-155 | Log setup, file naming, dual output |
| Section 3: DB Helpers | 158-199 | `get_db_connection()`, `@retry_on_deadlock()` decorator |
| Section 4: Pre-Filter | 200-390 | Abbreviations, category sets, normalization, tokenization, similarity scoring, `pre_filter_pairs()` |
| Section 5: LLM Evaluation | 390-645 | Prompt template, Claude/OpenAI API calls, response parsing, `evaluate_candidates()` |
| Section 6: Calibration | 645-690 | Decision labels, `rank_and_select_top_n()` |
| Section 7: Orchestration | 690-1200 | All DB read/write functions, `process_listing_pair()` full flow |
| Section 8: Batch Mode | 1205-1300 | `discover_listing_pairs()`, `run_batch()` |
| Section 9: Entry Point | 1305-1421 | `parse_args()`, `main()` |
