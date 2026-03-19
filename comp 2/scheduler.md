# VM TT Scheduler Reference: `ca_product_info_daily`

## Scope

This document covers exactly one scheduler entry on VM TT:

- Task name: `ca_product_info_daily`
- VM: `VM TT` (`DESKTOP-KD0HLJU`)
- Schedule intent: daily midnight run of the CA SG 3-script pipeline

Anything outside this task (other tasks/VMs) is out of scope.

---

## 1. Scheduler Identity

| Field | Value |
|---|---|
| Task | `ca_product_info_daily` |
| Host VM | `VM TT` (`DESKTOP-KD0HLJU`) |
| Task Path | `\` |
| Action executable | `C:\Users\Admin\AppData\Local\Programs\Python\Python313\python.exe` |
| Action arguments | `C:\Users\Admin\Desktop\ca_sg\ca_sg_pipeline.py` |
| Working directory | `C:\Users\Admin\Desktop\ca_sg` |
| Trigger type | `MSFT_TaskDailyTrigger` |
| Start boundary | `2026-03-10T00:00:00+08:00` |
| Interval | Every `1` day |
| Principal | `Admin` |
| Logon type | `InteractiveToken` |
| Run level | `HighestAvailable` |

Verified from live VM TT on 2026-03-19.

---

## 2. Runtime Policy (Task Scheduler)

| Setting | Value | Operational meaning |
|---|---|---|
| `MultipleInstancesPolicy` | `IgnoreNew` | If a previous run is still active when the next trigger fires, the new trigger is skipped |
| `StartWhenAvailable` | `true` | Task can start later if missed at trigger time |
| `ExecutionTimeLimit` | `PT0S` | No scheduler-side hard timeout |
| `AllowDemandStart` | `true` | Task can be started manually |
| `WakeToRun` | `false` | Task does not wake sleeping machine |
| `DisallowStartIfOnBatteries` | `false` | Battery state does not block run |
| `StopIfGoingOnBatteries` | `false` | Running task is not stopped for battery transition |

Verified from task XML and live task settings on 2026-03-19.

---

## 3. What the Task Starts at 00:00

The scheduled action launches `ca_sg_pipeline.py`, which is a detached launcher, not the main worker.

Execution chain:

1. Task Scheduler starts:
   - `python.exe C:\Users\Admin\Desktop\ca_sg\ca_sg_pipeline.py`
2. `ca_sg_pipeline.py`:
   - checks PID file (`ca_sg_pipeline.pid.json`)
   - prevents duplicate worker start if existing PID is still running
   - starts `ca_sg_pipeline_worker.py` as a detached background process
   - exits quickly
3. `ca_sg_pipeline_worker.py` executes the real pipeline (3 scripts, sequential):
   - `ca_product_info.py`
   - `ca_my_products_sales_sync.py`
   - `ca_comp_variation_matcher.py --live`

---

## 4. The 3-Script Pipeline (Single Task Scope)

This task is the scheduler entry for exactly this 3-step chain:

| Order | Worker job id | Script | Main responsibility | Timeout | Retry |
|---|---|---|---|---|---|
| 1 | `product_info` | `ca_product_info.py` | Refresh product metadata in `Shopee_My_Products` from Shopee APIs, including link/sheet-name alignment workflows | `7200s` | `1` |
| 2 | `shopee_sales_sync` | `ca_my_products_sales_sync.py` | URL normalization, ID/model resolution, Shopee daily sales sync, SiteGiant sales sync | `14400s` | `1` |
| 3 | `comp_variation_matcher` | `ca_comp_variation_matcher.py --live` | Variation matching with deterministic + LLM pipeline, writes live match results | `7200s` | `1` |

Worker-level controls:

- Rest between scripts: `60s`
- Retry delay: `120s`
- Progress watchdog interval: `15s`
- Stuck detection: no log growth for timeout window triggers kill and retry logic

---

## 5. What This Scheduler Mainly Does

Main function of `ca_product_info_daily` is nightly CA SG data maintenance for comp-analysis.

At a high level, each midnight run:

1. Refreshes and normalizes our product master records.
2. Updates rolling sales metrics from Shopee + SiteGiant.
3. Rebuilds/updates competitor variation matching outputs.

Primary operational outcome:

- Fresh product metadata + sales windows + variation match data ready by early morning, with script-level logs under `C:\Users\Admin\Desktop\ca_sg\logs`.

---

## 6. Observed Run Behavior (Recent Snapshot)

### Latest confirmed run (2026-03-19, UTC+8)

- Start: `2026-03-19 00:00:03`
- End: `2026-03-19 03:02:49`
- Overall: `ALL PASSED`

Per-script timings from pipeline log:

- `product_info`: `150s`
- `shopee_sales_sync`: `480s`
- `comp_variation_matcher`: `10216s` (~2h50m)

### Prior recent runs

| Date (UTC+8) | Overall | Notes |
|---|---|---|
| 2026-03-18 | `ALL PASSED` | Full 3-script chain completed |
| 2026-03-17 | `ALL PASSED` | Full 3-script chain completed |
| 2026-03-16 | `SOME FAILURES` | Historical run included legacy `ads_metrics` stage and failures |
| 2026-03-15 | `SOME FAILURES` | Historical run included legacy `ads_metrics` stage and failures |
| 2026-03-14 | `SOME FAILURES` | Historical run included legacy `ads_metrics` stage and failures |

Note: Current live worker definition is the 3-script chain documented above (no `ads_metrics` stage in current `PIPELINE` list).

---

## 7. Artifacts and Paths

### Scheduler and pipeline scripts

- `C:\Users\Admin\Desktop\ca_sg\ca_sg_pipeline.py`
- `C:\Users\Admin\Desktop\ca_sg\ca_sg_pipeline_worker.py`
- `C:\Users\Admin\Desktop\ca_sg\ca_product_info.py`
- `C:\Users\Admin\Desktop\ca_sg\ca_my_products_sales_sync.py`
- `C:\Users\Admin\Desktop\ca_sg\ca_comp_variation_matcher.py`

### State files

- PID file: `C:\Users\Admin\Desktop\ca_sg\ca_sg_pipeline.pid.json`

### Logs directory

- `C:\Users\Admin\Desktop\ca_sg\logs`

Key log patterns:

- Launcher log: `pipeline_launcher_YYYY-MM-DD.log`
- Worker summary log: `pipeline_YYYY-MM-DD.log`
- Script logs:
  - `product_info_YYYYMMDD_HHMMSS.log`
  - `shopee_sales_sync_YYYYMMDD_HHMMSS.log`
  - `comp_variation_matcher_YYYYMMDD_HHMMSS.log`

---

## 8. Health Check Checklist (Single Task)

Use these checks when validating this scheduler:

1. Task exists and is enabled:
   - `Get-ScheduledTask -TaskName ca_product_info_daily`
2. Trigger remains midnight daily:
   - check `StartBoundary` and `DaysInterval`
3. Action still points to `ca_sg_pipeline.py` in `C:\Users\Admin\Desktop\ca_sg`
4. Last run result is `0` and next run time is tomorrow 00:00 local
5. Latest `pipeline_YYYY-MM-DD.log` contains:
   - `CA SG PIPELINE STARTING`
   - script start/complete lines for all 3 jobs
   - `Overall: ALL PASSED` (or explicit failure lines for incident triage)
6. No stale active PID if task shows `Ready` but run appears blocked

---

## 9. Known Operational Caveats

1. `InteractiveToken` logon type:
   - behavior can depend on interactive session context for user `Admin`.
2. `IgnoreNew` instance policy:
   - if one run crosses midnight, next scheduled trigger is skipped.
3. `ExecutionTimeLimit=PT0S`:
   - scheduler will not auto-kill; worker watchdog/timeouts are the real protection.
4. Encoding noise in matcher logs:
   - `UnicodeEncodeError` lines may appear in console logging paths for some characters, while overall run still succeeds.

---

## 10. Baseline Snapshot (As of 2026-03-19)

| Metric | Value |
|---|---|
| Task state | `Ready` |
| Enabled | `true` |
| Last run time | `2026-03-19 00:00:00 +08:00` |
| Last result | `0` |
| Next run time | `2026-03-20 00:00:00 +08:00` |
| Trigger drift | None observed |
| Active related pipeline processes after completion | None observed |

---

## 11. Related Bot Documentation

This scheduler drives these three scripts:

- `bot-scripts/ca_product_info.md`
- `bot-scripts/ca_my_products_sales_sync.md`
- `bot-scripts/ca_comp_variation_matcher.md`

