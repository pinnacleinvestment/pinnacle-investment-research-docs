# Pinnacle US Alpha Linear V1

## 1. Metadata

| Field | Value |
|-------|--------|
| **Type** | Alpha Model |
| **Market** | US |
| **Version** | v1.0 |
| **Owner** | Quant Team |
| **Author** | Ricky The Ising |
| **Last Updated** | 19 Nov 2025 |
| **Production Start** | 1 Feb 2024 |
| **Production End** | 31 Jan 2025 |
| **Related Model** | |
| **Code** | [Link to Code](https://github.com/pinnacleinvestment/us-production/tree/1a2b7aac27504fc1ccb4320abe7d53da855feca8/src/usprod/model/alphalinearv1) |

## 2. Objective

The objective of **US Alpha Linear V1** is to compute the **expected alpha** for each stock in the **S&P 500 universe**. The model generates by value, quality, and momentum factors, which can then be **ranked or sorted** to guide portfolio construction and stock selection.

## 3. Model

### Factors

The alpha model uses three primary inputs: `value_score`, `quality_score`, and `mom_riskadj_21d_252d`. Each input is defined as follows:

#### 1. Value Score

The `value_score` calculation differs based on whether the firm is financial or non-financial. The scores are standardized at each timestamp $t$ across companies within each group.

- **For Non-Financial Firms:**
  - 40% * `cfo2ev_sa_rank`
  - 30% * `ebitda2ev_sa_rank`
  - 15% * `nic_sa_rank`
  - 15% * `sp_sa_rank`

- **For Financial Firms:**
  - 50% * `nic_sa_rank`
  - 50% * `sp_sa_rank`

The scores above is then standardized at each timestamp $t$ and across each companies in the group (non-financial with non-financial, financial with financial)

#### 2. Quality Score

The `quality_score` is defined as:

- 50% * `chcsho_252d`
- 25% * `asset_turn_sa_rank`
- 25% * `cash_ratio_sa_rank`

#### 3. Risk-adjusted Momentum (Sharpe Ratio)

The `mom_riskadj_21d_252d` at time $t$ is calculated as:

$$\text{mom-riskadj-21d-252d} = \frac{\mu_{t-21}}{\sigma_{t-21}}$$

where:
- $\mu_t = \text{Rolling Average}(r_t, 231)$
- $\sigma_t = \text{Rolling Standard Deviation}(r_t, 231)$

Here, $r_t$ represents the daily return at time $t$.

### Data Preprocessing

#### Outliers Handling

We remove outliers from the data before proceeding with the fitting process by winsorizing the top and bottom 2.5% of the feature and target variables.

#### Missing Values

After handling outliers, we address missing values in the feature set by imputing each missing value with the mean of that feature at the same time period $t$, ensuring each missing value is filled using the mean from its corresponding period.

### Model Construction

#### Linear Regression

At month $t$, we estimate to coefficient ${\beta_t} = [\beta_{\text{value}, t}, \beta_{\text{quality}, t}, \beta_{\text{momentum}, t}]$ by regressing the 1 month-forward return of companies in the **S&P 500** with the 3 features of the model:

$$r_{t} = {\beta_t} {X_{t-1}}$$

where ${X}$ is the feature matrix across stocks up to period $t$.

#### Momentum Crash Adjustment

After estimating the coefficient for each feature, we adjust the momentum coefficient to incorporate with momentum crash indicators as follows:

$$\beta_{\text{momentum}, t}^{adj} = \beta_{\text{momentum, t}} * \max\left(\dfrac{\sigma_{target}}{\sigma_{\text{momentum}, t}}, 2\right) * (-1)^{I_{C, t}}$$

where
- $\sigma_{target} = 0.15$
- $\sigma_{\text{momentum}, t} = \sqrt{\dfrac{\sum_{i=0}^{125} r_{t-i}^2}{126}} \times \sqrt{252}$
- $I_{C, t}$ is the momentum-crash indicator whose value is 1 if $market_{21, t} > 0 \text{ and } market_{504, t} < 0 \text{ and } \beta_{mom, t} < 0$, where:
    - $market_{n, t}$ is defined as the $n$ day rolling average of $r_{market, t}$, which is the equal-weighted 1-day return of stocks in the universe
    - $\beta_{mom, t}$ is defined as the momentum beta, estimated with the following 126-day rolling regression:

      $r_{MOM, t} - r_{f, t} = \alpha + \beta_{mom} (r_{market, t} - r_{f, t}) + \epsilon_t$

      where:
      - $r_{MOM, t}$ is defined as the spread return between winners and losers
      - $r_{f, t}$ is the risk-free rate at time $t$ *(Bloomberg: US0003M INDEX)*

#### Estimation Period

The coefficients are estimated every month using data starting from **December 30, 1994**, with the first estimated coefficients available on **December 31, 2001**.

### Final Scores

The final expected alpha for each stock is computed as follows:

1. Obtain the **regression coefficients** from the linear regression model applied to the three factors (value, quality, momentum).
2. Adjust the **momentum coefficient** according to the momentum crash adjustment.
3. Multiply each factor’s **latest standardized value** by its corresponding (adjusted) coefficient.
4. Sum the contributions across all three factors to generate the **final score (expected alpha)** for each stock:

$$
  \alpha_{i,t} = \beta_{\text{value}, t} \cdot X_{\text{value}, i, t} + \beta_{\text{quality}, t} \cdot X_{\text{quality}, i, t} + \beta_{\text{momentum}, t}^{adj} \cdot X_{\text{momentum}, i, t}
$$

## 4. Revision History
- **v1.0 (2025-11-19)**: Initial documentation for US Alpha Linear V1.