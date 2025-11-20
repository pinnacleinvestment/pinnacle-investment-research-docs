# Pinnacle US Alpha Linear V3

## 1. Metadata

| Field | Value |
|-------|--------|
| **Type** | Alpha Model |
| **Market** | US |
| **Version** | v3.0 |
| **Owner** | Quant Team |
| **Author** | Ricky The Ising |
| **Last Updated** | 19 Nov 2025 |
| **Production Start** | 1 Aug 2025 |
| **Production End** | — |
| **Related Model** | |
| **Code** | [Link to Code](https://github.com/pinnacleinvestment/us-production/tree/66ddfdc4fae0648bf5aeb35b0f33cb1cc05f2b3b/src/usprod/model/alphalinearv3) |

## 2. Objective

The objective of **US Alpha Linear V3** is to compute the **expected alpha** for each stock in the **S&P 500 universe**. The model generates by value, quality, and momentum factors, which can then be **ranked or sorted** to guide portfolio construction and stock selection.

## 3. Features

The alpha model uses the following features: Cash Flow, EBITDA to EV, Asset Turnover, Cash Ratio, Operating Leverage, Share Buyback, Momentum Score, Volatility

### 3.1. Cash Flow

The Cash Flow factor (`cash_flow_alphalinearv3`) is defined as the equally-weighted average of `fcf2ev_sa_rank`, `cfo2ev_sa_rank`, `cfroi2ev_sa_rank`, `cfoa2ev_sa_rank`.

### 3.2. EBITDA to EV

The EBITDA to EV factor (`ebitda2ev_sa_rank`) is defined as the sector neutral rank of EBITDA (`ebitda`) divided by EV (`ev`).

### 3.3. Asset Turnover

The Asset Turnover factor (`asset_turn_sa_rank`) is defined as the sector neutral rank of Revenue (`revenue`) divided by Total Assets (`total_asset`)

### 3.4. Cash Ratio

The Cash Ratio factor (`cash_ratio_sa_rank`) is defined as the sector neutral rank of Total Cash (`cash`) divided by Total Assets (`total_asset`)

### 3.5. Operating Leverage

The Operating Leverage factor (`operating_leverage_sa_rank`) is defined as the sector neutral rank of Operating Leverage (`operating_leverage`), where:

- Operating Leverage = Operating Liability (`operating_liability`) divided by Net Operating Asset (`net_operating_asset`).

- Operating Liability = Total Asset (`total_asset`) - Total Equity (`total_equity`) - Total Debt (`total_debt`) - Preffered Dividend (`pfd_dividend`)

- Net Operating Asset = Total Equity (`total_equity`) + Total Debt (`total_debt`) + Preferred Dividend (`pfd_dividend`) - Total Cash (`cash`)

### 3.6. Share Buyback

The Share Buyback factor (`chcsho_252d`) is defined the 252-day change of share outstanding adjusted for capital change.

### 3.7. Momentum Score

The Momentum Score factor (`momentum_score_alphalinearv3`) is defined as the equally-weighted average of `mom_residual_21d_252d` and `mom_residual_21d_189d`, where:

1. `mom_residual_21d_252d` for stock $i$ is defined as $\sum_{j=21}^{252} \epsilon_{t-j, i}$

2. `mom_residual_21d_189d` for stock $i$ is defined as $\sum_{j=21}^{189} \epsilon_{t-j, i}$

Here, residual return $\epsilon_t = r_{t,i} - \beta_{t,i} \cdot r_{\text{Market}, t}$.

### 3.8. Features Preprocessing

#### 3.8.1. Missing Values

Missing values in the feature set are imputed cross-sectionally. For each date $t$, any missing value is replaced with the mean of that feature on the same date. This ensures the imputation is time-consistent and avoids introducing look-ahead bias.

#### 3.8.2. Normalization (Cross-Sectional Ranking)

Each features is transformed into a **cross-sectional ranked score** for every date. Values are ranked from 0 to 1 and then scaled to the range [-1,1]

### 3.9. Feature Engineering

Additional transformed features are created:

#### 3.9.1. Sector-specific factor

Activates the cash-flow factor only for stocks in sector 3 (technology).

```
result["cash_flow_alphalinearv3_sector_3"] = (
    result["cash_flow_alphalinearv3"] *
    (cross_sectional_feature_df["stock_sector"].astype(int) == 3).astype(int)
)
```

#### 3.9.2. Volatility-adjusted momentum

The momentum score is scaled by the momentum **factor volatility**, which is the same for all stocks at a given time but varies across dates.

```
result["momentum_score_alphalinearv3_vol"] = (
    -result["momentum_score_alphalinearv3"].multiply(
        time_series_feature_df["mom_vol"], axis=0, level=0
    )
)
```

## 4. Target

The model target is the **risk-adjusted (factor-neutral) monthly return**, not the raw next-month return. Steps:

### 4.1. Compute Monthly Return

Monthly returns are calculated from month-end adjusted prices:

$$r_{t+1} = P_{t+1} / P_{t} - 1$$

### 4.2. Preprocess Risk Factors (Beta & Size)

Before regression, the **beta** and **size** exposures are processed with:

- Winsorization at 2.5% / 97.5%
- Missing values filled with the cross-sectional mean
- Standardization (zero mean, unit variance)

This produces stable, scaled risk-factor inputs for residualization.

### 4.3. Residualize Monthly Returns

For each month, regress returns on the preprocessed beta and size:

$$r_{i,t} = \alpha_t + \gamma_{1,t} \text{Beta}_{i,t} + \gamma_{2,t} \text{Size}_{i, t} + \varepsilon_{i,t}$$

The residual $\varepsilon_{i,t}$ becomes the model's target:

$$\text{target}_{i,t} = \varepsilon_{i,t}$$

### 4.4. Preprocess the Residual Target

The residuals are then processed using:
- Winsorization at 1% / 99%
- Missing values filled with the cross-sectional mean
- Standardization (zero mean, unit variance)

## 5. Model

### 5.1. Constrained Linear Regression

At each month $t$t, the model estimates a non-negative coefficient vector by regressing the risk-adjusted return (residual target) on all features, including two engineered interaction terms.

#### Base Features

$$X_{i,t}^{base} = [X_{\text{CF}}, X_{\text{EBITDA}/EV}, X_{\text{AT}}, X_{\text{CR}},X_{\text{OL}}, X_{\text{BB}}, X_{\text{MOM}}]_{i,t}$$

Corresponding coefficients:

$$\beta_{t}^{base} = [\beta_{\text{CF}}, \beta_{\text{EBITDA}/EV}, \beta_{\text{AT}}, \beta_{\text{CR}},\beta_{\text{OL}}, \beta_{\text{BB}}, \beta_{\text{MOM}}]_{t}$$

#### Regression Equation

$$\varepsilon_{i,t} = \beta_t^{base} \cdot X_{i,t}^{base} + \beta_{\text{CF x Sec3}, t} (X_{\textbf{CF},i,t} \cdot 1_{Sector=3}) - \beta_{\text{Mom x Vol}, t} (X_{\text{MOM},i,t} \cdot \text{MomVol}_t)$$

subject to the **non-negativity constraint**: $\beta \geq 0$

All coefficients must be **positive**, enforcing that each factor contributes positively to expected alpha.

#### Momentum Exposure Interpretation

Momentum enters the model through two components:

$$\beta_{\text{MOM}, t} X_{\text{MOM},t} - \beta_{\text{Mom x Vol}, t} (X_{\text{MOM},i,t} \cdot \text{MomVol}_t)$$

Thus, the **effective momentum exposure** is:

$$\text{MomentumExposure}_{i,t} = X_{\text{MOM},i,t} (\beta_{\text{MOM}, t} - \beta_{\text{Mom x Vol}, t} \cdot \text{MomVol}_t)$$

This structure allows momentum exposure to increase or decrease depending momentum factor volatility. When volatility is high, it's possible to have negative momentum exposure

## 6. Revision History
- **v1.0 (2025-11-19)**: Initial documentation for US Alpha Linear V3.