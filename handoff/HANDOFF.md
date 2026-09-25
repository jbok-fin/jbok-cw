# jbok-cw Capacity/Finance Notebooks — Reproduction Handoff

Self-contained reconstruction handoff for the six core notebooks. Everything an agent needs
to locate, rebuild, and validate them is in this one file. No external links or companion docs
are required.

**Provenance tags:** 🟩 verified from repo code (cell/line cited) · 🟦 verified from live runtime ·
🟨 documentation/assumption · 🟥 blocked/unknown (evidence to resolve is stated). Where a
notebook comment/markdown conflicts with executable code, **code wins** and the conflict is flagged.

---

## 1. Document control and scope

- **Scope:** the six notebooks `01.1`, `01.2`, `01.3`, `01.4`, `dcgm_metrics_based_bmaas_tflops`,
  `time-series-fcst-sparkcaster_1.2`. (`00_sparkcaster_sandbox` is read-only QA, out of this set.)
- **Method:** cell-level inspection of the current committed source of all six notebooks, plus
  read-only runtime inspection. Three code fixes (D-2/D-3/D-4) were applied and committed.
- **Runtime validation status:** none. Catalog credentials are unavailable in this workspace and
  are **out of scope by owner decision**, so no notebook has had a bounded/full run. All
  code-level claims are static; no notebook is `READY` in the strict sense.
- **Safety invariants used here:** read-only by default; no secrets printed; destructive writes
  flagged; no expensive/full-table Spark job run without explicit authorization.

## 2. Source files and canonical versions

| Repo | URL | Branch | Commit | State |
|---|---|---|---|---|
| jbok-cw | `https://github.com/jbok-fin/jbok-cw.git` | `fix/handoff-d2-d3-d4` | `58a0276` | 🟦 clean tree; HEAD code == on-disk for all core notebooks |
| data-engineering | (CoreWeave internal) | `main` | `79c418e7` | 🟦 used for the `NessieSparkClient` SDK + StarRocks view DDL |

Canonical notebook paths (all under `jbok-cw/notebooks/`, 🟩 committed `58a0276`):

| Key | Path | Kernel |
|---|---|---|
| 01.1 | `01.1_dcgm_raw_impute_sparkcaster_1.1.5.ipynb` | Spark via Enterprise Gateway |
| 01.2 | `01.2-dcgm_eda_sparkcaster_1.2.4.ipynb` | Spark via Enterprise Gateway |
| 01.3 | `01.3_dcgm_eda_parquet_export_1.0.0.ipynb` | Spark via Enterprise Gateway |
| 01.4 | `01.4_dcgm_eda_local_visualizations_monthly_1.0.0 (9).ipynb` | **local python (no Spark)** |
| bmaas | `dcgm_metrics_based_bmaas_tflops.ipynb` | Spark via Enterprise Gateway |
| tsfcst | `time-series-fcst-sparkcaster_1.2.ipynb` | Spark via Enterprise Gateway + driver pip |

> 🟥 **`nbstripout` is not installed** in `jbok-cw` (not in `.git/config`), so notebook *outputs*
> pollute git diffs. Canonical drift is otherwise resolved (HEAD == on-disk). Also present but
> out of scope: `dcgm_training_classifier.ipynb`, `demand_fcst.ipynb`,
> `dcgm_raw_impute_hyperparameter_tuning.ipynb`, `notebooks/archive/*`.

## 3. Runtime and namespace baseline

🟦 **Verified live (2026-09-21T16:51Z, pod `capacity-finance-analytics-v8-0`, ns `kubeflow-jbok`):**

| Item | Value |
|---|---|
| Notebook image | `coreweave.jfrog.io/cw-data-engineering-oci-prod-local/kubeflow:main.29512104924.1` |
| Image digest | `sha256:c457624dd4c314c03bd0a866e4b6e49b4b630e45b488801f5fc857fbbdabfebb` |
| Pod resources | requests cpu:16 / mem:128Gi |
| Workspace PVC | `capacity-finance-analytics-v8-volume` (1000Gi, class `shared-vast`), mounted at `/home/coreweave` |
| Python / Java | 3.11.11 / OpenJDK 17.0.19 |
| Spark / PySpark | 3.5.5 / 3.5.5 |
| dbt-core / Airflow | 1.9.10 / 2.10.5 |
| Base-image libs present | pandas 2.1.4, numpy 1.26.4, scikit-learn 1.9.0, pyarrow 15.0.2, s3fs 2026.6.0, keyring 24.3.1, keyrings.cryptfile 1.3.9 |
| Base-image libs **absent** (installed per-run by the notebooks) | statsmodels, prophet, xgboost, matplotlib, seaborn, scipy, bokeh, holoviews, hvplot, datashader, jinja2 |
| Spark connector jars | `iceberg-spark-runtime-3.5_2.12-1.10.0`, `iceberg-aws-bundle-1.10.0`, `nessie-spark-extensions-3.5_2.12-0.105.7`, `starrocks-spark-connector-3.4_2.12-1.1.3`, `hadoop-aws-3.3.4`, `aws-java-sdk-bundle-1.11.901` |

🟦 **Session builder:** `NessieSparkClient` at `data-engineering/sdk/spark/nessie/client.py`. With
`dbtcaster=True` it builds `appName("`SparkClient` for Nessie Catalog").getOrCreate()` and sets the
S3A/CAIOS Hadoop configs (`fs.s3a.endpoint` = the Nessie endpoint, path-style access, credentials
provider) and the Nessie/Iceberg catalogs; it exposes `.spark`, `.sql()`, `.db_prepare()`,
`.table_ref()`, `.db_load()`.

🟦 **Endpoints (reachable):** gateway `spark-gateway.kubeflow.svc.cluster.local:8888` (HTTP 200;
service lives in the **`kubeflow`** namespace, not inspectable); Nessie
`http://kf-proxy.nessie.svc.cluster.local:19120/api/v2` (HTTP 200); external Nessie endpoint
constant `http://nessie-prd.cwobject.com`; StarRocks FE
`kube-starrocks-warehouse-fe-service.starrocks.svc.cluster.local:9030` (TCP open).

🟦 **RBAC / namespace visibility is PARTIAL.** In-cluster SA `system:serviceaccount:kubeflow-jbok:default-editor`,
namespaced only. Can get/list pods, services, configmaps, PVCs, `notebooks.kubeflow.org`; can
manage `sparkapplications`. **Cannot read Secrets**, cannot list other namespaces, cannot inspect
the gateway namespace, gateway/driver pod specs, or catalog/Secret configuration.

🟦 **Spark resources by notebook:** 01.1 / 01.2 / 01.3 / bmaas use the gateway-managed session from
`NessieSparkClient` with `spark.sql.shuffle.partitions=2000` and adaptive query execution on. tsfcst
(CELL 6) uses executor 16g/4cores/20 instances, dynamic allocation 5–25, `shuffle.partitions=400`,
driver 8g, `maxResultSize 4g`, Arrow on.

## 4. Execution prerequisites and credentials

🟩 All Spark notebooks obtain credentials the same way (01.1 CELL 2, 01.2 CELL 2–3, 01.3 CELL 2,
bmaas CELL 2–3, tsfcst CELL 1/23):

- Encrypted keyring `CryptFileKeyring` at `$KEYRING_CRYPTFILE_PATH` =
  `~/.local/share/python_keyring/cryptfile_pass.cfg` (home from `Path.home()`, not hardcoded);
  master password entered via `getpass`.
- **Credential names (values never to be printed):**
  - CAIOS — service `caios`, keys `access_key`, `secret_key` (used by all Spark notebooks for
    Nessie/CAIOS/S3A). Required perm: read/write the `sandbox`/`nessie` catalogs used.
  - StarRocks — service `starrocks`, keys `username`, `password`. **Used only by tsfcst**
    (01.1 collects them but never uses them). Required perm: read `sandbox_finance` on StarRocks.
  - CW-S3 (tsfcst only) — env `CW_S3_ACCESS_KEY` / `CW_S3_SECRET_KEY` (or `getpass`). Required
    perm: write to `s3://jbok-sandbox-test` via endpoint `cwlota.com`.

🟥 **BLOCKER (owner: out of scope):** no keyring config exists in this workspace and the SA cannot
read Secrets, so no catalog auth is possible here → no runtime reproduction. To make headless
runs possible elsewhere, replace the interactive `getpass`/keyring with env or a mounted Secret
(see §13 scaffolding); record only credential names + verification status, never values.

🟥 **BLOCKER (tsfcst):** `/tmp/sparkcaster_wheels.zip` (executor wheelhouse) is absent; the base
image also lacks statsmodels/prophet. Forecaster's distributed path cannot run until it is staged.

## 5. Notebook dependency graph

```
 Victoria Metrics / Redfish ingestion (upstream; data-engineering intake DAGs, not these notebooks)
        │
        ▼
 nessie.staging_capacity_finance.dcgm_metrics_raw   ← raw ~15-min DCGM telemetry
        │                                             (schema drift risk: is_hpc_verification_job)
        ├──────────────► [01.1 impute] ── writes ──► sandbox.sandbox_finance.dcgm_metrics_raw_imputed_v2
        │                                                   │  exposed by StarRocks views
        │                                                   │  v_dcgm_metrics_raw_imputed (sandbox_finance, staging_finance)
        │                                                   ▼
        │                                             [01.2 EDA] ── writes ──► sandbox.sandbox_finance.dcgm_eda_* (7 tables)
        │                                                   │  ALSO READS: dcgm_metrics_summary_imputed_v2  ← 🟨 LEGACY/deprecated & STALE (D-15 open; see §12)
        │                                                   ▼
        │                                             [01.3 export] ─► /home/coreweave/dcgm_eda_parquet/<tag>/ (7 parquet dirs + manifest.json)
        │                                                   ▼
        │                                             [01.4 viz]  ─► /home/coreweave/dcgm_eda_viz_outputs/ (PNG/zip/HTML)  [LOCAL]
        │
        └──────────────► [bmaas] reads dcgm_metrics_raw (+ redfish_stats_power_shelf_rail_output_power_watts, dim_customer)
                                 ── writes ──► sandbox.sandbox_finance.dcgm_metrics_bmaas  (→ StarRocks view v_dcgm_metrics_bmaas)

 StarRocks sandbox_finance.dcgm_metrics_summary_imputed_daily   ← 🟨 created NATIVELY in StarRocks via SQL (not in repo, not a notebook)
        │
        ▼  (tsfcst CELL 7 JDBC → Iceberg sync)
 sandbox.sandbox_finance.dcgm_metrics_summary_imputed_daily_sync
        ▼
 [tsfcst] ─► that sync table (overwrite) + s3://jbok-sandbox-test/jbok/time-series-fcst-sparkcaster/<RUN_TS>/*.{csv,xlsx}
```

**Execution order:** `01.1 → 01.2 → 01.3 → 01.4` is a strict chain. `bmaas` and `tsfcst` are
independent branches off shared raw/summary data — **not** downstream of 01.2. 🟨 **Lineage resolved
(owner):** the summary tables are created **natively in StarRocks via SQL** (`dcgm_metrics_summary_
imputed_daily`, `_monthly`, and `dcgm_histogram`) — not by any notebook or repo job. 01.2's input
`dcgm_metrics_summary_imputed_v2` is **legacy** (formerly written by 01.1, now deprecated/superseded)
— see §12 and D-15.

**Readiness matrix** (status: none are `READY` — all runtime-unvalidated; credentials out of scope):

| NB | Commit | Inputs | Outputs | Runtime/dep needs | Bounded-run method | Destructive write | Status | Evidence/blocker |
|---|---|---|---|---|---|---|---|---|
| 01.1 | `58a0276` | `dcgm_metrics_raw` | `dcgm_metrics_raw_imputed_v2` | gateway+CAIOS | date/sample-bounded read → repro schema | **DROP+overwrite** (CELL 19) | BLOCKED (creds); code fixed | D-4 fixed; CV not run |
| 01.2 | `58a0276` | `dcgm_metrics_raw_imputed_v2`, `dcgm_metrics_summary_imputed_v2` (🟨 LEGACY/stale) | 7× `dcgm_eda_*` | gateway+CAIOS | small `RAW_SAMPLE_FRACTION` → repro schema | overwrite ×7 | BLOCKED (creds); D-15 legacy/stale summary input | D-2 fixed; D-15 OPEN (see §12) |
| 01.3 | `58a0276` | 7× `dcgm_eda_*` | local parquet + manifest | gateway+CAIOS | isolated `EXPORT_TAG` | local FS overwrite | BLOCKED (creds) | no strict gate |
| 01.4 | `58a0276` | latest parquet export | PNG/zip/HTML | **none (local)** | run vs existing export | none (local files) | READY_WITH_CONDITIONS | needs an export on disk |
| bmaas | `58a0276` | `dcgm_metrics_raw`, `redfish_stats_power_shelf_rail_output_power_watts`, `dim_customer` | `dcgm_metrics_bmaas` | gateway+CAIOS | bounded → repro schema | **DROP+overwrite** (CELL 22) | BLOCKED (creds); code fixed | D-3 fixed; unmapped zone accepted |
| tsfcst | `58a0276` | StarRocks `dcgm_metrics_summary_imputed_daily` (🟨 StarRocks-native) → `_sync` | `_sync` + S3 forecasts | gateway+CAIOS+StarRocks+**wheels** | 1 metric, short horizon → scratch S3 | **DROP+overwrite** sync (CELL 7) | BLOCKED (creds + wheels) | source lineage resolved; exposure fallback accepted |

---

## 6. Notebook 01.1 — impute contract & reconstruction

**Role:** impute NULL `redfish_power` in raw DCGM metrics with the best of four Spark MLlib
regressors; write the imputed raw table. Daily/monthly summaries are produced downstream in
StarRocks, **not here** (D-4).

**Session/config (CELL 1–4):** pip-installs (unpinned) keyring, ipython-secrets, pyarrow, fsspec,
s3fs, statsmodels, matplotlib, scikit-learn, xgboost, keyrings.cryptfile, jupyter_bokeh, reportlab;
pinned bokeh==3.6.2, panel==1.5.2, holoviews==1.19.0, hvplot==0.10.0, datashader==0.16.3,
dask[dataframe]==2024.9.1, distributed==2024.9.1. `NessieSparkClient(svc_url=".../api/v2",
nessie_endpoint="http://nessie-prd.cwobject.com", caios_*, dbtcaster=True)`; Spark: adaptive.enabled,
adaptive.coalescePartitions.enabled, adaptive.skewJoin.enabled all `true`,
`spark.sql.shuffle.partitions=2000`, logLevel ERROR (all confirmed CELL 4).

**Input (CELL 5):** `ness.sql("select * from nessie.staging_capacity_finance.dcgm_metrics_raw")`
(full `SELECT *`, no date filter). Feature-eng inputs (CELL 6): tensor_util, gpu_util, mem_copy_util,
tensor_tflops, vram_usage, chip_power, gpu_peak_power_node_watts, peak_tflops_unit,
gpu_count_expected, product_resolved, region, is_training.

**Feature engineering (CELL 6) — code uses epsilon `0.0001`; print strings say `0.001` (5× cosmetic mismatch, FLAGGED):**
- `tensor_to_gpu_ratio_index = tensor_util/(gpu_util+0.0001)`
- `memory_intensity_index = mem_copy_util/(gpu_util+0.0001)`
- `compute_to_memory_ratio_index = tensor_tflops/(vram_usage+0.0001)`
- `power_pct_max_index = chip_power/(gpu_peak_power_node_watts+0.0001)`
- `compute_pct_max_index = tensor_tflops/((peak_tflops_unit*gpu_count_expected)+0.0001)`
- `gen = current_gen` if `product_resolved ∈ {GB200,GB300,B200,B300}` else `prior_gen`
- `region_rollup = EU` if `trim(coalesce(region,'')).startswith("EU")` else `NAM`
- `training_inference = active_training` if **`is_training == 1`** else `non_training`
  (🟩 CORRECTION: notebook markdown claims `is_training == "slurm"` — code uses integer `1`).

**Training/selection (CELL 7–12):**
- `num_cols` (22) cast to Double, `.na.drop()` **with no subset → drops rows with ANY null feature**
  (CELL 7; the `subset=["redfish_power"]` is commented; the "only drop null target" comment is false, FLAGGED).
- `feature_cols` = `num_cols − {redfish_power}` (21, **includes `is_hpc_verification_job`**); target `redfish_power`.
- `randomSplit([0.7,0.3], seed=42)`, then `sample(False, 0.02, 42)` train / `0.02, 43` test
  (`SAMPLE_FRAC=0.02`; inline comment "3%" is wrong, FLAGGED). Final retrain `FINAL_SAMPLE_FRAC=0.10` (CELL 12).
- Candidate models (CELL 9): Simple LR on `[chip_power]` (maxIter=100, standardization=False);
  Ridge on all features via `Pipeline([StandardScaler(withStd=True,withMean=False), LinearRegression(regParam=0.01, elasticNetParam=0.0, maxIter=100, standardization=False)])`;
  GBT (maxIter=30, maxDepth=12, stepSize=0.2, maxBins=32, subsamplingRate=0.6, minInstancesPerNode=100, seed=42);
  RF (numTrees=150, maxDepth=15, featureSubsetStrategy="sqrt", minInstancesPerNode=50, subsamplingRate=0.7, seed=42).
- Metrics train/test R²/RMSE/MAE; **selection = max test R²**.
- 🟩 **CROSS-VALIDATION IS NOT EXECUTED.** `NUM_FOLDS=5` and CV classes are imported but no
  `CrossValidator` is ever constructed/fit; hyperparameters are hardcoded from prior offline tuning.
  Any "5-fold CV" claim in markdown is false.

**Imputation (CELL 13):** `VectorAssembler(feature_cols, "features_raw", handleInvalid="keep")` on
full `df`; `final_model.transform`; replace `redfish_power` only where original is null; drop
`features_raw`,`prediction`. `imputed_flag` and the post-imputation null-verification block are
**commented out** (no provenance flag, no null re-check). 🟥 RISK: a tree model (GBT/RF) with
`handleInvalid="keep"` is not guaranteed to score null-feature rows; null-target+null-feature rows
may remain null and this is never verified.

**Persistence/validation (CELL 15/19/21):** the **only active write** is
`sandbox_finance.dcgm_metrics_raw_imputed_v2` — `ness.db_prepare` → `table_ref` →
**`DROP TABLE IF EXISTS` + `.format("iceberg").mode("overwrite").saveAsTable(...)` (DESTRUCTIVE, FLAGGED)**.
`assert "is_hpc_verification_job" in df_imputed.columns` guards the write (CELL 19). Daily/monthly
summary code (PART 2–5, `summary_sql`/`summary_sql_monthly`) is fully commented / non-executing —
and the previously crashing references to undefined summary globals were removed (D-4 fix). Read-back
(CELL 21) checks the imputed table for `region_rollup`, `gen` and shows `product_resolved`, `is_training`.

**Reconstruction recipe:** install deps → open CryptFileKeyring + CAIOS creds →
`NessieSparkClient(dbtcaster=True)` + AQE/shuffle=2000 → `select * from …dcgm_metrics_raw` →
feature-eng (ε=0.0001, `is_training==1`) → build `num_cols`, `.na.drop()`, cast Double → split(0.7/0.3,
seed42) + sample 0.02 → train 4 models, pick max test R² (no CV) → retrain on 0.10 →
assemble(keep)+transform+replace-null-only → assert `is_hpc_verification_job` → DROP+overwrite
`dcgm_metrics_raw_imputed_v2` → read-back. **Bounded variant:** add a `datestamp` filter or sample
fraction to the CELL 5 read; write to `sandbox.sandbox_<you>_repro.dcgm_metrics_raw_imputed_v2`.

## 7. Notebook 01.2 — EDA contract & reconstruction

**Role:** segmented EDA → 7 `dcgm_eda_*` tables. **Inputs (CELL 5):** `RAW_TABLE =
sandbox.sandbox_finance.dcgm_metrics_raw_imputed_v2` (🟩 D-2 FIXED — the earlier `raw_impute_v2`
bug is gone). 🟨 **`SUMMARY_TABLE = sandbox.sandbox_finance.dcgm_metrics_summary_imputed_v2` is
LEGACY/deprecated & STALE (D-15, OPEN).** An Option-B repoint to the StarRocks-native daily was
prototyped and then **reverted** (see D-15 in §12/§14 for why it's non-trivial).
`OUTPUT_SCHEMA=sandbox.sandbox_finance`, `OUTPUT_PREFIX=dcgm_eda`, `RAW_TIME_COL=datestamp`.

**Config (CELL 5):** CORE_RAW_METRIC_COLS = tensor_util,gpu_util,tensor_tflops,tensor_tflops_sm,
chip_power,redfish_power,sm_active; extended candidates dram_active,mem_copy_util,vram_usage,
sm_clock,sm_occupancy,TFLOPS_per_watt_efficiency; index cols auto-detected by `_index` suffix;
`RAW_SAMPLE_FRACTION=0.0005`, `SAMPLE_SEED=42`, `SCATTER_RENDER_LIMIT=10000` (unused); density
`NUM_BINS=64`, `PERCENTILE_RANGE=(0.01,0.99)`, day grain; `CORR_METHODS=[pearson,spearman]`;
`SEGMENT_DIMS=[gen,region_rollup,customer_segment,product_segment,is_sunk_active]`
(🟩 markdown claiming `is_training` is stale). `RUN_ID/RUN_TIMESTAMP` = UTC per run.

**Schema resolution (CELL 7):** summary time candidates `[day,date,datestamp,timestamp,ts]`;
percentile priority `[p50,p95,p99]`; summary features resolved as `{base_metric}_{pct}`; a
`raw_fallback` mode exists but is excluded from the percentile groups actually used (effectively
dead). Segment dims are trimmed to those present in BOTH raw and summary with only a warning
(🟥 silent scope reduction). RAISES only on: unresolved summary time/metrics; no density metrics;
`<2` overlapping validation metrics.

**Algorithms:** sampled raw `.sample(False,0.0005,42)`, cast double, fill segment nulls `"unknown"`,
`.na.drop(subset=CORE)`, stamp `segment_dim/value="all"`. Summary correlations per percentile tier ×
segment (baseline all/all + per-dim values via `.distinct().collect()`), pandas `.corr()` pearson/
spearman, upper-triangle only, normalize `_pXX` → base metric. Auto-validation = top-5 |corr| per
(percentile,method); raw validation vs Spark ML `Correlation` on the sample (report only).
Correlation-over-time: to timestamp, floor to day, **skip buckets `<3` rows**. Trend: melt
percentile cols, count/mean/median by day/metric/percentile/segment. Density: `percentile_approx(col,
[0.01,0.99], 1000)`, expand ±0.1%, 64 uniform bins + ±inf, `Bucketizer(handleInvalid="keep")` then
`bin-1`, retain bins 0..63, pairwise 2D counts in Spark (no full collect), reuse edges for over-time.

**Outputs (7, all `.write.mode("overwrite").saveAsTable()`)** — explicit StructTypes where defined:
- `dcgm_eda_raw_sample` — inferred: `datestamp`(source type), metrics→double, segment dims→string, `segment_dim/value`→string.
- `dcgm_eda_corr_summary` — StructType (CELL 12): metric_x/y (string, non-null), base_metric_x/y (string), method (string), summary_percentile (string), summary_feature_group (string), correlation_value (double), obs_count (long; **always -1**, unpopulated), run_id (string), run_timestamp (timestamp), summary_time_col (string), segment_dim/value (string, non-null).
- `dcgm_eda_corr_over_time` — StructType: time_bucket (timestamp), metric_x/y, base_metric_x/y, correlation_value (double), method, summary_percentile, run_id, time_grain, segment_dim/value.
- `dcgm_eda_density_bins` — inferred: metric_x, metric_y, x_bin (int), y_bin (int), count (long), segment_dim/value.
- `dcgm_eda_density_over_time` — inferred: + time_bucket.
- `dcgm_eda_bin_metadata` — StructType: metric_name, bin_number (int), bin_min/max (double), num_bins (int), percentile_low/high (double), run_id, segment_dim/value.
- `dcgm_eda_trend_summary` — StructType: time_bucket, metric_name, base_metric, summary_feature_group, observation_count (long), mean_value/median_value (double), run_id, segment_dim/value.

Validation (CELL 20) checks only readability + presence of key columns — **no** row-count, dup,
freshness, or range checks. Risks: `reduce()` over an empty density list raises `TypeError` if all
segments skipped (CELL 15/16); segment-value discovery sources differ across tables (raw vs summary)
→ possible misalignment; density default `[0,1]` fallback for all-null metrics.

**Reconstruction recipe:** config → load raw+summary → resolve schema (enforce the 3 raises) →
sampled raw → summary corr (+auto pairs) → raw validation → corr-over-time → persist corr (2 explicit
StructTypes) → density (guard empty `reduce`/`None`) → persist density (+bin_metadata) → trend →
persist → read-back key columns. **Bounded variant:** keep `RAW_SAMPLE_FRACTION` small; write to a
repro schema; FAIL if segmentation collapses to baseline-only.

## 8. Notebook 01.3 — parquet export contract & reconstruction

**Role:** read the 7 `dcgm_eda_*` tables and write local parquet + `manifest.json`. **Inputs (CELL 4/5):**
`spark.table()` on the 7 tables (short keys map to full names via `TABLES`); read-only, loaded lazily.

**Export layout (CELL 4/6/7):** `EXPORT_ROOT=/home/coreweave/dcgm_eda_parquet` (LOCAL disk — **no S3
bucket/prefix**); `EXPORT_TAG=utcnow().strftime('%Y%m%d_%H%M%S')`; per-run dir `EXPORT_ROOT/EXPORT_TAG/`;
one parquet dir per dataset (short key names) + `manifest.json`; `COMPRESSION='snappy'`; **no
partitioning**; `OVERWRITE=True` → `mode='overwrite'`. Full snapshot per run.

**`manifest.json` fields (CELL 7):** top-level `export_tag`, `export_root`, `created_utc`, `datasets`
{per dataset: `source_table`, `parquet_path`, `has_month_period`, `schema_preview` (first 12
`name:type` + "N total"), `time_column`, `column_count`, `columns`, `segmentation`{source_columns,
dimension_columns, has_segmentation}}, `segmentation_support`, `validation_results` [{dataset, valid,
issues}]. 🟩 **Does NOT contain:** row counts, schema fingerprint/hash, time range/min-max, a run-id
distinct from export_tag, or source git commit.

**month_period (CELL 6/7):** `date_format(to_timestamp(col),'yyyy-MM')` added to `raw_sample` (from
`datestamp`), `corr_over_time`, `density_over_time`, `trend_summary` (from `time_bucket`).

🟥 **Gating defects:** schema completeness is recorded (`validation_results[].valid`) but **export
proceeds unconditionally** — invalid/partial exports look successful; `valid=False` only when exactly
one of `segment_dim`/`segment_value` is present. CELL 6 `validate_schema_completeness` has an
inverted `missing = [col for col in required_cols if col in df.columns]` (name says missing, holds
present). No `_SUCCESS`/completeness marker; no "latest export" pointer (left to 01.4); no read-back
after write. `pyarrow/fsspec/s3fs` installed but unused (writes are local Spark parquet). Only CAIOS
read perms needed (source tables); no S3 write perms.

**Reconstruction recipe (clean-room, isolated tag):** after the Nessie session, override
`EXPORT_TAG='reconstruct_cleanroom_v100'` (non-timestamp), keep `TABLES`/`TIME_COLUMN_MAP`, run
load+export → `/home/coreweave/dcgm_eda_parquet/reconstruct_cleanroom_v100/{manifest.json, 7 dirs}`.
For a *safe* contract add: gate on `validation.valid`, fix the `missing` mis-naming, write a
`_SUCCESS` marker + per-dataset row-count read-back, and record row counts / source commit in the
manifest (none exist today).

## 9. Notebook 01.4 — local visualization contract & reconstruction

**Role:** LOCAL (no Spark) — render PNG/HTML charts from the 01.3 parquet export. **Export discovery
(CELL 3/4):** `discover_export_root(PARQUET_ROOT, EXPORT_TAG)`: if `EXPORT_TAG` set → that dir (else
`FileNotFoundError`); elif `PARQUET_ROOT/manifest.json` exists → `PARQUET_ROOT`; else pick the child
dir with a `manifest.json` **sorted by name descending (lexicographic, NOT mtime)**. `manifest.json`
is parsed but **never validated against on-disk datasets**. Paths: `PARQUET_ROOT=/home/coreweave/
dcgm_eda_parquet`, `OUTPUT_DIR=/home/coreweave/dcgm_eda_viz_outputs`, `FIG_DIR=OUTPUT_DIR/figures`,
`HTML_REPORT_DIR=OUTPUT_DIR/html_reports`, `PNG_ZIP_PATH=OUTPUT_DIR/dcgm_eda_png_summary_pack.zip`.

**Inputs/params (CELL 2/4):** reads all 7 datasets via `pyarrow.dataset` into pandas; `bin_metadata`
and `trend_summary` are loaded but not charted (row-count only). `MAX_MONTHS=15`, `MAX_PAIRPLOT_ROWS
=4000`, `MAX_SCATTER_ROWS_PER_MONTH=2500`, `MAX_RAW_SAMPLE_ROWS=2_000_000`, `MAX_SEGMENT_VALUES_PER_
DIM=10`, `MIN_SEGMENT_ROWS=100`, `SEED=42`, `DPI=140` (but `save_figure` hardcodes `dpi=160`),
`CORR_GROUPS_TO_RENDER=['p50','p95']`, `SEGMENT_DIMS_TO_RENDER=[gen,region_rollup,customer_segment,
product_segment,is_training]`.

**Charts (filenames end `_{segment_context}.png`):** `01_core_density_*` (density_bins, log1p sum),
`02_monthly_density_*` (density_over_time), `03_histograms_*` / `04_boxplots_*` /
`05_monthly_metric_boxplots_*` (raw_sample), `06_corr_heatmap_{p50,p95}_*` (corr_summary, filtered on
`summary_feature_group`), `07_monthly_correlation_heatmaps_*` (corr_over_time, filtered on
`summary_percentile`), `08_pairwise_scatter_matrix_*` / `09_monthly_scatter_*` (raw_sample),
`11_drift_heatmap_*` (raw_sample IQR-scaled). Writes **local files only** — no catalog writes.

🟥 **Risks:** `load_parquet_dataset` silently drops requested columns absent from schema — but
`render_correlation_heatmap` accesses `summary_feature_group` and `render_monthly_correlation_drift`
accesses `summary_percentile` unconditionally → **KeyError if missing**, and the two datasets are
filtered on **different** column names for the same `p50/p95` literal (asymmetric — verify against
the 01.3 export). Hardcoded customer literals `Bigbird`/`Internal` (CELL 9, diagnostic). No `Agg`
backend set → headless needs `MPLBACKEND=Agg`. `scipy`, `EFFICIENCY/WORKLOAD/DIAGNOSTIC_INDEX_METRICS`,
`pyarrow.compute` defined/loaded but unused.

**Reconstruction recipe (headless):** `MPLBACKEND=Agg jupyter nbconvert --to notebook --execute
--ExecutePreprocessor.timeout=-1 "01.4_…(9).ipynb"`; set `EXPORT_TAG` in CELL 2 to pin the export
(don't rely on lexicographic auto-pick); point `PARQUET_ROOT`/`OUTPUT_DIR` at explicit dirs. This is
the only notebook runnable with **no credentials**, given an export on disk.

## 10. BMaaS TFLOPs contract & reconstruction

**Role:** two-stage stacked model deriving BMaaS tensor-TFLOPs from Redfish rack power. **Inputs:**
`nessie.staging_capacity_finance.dcgm_metrics_raw` (training, CELL 5); `nessie.staging_victoria_
metrics.redfish_stats_power_shelf_rail_output_power_watts` (scoring, CELL 16, 18 hardcoded zones,
dedup `MAX(value)` per device slot → `SUM` to per-rack `redfish_power`); `nessie.model_conformed.
dim_customer` (CELL 18, left join `org_id=cluster_org`, cols customer_name/flag_is_coreweave).

**Mappings (CELL 17/18):** `zone_mapping` covers **17** zones (zone → product_resolved,
gpu_count_expected, node_per_rack, peak_power_unit, peak_tflops_unit); `US-WEST-02B` is in the CELL 16
filter but **absent** from `zone_mapping`. Derived `gpu_peak_power_node_watts = gpu_count_expected*
peak_power_unit`, `gpu_peak_power_rack_watts = ×node_per_rack`. `customer_segment` CASE (Bigbird / AI
Lab [`Mistral AI`,`Cohere`, case-sensitive] / Internal / External-Other); `product_segment` from
`product_specs` (HGX/PCIE/MGX/Workstation); `gen` current/prior. `RACK_GPUS=72` hardcoded (CELL 21).

**Feature eng (CELL 6):** same `_index` formulas as 01.1 (ε=0.0001) + `redfish_power_pct_max`; `gen`,
`region_rollup`, `training_inference` (here **`is_training=="slurm"`**). Only 8 columns feed the model
(`num_cols`, CELL 7): tensor_tflops, chip_power, redfish_power, peak_power_unit,
gpu_peak_power_node_watts, peak_tflops_unit, gpu_count_expected, redfish_power_pct_max; all `*_index`/
gen/region/training_inference are computed but unused.

**Model (CELL 8–12):** Stage-1 `LinearRegression(chip_power ← 6 base features, regParam=0.01,
elasticNetParam=0.0, maxIter=100, standardization=False)` → `chip_power_hat`. Stage-2 candidates on
`base + chip_power_hat`: Simple LR (redfish_power only, `is_stacked=False`, excluded from selection);
Ridge (StandardScaler→LR same params); GBT (maxIter=30,maxDepth=12,stepSize=0.2,maxBins=32,
subsamplingRate=0.6,minInstancesPerNode=100,seed=42); RF (numTrees=150,maxDepth=15,sqrt,
minInstancesPerNode=50,subsamplingRate=0.7,seed=42). Split 0.7/0.3 seed42 then sample 0.02 (final
0.10). **Best = max test R² among stacked only.** 🟩 CV NOT run (NUM_FOLDS=5 defined, no
CrossValidator). 🟥 **In-sample stacking leakage** (CELL 9 NOTE, quoted): Stage-2 trains on Stage-1's
in-sample `chip_power_hat` on the training split — train metrics optimistic (test `chip_power_hat` is
out-of-sample).

**Scoring/aggregation (CELL 20/21):** score rack power at node-equivalent (`redfish/node_per_rack`),
predict, rescale `×node_per_rack`; monthly rollup at grain `(month, cluster_org, region, zone,
product_resolved, gpu_count_expected, node_per_rack, customer_name, customer_segment, product_segment,
gen)`; percentiles via `percentile_approx(...,1000)`; `rack_peak_tflops = peak_tflops_unit*72`;
`tensor_util_p50/p95 = tflops_total_pXX / rack_peak_tflops`.

**Output (CELL 22, 🟩 D-3 FIXED):** `sandbox.sandbox_finance.dcgm_metrics_bmaas` (write **and**
read-back CELL 28 both use this name now). **`DROP TABLE IF EXISTS` + iceberg overwrite `saveAsTable`
(DESTRUCTIVE)**. Only the monthly rollup is written. Downstream: StarRocks view `v_dcgm_metrics_bmaas`.
🟩 CELL 27 changelog still claims `..._tflops` via `db_load` — stale/wrong vs live code.

🟥 **Risks:** `US-WEST-02B` filtered but unmapped → null specs → dropped by `handleInvalid="skip"`
(silent undercount — accepted risk per owner); `rack_peak_tflops=×72` wrong for H100/H200 (24 GPU/
rack) and B200 (40 GPU/rack) zones present in the map; leakage above; destructive write.

**Reconstruction recipe:** session → load raw → feature-eng → 8 `num_cols` na.drop cast → Stage-1 LR
+ Stage-2 (pick best stacked by test R², no CV) → retrain 0.10 → **preflight: assert
`set(filter_zones)==set(zone_mapping)` and no null map outputs (fail loudly instead of silent drop)**
→ score rack power (node-equiv) → monthly rollup (replace flat 72 with per-zone
`gpu_count_expected*node_per_rack`) → DROP+overwrite `dcgm_metrics_bmaas` → read-back.

## 11. Time-series forecast contract & reconstruction

**Role:** demand forecasting to 2031 with an exposure-weighted hierarchy rollup. **Inputs:** StarRocks
JDBC `jdbc:mysql://kube-starrocks-warehouse-fe-service.starrocks.svc.cluster.local:9030/sandbox_
finance`, `dbtable=dcgm_metrics_summary_imputed_daily` → overwrite Iceberg
`sandbox.sandbox_finance.dcgm_metrics_summary_imputed_daily_sync` (CELL 7); then **reads the `_sync`
table** (CELL 8; 🟩 markdown naming the non-`_sync` table is wrong). Executor wheelhouse
`/tmp/sparkcaster_wheels.zip` ships `statsmodels, scipy, pandas, numpy, patsy` (**Prophet NOT
included**).

**Grouping/metrics/horizon:** `region_summary = EU if region.startswith('EU') else NAM` (CELL 10).
🟩 METRICS actually forecast (CELL 11) = **`redfish_power_fleet_p50/p95`, `tensor_tflops_fleet_p50/p95`
only** (markdown's 8-metric/util list is wrong; none are util so the `[0,1]` clip never fires).
GROUPINGS (8) key on physical columns incl. `region_rollup` (🟥 the 3 `region_summary*` groupings use
`region_rollup`, which must exist or baseline `groupBy` throws — hierarchy path uses `region_summary`).
`TRAIN_SPLIT=0.7`; `FORECAST_END_DATE=2031-05-31`; `FORECAST_DAYS=(END−max(day)).days` (derived, not
1100); per series `horizon=n_test+FORECAST_DAYS`, skip series `<21` points. Backtest (CELL 48):
`BT_HOLDOUT_DAYS=60`, `BT_METRICS`=first 2 present, `BT_CUTS=[All,region_summary,product_resolved,
customer_segment]`.

**Exposure weights (CELL 37/38):** `BASE_DIMS=[region_summary,product_resolved,customer_segment]`;
`weight = node_count_daily_avg × gpu_count_expected`. `COLMAP._resolve` ladder: node_count ←
first of (`node_count_daily_avg`,`node_count`,`record_count`); gpu_count ← first of
(`gpu_count_expected`,`gpu_count`). `EXPOSURE_MODE = node_x_gpu → node_only → gpu_only → unweighted
(lit 1.0)`. 🟥 **Silent fallback** — only base dims (product_resolved, customer_segment, day) are
asserted; missing exposure columns degrade weighting with only a printed `⚠️ FALLBACK` (owner: those
columns are always present, so this should never fire; accepted risk). `record_count` satisfying
node_count silently yields `node_only`.

**Models (CELL 13):** ExponentialSmoothing(seasonal_periods=7, trend/seasonal add); ARIMA(5,1,0);
SARIMAX(1,1,1)(1,1,1,7); Prophet(interval_width=0.8, native intervals, only `if has_prophet`);
Holt-Winters(seasonal_periods=7, trend add, seasonal mul, damped). Select **min MAE**. Intervals:
`_Z_P10_P90=1.2816`, residual-std band (Prophet uses native); clip `≥0` always, `≤1` only for util
metrics. 🟥 **Prophet absent on executors** → distributed path fits **4 of 5** models.

**Outputs:** Iceberg `_sync` (overwrite); S3 to `s3://jbok-sandbox-test/jbok/time-series-fcst-
sparkcaster/<RUN_TS>/` (endpoint `cwlota.com`, region `US-EAST-04A`), CSV + XLSX (XLSX skipped when
`>1,048,575` rows): `all_models_results`, `best_models_results`, `forecast_daily_values`,
`forecast_monthly_values`, `forecast_monthly_values_all`, `forecast_monthly_values_all_with_history`,
`forecast_daily_values_hier[_<POC_METRIC>]`, `forecast_monthly_values_hier[_<POC_METRIC>]`,
`backtest_summary_hier_vs_baseline`, `backtest_detail_hier_vs_baseline`. 🟥 Hierarchy parent =
`Σ(child·weight)/Σ(weight)` — a weighted average of p50/p95 is **NOT a true percentile** (approximate
coherence, flagged in code). CELL 34/35 legacy/dead (perf note, `forecasts.zip`).

**Reconstruction recipe:** extract hardcoded params (StarRocks/Nessie/S3 endpoints, FORECAST_END_DATE,
TRAIN_SPLIT, wheel path, Spark sizing, METRICS/GROUPINGS/BASE_DIMS) → JDBC sync → prep+cache (fix
`region_rollup`→`region_summary` refs) → build wheelhouse (add prophet for 5-model parity) → tasks
(avg per day) → distributed fit (skip <21, 4–5 models, min MAE, ±1.2816σ band, clip) → export baseline
CSV/XLSX → optional hierarchy (exposure ladder, weighted rollup, backtest) → **record `exposure_mode`
and the weighted-percentile approximation in outputs**. **Bounded variant:** 1 metric, short
`FORECAST_END_DATE`, scratch S3 prefix; requires the wheel bundle + StarRocks creds.

---

## 12. Input and output data contracts

🟥 Column *Spark types*, natural keys, partitioning, row counts, and date ranges are **NOT verified**
(no catalog auth). Column *names* are from code; where a StarRocks view DDL exists it is authoritative
for the column list. Complete each with `DESCRIBE <table>` + bounded `count/min/max` once creds exist.

**Inputs**

| Object | Engine | Key columns referenced (🟩 code) | Notes |
|---|---|---|---|
| `nessie.staging_capacity_finance.dcgm_metrics_raw` | Iceberg | datestamp, redfish_power (target), chip_power, tensor_tflops, tensor_tflops_sm, tensor_util, gpu_util, sm_active, sm_clock, sm_occupancy, dram_active, mem_copy_util, vram_usage, peak_power_unit, gpu_peak_power_node_watts, peak_tflops_unit, gpu_count_expected, **is_hpc_verification_job**, is_training, product_resolved, region, cluster_org | time col `datestamp`; ~900M rows (🟨 code comment). Source of 01.1 + bmaas. |
| `nessie.staging_victoria_metrics.redfish_stats_power_shelf_rail_output_power_watts` | Iceberg | zone, value, datestamp, cluster_org, region, data_hall, rack, deviceslot, serial | bmaas scoring source. |
| `nessie.model_conformed.dim_customer` | Iceberg | org_id, customer_name, flag_is_coreweave | bmaas enrichment (join org_id=cluster_org). |
| StarRocks `sandbox_finance.dcgm_metrics_summary_imputed_daily` | StarRocks | day, product_resolved, customer_segment, product_segment, region_rollup, node_count_daily_avg, gpu_count_expected, record_count, redfish_power_fleet_p50/p95, tensor_tflops_fleet_p50/p95 | tsfcst source. 🟨 **created natively in StarRocks via SQL** (not in repo). Also `_monthly` and `dcgm_histogram` are StarRocks-native. |
| `sandbox.sandbox_finance.dcgm_metrics_summary_imputed_v2` | Iceberg | `{metric}_{p50/p95/p99}` percentile columns + segment dims | 01.2 input. 🟨 **LEGACY & STALE** — formerly written by 01.1, now deprecated/unrefreshed. Repointing 01.2 to the StarRocks-native daily is **non-trivial** (column-naming mismatch drops all 7 CORE metrics) — see D-15 in the lineage note below. |

**Outputs**

| Object | Producer | Write mode | Notes |
|---|---|---|---|
| `sandbox.sandbox_finance.dcgm_metrics_raw_imputed_v2` | 01.1 | DROP+overwrite (iceberg) | **48-col schema authoritative via StarRocks view** `v_dcgm_metrics_raw_imputed` (below). |
| `sandbox.sandbox_finance.dcgm_eda_{raw_sample,corr_summary,corr_over_time,density_bins,density_over_time,bin_metadata,trend_summary}` | 01.2 | overwrite | schemas in §7. |
| `/home/coreweave/dcgm_eda_parquet/<tag>/` (+manifest.json) | 01.3 | local overwrite | snappy, unpartitioned, per-run tag, no cleanup. |
| `/home/coreweave/dcgm_eda_viz_outputs/` | 01.4 | local files | PNG/zip/HTML. |
| `sandbox.sandbox_finance.dcgm_metrics_bmaas` | bmaas | DROP+overwrite (iceberg) | monthly rollup; downstream StarRocks view `v_dcgm_metrics_bmaas`. |
| `sandbox.sandbox_finance.dcgm_metrics_summary_imputed_daily_sync` | tsfcst | DROP+overwrite (iceberg) | StarRocks→Iceberg copy. |
| `s3://jbok-sandbox-test/jbok/time-series-fcst-sparkcaster/<RUN_TS>/*.{csv,xlsx}` | tsfcst | overwrite | forecast + backtest outputs. |

🟩 **`dcgm_metrics_raw_imputed_v2` authoritative column list** (from
`data-engineering/dbo/{sandbox_finance,staging_finance}/views/v_dcgm_metrics_raw_imputed.sql`, 48
columns): datestamp, node, gpu_util, tensor_util, chip_power, dram_active, mem_copy_util, vram_usage,
sm_active, sm_clock, sm_occupancy, region, zone, cluster, cluster_org, cw_sku, model, serial,
redfish_power, customer_name, flag_is_coreweave, model_imputed, product, product_resolved,
customer_segment, is_training, is_sunk_running, is_hpc_verification_job, is_multinode, infiniband_data,
gpu_count_expected, peak_tflops_unit, peak_power_unit, product_segment, memory_boundness_index,
TFLOPS_per_watt_efficiency, compute_occupancy_index, tensor_util_dram_index, tensor_tflops_sm,
tensor_tflops, gpu_peak_power_node_watts, tensor_to_gpu_ratio_index, memory_intensity_index,
compute_to_memory_ratio_index, power_pct_max_index, compute_pct_max_index, gen, region_rollup,
training_inference. (Column comments in the DDL give business meaning; Spark types still need `DESCRIBE`.)

🟨 **DATA-LINEAGE RESOLUTION (owner-provided 2026-09-21; 🟩 corroborated by repo absence-of-producer):**
- **`dcgm_metrics_summary_imputed_daily`, `dcgm_metrics_summary_imputed_monthly`, `dcgm_histogram`** —
  created **natively in StarRocks via SQL queries**, not by any notebook or by a repo job (grep of
  `data-engineering` and `jbok-cw` returns no producer, consistent with native StarRocks DDL). tsfcst
  consumes the `_daily` one over JDBC. 🟥 The exact StarRocks SQL/DDL that materializes them is
  external to both repos — capture it separately if full reproduction of the summary layer is needed.
- **`dcgm_metrics_summary_imputed_v2`** — **LEGACY.** It was produced by an older version of `01.1`
  (whose summary writes are now disabled, D-4) and has been **superseded**. It is NOT maintained.
- **Current serving layer** = the dbo StarRocks views (present in
  `data-engineering/dbo/{staging_finance,sandbox_finance}/views/`): `v_dcgm_metrics_bmaas`,
  `v_dcgm_metrics_raw`, `v_dcgm_metrics_raw_imputed`; plus the StarRocks-native tables
  `dcgm_metrics_summary_imputed_daily`, `dcgm_metrics_summary_imputed_monthly`, `dcgm_histogram`.

➡️ **Consequence (D-15) — OPEN (a naive repoint is NOT safe; Option B was prototyped then reverted):**
`01.2` reads the deprecated `dcgm_metrics_summary_imputed_v2` (stale/unrefreshed). Only the 3
summary-driven outputs are affected (`dcgm_eda_corr_summary`, `dcgm_eda_corr_over_time`,
`dcgm_eda_trend_summary`); the 4 raw/density outputs already read the current
`dcgm_metrics_raw_imputed_v2`. tsfcst's source lineage was already correct (StarRocks-native daily).

**Why repointing to the StarRocks-native daily is non-trivial** (verified against the daily's actual
columns, 2026-09-21): 01.2's resolver matches `f"{base}_{pct}"` for base ∈ the 7 CORE metrics + extended
+ raw `_index` columns. In the current daily:
- **All 7 CORE metrics resolve to nothing.** The util percentiles (`tensor_util`, `gpu_util`,
  `sm_active`, `dram_active`, `mem_copy_util`, `sm_occupancy`) were **moved to `sandbox_finance.dcgm_histogram`**
  and are absent from the daily; the power/tflops/vram percentiles exist only as **`*_fleet_p50/p95/p99`**
  (e.g. `chip_power_fleet_p50`, `redfish_power_fleet_p50`, `tensor_tflops_fleet_p50`), which the resolver
  does not try.
- **5 of 8 `_index` metrics don't match** because the daily dropped the `_index` suffix
  (`tensor_to_gpu_ratio_p50` vs candidate `tensor_to_gpu_ratio_index_p50`, etc.).
- Only **5 metrics would resolve** — `sm_clock`, `TFLOPS_per_watt_efficiency`, `memory_boundness_index`,
  `compute_occupancy_index`, `tensor_util_dram_index` — which is ≥2, so the notebook **runs without
  error but silently analyzes the wrong/narrow metric set** (D-10 class).
- **`dcgm_histogram` cannot substitute** in the summary-correlation path: it is bins/counts, not
  `{metric}_pXX` percentile columns; and 01.2's density path already builds its own histograms from raw.

A correct D-15 fix therefore needs, at minimum: (a) a fail-loud guard so CORE dropout raises instead of
silently narrowing; (b) resolver support for `_fleet` and suffix-stripped `_index` names; and (c) an
explicit decision on the pure-util metrics (accept their loss from the summary correlations, or derive
percentiles from `dcgm_histogram`). Both 01.2 and tsfcst otherwise remain BLOCKED only on credentials
(tsfcst also on the wheel bundle).

## 13. Validation and acceptance tests

Least-expensive-first. `READY` requires a passed Tier-C bounded run (needs creds) — not claimable now.

- **Tier A — static, no creds (RUN; PASS):** `nbconvert --to script | py_compile` for 01.1/01.2/bmaas
  → compile; grep confirms no stale `dcgm_metrics_raw_impute_v2` / `dcgm_metrics_bmaas_tflops`.
- **Tier B — read-only catalog (needs creds):** per input, `DESCRIBE` + bounded `count/min/max` to
  confirm existence, schema, scale, freshness; resolves the §12 lineage + type gaps.
- **Tier C — bounded writes to `sandbox_<you>_repro` (needs creds + authorization):** per-notebook
  recipes above; each emits a run manifest.
- **Tier D — full run:** only after A–C pass and production-scale execution is authorized.

**Acceptance tests (explicit):**
1. Source & output table names exactly match §12 (no `impute`/`_tflops` drift).
2. Required columns + Spark types present (via `DESCRIBE`).
3. Imputation: `redfish_power` null-count after 01.1 == 0 for rows with non-null features; originally
   non-null values unchanged.
4. Feature-eng mappings reproduce (ε=0.0001; `gen`/`region_rollup`/`training_inference` per notebook —
   note 01.1 uses `is_training==1`, bmaas uses `=="slurm"`).
5. Model metadata: 70/30 seed42 split; selected model + hyperparameters recorded; **CV not claimed**.
6. Segment coverage: 01.2 emits baseline + each expected `SEGMENT_DIMS` value; FAIL on collapse.
7. Percentile-group resolution recorded ({metric}_{p50/p95/p99}).
8. Correlation output grain (per percentile×method×segment; upper-triangle).
9. Density bin range + metadata consistency (64 bins, (0.01,0.99), edges reused over-time).
10. Daily time bucketing (floor to day; skip `<3`-row buckets).
11. Output-table readability + key columns present.
12. Overwrite safety/idempotency: destructive DROP+overwrite writes go only to a repro schema in
    validation; re-run yields same row counts.
13. Failure on missing required inputs (preflight table existence).
14. Detection of silent fallback/scope reduction: record `exposure_mode` (tsfcst), zone-coverage
    (bmaas), segmentation completeness (01.2), export `validation.valid` (01.3); mark run DEGRADED if any fired.

**Scaffolding (proposals, not wired in):** (a) credential shim preferring `${SERVICE}_{KEY}` env or a
mounted Secret over interactive keyring, never printing values; (b) fail-loud preflight asserting
creds/wheels/table-existence before compute; (c) a run manifest per run recording notebook code
sha256, repo commit `58a0276`, runtime image digest, package versions, parameters, input/output
objects + row counts, `exposure_mode`/degraded fallbacks, and validation outcomes.

## 14. Known defects, risks, and required decisions

**Fixed (committed `58a0276`):** D-2 (01.2 reader → `dcgm_metrics_raw_imputed_v2`); D-3 (bmaas output
unified to `dcgm_metrics_bmaas`); D-4 (01.1 clean-run crash from undefined summary globals removed;
summaries are downstream StarRocks). Tier-A static-verified.

**Accepted risks (owner-deferred; documented, not fixed):**

| Risk | Evidence | Consequence |
|---|---|---|
| Credentials out of scope | §4 | no runtime validation possible here. |
| Exposure-weight silent fallback (tsfcst) | CELL 37/38 | owner: exposure cols always present → should never fire; if it does, rollups become unweighted averages silently. |
| bmaas unmapped zone `US-WEST-02B` | CELL 16 vs 17 | rows dropped via `handleInvalid="skip"`; zone undercounted. |
| `rack_peak_tflops=×72` for non-GB200/GB300 | bmaas CELL 21 | wrong `tensor_util_*` for H100/H200/B200 zones. |
| Executor wheel bundle absent; Prophet not on executors | §3/§11 | tsfcst distributed path blocked / 4-of-5 models. |
| 01.2 silent segmentation collapse | CELL 7 | EDA tables lose dims with only a warning. |
| 01.3 exports when invalid; no tag cleanup | CELL 6/7 | invalid exports look complete; disk grows. |
| Hardcoded paths/endpoints/dates/customer literals | multiple | not portable/headless-safe. |
| `nessie-prd` endpoint with `sandbox` catalog | all Spark NBs | sandbox writes traverse prod Nessie — confirm isolation. |
| 01.1 full-history `SELECT *` | CELL 5 | ~900M rows/run; no bounded mode. |
| `nbstripout` not installed | §2 | output noise in diffs. |
| Destructive DROP+overwrite writes | 01.1 C19, bmaas C22, tsfcst C7 | no backup beyond Iceberg history; write to repro schema in validation. |
| In-sample stacking leakage (bmaas) | CELL 9 NOTE | optimistic training metrics. |
| CV imported but never run (01.1, bmaas) | CELL 8/9 | "5-fold CV" claims false; hyperparameters hardcoded. |
| **D-15 (OPEN): 01.2 reads legacy/stale `dcgm_metrics_summary_imputed_v2`** | §12 lineage | Naive repoint to the StarRocks-native daily silently drops all 7 CORE metrics (util moved to `dcgm_histogram`; power/tflops are `*_fleet_*`; 5 index cols lost `_index`) → 01.2 would run over the wrong 5 metrics with no error. Option B prototyped then reverted. Needs resolver work (see §12). |

**Lineage resolved (2026-09-21, owner):** the summary/histogram tables are created natively in
StarRocks via SQL (out of both repos); `dcgm_metrics_summary_imputed_v2` is legacy; serving layer is
the dbo views + StarRocks-native tables. See §12.

**Required owner decisions still open:** (1) **D-15** — how to repoint 01.2's summary: add resolver
support for `_fleet`/suffix-stripped `_index` names + a fail-loud CORE-dropout guard, and decide whether
the pure-util metrics (now only in `dcgm_histogram`) are dropped from the summary correlations or derived
from the histogram; capture the external StarRocks SQL/DDL if the summary layer must be reproduced;
(2) confirm sandbox↔prod isolation at `nessie-prd`; (3) authorize a bounded validation run + method if
runtime validation is ever wanted; (4) approve `nbstripout --install`.

## 15. Execution status and blockers

- **01.4** — READY_WITH_CONDITIONS: runnable locally (no creds) against an existing 01.3 export;
  set `MPLBACKEND=Agg` and pin `EXPORT_TAG`.
- **01.1, 01.2, 01.3, bmaas, tsfcst** — BLOCKED for runtime: **catalog credentials unavailable (out of
  scope)**. Code is fixed + Tier-A static-validated. Notes: 01.2 still reads the legacy/stale
`dcgm_metrics_summary_imputed_v2` for its 3 summary-driven outputs (D-15 OPEN — repoint is non-trivial,
see §12); its 4 raw/density outputs are unaffected. tsfcst additionally needs the executor wheel bundle.
The summary-producer lineage is resolved (StarRocks-native) and is no longer a blocker.
- **No notebook is `READY`** — a passed bounded run is required and cannot occur without credentials.

## 16. Change history

- **2026-09-21 (D-15 Option B prototyped, then REVERTED):** Prototyped repointing `01.2`'s summary to
  the StarRocks-native daily via JDBC, then reverted `01.2` to `58a0276` after discovering the daily no
  longer carries the 7 CORE percentile metrics (util moved to `dcgm_histogram`; power/tflops are
  `*_fleet_*`; several `_index` cols renamed) — so a naive repoint would silently analyze the wrong 5
  metrics. D-15 remains OPEN with the resolver requirements documented in §12. Working tree is clean.
- **2026-09-21 (lineage resolution):** Owner confirmed the summary/histogram tables
  (`dcgm_metrics_summary_imputed_daily`, `_monthly`, `dcgm_histogram`) are created **natively in
  StarRocks via SQL** (external to both repos); `dcgm_metrics_summary_imputed_v2` is **legacy** (old
  01.1 output, superseded); serving layer is the dbo views (`v_dcgm_metrics_bmaas`, `v_dcgm_metrics_raw`,
  `v_dcgm_metrics_raw_imputed`) + those StarRocks-native tables. Recorded new issue **D-15** (01.2 reads
  the deprecated `_v2`; repoint to the StarRocks-native daily). The §12 lineage BLOCKER is downgraded to
  resolved; tsfcst source lineage is no longer a blocker.
- **2026-09-21 (this upgrade):** Rewrote the handoff into the 16-section reconstruction structure.
  Added cell-level contracts for all six notebooks (verified against current source of `58a0276`),
  the runtime/connector/version baseline, the `NessieSparkClient` session facts, the authoritative
  48-column `v_dcgm_metrics_raw_imputed` schema, and the data-lineage finding (summary producer not in
  `data-engineering` → BLOCKED). Corrected several doc-vs-code conflicts (CV not executed; ε=0.0001;
  01.1 `is_training==1` vs bmaas `=="slurm"`; tsfcst reads `_sync` and forecasts 4 metrics with 4/5
  models on executors; 01.2 outputs have explicit StructTypes; 01.3 manifest lacks row counts/
  fingerprint/commit).
- **2026-09-21 (earlier):** Fixed D-2/D-3/D-4 and committed to `jbok-cw` `58a0276`; D-7 canonical
  drift resolved (HEAD == on-disk); D-5 accepted (exposure cols always present); D-1 out of scope;
  D-6/D-8–D-14 accepted risks.
- **2026-09-21 (initial):** Built the original handoff from read-only inspection: access map,
  namespace inventory, discrepancy register (D-1…D-14), data contracts, validation ladder.

*End of handoff.*
