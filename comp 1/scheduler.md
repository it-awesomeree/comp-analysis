# VM3 Smart Scheduler — Comprehensive Debugging Reference

**VM**: VM3 (DESKTOP-6C5NRUO) | Windows 10 Education 10.0.19045 | 4 GB RAM | Python 3.13.5
**Task Created**: January 13, 2026
**Document Date**: March 6, 2026

---

## 1. Task Scheduler Configuration ("Sales Data")

### Raw XML Export

```xml
<?xml version="1.0" encoding="UTF-16"?>
<Task version="1.3" xmlns="http://schemas.microsoft.com/windows/2004/02/mit/task">
  <RegistrationInfo>
    <Date>2026-01-13T16:59:51.8816509</Date>
    <Author>DESKTOP-6C5NRUO\Admin</Author>
    <URI>\Sales Data</URI>
  </RegistrationInfo>
  <Principals>
    <Principal id="Author">
      <UserId>S-1-5-21-3048672443-3499528760-1085260041-1000</UserId>
      <LogonType>InteractiveToken</LogonType>
    </Principal>
  </Principals>
  <Settings>
    <DisallowStartIfOnBatteries>true</DisallowStartIfOnBatteries>
    <StopIfGoingOnBatteries>true</StopIfGoingOnBatteries>
    <ExecutionTimeLimit>PT0S</ExecutionTimeLimit>
    <MultipleInstancesPolicy>IgnoreNew</MultipleInstancesPolicy>
    <StartWhenAvailable>true</StartWhenAvailable>
    <IdleSettings>
      <Duration>PT10M</Duration>
      <WaitTimeout>PT1H</WaitTimeout>
      <StopOnIdleEnd>true</StopOnIdleEnd>
      <RestartOnIdle>false</RestartOnIdle>
    </IdleSettings>
    <UseUnifiedSchedulingEngine>true</UseUnifiedSchedulingEngine>
  </Settings>
  <Triggers>
    <CalendarTrigger>
      <StartBoundary>2026-03-07T16:00:00+08:00</StartBoundary>
      <ScheduleByDay><DaysInterval>1</DaysInterval></ScheduleByDay>
    </CalendarTrigger>
  </Triggers>
  <Actions Context="Author">
    <Exec>
      <Command>C:\Windows\System32\cmd.exe</Command>
      <Arguments>/c ""C:\Users\Admin\AppData\Local\Programs\Python\Python313\python.exe"
        "C:\Users\Admin\Desktop\Shopee Comp My links Api\scheduler.py""</Arguments>
    </Exec>
  </Actions>
</Task>
```

### Settings Explained for Debugging

| Setting | Value | Debugging Implication |
|---|---|---|
| **StartBoundary** | `2026-03-07T16:00:00+08:00` | The date when the daily trigger begins. If set to a future date, no runs fire until then. **Check this first when pipeline doesn't fire.** |
| **DaysInterval** | 1 | Fires every day at the StartBoundary time (4:00 PM MYT) |
| **LogonType** | `InteractiveToken` | **Task only runs when Admin is logged into the GUI session.** If VM is at the lock screen or logged out, the task silently won't fire. |
| **MultipleInstancesPolicy** | `IgnoreNew` | If the previous pipeline run (10-11h) is still running when the next 4 PM trigger fires, the new run is **silently skipped**. No error logged in scheduler.log. Task Scheduler records result `2147946720`. |
| **ExecutionTimeLimit** | `PT0S` | No time limit at the Task Scheduler level. A stuck pipeline runs forever until manually killed. Per-job timeouts in scheduler.py are the actual safety net. |
| **StartWhenAvailable** | `true` | If a scheduled run is missed (e.g., VM was off at 4 PM), it fires as soon as the VM comes back. Good for recovery. |
| **DisallowStartIfOnBatteries** | `true` | Could silently prevent execution if VM reports battery power. |
| **StopIfGoingOnBatteries** | `true` | Could kill a running pipeline mid-execution if power state changes. |
| **WorkingDirectory** | **(not set)** | CWD defaults to `C:\Windows\System32`. The scheduler.py compensates via `Path(__file__).resolve().parent` and sets `cwd` per subprocess. Compare with `RunSchedulerNow` which correctly sets WorkingDirectory. |
| **RestartCount** | 0 (default) | No automatic retry at Task Scheduler level if the task crashes. |
| **RunOnlyIfIdle** | `false` | IdleSettings (StopOnIdleEnd, etc.) are irrelevant since RunOnlyIfIdle is false. |

### Exit Codes Reference

| Code | Meaning | What to check |
|---|---|---|
| `0` | SUCCESS | All good |
| `1` | GENERAL FAILURE | Check scheduler.log tail |
| `267009` | CURRENTLY RUNNING | Task is mid-execution |
| `267011` | TASK NOT STARTED | Never ran (e.g., BootResumePipeline) |
| `267014` | TASK TERMINATED | Task was manually stopped or killed |
| `2147946720` | TASK ALREADY RUNNING | IgnoreNew blocked a new instance |
| `4294967295` (-1) | CRASHED | Process crashed. Check scheduler.log for last entry before crash |

---

## 2. Scheduler Script Architecture (`scheduler.py`)

**Path**: `C:\Users\Admin\Desktop\Shopee Comp My links Api\scheduler.py`
**Version on disk**: v2.4 (447 lines)
**Version in recent log runs**: v2.2 — v2.4 will activate on the next fresh scheduler.py launch (see Known Issue #1)

### Configuration Constants

```python
TZ = "Asia/Kuala_Lumpur"                    # All timestamps in MYT (UTC+8)
GAP_MINUTES = 5                             # Wait between jobs
SAFETY_TIMEOUT_MIN = 360                    # Default safety net (overridden per-job)
MAX_JOB_RETRIES = 2                         # 2 retries = 3 total attempts
RETRY_WAIT_SECONDS = 30                     # Wait between retry attempts
TOKEN_REFRESH_URL = "https://employee.awesomeree.com.my/api/shopee/refresh-tokens"
CHATBOT_API_KEY = "b8c27f6840e1..."         # Hardcoded in plaintext
TOKEN_REFRESH_RETRIES = 3                   # With exponential backoff (10s, 20s)
TOKEN_REFRESH_TIMEOUT = 120                 # seconds
CHECKPOINT_MAX_AGE_HOURS = 18               # Checkpoint valid for 18h
```

### Execution Flow

```
Task Scheduler fires "Sales Data" at 4:00 PM
  +-- cmd.exe /c python scheduler.py
       |-- 1. acquire_lock() -- write PID to scheduler.lock
       |-- 2. load_checkpoint() -- check for interrupted run (18h max age)
       |-- 3. preflight_refresh_tokens() -- POST to web app (3 retries)
       |-- 4. For each job 1-7:
       |     |-- Per-job token refresh (except job 1)
       |     |-- save_checkpoint(job_index)
       |     |-- run_script_with_retry(job)
       |     |     |-- Attempt 1: subprocess.run(python, script, cwd=script_dir,
       |     |     |              timeout=job.timeout_minutes*60)
       |     |     |-- On fail: refresh tokens -> wait 30s -> retry
       |     |     +-- Up to 3 total attempts
       |     +-- Wait 5 min before next job
       |-- 5. clear_checkpoint()
       |-- 6. Log PIPELINE SUMMARY
       +-- 7. release_lock()
```

### How Scripts Are Executed

```python
subprocess.run(
    [sys.executable, str(script_path)],   # python.exe + full script path
    cwd=str(script_path.parent),          # CWD set to script directory
    shell=False,                          # Direct execution, not through cmd
    check=False,                          # Don't raise on non-zero exit
    timeout=timeout_minutes * 60          # Per-job safety timeout
)
```

### Key Files

All under `C:\Users\Admin\Desktop\Shopee Comp My links Api\`:

| File | Purpose | When to check |
|---|---|---|
| `scheduler.py` | Main pipeline orchestrator (v2.4) | When modifying pipeline logic |
| `scheduler.log` | Main log — all job starts/ends/failures | **Always first** |
| `scheduler.lock` | PID-based lock file, prevents concurrent instances | If "Another scheduler instance is already running" error |
| `scheduler_checkpoint.json` | Resume state: `{job_index, start_time, results, saved_at}` | If pipeline resumes mid-way or checkpoint is stale |
| `pipeline2_resume.log` | Log from ResumePipeline2 / RunPipeline2 helper tasks | When recovery helper tasks are involved |
| `overnight_runner.log` | Old runner (v1, deprecated since Feb 26) | Historical reference only |

### Concurrency Protection (v2.4)

The lock file mechanism uses Windows `kernel32.OpenProcess()` to check if the stored PID is alive:
- If PID alive -> refuses to start, logs error
- If PID dead -> takes over (stale lock), logs warning
- On exit -> releases lock (only if PID matches)

**Weakness**: If the scheduler is killed via `taskkill /f` or crashes hard, `release_lock()` never runs. The lock file remains, but the next run detects the stale PID and takes over — this is the designed recovery path.

---

## 3. The 7-Script Pipeline

| # | Script | Purpose | Timeout | Typical Duration |
|---|--------|---------|:-------:|:----------------:|
| 1 | `ca_shopee_listing_to_db.py` | SEED — creates new product rows from Shopee listings | 60 min | 15-17 min |
| 2 | `our_variation_preprocessing.py` | Backfills `our_variation` column before AI matching | 180 min | 0.1-174 min |
| 3 | `ca_ai_variation_match.py` | Resolves IDs, cleans data, AI variation matching | 30 min | 0.3-9 min |
| 4 | `shopee_comp_shopee_sales.py` | Shopee order sales sync (needs resolved IDs) | 60 min | 2-9 min |
| 5 | `Shopee-mylinks-sales-data-merged.py` | Product info + SiteGiant sales data | 300 min | 26-254 min |
| 6 | `ca_shopee_ads_metrics.py` | Ads metrics scraping via Chrome CDP | 420 min | 1-316 min |
| 7 | `ca_similarity_check.py` | AI similarity check (needs images/descriptions from #5) | 180 min | 0.3-65 min |

**Total pipeline**: ~620-660 min (10-11 hours), typically 4 PM to 2-4 AM next day

**Data dependency chain**: `1 -> 2 -> 3 -> 4 -> 5 -> 6 -> 7` (strictly sequential, each depends on prior)

### Per-job Timeout Notes

- `SAFETY_TIMEOUT_MIN = 360` remains as the `JobDef` dataclass default, but every job now overrides it with a custom `timeout_minutes` value.
- Job 6 (`ca_shopee_ads_metrics.py`) has the longest timeout at 420 min (7 hours) because its estimated runtime is ~300 min with high variance.
- Job 3 (`ca_ai_variation_match.py`) has the tightest timeout at 30 min (estimated ~4 min).
- When a timeout fires, the log shows: `TIMEOUT: {script} exceeded {timeout_minutes} min safety limit. Process killed.`

### Duration Variance Notes

Jobs 2, 5, and 6 have extreme variance (100x+). Likely driven by:
- **Job 2**: Whether new variations exist to backfill
- **Job 5**: Number of products to fetch from SiteGiant API + Shopee API rate limits
- **Job 6**: Chrome CDP scraping speed, number of items with ads, network latency

---

## 4. Helper Task Ecosystem

| Task | State | Trigger | Action | Purpose |
|---|---|---|---|---|
| **BootResumePipeline** | Ready (never run) | On boot | `boot_resume_pipeline.ps1` | Waits 2 min after boot, if `scheduler_checkpoint.json` exists, resumes pipeline with 14h safety timeout |
| **ResumePipeline2** | Running | Manual | `resume_pipeline2.ps1` | Runs jobs 5-7 with per-job timeouts, then chains full scheduler.py (14h safety net) |
| **ChainTodayPipeline** | **Disabled** | Manual | `chain_today_pipeline.ps1` | Formerly waited for ResumePipeline2 then started pipeline. Now redundant — resume_pipeline2.ps1 handles chaining internally |
| **RunPipeline2** | Ready | Manual | `run_pipeline2.ps1` | Checks for running scheduler.py via Win32_Process, if clear starts it with 14h safety timeout |
| **RunSchedulerNow** | Ready | Manual | `python scheduler.py` (WorkingDirectory set correctly) | Quick manual trigger for full pipeline |
| **RunAdsNow** | Ready | Manual | `run_ads_karlmobel.bat` | Individual runner for ca_shopee_ads_metrics.py |
| **RunMergedNow** | Ready | Manual | `run_merged_no_sitegiant.bat` | Individual runner for Shopee-mylinks-sales-data-merged.py |
| **RunMergedWider** | Ready | Manual | `run_merged_wider.bat` | Variant of merged runner |
| **RunShopSalesNow** | Ready | Manual | `run_shopee_sales.bat` | Individual runner for shopee_comp_shopee_sales.py |
| **RunPreprocessNow** | Ready | Manual | `run_preprocess.bat` | Individual runner for our_variation_preprocessing.py |
| **ManualRun_VariationPreprocessing** | Ready | Manual | `wait_and_run_variation.bat` | Another manual variation runner |
| **ShopeeSessionKeepalive** | Ready | Every 3 days | `run_session_keepalive.bat` | Keeps Shopee browser session alive |
| **StartChromeCDP9222** | Ready | On logon | `chrome.exe --remote-debugging-port=9222` | Launches Chrome in CDP mode for Job 6 ads scraping |

### Helper Script Details

#### `resume_pipeline2.ps1` (65 lines) — Recovery script with timeout enforcement

```
1. Defines $jobDefs with per-job timeouts:
   - Shopee-mylinks-sales-data-merged.py -> 240 min
   - ca_shopee_ads_metrics.py -> 420 min
   - ca_similarity_check.py -> 120 min
2. For each job:
   - Start-Process -PassThru (non-blocking start)
   - Wait-Process -Timeout (enforced timeout)
   - If !HasExited -> Stop-Process -Force (kill stuck job)
   - 5-min gap between jobs
3. After jobs 5-7 complete:
   - Wait 5 min
   - Start scheduler.py for today's full pipeline
   - 14-hour overall safety net on scheduler.py
   - If scheduler.py exceeds 14h -> force kill
```

#### `run_pipeline2.ps1` (35 lines) — Safe manual trigger with collision detection

```
1. Queries Win32_Process for any python process with "scheduler.py" in CommandLine
2. If found -> logs "ABORT: scheduler.py already running (PID=X)" and exits with code 1
3. If clear -> starts scheduler.py via Start-Process -PassThru
4. Waits up to 14 hours (Wait-Process -Timeout)
5. If scheduler.py exceeds 14h -> Stop-Process -Force
```

#### `boot_resume_pipeline.ps1` (36 lines) — Boot recovery with timeout

```
1. Waits 120s for system stabilization
2. Checks if scheduler_checkpoint.json exists
3. If yes:
   - Starts scheduler.py via Start-Process -PassThru
   - 14-hour safety timeout (Wait-Process -Timeout)
   - If scheduler.py exceeds 14h -> Stop-Process -Force
4. If no checkpoint: logs "Nothing to resume" and exits cleanly
```

#### `chain_today_pipeline.ps1` (21 lines) — DISABLED

```
1. Polls Get-ScheduledTask 'ResumePipeline2' every 60s until not Running
2. Waits 5 min
3. Starts scheduler.py via Start-Process -Wait (no timeout protection)
```

Now redundant because resume_pipeline2.ps1 handles chaining internally.

---

## 5. Token Refresh System

The scheduler refreshes Shopee API tokens via:

```
POST https://employee.awesomeree.com.my/api/shopee/refresh-tokens
Headers: Content-Type: application/json, x-api-key: <CHATBOT_API_KEY>
Body: {"region": "MY"}
```

**When tokens are refreshed**:
1. **Pre-flight**: Once before the entire pipeline starts
2. **Per-job**: Before each job (except job 1)
3. **On retry**: After each job failure, before retrying

**Response format**: `{success: bool, refreshed: int, valid: int, failed: int, failedShopIds: []}`

**Retry behavior**: 3 attempts with exponential backoff (10s, 20s). On total failure, logs error but **pipeline continues** — tokens may still be valid from a previous refresh.

**What to check when scripts fail with auth errors**:
1. Token refresh log entries — look for `failed > 0` or `failedShopIds`
2. Token refresh network errors — `HTTP error`, `network error`, `timeout`
3. Whether the web app endpoint itself is down

---

## 6. Known Issues & Debugging Scenarios

### Issue 1: Version Mismatch — v2.4 on disk, v2.2 in runs `ACTIVE — self-resolving`

**Symptom**: Log shows "SMART SCHEDULER v2.2 STARTED" but file on disk contains v2.4.
**Cause**: `scheduler.py` was updated on disk but running/recently-started instances loaded the old v2.2 code.
**Impact**: Lock file, checkpoint, and per-job custom timeout features from v2.4 are NOT active in v2.2 runs.
**Resolution**: Self-resolving — the next time scheduler.py is started fresh (by "Sales Data" task on Mar 7, or via RunSchedulerNow/ResumePipeline2 chain), it will load v2.4 from disk. Verify by checking the version line in scheduler.log after the next launch.

### Issue 2: Concurrent Instance Collisions `MITIGATED`

**Symptom**: Multiple "SMART SCHEDULER v2.2 STARTED" entries in the log with overlapping timeframes.
**Cause**: v2.2 has no lock file mechanism.
**Current mitigation**: `run_pipeline2.ps1` now checks for running scheduler.py via `Win32_Process` query before starting a new instance. `resume_pipeline2.ps1` runs individual scripts (not scheduler.py) for jobs 5-7, then chains into a single scheduler.py run.
**Full resolution**: Once v2.4 runs in production, the PID-based lock file will prevent any concurrent scheduler instances regardless of how they're launched. Look for "Lock acquired" in the log to confirm.

### Issue 3: StartBoundary Set to Future Date `ACTIVE — intentional`

**Symptom**: "Sales Data" task's `NextRunTime` is March 7, 2026 at 4:00 PM. Task will not fire today (March 6).
**Context**: The trigger's `StartBoundary` was changed from the old 3:00 AM schedule to 4:00 PM starting March 7. Today's pipeline is being run manually via ResumePipeline2. Automatic daily runs will resume from March 7 onward.
**Future debugging**: If the pipeline doesn't fire on a given day, always check StartBoundary first:

```powershell
Get-ScheduledTask -TaskName "Sales Data" | Select -Expand Triggers | Select StartBoundary
```

### Issue 4: Task Silently Skipped (IgnoreNew) `DESIGN CHARACTERISTIC`

**Symptom**: No 4 PM entry in scheduler.log. Task shows `LastResult = 2147946720`.
**Cause**: `MultipleInstancesPolicy = IgnoreNew`. If the previous pipeline (10-11h) is still running when the next trigger fires, the new run is silently skipped.
**Note**: This is by design — running two pipelines simultaneously would cause data corruption. The risk is that a stuck pipeline could block all subsequent runs indefinitely. The per-job timeouts (v2.4) mitigate this by ensuring no single job runs forever.

### Issue 5: Task Won't Start — Interactive Logon Required `ACTIVE`

**Symptom**: Task is Ready, NextRunTime is correct, but it never fires.
**Cause**: `LogonType = InteractiveToken` requires Admin to be logged into the VM's GUI session.
**Debugging**: `query user` on the VM to check if Admin is logged in.
**Fix**:

```powershell
$task = Get-ScheduledTask -TaskName "Sales Data"
$task.Principal.LogonType = "Password"
Set-ScheduledTask -InputObject $task -User "Admin" -Password "<password>"
```

### Issue 6: Job Stuck / Safety Timeout `IMPROVED`

**Symptom**: A job exceeds its per-job timeout and is killed. Log shows: `"TIMEOUT: {script} exceeded {timeout_minutes} min safety limit. Process killed."`
**Per-job timeout values (v2.4)**: 60 / 180 / 30 / 60 / 300 / 420 / 180 min (jobs 1-7).
**Cause**: Script hung — usually Chrome CDP timeout (Job 6), API rate limiting (Job 5), or database lock (any job).
**Improvement**: Previously all 7 jobs shared a flat 6-hour (360 min) timeout. Now each job has a tailored timeout sized to its expected duration. Additionally, all helper scripts (`resume_pipeline2.ps1`, `run_pipeline2.ps1`, `boot_resume_pipeline.ps1`) enforce their own timeout layers — when running via a helper, a stuck job gets killed from **two layers** (Python `subprocess.run(timeout=...)` AND PowerShell `Wait-Process -Timeout`) — whichever fires first.
**Debugging**: Check the individual script's log file for the last error before the kill.

### Issue 7: Stale Lock File `FUTURE SCENARIO (v2.4)`

**Symptom**: Log shows "Another scheduler instance is already running (PID XXXX). Exiting." but no Python process with that PID exists.
**Cause**: Previous process was killed without `release_lock()` running (e.g., `taskkill /f`, system crash).
**Resolution**: The scheduler auto-detects stale PIDs using `kernel32.OpenProcess()`. If the PID is truly dead, the next run takes over automatically and logs "Stale lock file found." If auto-detection fails, manually delete `scheduler.lock`.
**Current state**: No lock file exists. This scenario will only arise once v2.4 is actively running.

### Issue 8: Checkpoint Stuck `FUTURE SCENARIO (v2.4)`

**Symptom**: Pipeline always resumes from job N instead of starting fresh.
**Cause**: Checkpoint file exists and is less than 18 hours old.
**Debugging**: Read `scheduler_checkpoint.json` — check `job_index` and `saved_at`.
**Fix**: Delete `scheduler_checkpoint.json` to force a fresh start.
**Current state**: No checkpoint file exists. This scenario will only arise once v2.4 is actively running.

---

## 7. Debugging Checklist

When the pipeline doesn't run or fails:

```
1. CHECK TASK STATE
   Get-ScheduledTaskInfo -TaskName "Sales Data"
   -> LastRunTime, LastResult, NextRunTime

2. CHECK TRIGGER
   Get-ScheduledTask -TaskName "Sales Data" | Select -Expand Triggers | Select StartBoundary
   -> Is StartBoundary in the past? If future, task won't fire until then.

3. CHECK LOG TAIL
   Get-Content "C:\Users\Admin\Desktop\Shopee Comp My links Api\scheduler.log" -Tail 50
   -> Look for last STARTED/DONE/FAIL/TIMEOUT/EXCEPTION entry
   -> Check version: v2.2 (no lock/checkpoint) vs v2.4 (full protection)

4. CHECK RUNNING PROCESSES
   Get-Process python -EA SilentlyContinue | Select Id, StartTime, CommandLine
   Get-Process powershell -EA SilentlyContinue | Select Id, StartTime, CommandLine
   -> Is a pipeline currently running? Is it stale (24h+)?

5. CHECK LOCK & CHECKPOINT (v2.4)
   Test-Path "...\scheduler.lock"
   Test-Path "...\scheduler_checkpoint.json"
   -> If lock exists but no python process, it's stale -- delete it
   -> If checkpoint exists, pipeline will resume from that job index

6. CHECK HELPER TASKS
   Get-ScheduledTask | Where TaskPath -eq "\" | Where State -eq "Running"
   -> Are ResumePipeline2 or other helpers running?
   -> Check pipeline2_resume.log for their progress

7. CHECK ADMIN SESSION
   query user
   -> Admin must be logged in (InteractiveToken logon requirement)

8. CHECK TOKEN REFRESH
   Search scheduler.log for "Token refresh FAILED" or "failed:"
   -> Token issues cause script auth failures

9. CHECK CHROME CDP (for Job 6)
   Get-Process chrome -EA SilentlyContinue
   -> Chrome with --remote-debugging-port=9222 must be running for ads scraping
   -> StartChromeCDP9222 task starts it on logon
```

---

## 8. Architecture Diagram

```
                     Task Scheduler
                          |
          +---------------+---------------+
          |               |               |
    "Sales Data"    "BootResume      Manual Tasks
    Daily 4PM MYT    Pipeline"      (RunSchedulerNow,
          |          (On Boot)       RunPipeline2,
          |               |          ResumePipeline2)
          v               v               |
     cmd.exe /c      PS: check           |
          |          checkpoint           |
          v               |               |
    +---------------------+---------------+
    |              scheduler.py (v2.4)
    |     +----------------------------------------------+
    |     |  1. acquire_lock()                           |
    |     |  2. load_checkpoint() (18h max age)          |
    |     |  3. preflight_refresh_tokens()               |
    |     |  4. For each job 1-7:                        |
    |     |     +-- per-job token refresh                |
    |     |     +-- save_checkpoint()                    |
    |     |     +-- run w/ retry (3x, per-job timeout)   |
    |     |     +-- wait 5 min                           |
    |     |  5. clear_checkpoint()                       |
    |     |  6. release_lock()                           |
    |     +----------------------------------------------+
    |                    |
    v                    v
  scheduler.log    scheduler.lock / scheduler_checkpoint.json
```

### ResumePipeline2 flow (when used for recovery)

```
  resume_pipeline2.ps1
    +-- Job 5: Shopee-mylinks-sales-data-merged.py (timeout: 240m)
    +-- Wait 5 min
    +-- Job 6: ca_shopee_ads_metrics.py (timeout: 420m)
    +-- Wait 5 min
    +-- Job 7: ca_similarity_check.py (timeout: 120m)
    +-- Wait 5 min
    +-- scheduler.py (today's full pipeline, timeout: 14h)
         +-- [runs all 7 jobs as normal with v2.4 logic]
```

### RunPipeline2 flow (safe manual trigger)

```
  run_pipeline2.ps1
    +-- Check: is scheduler.py already running? (Win32_Process query)
    |   +-- YES -> ABORT with "scheduler.py already running (PID=X)"
    |   +-- NO  -> continue
    +-- Start scheduler.py (Start-Process -PassThru)
    +-- Wait up to 14h (force kill if exceeded)
```

### BootResumePipeline flow (on VM boot)

```
  boot_resume_pipeline.ps1
    +-- Wait 120s (system stabilization)
    +-- Check: does scheduler_checkpoint.json exist?
    |   +-- NO  -> log "Nothing to resume", exit
    |   +-- YES -> continue
    +-- Start scheduler.py (Start-Process -PassThru)
    +-- Wait up to 14h (force kill if exceeded)
```

---

## 9. Other Scheduled Tasks on VM3

| Task | Schedule | Script Location | Purpose |
|---|---|---|---|
| **Shopee Auto Approve Cancellation** | Daily 8:45 AM | `C:\Users\Admin\Desktop\Shopee Auto Approve Cancellation\schedule.py` | Auto-approves cancellation requests |
| **ShopeeSessionKeepalive** | Every 3 days 10:00 AM | `run_session_keepalive.bat` | Keeps Shopee browser session alive |
| **StartChromeCDP9222** | On logon | `chrome.exe --remote-debugging-port=9222` | Chrome CDP for Job 6 ads scraping. **Critical dependency.** |
