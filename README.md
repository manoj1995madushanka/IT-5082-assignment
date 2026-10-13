# IT5082 Optimization Methods – Cardinality-Constrained Portfolio Optimisation

**The Smart Investor.** Invest **$10,000** in **at most 5** of the S&P 500 companies so that the portfolio reaches a **20% expected annual return with minimum risk**.

The problem is solved with two approaches and compared:

| Branch | Method | Status              |
|---|---|---------------------|
| `exact-method` | **Exact:** cardinality-constrained Mean-Absolute-Deviation model as a Mixed-Integer Linear Program (PuLP + HiGHS/CBC Branch & Bound) | exact-method branch |
| *(heuristic branch)* | Metaheuristic | partner             |

## Dataset
[S&P 500 stock data](https://www.kaggle.com/datasets/camnugent/sandp500) (Kaggle `camnugent/sandp500`, CC0). Daily prices of 505 companies, 2013-02-08 → 2018-02-07. The raw file is committed in `data/raw/`; if it is missing, notebook 01 downloads it through `kagglehub` (no API token needed).

Cleaning: 35 tickers with incomplete history and 13 tickers with **unadjusted splits/spin-offs** (e.g. EBAY −57% on the PayPal spin-off) are removed → **457 stocks**. Weekly returns 2013–2016 (203 weeks) instantiate the model; 2017-01 → 2018-02 is the out-of-sample test.

## Mathematical model (exact method)

$$
\begin{aligned}
\min\ & \tfrac{1}{T}\textstyle\sum_t d_t \\
\text{s.t. } & d_t \ge \pm \textstyle\sum_i (r_{ti}-\mu_i)\, w_i \quad \forall t \\
& \textstyle\sum_i w_i = 1, \quad \sum_i \mu_i w_i \ge \rho, \quad \sum_i z_i \le K \\
& l\,z_i \le w_i \le u\,z_i, \quad z_i \in \{0,1\}
\end{aligned}
$$

with $K=5$, $l=5\%$, $u=40\%$, $\rho = 20\%/52$. A whole-share variant adds integer share counts $x_i$ with $B w_i = p_i x_i$ and at most 1% cash.

## Repository layout
```
data/raw/                        Kaggle CSV
data/processed/                  cleaned weekly returns + daily prices (written by notebook 01)
notebooks/01_data_exploration.ipynb   data loading, cleaning and exploration
notebooks/02_exact_method.ipynb       formulation, MILP code, verification, experiments, backtest, discussion
results/exact/                   saved experiment results (CSV / JSON)
results/figures/                 figures for the report
docs/exact_method_report.md      report draft for the exact-method sections
code_appendix.txt                plain-text code appendix (generated from the notebooks)
```

## How to run
```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
jupyter notebook                 # run notebooks/01 then notebooks/02 top to bottom
```
Notebook 02 has a switch in its parameter cell:
* `RUN_EXPERIMENTS = False` (default): loads the saved results in `results/exact/`; the whole notebook runs in about a minute, including all verification checks.
* `RUN_EXPERIMENTS = True`: recomputes every experiment (~2.5 h; the full-size solves run for 30 min each). Rows are saved as they finish, so an interrupted run resumes where it stopped.

Regenerate the plain-text code appendix:
```bash
jupyter nbconvert --to script --stdout notebooks/01_data_exploration.ipynb notebooks/02_exact_method.ipynb > code_appendix.txt
```

## Comparing with the heuristic
For a fair comparison, the heuristic notebook should:
1. read the **same instance**: `data/processed/weekly_returns_train.csv` and `daily_close_clean.csv`;
2. copy the parameter cell, the `Instance` class / `random_subinstance` and **`evaluate_weights`** from notebook 02 unchanged;
3. minimise `evaluate_weights(...)["mad"]`; a portfolio counts only if `["feasible"]` is `True`.

Exact reference values:
* `results/exact/exact_solution.json` – the full problem;
* `results/exact/scalability.csv` – the same `n`/`seed` subsets, with certified lower bounds, so the heuristic's true optimality gap can be computed.
