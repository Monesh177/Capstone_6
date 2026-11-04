# Telecom Medallion Pipeline – Operations Runbook

## 1. What runs where

| Item | Value |
|---|---|
| Job | Capstone6 Telecom Medallion Pipeline (id 634660669542804), serverless |
| Trigger | File arrival on `/Volumes/capstone6_telecom/bronze/ext_raw_landing/` (min 300 s between runs, waits 60 s after the last change); runs can also be started manually |
| Concurrency | 1 run at a time; extra triggers are queued |
| Alerts | Failure email to the job owner |
| Catalog | `capstone6_telecom` (schemas `bronze`, `silver`, `gold`) |
| Secrets | Azure SQL user/password in secret scope `azure-sql-scope` (keys `sql-user`, `sql-pwd`) |
| Auto Loader state | Managed volume `capstone6_telecom.bronze.autoloader_metadata` (`checkpoints/telecom_bronze_v2`, `schemas/telecom_bronze_v2`) |

| Task | Notebook | Purpose | Retries |
|---|---|---|---|
| bronze_jdbc | notebooks/bronze/01_Bronze_JDBC | Azure SQL masters, insert-only MERGE (new version only when a row changes) | 2 (covers Azure SQL resume) |
| bronze_adls | notebooks/bronze/02_Bronze_ADLS | KPI, outage and complaint files via Auto Loader (availableNow) | 1 |
| bronze_profiling | notebooks/bronze/03_Bronze_Data_Profiling | Writes `bronze_profiling_summary` | 1 |
| silver_processing | notebooks/silver/04_Silver_Processing | DQ rule engine, Silver tables, rejected / warning records, rule results | 1 |
| gold_analytics | notebooks/gold/05_Gold_Analytics | Dimensions, facts, outage impact, daily quality, SLA summary | 1 |
| pipeline_validation | tests/06_Pipeline_Validation | Bronze / Silver / Gold checks, writes `*_validation_summary`; fails on blocking checks | 0 |

Every task is idempotent. Re-running the job with no new data gives the same tables.

## 2. Daily checks

1. Job run page: the latest run is SUCCESS.
2. Validation result:
   ```sql
   SELECT layer, status, COUNT(*) FROM (
     SELECT * FROM capstone6_telecom.bronze.bronze_validation_summary UNION ALL
     SELECT * FROM capstone6_telecom.silver.silver_validation_summary UNION ALL
     SELECT * FROM capstone6_telecom.gold.gold_validation_summary) GROUP BY ALL;
   ```
   Expected: no `FAIL`. `WARN` rows are region-consistency issues in the source data. Report them to the source owner; they never drop records.
3. DQ volume: `silver_rejected_records` and `silver_warning_records` by `rule_id`. Investigate when the counts jump.

## 3. Failure handling

| Failed task | Likely cause | Action |
|---|---|---|
| bronze_jdbc | Azure SQL serverless resuming, network, or secret | Task retries twice. Check that the secret scope can be read. Then repair the run |
| bronze_adls | Schema change in landing files, volume access | Check `_rescued_data` and the error. Fix the source or schema, then repair the run |
| silver_processing | A detected issue has no catalogue rule (assert), or a data type change | Add or adjust the rule in `DATASETS` in 04_Silver_Processing, then repair the run |
| gold_analytics | Join uniqueness assert (duplicate keys in Silver) | Check uniqueness in Silver, then repair the run |
| pipeline_validation | A blocking check failed | Read the `FAIL` rows in the summary tables (expected vs actual). Fix the cause, then repair from the failed upstream task |

**Repair:** on the failed run's page click *Repair run*, or use `databricks jobs repair-run <run_id> --rerun-all-failed-tasks --rerun-dependent-tasks`. Tasks that already succeeded are not re-run. Because every task is idempotent, a repair cannot duplicate data.

## 4. Replay and reload

* **New data:** drop new files with **new file names** into the landing folders. The trigger starts the job. Auto Loader loads only files it hasn't seen before.
* **Corrected file with the same name:** Auto Loader ignores it. Rename the file, or do a full reset:
  1. Pause the job trigger.
  2. Truncate the 6 Bronze source tables.
  3. Delete the Auto Loader checkpoints and schemas under `autoloader_metadata/.../telecom_bronze_v2`.
  4. Run the job manually.
  5. Unpause the trigger.
  6. Confirm the validation summaries show 0 FAIL. Then run `tests/07_Final_Regression` and re-baseline it if the new data is accepted.
* **Changed Azure SQL master row:** the next run appends a new version, and Silver uses the latest `_ingested_at` per key.

## 5. Release / regression procedure

1. Work on a feature branch in the Git folder. Commit, push and open a PR to `main`.
2. Run the job manually. All 6 tasks must succeed.
3. Run `tests/07_Final_Regression`. It must pass 43/43 against the UAT baseline, or the baseline is updated with sign-off.
4. Record the run IDs in `docs/UAT_Signoff.md`.

## 6. Consumers

* AI/BI dashboard on the Gold tables (network quality, outage impact, SLA by region).
* Genie space on the Gold tables for ad-hoc questions.
* Regional row-level security is not yet configured (needs the group-to-region mapping).
