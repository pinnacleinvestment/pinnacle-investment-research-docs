# Pinnacle US Stock Picking Strategy

## 1. Metadata
Type: Strategy  
Market: US  
Version: v1.0  
Owner: Quant Team  
Author: Ricky  
Last Updated: 2025-11-19  
Related Model: [US Alpha Model v1](../../../alpha-models/us/alpha-linear-v1.md)  
Code: https://github.com/pinnacleinvestment/us-production/tree/1a2b7aac27504fc1ccb4320abe7d53da855feca8/src/usprod/strategy/stockpickv1


## 2. Objective

Construct a US equity portfolio that maximizes exposure to the US Alpha Model v1 scores, using the **S&P 500 constituents as the investment universe**, while maintaining disciplined risk controls on sector, size, active weight, and turnover.

## 3. Method Summary

- Alpha from US Alpha Model v1.
- Momentum crash adjustment.
- Sector momentum for sector tilt.
- Optimization to maximize alpha subject to constraints.

### Portfolio Construction

Portfolio weights $w$ is derived from the following optimization problem

$$
  \text{Maximize } {\mu'}{w}
$$

Subject to:

$$
  {1'} {w} = 1
$$

$$
  w \geq 0
$$

$$
{w - b} \leq 0.05
$$

$$
{w - b} \geq -0.05
$$

$$
\sum_{i \in S_j} w_i - \text{sector-allocation}\_j \leq 0.02 \text{ for each sector }j
$$

$$
\sum_{i \in S_j} w_i - \text{sector-allocation}\_j \geq -0.02 \text{ for each sector }j
$$

$$
{z_{\textbf{size}}'} ({w - b}) \leq 0.45
$$

$$
{z_{\textbf{size}}'} ({w - b}) \geq -0.45
$$

$$
{1'} |{w - w_{\text{prev}}}| \leq 0.2
$$

Where:
- ${\mu}$ = expected alpha (from US Alpha Model v1).
- ${b}$ = benchmark weights.
- $S_j$ = set of stocks in sector $j$.
- $\text{sector-allocation}_j$ = sector allocation for sector $j$ (from Sector Momentum Strategy).
- $z_{\textbf{size}}$  is the standardized log of market capitalization.
- $w_{\text{prev}}$  is the previous portfolio weight, normalized so that the sum equals 1.

### Sector Momentum Strategy

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

## 4. Revision History

- **v1.0 (2025-11-19)**: Initial documentation for US Stock Picking Strategy v1.