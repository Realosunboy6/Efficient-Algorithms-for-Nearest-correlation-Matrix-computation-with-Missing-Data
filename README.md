# THESISCODE.ipynb — Data Preparation Pipeline for Large-Scale Stock-Market Analysis

This repository contains the data-preparation notebook and assets used in the thesis "Efficient Algorithms for Nearest Correlation Matrix Computation with Missing Data." The primary purpose of the code is to load raw per-sector CSV price data, clean it, align it into a rectangular time × asset matrix (preserving missing entries), and produce the inputs (price matrix, masks, metadata) required to run, evaluate and benchmark nearest-correlation-matrix algorithms and related experiments.

Table of contents
- Overview
- What this code is used for (detailed)
- Repository layout
- Expected data format
- Dependencies
- How to run the notebook
- Key functions, variables and outputs produced
- Example workflows (reshape, returns, build mask, nearest-correlation projection)
- Troubleshooting & tips
- License & contact

---

## Overview

The notebook `THESISCODE.ipynb` implements a reproducible pipeline that:
- Loads many per-sector stock CSV files.
- Validates and inspects dataset composition (counts, tickers).
- Cleans and filters dates to the modeling window (example: 2020-01-02 → 2025-09-08).
- Parses date/time strings to DateTime objects.
- Converts long-form price data (date, ticker, close) into a wide time-series matrix (rows = dates, cols = tickers).
- Preserves missing entries and creates the observation mask used by algorithms that operate under missing data.
- Prepares inputs for downstream routines in the thesis (pairwise covariance/correlation estimation, MAP iterations, Anderson acceleration, weighted Frobenius projections, nearest-correlation projections, timing of algorithms).

---

## What this code is used for

Use-cases enabled by the notebook and repo:

1. Data ingestion and cleaning
   - Combine per-sector CSVs into a single canonical DataFrame.
   - Standardize Date, Ticker, Close (and Sector) columns.
   - Remove out-of-range dates and problematic rows.

2. Exploratory validation
   - Count observations and unique tickers per sector.
   - Print lists of all tickers and dataset composition for reproducibility.

3. Matrix construction
   - Create a TSFrame / wide matrix (dates × tickers) of closing prices.
   - Keep NaN / missing entries where price data do not exist (delistings, IPOs, holidays).

4. Missing-data bookkeeping
   - Construct boolean masks indicating observed vs missing entries to feed to nearest-correlation-matrix solvers and to run controlled experiments.

5. Downstream numerical experiments
   - Compute returns and pairwise covariances/correlations while preserving missing data semantics.
   - Evaluate and benchmark nearest-correlation-matrix algorithms (MAP, Anderson acceleration, weighted Frobenius projection) using the prepared matrices.
   - Time experiments using TimerOutputs to collect performance data.

6. Reproducibility
   - Provide a single preprocessing pipeline so subsequent algorithm runs and figures in the thesis can be reproduced from the same cleaned dataset.

---

## Repository layout

- `THESISCODE.ipynb` — The main Julia notebook that performs the data ingestion and preprocessing and demonstrates dataset checks and reshaping into a TSFrame/wide matrix.
- `Presentation.pdf` — Presentation slides related to the thesis.
- `thesis asset/` — (Not committed here) expected location for per-sector CSV files used by the notebook.
- `README.md` — This document.

---

## Expected input CSV format

Each sector CSV should be UTF-8 encoded with at least the following columns (examples shown as column names used in the notebook):
- `Date` — date/time string. Example formats used by the notebook: `"yyyy-mm-dd HH:MM:SS-HH:MM"`. Timezone offsets may be present.
- `Ticker` — ticker symbol (string).
- `Close` — daily closing price (float).
- `Sector` — sector name (string) — used for grouping/metadata.

Files are typically named like:
- `Consumer_Discretionary.csv`
- `Information_Technology.csv`
- `Utilities.csv`
- etc.

The notebook's `load_stocks(data_dir, stock_files)` function iterates this list and concatenates available CSVs. Missing files are skipped with a message.

---

## Dependencies

The notebook is written for Julia and uses the following packages (imported in the first cell):

- CSV
- DataFrames
- Dates
- TSFrames
- Statistics
- PortfolioAnalytics
- LinearAlgebra
- TimerOutputs
- NearestCorrelationMatrix
- NaNStatistics
- LinearAlgebra.LAPACK

Install packages in Julia with:
```julia
using Pkg
Pkg.add.(["CSV","DataFrames","Dates","TSFrames","Statistics","PortfolioAnalytics",
          "LinearAlgebra","TimerOutputs","NearestCorrelationMatrix","NaNStatistics"])
```

Note: package versions and environments are not pinned in this repo. For reproducible experiments, instantiate a project environment and pin versions as needed.

---

## How to run

1. Place your per-sector CSV files in a local folder, e.g., `thesis asset/`.
2. Update the `data_dir` and `stock_files` list inside the notebook or script to point to your files.
3. Open `THESISCODE.ipynb` in Jupyter (IJulia) and run the cells sequentially, or convert the notebook to a script:
   ```bash
   jupyter nbconvert --to script THESISCODE.ipynb
   ```
4. Run the converted script in a Julia REPL or execute the notebook in JupyterLab.

---

## Key functions, variables and outputs

- Function:
  - `load_stocks(data_dir::String, stock_files::Vector{String})`  
    - Reads listed CSVs, selects `:Date, :Ticker, :Close, :Sector`, concatenates into `combined_df`.
- Important variables produced:
  - `combined_df` — long-format DataFrame with all loaded observations.
  - `unique_tickers` / `tickers` — list of tickers (columns for the wide matrix).
  - `filtered_df` / `daily_df` — date-filtered and parsed long DataFrame (DateTime parsed).
  - `tsframe` or `wide_df` — TSFrame or wide DataFrame with rows = dates and columns = tickers and price values (Float64? to allow missing).
  - `price_matrix` — numeric matrix (dates × tickers) with missing values (NaN or missing) as appropriate.
  - `mask` — boolean matrix (same dims as price_matrix) where true indicates observed data; used as the observation pattern for algorithms.
  - `to` — TimerOutput object collecting timing information for experiments.

---

## Example workflows

1) Reshape long DataFrame → wide TSFrame / Matrix (illustrative Julia code)
```julia
# assume filtered_df has columns: Date::DateTime, Ticker::String, Close::Float64
using TSFrames, DataFrames

# pivot to wide: rows are sorted unique dates, columns are tickers
dates = sort(unique(filtered_df.Date))
tickers = sort(unique(filtered_df.Ticker))

# Create empty Matrix with missing
price_matrix = Matrix{Union{Missing, Float64}}(missing, length(dates), length(tickers))

# Build index maps
date_to_i = Dict(d => i for (i,d) in enumerate(dates))
ticker_to_j = Dict(t => j for (j,t) in enumerate(tickers))

for row in eachrow(filtered_df)
    i = date_to_i[row.Date]; j = ticker_to_j[row.Ticker]
    price_matrix[i,j] = row.Close
end

# Optionally convert to a TSFrame if you prefer:
using TSFrames
ts = TSFrame(price_matrix, index=dates, columns=tickers)
```

2) Build observation mask
```julia
mask = .!ismissing.(price_matrix)  # true where observed
```

3) Compute returns preserving missing entries (simple forward difference)
```julia
# log returns example
using Statistics
returns = similar(price_matrix, Union{Missing, Float64})
for j in 1:size(price_matrix,2)
    col = price_matrix[:,j]
    for i in 2:size(price_matrix,1)
        if !ismissing(col[i]) && !ismissing(col[i-1])
            returns[i,j] = log(col[i]) - log(col[i-1])
        else
            returns[i,j] = missing
        end
    end
end
```

4) Pairwise sample correlation (pairwise deletion)
- Compute pairwise covariances/correlations between columns using observed pairs only. (Use NaNStatistics or manually compute using masks.)

5) Projection to nearest valid correlation matrix
- The `NearestCorrelationMatrix` package referenced at the top of the notebook is intended for projecting an estimated (possibly indefinite) matrix to the nearest valid correlation matrix under an appropriate norm (e.g., Frobenius). Once you form a sample correlation matrix (with your chosen treatment for missing data), feed it to the package's projection routines to ensure positive semidefiniteness and diagonal = 1 constraints:
```julia
using NearestCorrelationMatrix

# Example (pseudocode)
R = pairwise_correlation_matrix  # computed from returns/covariances
R_projected = nearest_correlation_matrix(R)  # call into the package - check API
```

Refer to the NearestCorrelationMatrix package docs for exact function names and options (e.g., weighting, tolerances, maximum iterations).

---

## Troubleshooting & tips

- Date parsing problems:
  - If `DateTime.(...)` fails, inspect the exact format in the CSVs. Timezone offsets (e.g., `-05:00`) must be accounted for in the parsing pattern.
- CSV encoding:
  - CSV files must be UTF-8 encoded. Non-UTF8 content may cause `CSV.read` to error.
- Memory:
  - Building a large dense matrix for many tickers and many dates may use a lot of RAM. Consider working with sparse representations or chunking when feasible.
- Missing files:
  - The loader prints "File not found: ..." and continues. Ensure the filenames listed in `stock_files` match your local file names.
- Reproducibility:
  - Use a Julia project environment and `REPL` activation (`Pkg.activate`) to reproduce exact package versions and experiment runs.
- Speed:
  - Preallocate matrices and avoid growing arrays in hot loops for better performance. Use `TimerOutputs` (already included) to profile timing.

---

## Reproducing experiments

1. Prepare per-sector CSV files in the `thesis asset` folder (or any folder but update `data_dir` accordingly).
2. Run the notebook from top to bottom, making sure packages are installed and `data_dir` path is correct.
3. After producing the wide price matrix and masks, compute returns and the sample correlation/covariance matrix according to the experimental design you wish to reproduce.
4. Use the masking information and the NearestCorrelationMatrix routines to run algorithmic experiments and collect timing via the `to` TimerOutput.

---

## License & contact

- Author / Owner: Realosunboy6
- For licensing, cite the thesis or contact the repository owner if you need explicit permission to reuse data or code for non-academic projects.
- Contact / Questions: (use your GitHub profile) https://github.com/Realosunboy6

---
