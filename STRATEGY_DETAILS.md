# FX Statistical Arbitrage - Detailed Strategy Documentation

## Executive Summary

This document provides an in-depth technical explanation of the **Residual Relative Momentum Strategy** for FX pairs. The strategy exploits statistical mispricings by removing common factors using PCA and trading the persistent directional drift in pair-specific residuals.

---

## Table of Contents

1. [Mathematical Framework](#mathematical-framework)
2. [Data Pipeline](#data-pipeline)
3. [PCA Factor Extraction](#pca-factor-extraction)
4. [Signal Generation](#signal-generation)
5. [Position Construction](#position-construction)
6. [Risk Management](#risk-management)
7. [Implementation Details](#implementation-details)

---

## Mathematical Framework

### 1. Price Normalization

All FX rates are normalized to **units of foreign currency per 1 USD**:

For direct quotes (EUR/USD, GBP/USD):
$$P_{\text{norm}} = P$$

For inverse quotes (USD/JPY, USD/CNH):
$$P_{\text{norm}} = \frac{1}{P}$$

**Result:** Uniform directional interpretation
- Positive return = USD strengthening
- Negative return = USD weakening

### 2. Log Returns

Computed on normalized prices:
$$r_t = \ln\left(\frac{P_t}{P_{t-1}}\right)$$

**Properties:**
- Compoundable over multiple periods
- Symmetric for bid/ask (log scale)
- Suitable for correlation analysis

### 3. Return Matrix

Cross-sectional return vector at time $t$:
$$\mathbf{r}_t = [r_{t,\text{EUR}}, r_{t,\text{GBP}}, r_{t,\text{JPY}}, r_{t,\text{CNH}}]^T$$

---

## Data Pipeline

### Step 1: Raw Data Loading

**Input:** 5-second OHLCV bars for each pair

```
CSV Format:
  utc: Unix timestamp (seconds)
  open, high, low, close: OHLC prices
  volumn: Volume indicator
```

**Output:** Loaded and validated DataFrame per pair

**Handling:**
- Invalid timestamps (NaN) are dropped
- Duplicate timestamps kept (last value)
- Forward-fill NOT used (prevents look-ahead bias)

### Step 2: Timestamp Alignment

All four pairs are **inner-joined** on timestamps:
$$\text{Common Timestamps} = \bigcap_{i=1}^{4} \{\text{Timestamps}_i\}$$

**Rationale:**
- Ensures no forward-filling
- Prevents data artifacts
- Guaranteed 4 prices available at every timestamp

### Step 3: Time-Aggregation to 30-Minute Bars

**Input:** 5-second prices and returns
**Method:** Resample using `last()` (close price)
**Output:** 30-minute OHLC bars

**Formula:**
$$P_{t}^{(30\text{min})} = \text{last}\{P_{s}^{(5\text{sec})} : \lfloor s \rfloor_{30\text{min}} = t\}$$

**Advantages:**
- Reduces noise
- Captures structural relationships
- Aligns with trader decision frequencies

---

## PCA Factor Extraction

### 1. Rolling PCA Setup

**Purpose:** Decompose returns into common and idiosyncratic components

**Window:** 48 bars (24 hours) of 30-minute returns

**Update:** Every bar (rolling window)

**Causality:** PCA model at time $t$ sees data from $[t-48, t-1]$ only

### 2. PCA Decomposition

**Input:** $48 \times 4$ return matrix $X_t$
```
X_t = [
  r_{t-47, EUR}  r_{t-47, GBP}  ...
  r_{t-46, EUR}  r_{t-46, GBP}  ...
  ...
  r_{t-1, EUR}   r_{t-1, GBP}   ...
]
```

**Decomposition:**
$$X_t = U \Sigma V^T$$

where:
- $U$: Orthogonal left singular vectors
- $\Sigma$: Singular values
- $V$: Right singular vectors (loadings)

**Principal Components:**
$$\text{PC}_k = X_t V_k$$

### 3. Component Interpretation

**PC1 (First Principal Component):**
- Explains 40-60% of total variance
- Represents **common USD factor**
- Nearly equal loadings across pairs (all strengthen/weaken with USD moves)

**Example loadings:** 
- EUR: +0.50
- GBP: +0.51
- JPY: +0.49
- CNH: +0.49

**Interpretation:** When PC1 score is positive, USD weakens (all pairs strengthen)

### 4. Residual Computation

Remove PC1 contribution from return vector:

$$\text{Residual}_t = \mathbf{r}_t - (\mathbf{r}_t^T \mathbf{w}_1) \mathbf{w}_1$$

where:
- $\mathbf{w}_1$: PC1 loadings (unit norm)
- $\mathbf{r}_t^T \mathbf{w}_1$: PC1 score
- $(\mathbf{r}_t^T \mathbf{w}_1) \mathbf{w}_1$: PC1 contribution

**Result:** Pair-specific returns net of USD common factor

---

## Signal Generation

### 1. Cumulative Residuals

Represent running "mispricing level":

$$\text{CumRes}_{t,j} = \sum_{s=1}^{t} \text{Residual}_{s,j}$$

**Interpretation:**
- Positive: Pair outperformed factor-adjusted expectation
- Negative: Pair underperformed
- Extreme values: Dislocation ripe for reversal/continuation

### 2. Rolling Z-Score Normalization

Account for time-varying volatility:

$$Z_{t,j} = \frac{\text{CumRes}_{t,j} - \mu_{t,j}}{\sigma_{t,j}}$$

where:
- $\mu_{t,j}$: Rolling mean (24-bar window)
- $\sigma_{t,j}$: Rolling std (24-bar window)

**Window:** 24 bars (12 hours) for both mean and std

**Min periods:** `len(window) // 2` to allow early signal

### 3. EWMA Smoothing

Reduce noise from single-bar shocks:

$$\tilde{Z}_{t,j} = \text{EWMA}(Z_{t,j}, \text{span}=3)$$

**Exponential weights:** Recent bars get more weight

**Span=3:** Halflife ≈ 1.5 hours

### 4. Momentum Signal

**Key hypothesis:** High Z-scores CONTINUE (momentum), not revert

$$\text{Signal}_{t,j} = +Z_{t,j}$$

**Rationale:**
- Cumulative residuals are slow-moving
- Extreme dislocations persist (mean-reversion is slow)
- Trading continuation captures this persistence

---

## Position Construction

### 1. State Machine Entry Logic

#### Pre-Entry Filters

**Top-K Extremes Filter:**
- Only trade assets in top-K by $|Z|$ across all pairs
- Default K=2 (trade only the 2 most extreme)
- Concentrates capital in highest-conviction signals

**Threshold Check:**
$$|Z_{t,j}| \geq \text{ENTRY\_Z} = 1.75$$

**Confirmation Requirement:**
- Must see same sign (strong positive or negative) for 2 consecutive bars
- Prevents whipsaws from single-bar spikes

#### Entry Decision

```
if (abs(Z[t]) >= 1.75) AND (top_k[t]) AND (confirmed_long[t]):
    position[t] = +1.0
elif (abs(Z[t]) >= 1.75) AND (top_k[t]) AND (confirmed_short[t]):
    position[t] = -1.0
else:
    position[t] = 0.0
```

### 2. State Machine Exit Logic

#### Exit Conditions

Applied **only after minimum hold period**:

**Momentum Decay:**
- Exit if $|Z_{t,j}| \leq \text{EXIT\_Z} = 0.9$
- Wide threshold allows signal to persist

**Direction Flip (requires full exit first):**
- If long and see extreme negative ($Z \leq -1.75$): exit first
- If short and see extreme positive ($Z \geq +1.75$): exit first
- No direct position flips (must go through zero)

**Minimum Hold Enforcement:**
$$\text{bars\_in\_position} \geq 6 \text{ (3 hours)}$$

```python
if bars_in_position >= MIN_HOLD_BARS:
    if abs(Z[t]) <= EXIT_Z:
        position[t] = 0.0
```

### 3. Cross-Sectional Neutrality

**Step 1:** Demean positions across assets
$$\tilde{p}_{t,j} = p_{t,j} - \bar{p}_t = p_{t,j} - \frac{1}{4}\sum_{k=1}^{4}p_{t,k}$$

**Step 2:** Normalize gross exposure
$$\text{GrossExp}_t = \sum_{j}|\tilde{p}_{t,j}|$$
$$p'_{t,j} = \tilde{p}_{t,j} / \text{GrossExp}_t$$

**Step 3:** Clip to bounds
$$p''_{t,j} = \text{clip}(p'_{t,j}, -1.0, +1.0)$$

**Step 4:** Re-neutralize
$$p_{\text{final}, t,j} = p''_{t,j} - \bar{p''}_t$$

**Result:** 
- $\sum_j p_{\text{final}, t,j} \approx 0$ (market neutral)
- Each position in $[-1, +1]$ (constraint satisfied)

---

## Risk Management

### 1. Position Constraints

| Constraint | Value | Rationale |
|-----------|-------|-----------|
| Per-asset max | 1.0 | Prevents single-pair domination |
| Net exposure | ≈ 0 | Market neutral (USD risk-free) |
| Gross exposure | ~1.0 | Normalized leverage |
| Min hold | 6 bars (3h) | Reduces turnover, avoids whipsaws |

### 2. Volatility Targeting (Optional)

Scales positions inversely with realized volatility:

$$\text{Scale}_t = \frac{\sigma_{\text{target}}}{\hat{\sigma}_t}$$

where:
- $\sigma_{\text{target}}$: Median realized vol (fixed)
- $\hat{\sigma}_t$: EWMA of squared returns (1-day halflife)

**Bounds:** $[0.75, 1.25]$ to prevent extreme leverage/deleveraging

### 3. Stop-Loss & Profit Targets

**Not explicitly implemented** (relies on exit thresholds and holding constraints)

**Implicit protection:**
- Position always decays toward zero after min hold period
- No position ever forced to stay open indefinitely

---

## Implementation Details

### 1. Chunked Data Loading

Handle large CSV files (multiple GB) using chunking:

```python
for chunk in pd.read_csv(file_path, chunksize=500_000):
    # Process chunk
    # Validate columns
    # Normalize prices
    # Append to all_chunks
```

**Benefits:**
- Low memory usage
- Robust error handling per chunk
- Can skip bad chunks

### 2. Bar Mapping (5-sec to 30-min)

Strictly causal floor alignment:

```python
signal_floor = returns_5sec.index.floor('30min')
# Each 5-sec bar uses position from its floor signal bar
positions_5sec = positions_slow.loc[signal_floor].values
```

**Causality:** Position at 5-sec bar $t$ uses signal from 30-min bar $\lfloor t \rfloor_{30\text{min}}$

### 3. PnL Computation

Standard mark-to-market using lagged positions:

$$\text{PnL}_t = \text{Position}_{t-1} \times r_t$$

**Interpretation:**
- Hold position from bar $t-1$ close
- Realize PnL over bar $t$
- Enter at bar $t$ close (next position change)

### 4. Transaction Costs

**Model:** Linear cost on turnover
$$\text{Cost}_t = \sum_j |\Delta p_{t,j}| \times \frac{\text{cost\_bp}}{10,000}$$

where:
- $\Delta p_{t,j}$: Position change for asset $j$
- cost_bp: Transaction cost in basis points

**Tested levels:**
- 0.5 bp (best execution, major pairs)
- 1.0 bp (realistic institutional)
- 2.0 bp (stressed markets)

### 5. Machine Learning Classifier (Optional)

#### Label Definition
Forward 2-bar gross PnL minus round-trip cost:

$$\text{Label}_{t,j} = \begin{cases}
1 & \text{if } \text{ForwardPnL} - 2 \times \text{cost\_bp} \times 1e-4 > 0 \\
0 & \text{otherwise}
\end{cases}$$

#### Features (Causal Only)
1. $Z_{t,j}$: Current Z-score
2. $|Z_{t,j}|$: Absolute Z-score (signal strength)
3. $\Delta Z_{t,j}$: Z-score change
4. $\Delta \text{PC1Var}$: Change in PC1 explained variance
5. Volatility ratio: Short / long realized vol
6. Session indicators: Asia (0-8 UTC), Europe (8-16 UTC)

#### Walk-Forward Training
- Training: 12 months
- Test: 3 months (rolling forward)
- Model: L2 Logistic Regression
- No hyperparameter tuning (prevents overfitting)

#### Probability-to-Size Mapping

**Linear version:**
$$\text{Size}(p) = \begin{cases}
0.50 & p \leq 0.30 \\
1.00 & p \geq 0.50 \\
0.50 + 0.50 \times \frac{p-0.30}{0.20} & \text{otherwise}
\end{cases}$$

**Concave version (sqrt):**
$$\text{Size}(p) = 0.50 + 0.50 \times \sqrt{\frac{p-0.30}{0.20}}$$

**Concave rationale:** Higher sizes for mid-confidence signals (sqrt gives more weight to [0.30, 0.50])

---

## References

### Methodological Papers

1. **Jolliffe, I. T.** (2002). "Principal Component Analysis" (2nd Edition). Springer-Verlag.
   - Comprehensive PCA reference

2. **Asness, C. S., Moskowitz, T. J., & Pedersen, L. H.** (2013). "Value and Momentum Everywhere."
   - Factor momentum framework

3. **Burnside, C., Eichenbaum, M., & Rebelo, S.** (2011). "Carry Trade and Momentum in Currency Markets."
   - FX momentum empirical evidence

### Implementation References

1. **Scikit-learn Documentation:** PCA, LogisticRegression, StandardScaler
2. **Pandas Documentation:** Time Series, Resampling, Rolling Windows
3. **NumPy Documentation:** Linear Algebra, Array Operations

---

**Last Updated:** January 29, 2026
