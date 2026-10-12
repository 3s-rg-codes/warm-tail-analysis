# warm-tail-analysis

Analysis code for *The Tail Stays Warm*.
It measures how slow the slowest *warm* invocations are on seven commercial FaaS platforms and checks the result against four previously published datasets.

The primary data is a 53-day run across nine provider-architecture combinations (AWS Lambda x86/ARM, Google Cloud Run Functions, Oracle Cloud Functions x86/ARM, Cloudflare Workers, Deno Deploy, Fastly Edge Compute, Fly.io), plus an AWS EC2 `t3.nano` baseline.
Only the analysis is in this repository.
The raw data has to be downloaded separately (see [Data](#data)).

```
data/             EC2 baseline run, benchmark configuration, aggregates behind the validation figures
figures/          Generated figures; figures/tertiary/ for the validation datasets
notebooks/        Main analysis of the primary dataset
tertiary-repos/   The same metrics on four third-party datasets, one folder per dataset
```

## Setup

```sh
uv sync
uv run jupyter lab
```

pandas is pinned below 3.0 because the committed notebook output was produced on 2.x. All paths
are relative to the notebook's own directory, so the kernel must run from there.

## Notebooks

| Notebook | Covers |
|---|---|
| `notebooks/00_tail_latency.ipynb` | Daily percentile stability, tail factors (TF = pXX / p50) per platform |
| `notebooks/01_patterns.ipynb` | Intraday and weekly patterns, per-provider events |
| `notebooks/02_cold_impact.ipynb` | Cold factor: all-calls / warm-only latency per percentile |
| `notebooks/02_warm_share_of_slowest.ipynb` | Share of warm starts among the slowest N% of calls; cold-start enrichment |
| `notebooks/03_tail_percentile_robustness.ipynb` | Bootstrap and order-statistic CIs for p99.9 and p99.99 |
| `notebooks/04_tail_latency_effect.ipynb` | Tail compounding across workflows (max-of-N, tail mass, CCDF slope) |
| `notebooks/05_language_runtime_tail.ipynb` | Go vs. JS runtime effect on the tail |
| `notebooks/baseline.ipynb` | FaaS platforms against the EC2 baseline |
| `tertiary-repos/overall_analysis.ipynb` | Cross-dataset tail factors and cold factors |
| `tertiary-repos/warm_share_of_slowest.ipynb` | Warm share of the slowest calls, validation datasets |

### Paper figures

| Figure | Notebook |
|---|---|
| Relative tail factors | `notebooks/00_tail_latency.ipynb` |
| Intraday tail factors per platform | `notebooks/01_patterns.ipynb` |
| ECDF above p95 | `notebooks/baseline.ipynb` |
| Warm share of the slowest calls | `notebooks/02_warm_share_of_slowest.ipynb` |
| Warm share, validation datasets | `tertiary-repos/warm_share_of_slowest.ipynb` |
| Tail factors, validation datasets | `tertiary-repos/overall_analysis.ipynb` |
| Intraday tail factors, validation datasets | `tertiary-repos/overall_analysis.ipynb` |
| Huawei tail factors per configuration | `tertiary-repos/overall_analysis.ipynb` |

The Huawei inputs to these figures are committed in `data/tertiary_aggregations/`, so the cross-dataset notebooks can run without the 63 GB raw trace.

## Data

### Primary dataset (required for `notebooks/`)

Schirmer et al., *New Kids: An Architecture and Performance Investigation of Second-Generation Serverless Platforms*.

1. Download the archive from <https://depositonce.tu-berlin.de/items/70f1556b-b046-494e-8fa3-c029cf1459d8>, extract it and put the SQLite file at the repository root as `data.db`.
2. Check it:

   ```sh
   sqlite3 data.db "SELECT COUNT(*), MIN(date_time), MAX(date_time) FROM data;"
   # 38072745|2025-11-13T12-30-01|2026-01-05T23-55-02
   ```

One row is one request.
Latency is `http_req_waiting` (ms), and `cold_starts = 1` marks a cold invocation.
Multi-architecture providers appear as separate providers (`awsARM`/`awsX86`,`oracleARM`/`oracleX86`).
The paper uses `experiment = 'warmstart'` (12,442,615 rows) and warm calls only, except for Fastly, which flags ~99% of its calls as cold, so all of its calls are used.

The notebooks add an index to `data.db` on first run.

`data/tail-data_1w.db` (the 11-day EC2 baseline), `data/all_configs.json` (benchmark matrix) and `data/oci-vcpus.json` (Oracle vCPU-to-memory mapping) are already in the repository.

### Validation datasets (optional)

Each one is independent.
Put the raw data in the listed folder and run the loader notebook from that folder.

| Key | Source | Raw data | Loader |
|---|---|---|---|
| BEF | Kuhlenkamp et al. 2020, [doi](https://dl.acm.org/doi/10.1145/3341105.3373948) | `full.csv` (~85 MB) from the authors' replication package → `tertiary-repos/kuhlenkamp_et_al/` | `full_duckdb.ipynb` → `full_data.duckdb` |
| SPM | Mahmoudi and Khazaei 2020, [doi](https://ieeexplore.ieee.org/document/9238484) | `git clone https://github.com/pacslab/serverless-performance-modeling.git .` inside `tertiary-repos/mahmoudi_et_al/` | `schema_scan_and_load.ipynb` → `experiments/results/results.duckdb` |
| HUA | Joosen et al. 2025, [doi](https://dl.acm.org/doi/10.1145/3689031.3696073) | 2025 request traces from <https://github.com/sir-lab/data-release> → `tertiary-repos/joosen_et_al/huawei-data/2025-public-requests/region-*/R*_*/part-*.csv` | `huawei_analysis.ipynb`, then `warm_share_of_slowest_huawei.ipynb` |
| NSD | Schirmer et al. 2023, [doi](https://dl.acm.org/doi/10.1145/3592533.3592808) | `data/short/gcf_data.sql` (~300 MB) from <https://github.com/umbrellerde/night-shift-code>, imported into a local MySQL database `benchmark` (below) | none, read live from MySQL |

```sh
mysql -u root -e "CREATE DATABASE benchmark;"
mysql -u root benchmark < gcf_data.sql
```

HUA (944M rows) needs a large-memory machine.
Set the `PRAGMA threads` / `memory_limit` values in its notebooks to match yours.
`warm_share_of_slowest_huawei.ipynb` has a `REGION_FILTER` for a dry run on a single region.

How each dataset maps to platform delay (ms) and warm starts:

| Key | Platform delay | Warm |
|---|---|---|
| BEF | `DLat` (seconds) | `CNew = 'Reused'` (platform-reported, not the derived `cold` column) |
| SPM | `client_elapsed_time * 1000 - elapsed_time` (s and ms) | `is_cold = FALSE` |
| HUA | `(totalCost_frontend - runtimeCost) * 1000` (s) | every request after the first on a `podID` |
| NSD | `google_time - (function_stop - function_start) * 1000` (ms and epoch s) | `isCold = 'False'` (not `type`) |

For HUA, `runtimeCost` is the function's own time in the language runtime.
`workerCost` is not its superset, so subtracting it would leave execution time in the figure.
