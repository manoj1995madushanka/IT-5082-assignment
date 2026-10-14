# IT5082 Optimization Methods: Cardinality-Constrained Portfolio Optimisation

**The Smart Investor.** Invest **$10,000** in **at most 5** of the S&P 500 companies so that the portfolio reaches a **20% expected annual return with minimum risk**.

The problem is solved with an exact method and a metaheuristic. The two are then compared on solution quality, runtime, scalability and feasibility.

| Method | Approach | Notebook |
|---|---|---|
| **Exact** | Cardinality-constrained Mean-Absolute-Deviation (CC-MAD) model as a Mixed-Integer Linear Program. It is solved by Branch & Bound with PuLP and HiGHS, and CBC is used as a cross-check. | `02_exact_method.ipynb` |
| **Heuristic** | Simulated Annealing on an equal-weight portfolio with a 15% volatility cap. | `portfolio_selection_simulated_annealing.ipynb` |
| **Comparison** | The same Simulated Annealing design applied to the CC-MAD problem. Weights come from a small LP, so both methods solve the identical problem. | `03_comparison.ipynb` |

## Dataset
[S&P 500 stock data](https://www.kaggle.com/datasets/camnugent/sandp500) (Kaggle `camnugent/sandp500`, CC0). It holds daily prices of 505 companies from 2013-02-08 to 2018-02-07. The raw file is committed in `data/raw/`. If it is missing, notebook 01 downloads it through `kagglehub` and no API token is needed.

Cleaning removes 35 tickers with an incomplete history. It also removes 13 tickers whose prices contain **unadjusted splits or spin-offs** (for example EBAY falls 57% on the PayPal spin-off). This leaves **457 stocks**. Weekly returns from 2013 to 2016 (203 weeks) instantiate the model. The period from January 2017 to February 2018 is the out-of-sample test.

## Mathematical model

$$
\begin{aligned}
\min\ & \tfrac{1}{T}\textstyle\sum_t d_t \\
\text{s.t. } & d_t \ge \pm \textstyle\sum_i (r_{ti}-\mu_i)\, w_i \quad \forall t \\
& \textstyle\sum_i w_i = 1, \quad \sum_i \mu_i w_i \ge \rho, \quad \sum_i z_i \le K \\
& l\,z_i \le w_i \le u\,z_i, \quad z_i \in \{0,1\}
\end{aligned}
$$

The parameters are $K=5$, $l=5\%$, $u=40\%$ and $\rho = 20\%/52$. A whole-share variant adds integer share counts $x_i$ with $B w_i = p_i x_i$ and allows at most 1% cash.

## Key results

| | Exact (MILP) | Heuristic (SA + LP) |
|---|---|---|
| Full problem (457 stocks) | MAD 0.010071 with a certified gap of 8.6% after 30 minutes | Found the same portfolio in 8 of 10 runs, at about 57 s per run |
| Instances with up to 100 stocks | Proven optimal on every instance | Matched the proven optimum in 61 of 63 runs |
| Runtime as the number of stocks grows | Grows exponentially. From 150 stocks the 300 s limit is reached. | Grows roughly linearly, from 1.4 s to 78 s per run |
| Feasibility | Guaranteed by the constraints | 103 of 103 runs feasible |

The best portfolio is **CLX, RE, AZO, FB and HUM**. As a whole-share plan it buys 23, 11, 3, 12 and 5 shares for $9,910.98 and leaves $89.02 in cash.

## Repository layout
```
data/raw/                     Kaggle CSV
data/processed/               cleaned weekly returns and daily prices (written by notebook 01)
notebooks/
  01_data_exploration.ipynb                     data loading, cleaning and exploration
  02_exact_method.ipynb                         MILP formulation, verification, experiments, backtest
  portfolio_selection_simulated_annealing.ipynb Simulated Annealing on the equal-weight model
  03_comparison.ipynb                           like-for-like comparison of the two methods
results/exact/                saved MILP results (CSV and JSON)
results/comparison/           saved comparison results (CSV)
results/figures/              figures used in the report
docs/                         group report (DOCX and PDF) and earlier drafts
Submission/                   members.txt, submission.txt and code_appendix.txt
```

## How to run
```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
jupyter notebook
```
Run notebook 01 first, because it writes `data/processed/`. The other notebooks can then be run in any order.

Notebooks 02 and 03 have a switch in their parameter cell.
* `RUN_EXPERIMENTS = False` is the default. The notebook loads the saved results and runs in a few minutes, including all verification checks.
* `RUN_EXPERIMENTS = True` recomputes every experiment. This takes about 2.5 hours for notebook 02 and 1.5 hours for notebook 03. Each result row is saved as soon as it finishes, so an interrupted run resumes where it stopped.

## Code appendix
The plain-text code appendix for the submission is `Submission/code_appendix.txt`. Regenerate it after any notebook change with this command.
```bash
{
  for nb in 01_data_exploration 02_exact_method portfolio_selection_simulated_annealing 03_comparison; do
    echo "# notebooks/$nb.ipynb"
    jupyter nbconvert --to script --stdout "notebooks/$nb.ipynb" 2>/dev/null
  done
} > Submission/code_appendix.txt
```
