jbok-cw

Capacity & finance analytics for CoreWeave long-term demand planning. This repo holds the notebooks, reusable code, and documentation behind GPU/CPU/network telemetry analysis, imputation, workload classification, and demand modeling.

What's here

This project turns raw infrastructure telemetry (DCGM, Redfish power, Victoria Metrics) into curated summary tables and demand models used for capacity planning and financial forecasting. Core workflows include ML-based imputation of missing metrics, daily/monthly aggregation, and correlation of GPU usage with tokens, CPU, and network utilization.

Repository layout

jbok-cw/
├── notebooks/   # Jupyter notebooks (imputation, summaries, forecasting)
├── src/         # Reusable Python: Spark/Nessie helpers, shared utils
├── docs/        # Design docs, data-model notes, runbooks
├── exports/     # Generated outputs (xlsx/csv/png) — gitignored, not tracked
└── .gitignore

Tech stack





Compute / transforms: Spark (PySpark), dbt



Catalog / lakehouse: Nessie + Iceberg



Serving / query: StarRocks



Metrics sources: DCGM, Redfish power, Victoria Metrics (PromQL)



ML: regression-based imputation, workload classification

Environment setup

Run inside the Kubeflow notebook (fresh volume, current Claude Code extension).

# clone
git clone https://github.com/jbok-fin/jbok-cw.git ~/jbok-cw
cd ~/jbok-cw

# keep notebook diffs clean (strip outputs on commit)
pip install nbstripout && nbstripout --install

Credentials

Credentials are stored in the notebook keyring, not in this repo. Never commit keys, tokens, or .env files.





CAIOS object storage access/secret keys (used by Spark + Nessie writes)



Set/retrieve via keyring in the notebook session

Example (as used in the notebooks):

import keyring
caios_access_key = keyring.get_password("caios", "access_key")
caios_secret_key = keyring.get_password("caios", "secret_key")

Key data references

Common tables written / read by the pipelines (update as these evolve):





nessie.staging_victoria_metrics.redfish_stats_power_reading — raw Redfish power



nessie.staging_capacity_finance.redfish_grouped — cleaned/aggregated power



sandbox.sandbox_finance.dcgm_metrics_raw_imputed_v2 — imputed raw DCGM metrics



sandbox.sandbox_finance.dcgm_metrics_summary_imputed_daily_v2 — daily summary



sandbox.sandbox_finance.dcgm_metrics_summary_imputed_monthly_v2 — monthly summary

Conventions





One canonical notebook per workflow — use git history instead of copy/v2/backup filenames.



Generated artifacts go in exports/ (gitignored); commit only source and docs.



Notebook outputs are stripped on commit via nbstripout.

Notes

Working notebook environment: VS Code connected to the Kubeflow pod (Remote-SSH via WSL port-forward). Long-running Spark/imputation jobs should run under tmux so a dropped connection never kills the run.