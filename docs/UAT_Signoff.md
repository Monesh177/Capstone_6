# Telecom Network Quality – UAT Sign-off (Day 5)

Project: Capstone_6 telecom medallion pipeline  
Job: Capstone6 Telecom Medallion Pipeline (job id 634660669542804)  
UAT baseline run: 42362058080150 (2026-11-04, all 6 tasks SUCCESS, 117/117 validation checks PASS)  
Final regression: `tests/07_Final_Regression`, 43/43 PASS against the baseline (2026-11-04)

## 1. Scope

| Layer | Outputs |
|---|---|
| Bronze | `capstone6_telecom.bronze.bronze_*` (6 sources: Azure SQL masters via JDBC, ADLS feeds via Auto Loader), `bronze_profiling_summary` |
| Silver | `capstone6_telecom.silver.silver_*` (6 entities), `silver_rejected_records`, `silver_warning_records`, `silver_dq_rule_catalogue`, `silver_dq_rule_results` |
| Gold | `dim_tower`, `dim_region`, `dim_cell`, `fact_network_kpi`, `fact_network_outage`, `fact_customer_complaint`, `fact_outage_impact`, `gold_network_quality_daily`, `gold_region_sla_summary` |
| Validation | `bronze_validation_summary`, `silver_validation_summary`, `gold_validation_summary` |

## 2. Acceptance criteria and evidence

| # | Criterion (handbook) | Evidence | Result |
|---|---|---|---|
| 1 | Bronze preserves raw evidence, one row per landed record / master version | Bronze 161 / 170 / 179 / 11,894 / 696 / 354 = source files and tables; `bronze_profiling_summary` | PASS |
| 2 | Data-quality rules catalogued with rule ID, severity, columns, action, validation query | `silver_dq_rule_catalogue` (129 rules) | PASS |
| 3 | Quarantine only by explicit rules, traceable to the source record | `silver_rejected_records`: 65 (43 exact duplicate copies, 22 latency -999), each with `rule_id` and `source_record_json` | PASS |
| 4 | Warnings kept in Silver and flagged | `silver_warning_records`: 36 (26 IQR latency outliers, 10 missing optional fields); `dq_warning_checks` on Silver rows | PASS |
| 5 | Silver reconciliation: Bronze = Silver + rejected | `silver_validation_summary` RECONCILIATION checks, all 6 tables | PASS |
| 6 | Rule reconciliation: Bronze validation query = detected = stored | `silver_validation_summary` RULE_RECONCILIATION | PASS |
| 7 | Region consistency across masters, telemetry and complaints | REGION_CONSISTENCY checks in all 3 summaries: 0 mismatches | PASS |
| 8 | Gold facts / dimensions at documented grain, reconciled to Silver | `gold_validation_summary` GRAIN, RECONCILIATION, TOTALS, DIMENSIONS | PASS |
| 9 | Outage-to-telemetry correlation and impact | `fact_outage_impact`: 696 outages, 145 with telemetry evidence, 51 with complaints; independent SQL recomputation = PASS | PASS |
| 10 | Network SLA by region and SLA tier | `gold_region_sla_summary` (179 regions); totals = facts | PASS |
| 11 | Idempotent replay | Repeated job runs (see section 3) leave Bronze unchanged and reproduce the same Silver / Gold counts; final regression 43/43 PASS | PASS |
| 12 | Automated execution on file arrival | Trigger on `/Volumes/capstone6_telecom/bronze/ext_raw_landing/`; runs 419429553750424 and 529707784858563 started by FILE_ARRIVAL, both SUCCESS | PASS |
| 13 | Credentials held in secret management | Azure SQL credentials in secret scope `azure-sql-scope`; no hard-coded secrets in committed notebooks | PASS |
| 14 | Regional security (row-level access by region) | Not implemented: needs the mapping of user groups to regions | OPEN |
| 15 | SLA pass/fail against targets | Not implemented: SLA targets per tier not provided | OPEN |

## 3. Job run evidence (2026-11-04, UTC)

| Run id | Trigger | Duration | Result |
|---|---|---|---|
| 151662357113612 | Manual | 6.0 min | SUCCESS |
| 42362058080150 | Manual (UAT baseline) | 12.2 min (incl. queue wait) | SUCCESS |
| 138871970452227 | Manual | 6.4 min | SUCCESS |
| 608811158167760 | Manual | 5.8 min | SUCCESS |
| 795358723769547 | Manual | 6.5 min | SUCCESS |
| 145072042043873 | Manual | 6.2 min | SUCCESS |
| 669143367519130 | Manual (after full reset) | 7.2 min | SUCCESS |
| 529707784858563 | File arrival | 6.7 min | SUCCESS |
| 419429553750424 | File arrival | 7.0 min | SUCCESS |
| 475550860521539 | Manual (first run) | 8.6 min | SUCCESS |

10 of 11 runs succeeded. The other run (1084877109854682) was cancelled by the user and did not fail. A typical end-to-end run takes 6–7 minutes on serverless compute.

## 4. Known limitations / open items

* Regional security (criterion 14) and SLA targets (criterion 15) need business input.
* Outages cover 2026-07-01 to 07-04, complaints 07-01 to 07-02, and telemetry 07-01 to 08-27. Impact analysis is limited to these periods.
* Auto Loader ignores files that are overwritten with the same name. Deliver corrected files under new names, or use the reset procedure in `runbook/Operations_Runbook.md`.
* The regression baseline matches the current landing data. Re-baseline after accepted new loads.

## 5. Sign-off

| Role | Name | Decision (Accept / Reject) | Date | Comments |
|---|---|---|---|---|
| Business owner | | | | |
| Data engineering lead | | | | |
| QA / tester | | | | |
