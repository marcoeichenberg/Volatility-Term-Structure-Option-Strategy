# Implied Volatility Term Structure Forecasting & Trading

## Overview

Implied volatility is not flat across maturities. It forms a term structure that shifts, tilts, and bends over time in response to market conditions. This project treats that term structure as a low-dimensional dynamical system, forecasts its evolution using a rolling PCA + AR(1) framework, and translates the forecast into two independent trading strategies driven by a shared signal engine.

The central empirical result is that the **slope of the ATM implied variance curve is forecastable out-of-sample at a 5-day horizon**. This result is established rigorously before any strategy design and motivates all subsequent work.

---

## Pipeline

```
Raw SPX options data (2.3M rows, 2010–2025)
        │
        ▼
Pre-processing: forward price, risk-free rate, IV mid, log-moneyness
        │
        ▼
ATM variance term structure (5 maturities × daily panel)
        │
        ▼
PCA decomposition → 3 factors (level, slope, curvature)
        │
        ▼
Rolling AR(1) walk-forward backtest (no lookahead)
        │
        ▼
Slope forecast + PC1 regime filter
        ┌──────────────┴──────────────┐
        ▼                             ▼
VIX Futures Strategy       Calendar Spread Strategy
```

---

## Data

Two datasets are used, spanning **January 2010 – December 2025**:

* **SPX options** (~2.3M rows):
  Includes bid/ask, implied volatility, Greeks (delta, gamma, vega, theta), open interest, and volume.

* **SPX daily close prices**:
  Used to anchor ATM strike selection.

* **VX continuous futures (absolute-price adjusted)**:
  Used for Strategy I (VIX futures trading).

---

## Pre-Processing

Three key quantities are derived:

### Forward Price & Risk-Free Rate

Estimated via put-call parity regression:

$$
C - P = DF \cdot (F - K) \quad \Rightarrow \quad C - P = a - bK
$$

where:

* $( b = DF = e^{-rT} )$
* $( a = DF \cdot F )$

This yields arbitrage-free forward prices without external rate inputs.

---

### Log Forward-Moneyness

$$
m = \log\left(\frac{K}{F}\right)
$$

A symmetric and model-consistent measure aligned with Black-Scholes assumptions.

---

### IV Mid

$$
\sigma_{\text{mid}} = \frac{\sigma_{\text{bid}} + \sigma_{\text{ask}}}{2}
$$

A clean, spread-neutral estimate of implied volatility.

---

## Methodology

### Step 1 — ATM Variance Term Structure

For each day and expiry:

* ATM strike = ( \arg\min |m| )
* Variance estimated via vega-weighted average:

$$
\hat{\sigma}_{ATM}^2(T) =
\frac{\sum_i \sigma_i^2 \cdot V_i}{\sum_i V_i}
$$

Five maturities are selected: **14, 30, 60, 90, 180 DTE**, producing a daily ( T \times 5 ) panel.

---

### Step 2 — PCA on Log-Variance

$$
X = \log(\hat{\sigma}_{ATM}^2) - \mu
$$

Three principal components are retained:

* **PC1**: level (linked to VIX)
* **PC2**: slope (term structure tilt)
* **PC3**: curvature

Log-variance ensures approximate Gaussianity and stability.

---

### Step 3 — Factor Analysis & Model Selection

| Factor | ADF p-value | Half-life (days) | Included |
| ------ | ----------- | ---------------- | -------- |
| PC1    | 0.000       | 15.3             | ✓        |
| PC2    | 0.000       | 3.8              | ✓        |
| PC3    | 0.000       | 1.6              | ✗        |

All factors are stationary.
PC3 is excluded because its signal decays faster than the 5-day forecast horizon.

---

### Step 4 — Rolling Walk-Forward Forecast

For each factor:

$$
\hat{f}_{t+h}^{(k)} = \hat{a}_k + \hat{b}*k \cdot \hat{f}*{t+h-1}^{(k)}
$$

Reconstruction:

$$
\hat{X}*{t+H} = \hat{F}*{t+H} \cdot B + \mu
$$

Slope definition:

$$
\text{slope}*{t+H} = X*{t+H}(14d) - X_{t+H}(180d)
$$

Strict walk-forward: no lookahead bias.

---

## Forecasting Results

* **Correlation (forecast vs realised slope): 0.652 (5-day horizon)**

### Regime Insight

Model performance degrades in high-volatility regimes (high PC1).

| Metric   | No Filter | With PC1 Filter |
| -------- | --------- | --------------- |
| Coverage | 89.9%     | 70.1%           |
| Hit-rate | 86.5%     | 90.7%           |

Filtering improves directional accuracy by removing low-quality forecasts.

---

## Strategy I — VIX Futures

### Signal

* Z-score of slope (60-day window)
* Threshold: ( |z| > 0.5 )
* PC1 regime filter applied
* Rebalanced every 5 days (1-day execution lag)

### Results

| Metric       | Value          |
| ------------ | -------------- |
| Sharpe Ratio | 0.36           |
| Max Drawdown | −20.86 VIX pts |
| Active Days  | 68.7%          |

Improves upon the always-short VIX risk premium by avoiding adverse regimes.

---

## Strategy II — Calendar Spreads

### Construction

* ATM calls:

  * ~30 DTE (20–40)
  * ~60 DTE (50–70)
* Positions:

  * Long calendar → rising short-term IV
  * Short calendar → falling short-term IV

### Features

* Same signal engine as Strategy I
* Position size: 0–3 contracts
* Scaled by signal strength
* Penalised in high-vol regimes
* Explicit transaction costs (bid-ask + slippage)

### Analytics

Includes:

* Greeks attribution (vega, theta, gamma, delta)
* Regime-based performance
* Signal strength quartiles
* Probability of Backtest Overfitting (PBO)

---

## Closing Remark

This project is built around a single empirical observation:
**the slope of the implied variance term structure contains predictive information.**

Everything else—modeling choices, filtering, and strategy design—follows from respecting that signal and its limitations.
