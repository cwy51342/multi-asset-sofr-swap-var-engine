# Multi-Asset Portfolio Value-at-Risk (VaR) Engine: SOFR Interest Rate Swaps & Equities

A quantitative market risk engine that prices a multi-asset portfolio combining **US Equities** and a **10-Year Payer SOFR Interest Rate Swap**, computing **1-Day 95% Value-at-Risk (VaR)** across **Parametric (Variance-Covariance)**, **Monte Carlo Simulation (Cholesky Decomposition)**, and **Historical Simulation** frameworks[cite: 19]. Both **Full Revaluation** (curve re-bootstrapping) and **First-Order Sensitivity (Key-Rate PV01)** PnL approximations are benchmarked side-by-side[cite: 19].

---

## Portfolio Composition

The portfolio is evaluated as of the valuation date $t_0 = \text{2023-10-30}$ across **34 risk factors** (4 equity price series + 30 SOFR zero-rate tenors spanning 1D to 40Y)[cite: 19]:

* **Equity Book ($4,000,000 Total Notional):** $1,000,000 notional allocated equally to each of 4 US equities: **Apple (`AAPL`)**, **Microsoft (`MSFT`)**, **Ford (`F`)**, and **Bank of America (`BAC`)**[cite: 19].
* **Interest Rate Derivatives Book ($100,000,000 Notional):** 10-Year Payer SOFR Swap (`isPay = True`), fixed strike $K = 4.20\%$, annual payment frequency ($\text{freq} = 1$)[cite: 19].

---

## Quantitative Methodology

### 1. Zero-Rate Curve Construction & Swap Pricing (`ZeroRateCurve` & `SofrSwap`)
* **Discount Factor Interpolation:**Piecewise linear interpolation (`scipy.interpolate.interp1d`) over continuous zero rates $r(t)$ across 30 tenors (1D, 1M–9M, 1Y–20Y, 25Y, 30Y, 35Y, 40Y) to compute discount factors[cite: 19]:
  $$DF(t) = e^{-r(t) \cdot t}, \quad t = \frac{d - t_0}{365}$$
* **SOFR Swap Valuation:** Projects implied forward rates $f_i = \frac{1}{\alpha_i}\left(\frac{DF_{i-1}}{DF_i} - 1\right)$ over each accrual period $\alpha_i$ and values the swap as[cite: 19]:
  $$\text{PV}_{\text{swap}} = N \cdot \delta_{\text{pay}} \sum_{i=1}^{M} DF_i \cdot \alpha_i \cdot (f_i - K)$$
* **Key-Rate PV01 Vector (`pv01`):** Perturbs each of the 30 zero-rate curve nodes individually by $\Delta r = +1\text{ bp}$ ($0.0001$) to extract the partial DV01/PV01 sensitivity vector $\frac{\partial \text{PV}}{\partial r_k}$[cite: 19].

### 2. Full Revaluation vs. Sensitivity-Based PnL
* **Full Revaluation (`pnl_full`):** Applies shocked zero-rate changes $\Delta \mathbf{r}$ to the base curve, reconstructs a new `ZeroRateCurve` object per scenario, and re-prices the swap exactly to capture non-linear convexity[cite: 19].
* **Sensitivity Approximation (`pnl_sen`):** Computes fast first-order Taylor PnL via the inner product of the key-rate sensitivity vector and rate shocks[cite: 19]:
  $$\Delta \text{PnL} \approx \sum_{j=1}^{4} N_{\text{eq}} R_{j} + \sum_{k=1}^{30} \frac{\text{PV01}_k}{0.0001} \Delta r_k$$

### 3. Three VaR Engines (1-Day Horizon, 95% Confidence)
1. **Parametric (Delta-Normal) VaR:** Combines the equity notional weights and key-rate PV01 weights into a 34-dimensional exposure vector $\mathbf{w}$, computing portfolio mean $\mu_p = \mathbf{w}^T \boldsymbol{\mu}$ and variance $\sigma_p^2 = \mathbf{w}^T \boldsymbol{\Sigma} \mathbf{w}$ to analytical 95% normal quantile $\text{VaR}_{0.95} = \vert{}\Phi^{-1}(0.05; \mu_p, \sigma_p)\vert{}$[cite: 19].
2. **Monte Carlo VaR (100,000 Scenarios):** Performs **Cholesky Decomposition** ($\mathbf{R} = \mathbf{L}\mathbf{L}^T$) on the 34×34 historical correlation matrix, maps uniform draws through the inverse normal CDF (`NormalDist().inv_cdf`), correlates the standard normal shocks ($\mathbf{Z}_{\text{corr}} = \mathbf{Z}\mathbf{L}^T$), and evaluates the 5th percentile of simulated PnL under both Full Revaluation and Sensitivity-Based pricing[cite: 19].
3. **Historical Simulation VaR:** Replays empirical joint daily changes in equity returns and SOFR zero-rate tenors directly against current portfolio holdings[cite: 19].

---

## Empirical Results (1-Day, 95% VaR)

| Methodology | Valuation Approach | 1-Day 95% VaR ($) | Notes |
| :--- | :--- | :---: | :--- |
| **Parametric VaR** | Delta-Normal ($\mu_p = \$21,305$, $\sigma_p^2 = 3.33 \times 10^{11}$) | **$927,749** | Closed-form variance-covariance |
| **Monte Carlo VaR** | Sensitivity-Based (100,000 Cholesky paths) | **$930,265** | Converges closely to Parametric VaR |
| **Monte Carlo VaR** | Full Revaluation (100,000 curve rebuilds) | **$935,547** | Captures swap discount factor convexity |
| **Historical VaR** | Sensitivity-Based (Empirical replay) | **$979,371** | Reflects fat-tailed historical rate shocks |
| **Historical VaR** | Full Revaluation (Empirical replay) | **$985,468** | Captures both fat tails and non-linear convexity |

---

## Repository Structure

```text
├── notebooks/
│   └── multi_asset_var_engine.ipynb   # Interactive Jupyter Notebook with outputs
├── src/
│   └── var_engine.py                  # Standalone Python module
├── data/                              # Historical SOFR curve & equity price CSVs
├── requirements.txt                   # Package dependencies
└── README.md                          # Documentation
```
