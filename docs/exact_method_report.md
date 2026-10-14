# Exact Method – Report Section (draft)

*Material for the PDF report, organised by marking criterion. All numbers come from `results/exact/*.csv` and the executed notebooks; figures are in `results/figures/`. Rewrite in your own words before submission (Turnitin).*

---

## 1. Problem selection (Option C – Smart Investor)
An investor has **$10,000** and may buy shares of at most **5** S&P 500 companies (they cannot track more). They need an **expected annual return of at least 20%** and want the **lowest possible risk**.

This is the **cardinality-constrained portfolio selection problem**. It is realistic: retail investors and fund managers face limits on the number of holdings, minimum lot sizes, and whole-share purchases. It is also non-trivial: the cardinality constraint makes it NP-hard (Bienstock, 1996; Moral-Escudero et al., 2006). With 457 candidate stocks there are about **1.64 × 10¹¹** possible subsets of ≤ 5 stocks, and each subset still requires its weights to be optimised.

## 2. Problem and data description
* **Dataset:** Kaggle *S&P 500 stock data* (`camnugent/sandp500`, CC0): daily OHLCV for 505 companies, 2013-02-08 → 2018-02-07, 619,040 rows.
* **Role of the data:** the data **instantiates** the model; nothing is trained. Weekly returns $r_{ti}$, mean returns $\mu_i$, and decision-date prices $p_i$.
* **Cleaning:**
  * 35 tickers removed for incomplete history (listed / joined after Feb 2013).
  * 13 tickers removed because their prices are **not adjusted for splits / spin-offs**, or contain bad ticks: EBAY −57% (PayPal spin-off), DISCA/DISCK −51% (2:1 split), NI −63%, LNT −49% then +101% the next day, and others.
  * **457 stocks** remain.
* **Estimation window:** 203 weekly returns, 2013-02 → 2016-12. **Out-of-sample window:** 2016-12-30 → 2018-02-07 (278 trading days).
* **Data observations that shape the model:**
  * 96% of stocks have fat-tailed weekly returns (median excess kurtosis 1.66), which favours a robust, linear risk measure (MAD).
  * Mean pairwise correlation is 0.29; the most correlated stocks are utilities (CMS–DTE 0.90).
  * Only 91 of 457 stocks reach 20% on their own, so the target forces combinations.

**Constraints:**
* at most K = 5 stocks;
* each held stock between 5% and 40% of the budget;
* fully invested;
* expected return ≥ 20% per year;
* whole-share variant: integer share counts, at most 1% ($100) left as cash.

## 3. Mathematical formulation
Indices: stocks $i = 1..N$ ($N = 457$), weeks $t = 1..T$ ($T = 203$). Parameters: $a_{ti} = r_{ti} - \mu_i$, $\rho = 0.20/52$, $K = 5$, $l = 0.05$, $u = 0.40$.
Variables: $w_i \in [0, u]$ (weight), $z_i \in \{0,1\}$ (stock bought), $d_t \ge 0$ (absolute deviation in week $t$).

$$
\begin{aligned}
\min\ & \tfrac{1}{T}\textstyle\sum_t d_t &&\text{(MAD risk)}\\
\text{s.t. } & d_t \ge \textstyle\sum_i a_{ti} w_i,\quad d_t \ge -\textstyle\sum_i a_{ti} w_i && \forall t\\
& \textstyle\sum_i w_i = 1, \qquad \textstyle\sum_i \mu_i w_i \ge \rho, \qquad \textstyle\sum_i z_i \le K\\
& l\,z_i \le w_i \le u\,z_i && \forall i\\
& z_i \in \{0,1\}
\end{aligned}
$$

**Whole-share variant:** add $x_i \in \mathbb{Z}_{\ge 0}$ with $B w_i = p_i x_i$, and replace full investment by $1 - 0.01 \le \sum_i w_i \le 1$.

**Model size (full instance):** 457 binary + 660 continuous variables, 1,323 constraints, ~186k non-zeros. The whole-share variant adds 457 integer variables.

*Design notes for the viva:*
* The pair of constraints on $d_t$ linearises $|\cdot|$: since $d_t$ is minimised, $d_t = |{\textstyle\sum_i} a_{ti}w_i|$ at the optimum.
* $l z_i \le w_i \le u z_i$ is a Big-M linking constraint (M = u).
* The 1% cash cap was added after the first whole-share run returned a portfolio that **kept 52% in cash**. Cash has zero risk, so without the cap the model "parked" money and bought volatile high-return stocks to meet the target on paper. This is a good example of a model doing exactly what it is told rather than what was meant.

## 4. Optimisation method and justification
* **Why MAD instead of Markowitz variance:** variance makes the objective quadratic (MIQP). MAD (Konno & Yamazaki, 1991) gives the same kind of risk–return trade-off with **linear** constraints. The model therefore stays an MILP that PuLP can express and open-source solvers can solve exactly, and MAD is more robust to the fat tails observed in the data.
* **Algorithm:** LP-based **Branch & Bound** with cutting planes (HiGHS 1.15 primary, CBC as cross-check) via PuLP 3.3.
  * The LP relaxation gives a lower bound; branching on fractional $z_i$ splits the search; nodes are pruned by bound, infeasibility or integrality.
  * Optimality is proven when the gap reaches 0. Under a time limit the solver still returns the best portfolio found together with a **certified gap**.
* **Why the problem is hard for B&B (key insight):** in the LP relaxation, $z_i = w_i/u$ satisfies the linking constraints, and then $\sum_i z_i = 1/u = 2.5 \le K$. The cardinality constraint therefore **never binds at the root**: the root bound equals the unconstrained MAD optimum. This was confirmed numerically: the unconstrained LP at a 20% target gives 0.008630, identical to the MILP root bound. All the work of enforcing "≤ 5 stocks" falls on branching.

## 5. Implementation (GitHub)
Two self-contained Jupyter notebooks, each code cell preceded by an explanation:
* `01_data_exploration.ipynb` – download, cleaning, exploratory analysis; exports `data/processed/` (the identical instance used by the heuristic).
* `02_exact_method.ipynb`, in order:
  * parameters and the `Instance` class;
  * `build_model` – the formulation line by line;
  * `solve` – reports status, bound, gap and nodes, and works around PuLP reporting "Optimal" on time-outs;
  * the shared evaluator `evaluate_weights`;
  * a worked LP-relaxation example;
  * **verification checks** (assert cells: brute force, feasibility, bounds, whole shares, second solver);
  * all experiments (resumable; `RUN_EXPERIMENTS = False` loads saved results);
  * the out-of-sample backtest and the discussion.

Hardware: Intel i7-9750H, 16 GB RAM; HiGHS uses a single worker.

## 6. Results
**Correctness.** On all **18** random instances with N = 8–20, the MILP optimum equals the brute-force enumeration optimum, with the same stocks; the maximum difference is 3.9 × 10⁻¹⁰. At N = 20 brute force needs 21,699 LPs and 127 s on average; the MILP needs 2.0 s.

**Full problem (N = 457, 30-minute limit):**

| Variant | Best MAD | Lower bound | Certified gap | Nodes |
|---|---|---|---|---|
| continuous weights | 0.010071 | 0.009207 | 8.6% | 28,016 |
| whole shares (≤1% cash) | 0.010052 | 0.009276 | 7.7% | 61,184 |

Both variants choose the same five stocks from five different sectors:

| Stock | Weight | Shares @ 2016-12-30 price |
|---|---|---|
| CLX (Clorox) | 27.7% | 23 @ $120.02 |
| RE (Everest Re) | 26.1% | 11 @ $216.40 |
| AZO (AutoZone) | 22.6% | 3 @ $789.79 |
| FB (Facebook) | 13.3% | 12 @ $115.05 |
| HUM (Humana) | 10.4% | 5 @ $204.03 |

Whole-share plan: **$9,910.98 spent, $89.02 cash**. Expected return 20.0% per year, in-sample volatility 9.9% per year.

The incumbent was found within ~41 s. The remaining time only raised the lower bound (root 0.00863 → 0.00921), so the time limit costs **proof**, not solution quality.

**Scalability (3 random subsets per N, 300 s limit):**

| N | 10 | 20 | 30 | 40 | 50 | 75 | 100 | 150 | 200 | 300 | 457 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| mean time (s) | 0.3 | 1.2 | 3.2 | 10.4 | 16.2 | 66.7 | 73.6 | 300* | 300* | 300* | 300* |
| proven optimal | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 | 0/3 | 0/3 | 0/3 | 0/1 |
| mean final gap | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 4.6% | 7.3% | 9.2% | 12.6% |
| root LP gap | 0.8% | 2.0% | 5.6% | 5.6% | 5.7% | 8.2% | 9.1% | 10.3% | 11.7% | 12.3% | 14.3% |

\* stopped at the time limit.

**Solver comparison (N ≤ 50, 15 instances):**
* CBC is 1.2–2.9× slower than HiGHS.
* On 2 instances CBC reported "optimal" but returned a portfolio that is 0.01% / 0.08% worse than HiGHS's, with a lower bound *above* HiGHS's verified solution. The HiGHS portfolios were independently verified (evaluator, fixed-subset LP, brute force).
* Changing CBC's presolve and cut settings changes whether this happens, which suggests numerical tolerances.

**Price of the 5-stock rule (efficient frontier):**
* Unconstrained MAD portfolios hold 11–39 stocks.
* Limiting to 5 raises the risk by up to 20% at a 10% return target, 17% at 20%, and 4% at 45% (upper estimates; most K = 5 points stopped at 300 s).
* High return targets are *easier* to solve: 40% and 45% are proven optimal in 103 s and 32 s, because few stocks qualify and the return constraint prunes most of the tree.

**Sensitivity to K (300 s each):** best MAD falls from 0.01127 (K = 3) to 0.01007 (K = 5), 0.00918 (K = 10) and 0.00896 (K = 15). Holding more stocks gives diminishing benefit.

**Out-of-sample (buy and hold, 2016-12-30 → 2018-02-07):**

| Portfolio | Final value | Return | Volatility | Max drawdown | Sharpe (rf = 0) |
|---|---|---|---|---|---|
| MILP, continuous | $11,429 | +14.3% | 10.5% | −6.9% | 1.23 |
| MILP, whole shares | $11,414 | +14.1% | 10.4% | −6.9% | 1.23 |
| Equal-weight 457 stocks | $11,781 | +17.8% | 8.4% | −7.6% | 1.91 |

## 7. Discussion and future work
1. **Exactness has a price that grows with N.** Proofs take seconds up to N ≈ 100 and are out of reach within minutes beyond N ≈ 150. Even then the method gives a usable portfolio *and a guarantee* (≤ 8.6% from optimal). This guarantee is what lets us measure the heuristic's true quality.
2. **The bottleneck is the weak LP relaxation**, not the LP size. Future work:
   * stronger formulations, e.g. valid inequalities that tie $\sum z_i$ to the weight bounds;
   * warm-starting the MILP with the heuristic's solution (a hybrid method);
   * symmetry and dominance-based pre-selection of stocks;
   * parallel B&B, or a commercial solver.
3. **Verification matters.** A widely used open-source solver (CBC) reported false optimality on 2 of 15 instances. Brute-force checks and a second solver caught it.
4. **Optimal ≠ best decision.** Out of sample, the portfolio earned +14.3%, below its 20% estimate and below the 1/N benchmark, while being slightly more resilient in the drawdown. The cause is estimation error in $\mu_i$ from 4 years of history, plus the diversification lost by holding only 5 stocks. Future work:
   * robust or shrinkage estimates of $\mu$;
   * rolling-window re-optimisation with transaction costs;
   * CVaR as an alternative linear risk measure.

## References
* Konno, H. & Yamazaki, H. (1991). Mean-absolute deviation portfolio optimization model and its applications to Tokyo stock market. *Management Science*, 37(5), 519–531.
* Bienstock, D. (1996). Computational study of a family of mixed-integer quadratic programming problems. *Mathematical Programming*, 74, 121–140.
* Moral-Escudero, R., Ruiz-Torrubiano, R. & Suárez, A. (2006). Selection of optimal investment portfolios with cardinality constraints. *IEEE Congress on Evolutionary Computation*, 2382–2388.
* Land, A. H. & Doig, A. G. (1960). An automatic method of solving discrete programming problems. *Econometrica*, 28(3), 497–520.
* Huangfu, Q. & Hall, J. A. J. (2018). Parallelizing the dual revised simplex method. *Mathematical Programming Computation*, 10, 119–142. (HiGHS)
