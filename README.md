# S&P 500 stock segmentation: risk/return vs adapted RFM

Customer Relationship Management coursework (CW1). The project segments S&P 500 shares in two independent ways and compares them:

- **Risk/return**: mean weekly return and weekly beta against ^GSPC.
- **Adapted RFM** (trading activity): Recency of the last high-volume day, Frequency of high-volume days, and Monetary (approximate traded value).

Each feature matrix is clustered with **Ward agglomerative clustering** and **K-means**, giving four solutions. These are profiled, compared, stress-tested and stored in a small **star-schema data mart** (SQLite).

> The clusters are **descriptive**. They summarise 2021–2024 behaviour and are not a forecast of future returns or a claim of reduced portfolio risk (see the out-of-sample test below).

## Repository layout

```text
.
├── README.md
├── requirements.txt
├── 20230796_CRM_CW_v3.ipynb      # final notebook, run with outputs saved
├── data/                         # raw snapshots used by the run (SHA-256 hashed)
├── outputs/                      # every table (.csv), figure (.png) and the data mart
├── report/                       # submitted report and the star-schema diagram
└── docs/
```

Keep the notebook in the repository root: it reads and writes `data/` and `outputs/` as relative paths.

## Data

| Item | Value |
|---|---|
| Ticker source | Wikipedia, [List of S&P 500 companies](https://en.wikipedia.org/wiki/List_of_S%26P_500_companies), retrieved 24 Nov 2026 |
| Price source | yfinance 0.2.66, `auto_adjust=False` (Close, Adj Close, Volume, dividends, splits) |
| Benchmark | ^GSPC (price index) |
| Study period | 1 Jan 2021 to 1 Jan 2025 (end exclusive), 1,005 trading sessions |
| Shares | 503 listed, 486 retained after validation (17 excluded, logged in `outputs/exclusion_log.csv`) |
| Weekly returns | Friday-ending weeks, complete weeks only, 207 aligned weeks |
| RFM | Observation date 31 Dec 2024; event = volume > 2 × mean of the preceding 20 sessions; F and M over the latest 60 sessions |

`data/` holds the raw snapshots (`sp500_tickers.csv`, `adj_close.csv`, `close.csv`, `volume.csv`, `dividends.csv`, `splits.csv`, `gspc_adj_close.csv`, `gspc_close.csv`). Their SHA-256 hashes are recorded in `outputs/retrieval_metadata.json`.

## How to run

1. Open the notebook in Google Colab (or Jupyter) with this repository as the working directory.
2. Install the dependencies: `pip install -r requirements.txt`.
3. Choose the data source in the first Task 1 cell:
   - `USE_CACHE = True` reloads the snapshots in `data/` and reproduces the reported results exactly. The provenance cell checks every file against its recorded hash.
   - `USE_CACHE = False` downloads fresh data from Wikipedia and Yahoo Finance. The constituent list and prices may have changed, so results can differ.
4. Run all cells in order (in Colab: *Runtime → Restart and run all*). Running cells out of order can make the hash check fail.

The last cell zips `data/` and `outputs/`. In Colab it downloads the archive; elsewhere it leaves the zip in the working directory.

## Method summary

- **Preparation:** log1p applied to Monetary; each matrix standardised with its own scaler; outliers kept and their influence reported.
- **Choosing k:** the highest silhouette for k = 3–8 whose smallest cluster holds at least 5% of shares, checked against the profiles. The same rule is used for both algorithms.
- **K-means:** k-means++, `n_init=50`, `random_state=42`.
- **Comparison:** ARI, AMI and Hungarian-matched agreement between algorithms on the same 486 shares.
- **Sensitivity:** correlations among R, F and log1p(M), drop-one feature influence, whitened-PCA check, and a Ward refit without Monetary.

## Key results (run `v3.0_20241231`)

| Solution | Selected k | Notes |
|---|---|---|
| Ward, risk/return | 5 | From a defensive low-beta segment to a high-beta, IT-heavy segment |
| K-means, risk/return | 5 | Splits more along return; ARI vs Ward 0.32 |
| Ward, RFM | 3 | Recent lower-value, high-value and stale segments |
| K-means, RFM | 4 | Splits the recent segment into frequent and infrequent; ARI vs Ward 0.32 |

- The risk/return and RFM segmentations are almost independent (ARI 0.04 for Ward, 0.02 for K-means). Sector association is stronger for risk/return (Cramér's V 0.43) than for RFM (0.26).
- Removing Monetary changes Ward RFM memberships substantially (ARI 0.29). Traded value stops differing between clusters (eta² 0.52 → 0.004), so M mainly separates liquidity levels.
- **Out-of-sample test:** clusters rebuilt on 2021–22 kept their beta ordering in 2023–24 (rank correlation 1.0), but return differences reversed (−0.5 for Ward, −0.8 for K-means).
- **Survivorship bias:** the current constituent list is applied to past data. The 72 later additions averaged 0.47% a week vs 0.25% for the rest.

All headline numbers are in `outputs/report_key_numbers.csv`.

## Main outputs

| File | Content |
|---|---|
| `outputs/feature_table_with_clusters.csv` | Five features and the four cluster labels for each share |
| `outputs/profile_named_*.csv` | Cluster profiles in original units |
| `outputs/selected_k.csv`, `k_diagnostics_*.png` | k selection evidence |
| `outputs/dendrogram_*_selected_k.png` | Ward dendrograms with the selected cut |
| `outputs/ward_vs_kmeans_crosstab_heatmaps.png` | Algorithm comparison |
| `outputs/rr_vs_rfm_named_heatmap.png` | Risk/return vs RFM segments |
| `outputs/monetary_removal_*.csv/.png` | Monetary removal sensitivity |
| `outputs/out_of_sample_test.csv` | Descriptive vs predictive check |
| `outputs/crm_datamart.sqlite` | Star-schema data mart (5 fact tables, 5 dimension tables) |
| `outputs/run_metadata.csv`, `retrieval_metadata.json` | Provenance and calculation record |

## Data mart

The base grain is one share on one trading day (`fact_daily_market`). Derived facts are kept at weekly, per-run and per-solution grain. Dimensions cover date, share, sector, benchmark and calculation run. Every derived row carries a `run_id`, so results from different windows or versions can coexist. See `report/datamart_star_schema.png` for the diagram.

## Limitations

- **Survivorship bias:** the constituent list is current, so shares removed from the index since 2021 are missing.
- **Price basis:** Adj Close includes dividends but ^GSPC is a price index (the effect on beta is negligible).
- **Liquidity proxy:** traded value is only a proxy; spreads and price impact are not measured.
- **Overlapping features:** R and F partly overlap.
- **Descriptive only:** the clusters describe the past; they do not predict.

## Author

Ranuli Alahakoon (20230796): individual coursework for the CRM module.
