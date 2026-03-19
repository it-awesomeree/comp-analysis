# VM3 Smart Scheduler - Pipeline Reference (Current)

**VM**: VM3 (`DESKTOP-6C5NRUO`)
**Task Name**: `Sales Data`
**Last Verified**: 2026-03-19 (MYT, UTC+8)
**Source of Truth**: Live VM3 `Sales Data` scheduled task + `C:\Users\Admin\Desktop\Shopee Comp My links Api\scheduler.py`

---

## Scope

This document covers the **normal recurring scheduler pipeline only**.

- Included: the daily 4:00 PM MYT pipeline configuration and the current `scheduler.py` job chain.
- Excluded: one-off/special-case triggers and temporary manual actions.

---

## 1. Task Scheduler Configuration (`Sales Data`)

### Recurring Trigger (normal schedule)

- **Frequency**: Daily
- **Time**: 4:00 PM MYT (`Asia/Kuala_Lumpur`, UTC+8)
- **Calendar interval**: Every 1 day

### Action

- **Command**: `C:\Windows\System32\cmd.exe`
- **Arguments**:

```bat
/c ""C:\Users\Admin\AppData\Local\Programs\Python\Python313\python.exe" "C:\Users\Admin\Desktop\Shopee Comp My links Api\scheduler.py""
```

- **WorkingDirectory**: Not explicitly set in the task action

### Key Runtime Settings

| Setting | Value | Operational effect |
|---|---|---|
| `LogonType` | `Interactive` | Task requires interactive user session context |
| `MultipleInstances` | `IgnoreNew` | New trigger is skipped if previous run is still active |
| `StartWhenAvailable` | `True` | Missed run can start when machine becomes available |
| `ExecutionTimeLimit` | `PT0S` | No Task Scheduler-level timeout |

---

## 2. Scheduler Script Runtime (`scheduler.py`)

**Path**: `C:\Users\Admin\Desktop\Shopee Comp My links Api\scheduler.py`

### Core Behavior

- Sequential chain execution (job-by-job)
- 5-minute gap between jobs
- Pre-flight token refresh before pipeline starts
- Token refresh again before each subsequent job
- Retry on failure (`MAX_JOB_RETRIES=2`, so up to 3 total attempts/job)
- Per-job timeout enforcement
- CPU watchdog (poll/kill if stalled)
- Checkpoint/resume using `scheduler_checkpoint.json` (same-day resume)

### Runtime Constants (current)

| Constant | Value |
|---|---|
| `GAP_MINUTES` | `5` |
| `MAX_JOB_RETRIES` | `2` |
| `RETRY_WAIT_SECONDS` | `30` |
| `WATCHDOG_POLL` | `60` |
| `WATCHDOG_STALL_LIMIT` | `10` |
| `WATCHDOG_GRACE` | `120` |
| `TOKEN_REFRESH_RETRIES` | `3` |
| `TOKEN_REFRESH_TIMEOUT` | `120` |

---

## 3. Active Pipeline (5 Scripts)

The live `JOBS` tuple currently contains **5 scripts** in this exact order:

| # | Script | Timeout (`timeout_minutes`) | Purpose |
|---|---|---:|---|
| 1 | `ca_shopee_listing_to_db.py` | 120 | Seed new product rows |
| 2 | `our_variation_preprocessing.py` | 300 | Backfill `our_variation` before AI match |
| 3 | `ca_ai_variation_match.py` | 60 | Resolve IDs / AI variation match |
| 4 | `shopee_comp_shopee_sales.py` | 90 | Shopee order sales sync |
| 5 | `Shopee-mylinks-sales-data-merged.py` | 540 | Product info + SiteGiant merged sales sync |

### Important change from prior documentation

The following scripts are **not in the active scheduler `JOBS` pipeline**:

- `ca_shopee_ads_metrics.py`
- `ca_similarity_check.py`

These two scripts were previously documented as Job 6 and Job 7, but are currently disabled from the pipeline chain.

---

## 4. Pipeline Execution Flow

1. Task Scheduler triggers `Sales Data` at 4:00 PM MYT.
2. `cmd.exe` launches `python.exe scheduler.py`.
3. Scheduler loads same-day checkpoint (if present) to resume from last unfinished job.
4. Scheduler performs pre-flight token refresh.
5. Scheduler runs each job sequentially using `run_script_with_retry(...)`.
6. Between jobs, scheduler waits `GAP_MINUTES` (5 min).
7. On completion, scheduler clears checkpoint and writes pipeline summary to log.

---

## 5. Files to Check During Incidents

All under `C:\Users\Admin\Desktop\Shopee Comp My links Api\`:

- `scheduler.py` - pipeline definition and control logic
- `scheduler.log` - orchestrator log with start/end/retry/timeout/watchdog entries
- `scheduler_checkpoint.json` - resume checkpoint state (if interrupted)

---

## 6. Quick Validation Commands (VM3)

```powershell
Get-ScheduledTaskInfo -TaskName "Sales Data"
Get-ScheduledTask -TaskName "Sales Data" | Select-Object -ExpandProperty Triggers
Get-Content "C:\Users\Admin\Desktop\Shopee Comp My links Api\scheduler.log" -Tail 80
```

For job-chain truth, always validate against the live `JOBS` tuple in:

`C:\Users\Admin\Desktop\Shopee Comp My links Api\scheduler.py`
