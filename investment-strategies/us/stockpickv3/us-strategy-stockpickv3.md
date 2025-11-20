# Pinnacle US Stock Picking Strategy

## 1. Metadata
Type: Strategy  
Market: US  
Version: v3.0  
Owner: Quant Team  
Author: Ricky  
Last Updated: 2025-11-19  
Related Model: [US Alpha Model v3](../../../alpha-models/us/alpha-linear-v3.md)  
Code: 


## 2. Objective

Construct a **US equity portfolio** (universe: **S&P 500**) that maximizes exposure to the US Alpha Model v3 expected alpha, while maintaining discplined risk controls on:
- Sector allocation (sector momentum-tilted benchmark weights)
- Size neutrality
- Beta exposure (beta timing based on VIX regime)
- Active weight constraints
- Turnover constraints

This strategy is long-only and fully invested.

## 3. Method Summary

- Alpha from US Alpha Model v3.
- Sector momentum for sector tilt.
- Dynamic beta target dependign on VIX rank.
- Optimization to maximize alpha subject to constraints.

## 4. Portfolio Construction

Let:
- $w$: portfolio weights
- $b$: benchmark weights
- $\mu$: expected alpha from Alpha Model v3
- $M_{sector}$: sector exposure matrix
- $M_{\beta}$: beta exposures
- $M_{\text{size}}$: size exposures
- $w_{\text{prev}}$: previous portfolio weights
- $\lambda_{\text{TO}}$: turnover penalty (set to 0.2)
- $\text{sector\_alloc}$: target sector allocation from sector momentum

### 4.1. Optimization Objective

If previous weights are available:

$$ \max_{w} \mu^T w - \lambda_{\text{TO}} \sum |w - w_{\text{prev}}|$$

Else:

$$ \max_{w} \mu^T w$$

## 4.2. Constraints

### 4.2.1. Fully invested and long-only

$$ w \geq 0 $$

$$ 1^T w = 1 $$

### 4.2.2. Active weight bounds

$$ w - b \geq -9\% $$

$$ w - b \leq 9\% $$

### 4.2.3. Size neutrality

$$ M_{\text{size}}^T (w-b) = 0 $$

### 4.2.4. Beta timing constraint

Let VIX rank = $r_{\text{VIX}}$.

If $r_{\text{VIX}} < 0.8$, then $\text{beta\_target} = 0$.  

Else:

$$\text{beta\_target} = -2 + 2.5r_{\text{VIX}}$$

Interpretation: If VIX rank is 0.8 or below, beta target is 0. If VIX rank is 1, then beta target is 0.5. Beta target scales linearly when VIX rank is between 0.8 to 1.

Constraint:

$$M_{\beta} (w-b) = \text{beta\_target}$$

### 4.2.5. Sector constraints

$$
\sum_{i \in S_j} w_i - \text{sector-allocation}\_j \leq 0.02 \text{ for each sector }j
$$

$$
\sum_{i \in S_j} w_i - \text{sector-allocation}\_j \geq -0.02 \text{ for each sector }j
$$

## 4.3. Sector Momentum Strategy

To determine the allocation for each sector, we employ the following sector momentum strategy:

1. **Calculate the 1-day return of sector $j$, $r_{s_j}$**: This is defined as the equal-weighted 1-day return of stocks within sector $j$:

$$r_{s_j} = \dfrac{\sum_{i \in S_j} r_i}{|S_j|}$$

2. **Define the momentum of sector $j$, $M_j$**: $M_j$ represents the 231-day (approximately 11 months) rolling average of the daily returns for sector $j$:

$$M_j = \text{Rolling Average}(r_{s_j}, 231)$$

3. **Compute the cross-sectional z-score across sectors**: Let $M$ be the vector of sector momentum values. For each sector $j$:

$$z_j = \dfrac{M_j - \bar{M}}{\text{stdev}(M)}$$

where $\bar{M}$ is the mean of sector momentum values and $\text{stdev}(M)$ is the standard deviation.

4. **Transform z-scores into positive values**: Apply the following transformation to $z_j$ to ensure all values are positive:

$$f(z_j) = 1+z_j, \text{ if } z_j > 0 \text{ and } \frac{1}{1-z_j} \text{ if } z_j \leq 0$$

5. **Compute the target allocation for sector $j$**: The target allocation for each sector $j$ is given by:

$$\text{sector-allocation}\_j = \dfrac{f(z_j) \times b_j}{\sum_{i} f(z_i) b_i}$$

where $b_i$ represents the total weight of sector $i$ in the benchmark.

## 4.4. VIX Timing Signal

To incorporate market-wide risk sentiment, we compute a VIX rolling percentile rank and use it as a lagged monthly risk-regime indicator.

Computation Steps:

1. Load and clean VIX data

2. Compute 3-year rolling percentile rank

$$\text{VIXRank}_t = \text{PercentileRank}(\text{VIX}_t \text{within last 3 years})$$

3. Convert to monthly signal with 1-month lag

- Aggregate to monthly frequency using the **maximum** rank within each month
- Shift by one month:

$$\text{VIXSignal}_t = \max(\text{VIXRank}_{t-1})$$

## 5. Revision History

- **v1.0 (2025-11-19)**: Initial documentation for US Stock Picking Strategy v3.