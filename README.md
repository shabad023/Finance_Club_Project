# NIFTY Options IV Surface Reconstruction

**Finance Club Open Project 2026 — IIT Roorkee**
Kaggle Competition · Metric: Mean Squared Error

---

## Problem

Given partially observed implied-volatility (IV) data for the NIFTY 50 options chain — 975 five-minute timestamps from 7–27 January 2026, 28 option strikes, roughly 20% of values missing — reconstruct the complete IV surface by predicting the missing entries as accurately as possible.

---

## Key Insight

The volatility smile is not one continuous curve across all 28 strikes. **Calls (CE) and puts (PE) sit on two separate curves** with a sharp discontinuity near the spot price. Fitting a single interpolation across all strikes drags every prediction near that boundary across the jump. Near expiry, where IV levels are large, those boundary errors dominate the squared-error metric. Fitting CE and PE **independently at each timestamp** removes the problem entirely and drives a ~2x improvement over the naive combined baseline.

---

## Approach

Pure interpolation — no machine learning, no lookahead, no external data.

**For each missing cell:**
1. Group the observed strikes at the **same timestamp** by option type (call or put separately).
2. Interpolate IV across log-moneyness within that group.
3. Use **shape-preserving cubic (PCHIP)** interpolation inside the observed strike range on normal days, where the smile is smooth. Use **linear** interpolation on expiry day (noisier regime) and for any extrapolation beyond the observed range.

Log-moneyness `k = log(strike / underlying_price)` is the natural x-axis for the volatility smile — it centres the curve around zero and is the standard representation in options modelling.

**Why no ML model on top?** A residual boosting model was tested. Once CE/PE are fitted separately, the interpolation residuals carry no learnable pattern — simpler won.

**Why no time-direction interpolation?** Using a strike's IV at other timestamps would require future observations for interior missing cells — lookahead bias, which the competition explicitly penalises. The method is strictly cross-sectional.

---

## Results

| Method | Validation MSE (6-seed avg) |
|---|---|
| Combined single-curve interpolation (baseline) | ~1.7e-4 |
| Separate CE/PE linear interpolation | ~9.4e-5 |
| Separate CE/PE hybrid (PCHIP + linear) | ~9.3e-5 |

**Public leaderboard score: 0.0000422616**

Error breakdown: normal-day cells achieve ~9e-6 (near-perfect); essentially all remaining error concentrates in the final 30 minutes before expiry (Jan 27, 15:00–15:25), where IV is genuinely extreme and noisy — an irreducible property of near-expiry options data, not a modelling gap.

---

## What Was Tested (and Why Each Lost)

| Method | Result | Why |
|---|---|---|
| Combined smile interpolation | Baseline | Interpolates across CE/PE jump |
| Residual GBM on interpolation error | No improvement | Residuals carry no pattern after separation |
| Stacking (interpolations as ML features) | Worse | Expiry-day spikes dominate ML training |
| Quadratic / cubic interpolation everywhere | Worse | Overshoots near-expiry noise |
| SVI parametric smile fit | Much worse | Optimiser unstable near expiry (small time-to-expiry) |
| IV² (variance-space) interpolation | Worse | No advantage over direct IV |
| Temporal smoothing across timestamps | Worse | Near-expiry smile moves too fast to average |
| Flat extrapolation at wings | Much worse | Smile genuinely keeps rising at the tails |
| Time-direction interpolation blend | Lookahead bias | Removed entirely (W_TIME = 0) |

---

## Repository Structure

```
├── iv_surface_FINAL_hybrid.ipynb   # Full notebook: EDA, model, validation, submission
├── filled_dataset.csv              # Original dataset with missing values filled
├── submission.csv                  # Kaggle submission (id||column, value format)
└── README.md
```

---

## Reproducing the Results

**Requirements:** Python 3.x, pandas, numpy, scipy — no additional installs.

```bash
# Run the notebook top-to-bottom in Kaggle or locally
# Update DATA_PATH in Cell 1 to point to your copy of dataset.csv
# Outputs: filled_dataset.csv and submission.csv
```

On Kaggle, set:
```python
DATA_PATH = '/kaggle/input/finclub-open-project-26/dataset.csv'
```

Locally, set it to your local path. The notebook is seeded (`RANDOM_SEED = 42`) and produces identical output on every run.

---

## Submission Format

The organizers require a two-column CSV with ids of the form `datetime||column`:

```
id,value
07-01-2026 09:15||NIFTY27JAN2624100PE,0.163428
07-01-2026 09:15||NIFTY27JAN2625500CE,0.113723
...
```

The notebook reproduces the organizers' `generate_solution` converter internally and writes this file directly. A `filled_dataset.csv` is also written as an intermediate for compatibility with the converter script.

---

## Design Decisions

**Linear extrapolation at the wings** — when a missing strike lies beyond the range of observed strikes, the interpolator extends the last slope linearly. Alternatives tested: flat (clamp to nearest) and capped. Both were worse because the smile genuinely rises at the tails and our predictions were already well-calibrated (no overshoot).

**PCHIP for normal days, linear for expiry day** — PCHIP is monotonicity-preserving and captures the gentle curvature of a smooth smile without overshooting. It is not applied on expiry day because the near-expiry smile is noisy, and curvature-fitting there tracks noise rather than structure.

**W_TIME = 0** — the time-interpolation function is defined in the notebook but disabled. Time interpolation uses future observations for interior missing cells, which is lookahead bias under the competition rules.

**6-decimal rounding** — predicted values are rounded to 6 decimal places before writing. Full-precision floats (17 digits) cause Excel to import cells as text. The rounding error (~1e-13 per cell) has no effect on the MSE.

---

## Competition

[Finance Club Open Project 2026 — IIT Roorkee](https://www.kaggle.com/competitions/finclub-open-project-26)
