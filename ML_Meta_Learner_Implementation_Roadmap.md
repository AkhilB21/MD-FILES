# ML Meta-Learner Implementation Roadmap

**Feature:** ML Confidence Score — a third opinion layer alongside Apollo and LayerSignal  
**Models:** Logistic Regression (diagnostic) → XGBoost (production)  
**Architecture:** Feed engine outputs (not raw OHLCV) to ML model, predict forward returns, integrate as "ML Confidence" column  
**Data Source:** Backtest parquets (APOLLO_DATA_REPOSITORY) + Kite Connect live feed (volume fix)  
**Last Updated:** 2026-08-16

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Prerequisite Gate Conditions](#2-prerequisite-gate-conditions)
3. [Feature Engineering](#3-feature-engineering)
4. [Target Definition](#4-target-definition)
5. [Critical Fix — Gap 2: Overlapping Window Leakage](#5-critical-fix--gap-2-overlapping-window-leakage)
6. [Critical Fix — Gap 3: Multicollinearity](#6-critical-fix--gap-3-multicollinearity)
7. [Phase 1: Dataset Construction](#7-phase-1-dataset-construction)
8. [Phase 2: Logistic Regression Diagnostic](#8-phase-2-logistic-regression-diagnostic)
9. [Phase 3: XGBoost Meta-Learner](#9-phase-3-xgboost-meta-learner)
10. [Phase 4: Dashboard Integration](#10-phase-4-dashboard-integration)
11. [Phase 5: Live Inference Pipeline](#11-phase-5-live-inference-pipeline)
12. [Acceptance Criteria](#12-acceptance-criteria)
13. [Risk Register](#13-risk-register)
14. [Milestone Timeline](#14-milestone-timeline)

---

## 1. Architecture Overview

### What This Feature Does

The ML meta-learner takes the **existing outputs** of the Apollo and LayerSignal engines and learns how their combinations predict forward stock returns. It does NOT replace the engines — it adds a quantitative probability layer on top.

```
Current System (Rule-Based):
  Apollo Score: 118  →  ENTRY
  LayerSignal: EL1   →  ENTRY
  User mentally combines: "both say buy, probably good"

With ML Meta-Learner:
  Apollo Score: 118, LayerSignal: EL1, Gates: 4/5, Quality: STRONG
  ML Confidence: 73%  →  "73% probability of 5%+ gain in 20 days"
  User: "Both engines agree AND the model says 73% — high conviction"
```

### Why Engine Outputs, Not Raw OHLCV

Your Apollo and LayerSignal engines already encapsulate domain knowledge — they've distilled raw price/volume/indicator data into actionable signals. Feeding raw OHLCV to the ML model would ignore all that engineering. By feeding engine outputs instead, the ML model's job is narrowed to: **learning which combinations of existing signals predict outcomes best.** This is the highest-ROI use of ML in your system.

### Data Flow

```
Daily Data Update (bridge_daily_update.py)
    │
    ├─→ Apollo Engine → Apollo_Score, Gate_1..5, Quality, FQS, Bucket
    ├─→ LayerSignal Engine → LS_Score, ELStatus, Entry_Type
    ├─→ Technical Indicators → RSI21, RSI36, RSI56, ADX, ATR_Pct, Exit_Pressure
    ├─→ Kite Connect (NEW) → Live volume, traded value, tick-level data
    │
    └─→ ML Inference Service
         │
         ├─→ Loads pre-trained XGBoost model (pickle file)
         ├─→ Constructs feature row from today's engine outputs
         ├─→ Predicts: P(forward_20d_return > 5%)
         ├─→ Outputs: ml_confidence (0.0 – 1.0)
         │
         └─→ Stored in cache alongside other stock data
              │
              └─→ Frontend reads ml_confidence as additional column
                 in Screener, IPO Tab, Blue Sky Tab, 52WH Tab, Scanner
```

---

## 2. Prerequisite Gate Conditions

**These must be verified BEFORE starting Phase 1.** If any gate fails, fix it first.

### Gate 1: Backtest Parquet Integrity (Gap 1)

**Problem:** If Apollo's backtest parquets were generated while Apollo had a known look-ahead bias bug, the meta-learner will learn on leaked data. Backtest accuracy will look strong. Live accuracy won't match.

**What to verify:**

1. Check the date the backtest parquets in `APOLLO_DATA_REPOSITORY` were last generated
2. Check the date the Apollo look-ahead bias fix (if any) was deployed
3. If parquets were generated **before** the fix → **regenerate them after the fix**
4. If parquets were generated **after** the fix, or no look-ahead bug existed → proceed

**How to check:**

```bash
# Check parquet file modification dates
ls -la APOLLO_DATA_REPOSITORY/*.parquet

# Check Apollo engine code history for look-ahead fixes
git log --oneline --all -- backend/engines/apollo.py | head -20
```

**Decision rule:**

| Parquet Generation | Look-Ahead Fix Status | Action |
|---|---|---|
| Before fix | Fix exists | **STOP. Regenerate parquets after fix.** |
| After fix | Fix exists | Proceed. Parquets are clean. |
| Any time | No fix needed (never existed) | Proceed. |
| Unknown | Unknown | Investigate Apollo code for look-ahead patterns before proceeding. |

**What to look for in Apollo code:** Any place where future data (tomorrow's close, next day's high) is used to compute today's Apollo_Score. Common culprits:

- Using `df['Close'].shift(-1)` or forward-looking operations
- Computing indicators using a window that extends into the future
- Using the current day's close as both the signal input AND part of the label

### Gate 2: Volume Data Availability

**Status:** Kite Connect live feed arriving this week. This resolves the volume data gap.

**What to verify:**

1. Kite Connect API provides `volume` and `traded_value` per stock per day
2. These fields are successfully integrated into the daily data pipeline
3. Backtest parquets include volume/traded_value columns
4. At least 6+ months of historical volume data is available (for meaningful training)

### Gate 3: Minimum Data Volume

The models need sufficient historical data:

| Model | Minimum Rows | Your Data | Status |
|---|---|---|---|
| Logistic Regression | 5,000 independent samples | ~150K rows (before purge) | Sufficient |
| XGBoost | 10,000 independent samples | ~150K rows (before purge) | Sufficient |

After the purge (Gap 2 fix), expect ~100K usable rows. Still sufficient.

---

## 3. Feature Engineering

### Feature Set (18 inputs)

All features are **engine outputs** — already computed by your existing pipeline. No new calculations needed.

| # | Feature | Type | Source | Range | Notes |
|---|---|---|---|---|---|
| 1 | `Apollo_Score` | Continuous | Apollo Engine | 0–200 | Composite score |
| 2 | `LS_Score` | Continuous | LayerSignal Engine | 0–100 | LayerSignal composite |
| 3 | `Exit_Pressure` | Continuous | Distribution metric | 0–100 | High = distribution |
| 4 | `RSI21` | Continuous | Technical indicator | 0–100 | Short-term momentum |
| 5 | `RSI36` | Continuous | Technical indicator | 0–100 | Medium-term momentum |
| 6 | `RSI56` | Continuous | Technical indicator | 0–100 | Long-term momentum |
| 7 | `ADX` | Continuous | Trend strength | 0–100 | >25 = trending |
| 8 | `ATR_Pct` | Continuous | Volatility | 0–10% | Normalized ATR |
| 9 | `Pct_Change` | Continuous | Daily return | -10% to +10% | Today's price change |
| 10 | `52W_Prox` | Continuous | Price position | 50–110% | % of 52-week high |
| 11 | `Traded_Value` | Continuous | Kite Connect | 0–1B+ | **Log-scaled** to reduce skew |
| 12 | `Gate_1` | Binary | 5-Gate System | 0 or 1 | Regime gate |
| 13 | `Gate_2` | Binary | 5-Gate System | 0 or 1 | Trend gate |
| 14 | `Gate_3` | Binary | 5-Gate System | 0 or 1 | Momentum gate |
| 15 | `Gate_4` | Binary | 5-Gate System | 0 or 1 | Volatility gate |
| 16 | `Gate_5` | Binary | 5-Gate System | 0 or 1 | Quality gate |
| 17 | `Bucket` | Categorical | Bucket System | L1/L2/L3/L4 | One-hot encoded |
| 18 | `MCap` | Categorical | Market cap | Large/Mid/Small | One-hot encoded |

### Feature Preprocessing

```python
# File: ml/features.py

import numpy as np
import pandas as pd
from sklearn.preprocessing import LabelEncoder

def preprocess_features(df: pd.DataFrame) -> pd.DataFrame:
    """
    Preprocess raw engine outputs into ML-ready features.
    
    Input: df with raw columns from backtest parquets
    Output: df with preprocessed numeric features ready for model input
    """
    features = pd.DataFrame()
    
    # Continuous features — copy as-is (already numeric)
    continuous_cols = [
        'Apollo_Score', 'LS_Score', 'Exit_Pressure',
        'RSI21', 'RSI36', 'RSI56', 'ADX', 'ATR_Pct',
        'Pct_Change', '52W_Prox'
    ]
    for col in continuous_cols:
        if col in df.columns:
            features[col] = pd.to_numeric(df[col], errors='coerce').fillna(0)
    
    # Traded_Value — log-scaled to handle extreme skew
    # Kite Connect provides this; stocks like Reliance trade 5000Cr while
    # small caps trade 5Cr. Without log scaling, the model sees a 1000x range.
    if 'Traded_Value' in df.columns:
        tv = pd.to_numeric(df['Traded_Value'], errors='coerce').fillna(1)
        features['Traded_Value'] = np.log1p(tv)  # log(1 + x) handles zeros
    
    # Gate columns — convert PASS/FAIL to 1/0
    gate_cols = ['Gate_1', 'Gate_2', 'Gate_3', 'Gate_4', 'Gate_5']
    for col in gate_cols:
        if col in df.columns:
            features[col] = (df[col] == 'PASS').astype(int)
    
    # Categorical features — one-hot encode
    # Bucket: L1, L2, L3, L4
    if 'Bucket' in df.columns:
        bucket_dummies = pd.get_dummies(df['Bucket'], prefix='Bucket', dtype=int)
        features = pd.concat([features, bucket_dummies], axis=1)
    
    # MCap: Large, Mid, Small
    if 'MCap' in df.columns:
        mcap_dummies = pd.get_dummies(df['MCap'], prefix='MCap', dtype=int)
        features = pd.concat([features, mcap_dummies], axis=1)
    
    # Drop any rows that are entirely NaN after preprocessing
    features = features.dropna(how='all')
    
    return features
```

---

## 4. Target Definition

### Forward Return Targets

Train **separate models** for each time horizon. This lets users choose the horizon that matches their trading style.

| Target Variable | Question | Threshold | Use Case |
|---|---|---|---|
| `target_5d_pos` | Will it gain 2%+ in 5 trading days? | 2% | Quick swing trades |
| `target_10d_pos` | Will it gain 3%+ in 10 trading days? | 3% | Short-term positional |
| `target_20d_pos` | Will it gain 5%+ in 20 trading days? | 5% | Trend following |
| `target_20d_cont` | What's the expected 20-day return %? | Continuous | Position sizing |

### Target Computation

```python
# File: ml/targets.py

def compute_forward_returns(df: pd.DataFrame, price_col: str = 'Close') -> pd.DataFrame:
    """
    Compute forward return targets for each stock-day.
    
    CRITICAL: This must be computed BEFORE any train/test splitting.
    The target uses FUTURE data by definition — that's the whole point.
    The Gap 2 purge ensures training rows don't leak test labels.
    """
    # Sort by symbol and date to ensure correct forward looking
    df = df.sort_values(['symbol', 'date']).copy()
    
    # Forward close prices (next 5, 10, 20 trading days)
    # groupby symbol ensures we don't accidentally use next stock's close
    for days in [5, 10, 20]:
        df[f'close_{days}d_ahead'] = df.groupby('symbol')[price_col].shift(-days)
    
    # Compute forward returns
    for days in [5, 10, 20]:
        threshold_map = {5: 0.02, 10: 0.03, 20: 0.05}
        threshold = threshold_map[days]
        
        fwd_return = (df[f'close_{days}d_ahead'] / df[price_col]) - 1
        df[f'fwd_return_{days}d'] = fwd_return
        df[f'target_{days}d_pos'] = (fwd_return >= threshold).astype(int)
    
    # Continuous target for position sizing
    df['target_20d_cont'] = df['fwd_return_20d'] * 100  # As percentage
    
    return df
```

---

## 5. Critical Fix — Gap 2: Overlapping Window Leakage

### The Problem (Read This Carefully)

Your 20-day forward-return label creates **overlapping windows**. Consider two consecutive rows for the same stock:

```
Row 1: Date = Oct 1,  Label = (Close_Oct21 / Close_Oct1) - 1
Row 2: Date = Oct 2,  Label = (Close_Oct22 / Close_Oct2) - 1
```

These two labels share **19 out of 20 price days** (Oct 2 through Oct 21). They are NOT independent observations. If you put Row 1 in training and Row 2 in testing, the test label contains 95% of the same price data as a training label. This is **information leakage** — your test metrics will be optimistically biased.

A simple time-based split ("train before Oct 1, test after Oct 1") does NOT prevent this. The last ~20 training rows (Sep 11–Oct 1) have labels that extend into the test set (Oct 1–Oct 21). The model has already "seen" those price movements through the leaked labels.

### The Solution: Purged Time-Series Split with Embargo

This is standard methodology from Marcos López de Prado's *Advances in Financial Machine Learning*. Three steps:

**Step 1: Purge** — Remove training rows whose label window overlaps into the test set.

**Step 2: Embargo** — Add a buffer zone after the purge boundary to prevent near-boundary rows from being almost identical.

**Step 3: Validate** — Confirm zero overlap between any training label's price window and any test label's price window.

### Implementation

```python
# File: ml/splits.py

import pandas as pd
import numpy as np

def purged_time_series_split(
    df: pd.DataFrame,
    date_col: str = 'date',
    symbol_col: str = 'symbol',
    test_start_date: str = '2025-10-01',
    max_label_horizon: int = 20,
    embargo_days: int = 5
) -> tuple[pd.DataFrame, pd.DataFrame]:
    """
    Split data into train and test with purged boundary.
    
    Parameters
    ----------
    df : DataFrame with columns [date, symbol, ...features..., ...targets...]
    test_start_date : Date string. All rows ON or AFTER this date go to test.
    max_label_horizon : The longest forward-return window used in any target (20 days).
        Any training row within this many trading days BEFORE test_start_date
        has a label that peeks into the test set. These must be purged.
    embargo_days : Additional buffer days AFTER the purge zone.
        Rows in this zone have labels that nearly overlap test labels.
        Removing them prevents the model from learning boundary artifacts.
    
    Returns
    -------
    (train_df, test_df) : DataFrames with no label overlap.
    """
    df[date_col] = pd.to_datetime(df[date_col])
    test_start = pd.Timestamp(test_start_date)
    
    # Step 1: Identify the purge boundary
    # For each symbol, find the last training-eligible date.
    # A training row on date D is INELIGIBLE if D + max_label_horizon >= test_start
    # i.e., D >= test_start - max_label_horizon
    # So the last eligible training date is: test_start - max_label_horizon - 1 trading day
    
    # Compute unique trading dates and map to ordinal indices
    all_dates = df[date_col].dt.normalize().unique()
    all_dates = np.sort(all_dates)
    
    # Find the index of test_start in the trading calendar
    test_start_idx = np.searchsorted(all_dates, test_start)
    
    # Purge zone: last (max_label_horizon) trading days before test_start
    purge_end_idx = test_start_idx  # exclusive
    purge_start_idx = max(0, purge_end_idx - max_label_horizon)
    
    # Embargo zone: embargo_days BEFORE the purge zone
    embargo_end_idx = purge_start_idx
    embargo_start_idx = max(0, embargo_end_idx - embargo_days)
    
    # The cutoff date for training is the embargo boundary
    train_cutoff_date = all_dates[embargo_start_idx]
    
    print(f"Trading calendar: {len(all_dates)} days")
    print(f"Test starts at index {test_start_idx} ({all_dates[test_start_idx].date()})")
    print(f"Purge zone: indices {purge_start_idx}–{purge_end_idx-1} "
          f"({all_dates[purge_start_idx].date()} to {all_dates[purge_end_idx-1].date()})")
    print(f"Embargo zone: indices {embargo_start_idx}–{embargo_end_idx-1} "
          f"({all_dates[embargo_start_idx].date()} to {all_dates[embargo_end_idx-1].date()})")
    print(f"Training cutoff: {train_cutoff_date.date()}")
    
    # Step 2: Split
    train_df = df[df[date_col] < train_cutoff_date].copy()
    test_df = df[df[date_col] >= test_start].copy()
    
    # Step 3: Validate — confirm zero label overlap
    _validate_no_overlap(train_df, test_df, date_col, symbol_col, max_label_horizon)
    
    print(f"\nFinal split: Train={len(train_df)} rows, Test={len(test_df)} rows")
    print(f"Purged {len(df) - len(train_df) - len(test_df)} rows from boundary zone\n")
    
    return train_df, test_df


def _validate_no_overlap(
    train_df: pd.DataFrame,
    test_df: pd.DataFrame,
    date_col: str,
    symbol_col: str,
    max_horizon: int
):
    """
    Validate that NO training row's label window overlaps with ANY test row's date.
    
    For each training row at date D, the label uses prices from D to D+max_horizon.
    For each test row at date T, it occupies date T.
    Overlap exists if D <= T <= D + max_horizon, i.e., T - max_horizon <= D <= T.
    
    This is O(n*m) in the worst case. For efficiency, we check at the boundary.
    """
    train_dates = train_df[date_col].dt.normalize().unique()
    test_dates = test_df[date_col].dt.normalize().unique()
    
    latest_train = max(train_dates)
    earliest_test = min(test_dates)
    
    # The latest training row's label extends to latest_train + max_horizon trading days
    # Convert max_horizon to calendar days (approximate: trading days * 1.5)
    max_calendar_extension = pd.Timedelta(days=max_horizon * 2)
    label_end = latest_train + max_calendar_extension
    
    if label_end >= earliest_test:
        overlap_days = (label_end - earliest_test).days
        raise ValueError(
            f"VALIDATION FAILED: Training labels extend {overlap_days} calendar days "
            f"into test set. Latest train date: {latest_train.date()}, "
            f"Earliest test date: {earliest_test.date()}. "
            f"Increase embargo_days or push test_start_date later."
        )
    
    print(f"Validation PASSED: No label overlap. "
          f"Latest train label ends before {label_end.date()}, "
          f"earliest test row is {earliest_test.date()}")
```

### Usage Example

```python
# In your training script:
from ml.splits import purged_time_series_split

# Load data with features AND targets already computed
full_df = pd.read_parquet('APOLLO_DATA_REPOSITORY/ml_ready.parquet')

# Split with purge + embargo
# Using Oct 1 2025 as test start (adjust based on your data range)
train_df, test_df = purged_time_series_split(
    df=full_df,
    date_col='date',
    symbol_col='symbol',
    test_start_date='2025-10-01',
    max_label_horizon=20,   # Match your longest target (20d)
    embargo_days=5         # 5 extra trading days of buffer
)

# Now train and evaluate safely:
X_train = train_df[feature_columns]
y_train = train_df['target_20d_pos']
X_test = test_df[feature_columns]
y_test = test_df['target_20d_pos']
```

### Why Embargo Days Matter

Even after purging the 20-day overlap zone, rows just outside the purge boundary are problematic. Consider a training row on Sep 10 (just before the purge zone starts Sep 11). Its 20-day label uses prices through Oct 1. A test row on Oct 2 uses prices from Oct 2. These share 19/20 days of price data — essentially the same observation in both sets.

The 5-day embargo pushes the training cutoff further back (to ~Sep 4), ensuring even near-boundary training rows have minimal price overlap with test rows.

### What This Costs You

For 297 stocks × ~500 trading days = ~150K rows:

```
Before purge:  ~150,000 rows
Purge zone:   ~297 stocks × 25 days (20 purge + 5 embargo) = ~7,425 rows removed
After purge:   ~142,575 rows (train: ~115K, test: ~27K)
```

You lose ~5% of your data. Negligible impact on model quality. Massive impact on metric reliability.

---

## 6. Critical Fix — Gap 3: Multicollinearity

### The Problem

Apollo_Score is a **composite** of RSI, ADX, ATR, Volume, and other indicators. If you feed both `Apollo_Score` AND `RSI21`, `ADX`, `ATR_Pct` into the same model, you have multicollinearity — multiple features that are mathematically related because one is computed from the others.

**Impact:**

| What | Affected? | Why |
|---|---|---|
| **Prediction accuracy** | Not affected | Models handle correlated features fine for prediction |
| **Feature importance rankings** | **Severely affected** | Coefficients become unstable — small data changes cause large coefficient swings |
| **"Gate_3 is 5x more important than Gate_2"** | **Unreliable** | This specific conclusion may be an artifact of multicollinearity, not a real signal |

### The Solution: Use Permutation Importance, Not Raw Coefficients

**For the Logistic Regression diagnostic phase** (Phase 2), do NOT interpret raw coefficients. Instead, use permutation importance to determine which features actually matter.

```python
# File: ml/diagnostic.py

from sklearn.inspection import permutation_importance
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import roc_auc_score
import matplotlib.pyplot as plt


def compute_feature_importance(model, X_train, y_train, X_test, y_test, feature_names):
    """
    Compute permutation importance instead of reading raw coefficients.
    
    How it works:
    1. Take a trained model with known test accuracy
    2. For each feature, randomly shuffle that feature's values in the test set
       3. Measure how much the model's accuracy drops
    4. The larger the drop, the more important that feature actually is
    
    Why this works:
    - It measures the feature's CONTRIBUTION TO PREDICTION, not its coefficient
    - It's immune to multicollinearity — shuffling one correlated feature
      doesn't change the others, so the model can't "compensate"
    - It's model-agnostic — works for logistic regression AND XGBoost
    """
    # Baseline accuracy
    baseline_auc = roc_auc_score(y_test, model.predict_proba(X_test)[:, 1])
    print(f"Baseline AUC-ROC: {baseline_auc:.4f}")
    
    # Permutation importance on test set (most reliable)
    perm_result = permutation_importance(
        model, X_test, y_test,
        n_repeats=30,           # 30 random shuffles per feature for stability
        random_state=42,
        scoring='roc_auc',      # Use AUC as the metric
        n_jobs=-1               # Parallel
    )
    
    # Build importance dataframe
    importance_df = pd.DataFrame({
        'feature': feature_names,
        'importance_mean': perm_result.importances_mean,
        'importance_std': perm_result.importances_std,
    }).sort_values('importance_mean', ascending=False)
    
    # Print ranked features
    print("\n=== Feature Importance (Permutation) ===")
    print("(Higher = more important. Negative = feature actually hurts)")
    for i, row in importance_df.iterrows():
        bar = '#' * int(row['importance_mean'] * 100)
        print(f"  {row['feature']:20s}  {row['importance_mean']:+.4f}  +/- {row['importance_std']:.4f}  {bar}")
    
    return importance_df


def plot_feature_importance(importance_df, save_path: str = None):
    """
    Horizontal bar chart of permutation importance.
    """
    fig, ax = plt.subplots(figsize=(10, 8))
    
    # Only plot features with non-zero importance
    df = importance_df[importance_df['importance_mean'] != 0].copy()
    df = df.sort_values('importance_mean')
    
    colors = ['#10B981' if x > 0 else '#EF4444' for x in df['importance_mean']]
    
    ax.barh(df['feature'], df['importance_mean'], xerr=df['importance_std'],
            color=colors, edgecolor='white', linewidth=0.5)
    ax.set_xlabel('Permutation Importance (AUC-ROC impact)')
    ax.set_title('Feature Importance — ML Meta-Learner')
    ax.axvline(x=0, color='gray', linewidth=0.8)
    
    plt.tight_layout()
    
    if save_path:
        plt.savefig(save_path, dpi=150, bbox_inches='tight')
        print(f"Saved to {save_path}")
    plt.close()
```

### Additional Safeguard: L1-Regularized Logistic Regression

If you want to use logistic regression coefficients at all (for the diagnostic phase), use L1 regularization (Lasso) instead of standard logistic regression. L1 regularization drives unimportant feature coefficients to exactly zero, which naturally handles multicollinearity by selecting one feature from each correlated group.

```python
# Standard logistic regression — coefficients unreliable with multicollinearity
from sklearn.linear_model import LogisticRegression
model_standard = LogisticRegression(max_iter=1000)  # L2 regularization (default)

# L1-regularized (Lasso) — coefficients reliable, unimportant features get zeroed
model_l1 = LogisticRegression(max_iter=1000, penalty='l1', solver='saga')
```

### Summary: What the Implementer Should Do

| Step | Action | Why |
|---|---|---|
| 1 | Never interpret raw logistic regression coefficients | Multicollinearity makes them unreliable |
| 2 | Always use `permutation_importance()` for feature ranking | Measures actual prediction contribution, immune to collinearity |
| 3 | If using logistic regression for diagnostics, use `penalty='l1', solver='saga'` | L1 regularization zeroes out redundant features |
| 4 | For XGBoost, use built-in `model.feature_importances_` (gain-based) | XGBoost handles collinearity natively via tree splitting |
| 5 | For XGBoost interpretation, SHAP values are optional but recommended | SHAP shows per-prediction feature contributions, accounts for interactions |

---

## 7. Phase 1: Dataset Construction

**Duration:** 1–2 days  
**Goal:** Produce a single, clean parquet file with features and targets, ready for model training.

### Step-by-Step

**Step 1: Load and merge backtest parquets**

```python
# File: ml/build_dataset.py

import pandas as pd
import glob
import os

# Load all parquet files from the repository
parquet_files = glob.glob('APOLLO_DATA_REPOSITORY/*.parquet')
print(f"Found {len(parquet_files)} parquet files")

df = pd.concat([pd.read_parquet(f) for f in parquet_files], ignore_index=True)
print(f"Total rows: {len(df)}")

# Required columns — verify they exist
required_cols = [
    'symbol', 'date', 'Close', 'High52W', 'CMP',
    'Apollo_Score', 'LS_Score', 'Exit_Pressure',
    'RSI21', 'RSI36', 'RSI56', 'ADX', 'ATR_Pct', 'Pct_Change',
    'Gate_1', 'Gate_2', 'Gate_3', 'Gate_4', 'Gate_5',
    'Bucket', 'MCap', 'Traded_Value'
]

missing = [c for c in required_cols if c not in df.columns]
if missing:
    print(f"WARNING: Missing columns: {missing}")
    print("These features will be skipped. Model may underperform.")
```

**Step 2: Compute features**

```python
from ml.features import preprocess_features

# Compute 52W_Prox if not already present
if '52W_Prox' not in df.columns:
    df['52W_Prox'] = (df['CMP'] / df['High52W']) * 100

# Preprocess all features
features_df = preprocess_features(df)
print(f"Feature matrix: {features_df.shape}")
print(f"Columns: {list(features_df.columns)}")
```

**Step 3: Compute targets**

```python
from ml.targets import compute_forward_returns

# Need symbol, date, and Close to compute forward returns
# Build a merged dataframe with both features and raw price data
df_with_targets = df.copy()
df_with_targets = compute_forward_returns(df_with_targets, price_col='Close')

# Merge features with targets
ml_df = pd.concat([features_df.reset_index(drop=True),
                    df_with_targets[['symbol', 'date',
                                      'target_5d_pos', 'target_10d_pos', 'target_20d_pos',
                                      'target_20d_cont',
                                      'fwd_return_20d']].reset_index(drop=True)],
                   axis=1)

# Drop rows where target is NaN (last 20 days of data have no forward return)
initial_count = len(ml_df)
ml_df = ml_df.dropna(subset=['target_20d_pos'])
print(f"Dropped {initial_count - len(ml_df)} rows with no forward return label")
print(f"Final dataset: {len(ml_df)} rows, {len(features_df.columns)} features")
```

**Step 4: Save the ML-ready dataset**

```python
output_path = 'APOLLO_DATA_REPOSITORY/ml_ready.parquet'
ml_df.to_parquet(output_path, index=False)
print(f"Saved to {output_path}")

# Quick stats
print(f"\n=== Dataset Statistics ===")
print(f"Symbols: {ml_df['symbol'].nunique()}")
print(f"Date range: {ml_df['date'].min()} to {ml_df['date'].max()}")
print(f"Target 20d positive rate: {ml_df['target_20d_pos'].mean():.1%}")
print(f"Target 10d positive rate: {ml_df['target_10d_pos'].mean():.1%}")
print(f"Target 5d positive rate: {ml_df['target_5d_pos'].mean():.1%}")
```

### Phase 1 Deliverable

- [ ] `ml/features.py` — Feature preprocessing module
- [ ] `ml/targets.py` — Target computation module
- [ ] `ml/build_dataset.py` — Dataset construction script
- [ ] `APOLLO_DATA_REPOSITORY/ml_ready.parquet` — Clean, ML-ready dataset
- [ ] Dataset stats printed: row count, symbol count, date range, positive class rates

---

## 8. Phase 2: Logistic Regression Diagnostic

**Duration:** 1 day  
**Goal:** Answer the question — "Do my engine outputs contain any predictive signal for forward returns?"  
**This is a diagnostic phase, not a production model.**

### Step-by-Step

```python
# File: ml/train_logistic.py

import pandas as pd
import numpy as np
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import (roc_auc_score, classification_report,
                              precision_recall_curve, average_precision_score)
from ml.splits import purged_time_series_split
from ml.diagnostic import compute_feature_importance, plot_feature_importance

# Load ML-ready dataset
ml_df = pd.read_parquet('APOLLO_DATA_REPOSITORY/ml_ready.parquet')

# Identify feature columns (everything except symbol, date, and target columns)
target_cols = ['target_5d_pos', 'target_10d_pos', 'target_20d_pos', 'target_20d_cont', 'fwd_return_20d']
meta_cols = ['symbol', 'date']
feature_cols = [c for c in ml_df.columns if c not in target_cols + meta_cols]

print(f"Features ({len(feature_cols)}): {feature_cols}")

# ============================================================
# GAP 2 FIX: Purged time-series split
# ============================================================
train_df, test_df = purged_time_series_split(
    df=ml_df,
    date_col='date',
    symbol_col='symbol',
    test_start_date='2025-10-01',  # Adjust to your data range
    max_label_horizon=20,
    embargo_days=5
)

X_train = train_df[feature_cols]
y_train = train_df['target_20d_pos']
X_test = test_df[feature_cols]
y_test = test_df['target_20d_pos']

# ============================================================
# GAP 3 FIX: L1-regularized logistic regression
# ============================================================
model = LogisticRegression(
    max_iter=2000,
    penalty='l1',       # L1 regularization (Lasso) — handles multicollinearity
    solver='saga',      # Required solver for L1
    C=1.0,              # Inverse regularization strength (lower = more regularization)
    class_weight='balanced'  # Handles imbalanced classes
)

model.fit(X_train, y_train)

# ============================================================
# Evaluation
# ============================================================
y_pred_proba = model.predict_proba(X_test)[:, 1]
auc = roc_auc_score(y_test, y_pred_proba)
avg_precision = average_precision_score(y_test, y_pred_proba)

print(f"\n=== Logistic Regression Results (20d target) ===")
print(f"AUC-ROC:           {auc:.4f}")
print(f"Avg Precision (AP): {avg_precision:.4f}")
print(f"\nClassification Report (threshold=0.5):")
print(classification_report(y_test, model.predict(X_test), target_names=['No Gain', '5%+ Gain']))

# ============================================================
# GAP 3 FIX: Permutation importance (NOT raw coefficients)
# ============================================================
importance_df = compute_feature_importance(
    model, X_train, y_train, X_test, y_test, feature_cols
)

plot_feature_importance(importance_df, save_path='ml/output/feature_importance_logistic.png')

# ============================================================
# Baseline comparison: How does the model compare to Apollo alone?
# ============================================================
# If Apollo_Score >= 80 is the engine's ENTRY threshold:
apollo_entry_mask = test_df['Apollo_Score'] >= 80
if apollo_entry_mask.sum() > 0:
    apollo_only_rate = test_df.loc[apollo_entry_mask, 'target_20d_pos'].mean()
    print(f"\n=== Baseline Comparison ===")
    print(f"Apollo ENTRY (Score>=80) 20d win rate: {apollo_only_rate:.1%}")
    print(f"ML Model AUC-ROC: {auc:.4f}")
    
    # How does the model perform when BOTH Apollo says ENTRY and ML says high confidence?
    test_analysis = test_df.copy()
    test_analysis['ml_prob'] = y_pred_proba
    both_signal = test_analysis[
        (test_analysis['Apollo_Score'] >= 80) & 
        (test_analysis['ml_prob'] >= 0.6)
    ]
    if len(both_signal) > 0:
        dual_win_rate = both_signal['target_20d_pos'].mean()
        print(f"Apollo ENTRY + ML >= 60%: win rate = {dual_win_rate:.1%} ({len(both_signal)} stocks)")

# ============================================================
# Decision Gate: Does signal exist?
# ============================================================
if auc < 0.52:
    print("\n⚠️ AUC < 0.52 — essentially no predictive signal above random.")
    print("Recommendation: Do NOT proceed to XGBoost. The features don't contain")
    print("enough signal. Focus on improving engine logic instead.")
elif auc < 0.56:
    print("\n⚡ AUC 0.52-0.56 — weak signal. XGBoost may squeeze out marginal improvement.")
    print("Recommendation: Proceed to XGBoost with caution. Set realistic expectations.")
else:
    print(f"\n✅ AUC {auc:.4f} — meaningful signal. Proceed to XGBoost.")
    print("The engine outputs contain genuine predictive information.")
```

### Phase 2 Deliverable

- [ ] `ml/train_logistic.py` — Logistic regression training script with purged split
- [ ] `ml/splits.py` — Purged time-series split module (Gap 2 fix)
- [ ] `ml/diagnostic.py` — Permutation importance module (Gap 3 fix)
- [ ] `ml/output/feature_importance_logistic.png` — Feature importance chart
- [ ] AUC-ROC, average precision, and baseline comparison numbers printed
- [ ] **Go/No-Go decision** based on AUC threshold (>= 0.56 to proceed to XGBoost)

---

## 9. Phase 3: XGBoost Meta-Learner

**Duration:** 2–3 days  
**Goal:** Train production XGBoost model that outputs ML Confidence scores.  
**Prerequisite:** Phase 2 AUC >= 0.56

### Step-by-Step

```python
# File: ml/train_xgboost.py

import pandas as pd
import numpy as np
import xgboost as xgb
from sklearn.metrics import roc_auc_score, classification_report, average_precision_score
from sklearn.model_selection import ParameterGrid
from ml.splits import purged_time_series_split
from ml.diagnostic import compute_feature_importance, plot_feature_importance

# Load ML-ready dataset
ml_df = pd.read_parquet('APOLLO_DATA_REPOSITORY/ml_ready.parquet')

target_cols = ['target_5d_pos', 'target_10d_pos', 'target_20d_pos', 'target_20d_cont', 'fwd_return_20d']
meta_cols = ['symbol', 'date']
feature_cols = [c for c in ml_df.columns if c not in target_cols + meta_cols]

# Same purged split
train_df, test_df = purged_time_series_split(
    df=ml_df, date_col='date', symbol_col='symbol',
    test_start_date='2025-10-01', max_label_horizon=20, embargo_days=5
)

X_train = train_df[feature_cols]
y_train = train_df['target_20d_pos']
X_test = test_df[feature_cols]
y_test = test_df['target_20d_pos']

# ============================================================
# Hyperparameter search (light — not exhaustive)
# ============================================================
# Financial data is noisy. Keep the model simple to avoid overfitting.
# Deeper trees and more estimators = more overfitting risk.

param_grid = {
    'max_depth': [3, 4, 5],           # Shallow trees (3-5) prevent overfitting
    'n_estimators': [100, 200],       # 100-200 trees is enough for tabular
    'learning_rate': [0.05, 0.1],     # Lower learning rate = more robust
    'min_child_weight': [10, 20],     # Higher = more conservative (prevents leaf with few samples)
    'subsample': [0.8],               # Row subsampling reduces overfitting
    'colsample_bytree': [0.8],        # Feature subsampling reduces overfitting
    'reg_alpha': [0.1],               # L1 regularization on tree weights
    'reg_lambda': [1.0],              # L2 regularization on tree weights
    'scale_pos_weight': [             # Handle class imbalance
        (y_train == 0).sum() / (y_train == 1).sum()
    ],
    'eval_metric': ['auc'],
    'random_state': [42],
    'tree_method': ['hist'],          # Fast histogram-based splitting
}

best_auc = 0
best_model = None
best_params = None

print(f"Searching {len(list(ParameterGrid(param_grid)))} hyperparameter combinations...")

for i, params in enumerate(ParameterGrid(param_grid)):
    model = xgb.XGBClassifier(**params)
    model.fit(X_train, y_train, eval_set=[(X_test, y_test)], verbose=False)
    
    y_pred = model.predict_proba(X_test)[:, 1]
    auc = roc_auc_score(y_test, y_pred)
    
    if auc > best_auc:
        best_auc = auc
        best_model = model
        best_params = params
    
    if (i + 1) % 10 == 0:
        print(f"  [{i+1}/{len(list(ParameterGrid(param_grid)))}] Best AUC so far: {best_auc:.4f}")

print(f"\nBest AUC-ROC: {best_auc:.4f}")
print(f"Best params: {best_params}")

# ============================================================
# Final evaluation
# ============================================================
y_pred_proba = best_model.predict_proba(X_test)[:, 1]

print(f"\n=== XGBoost Results (20d target) ===")
print(f"AUC-ROC: {best_auc:.4f}")
print(f"Avg Precision: {average_precision_score(y_test, y_pred_proba):.4f}")
print(classification_report(y_test, best_model.predict(X_test),
                          target_names=['No Gain', '5%+ Gain']))

# Compare against logistic regression baseline
print(f"\n=== Model Comparison ===")
print(f"Logistic Regression AUC: [from Phase 2 output]")
print(f"XGBoost AUC:           {best_auc:.4f}")

if best_auc < 0.56:
    print("⚠️ XGBoost did not significantly outperform logistic regression.")
    print("The signal is weak. Consider using the logistic model as-is, or focus on engine improvement.")
else:
    print("✅ XGBoost shows meaningful improvement. Proceed to dashboard integration.")

# ============================================================
# Feature importance (XGBoost native — gain-based, handles multicollinearity)
# ============================================================
importance_df = pd.DataFrame({
    'feature': feature_cols,
    'importance': best_model.feature_importances_
}).sort_values('importance', ascending=False)

print("\n=== XGBoost Feature Importance (Gain) ===")
for _, row in importance_df.iterrows():
    bar = '#' * int(row['importance'] * 200)
    print(f"  {row['feature']:20s}  {row['importance']:.4f}  {bar}")

plot_feature_importance(
    pd.DataFrame({
        'feature': feature_cols,
        'importance_mean': best_model.feature_importances_,
        'importance_std': [0] * len(feature_cols),  # XGBoost doesn't give std
    }),
    save_path='ml/output/feature_importance_xgboost.png'
)

# ============================================================
# Save the trained model
# ============================================================
import pickle

model_path = 'ml/models/xgboost_20d_v1.pkl'
with open(model_path, 'wb') as f:
    pickle.dump({
        'model': best_model,
        'params': best_params,
        'feature_cols': feature_cols,
        'target': 'target_20d_pos',
        'threshold': 0.05,  # 5% gain threshold
        'train_auc': best_auc,
        'train_date': pd.Timestamp.now().isoformat(),
    }, f)

print(f"\nModel saved to {model_path}")
```

### Phase 3 Deliverable

- [ ] `ml/train_xgboost.py` — XGBoost training script with hyperparameter search
- [ ] `ml/models/xgboost_20d_v1.pkl` — Serialized trained model
- [ ] `ml/output/feature_importance_xgboost.png` — Feature importance chart
- [ ] AUC-ROC comparison: Logistic Regression vs. XGBoost
- [ ] Final go/no-go decision for dashboard integration

---

## 10. Phase 4: Dashboard Integration

**Duration:** 2–3 days  
**Goal:** ML Confidence column visible across all discovery tabs.

### Step 1: ML Inference Service

```python
# File: backend/services/ml_service.py

import pickle
import numpy as np
import pandas as pd
from ml.features import preprocess_features

# Load model at service startup (not on every request)
_model_artifact = None

def load_model(model_path: str = 'ml/models/xgboost_20d_v1.pkl'):
    global _model_artifact
    with open(model_path, 'rb') as f:
        _model_artifact = pickle.load(f)
    print(f"ML model loaded: {_model_artifact['target']}, AUC={_model_artifact['train_auc']:.4f}")

def predict_ml_confidence(stock_data: list[dict]) -> list[dict]:
    """
    Add ml_confidence to each stock's data.
    
    This runs during the daily data update cycle, after engine outputs are computed.
    It operates on the in-memory cached data — no external API calls.
    """
    if _model_artifact is None:
        load_model()
    
    model = _model_artifact['model']
    feature_cols = _model_artifact['feature_cols']
    
    # Convert to DataFrame for preprocessing
    df = pd.DataFrame(stock_data)
    
    # Preprocess features (same pipeline as training)
    features = preprocess_features(df)
    
    # Ensure we have all required features
    missing = [c for c in feature_cols if c not in features.columns]
    if missing:
        print(f"Warning: Missing ML features: {missing}. Filling with 0.")
        for c in missing:
            features[c] = 0
    
    X = features[feature_cols]
    
    # Predict probability of positive return
    probabilities = model.predict_proba(X)[:, 1]
    
    # Add to stock data
    for i, stock in enumerate(stock_data):
        stock['ml_confidence'] = round(float(probabilities[i]), 3)
    
    return stock_data
```

### Step 2: API Endpoint

```python
# File: backend/api/routes/ml.py (new file)

@router.get('/ml/info')
async def get_ml_info():
    """Returns model metadata: target, AUC, feature count, train date."""
    if _model_artifact is None:
        return {'status': 'MODEL_NOT_LOADED'}
    return {
        'status': 'LOADED',
        'target': _model_artifact['target'],
        'train_auc': _model_artifact['train_auc'],
        'feature_count': len(_model_artifact['feature_cols']),
        'trained_at': _model_artifact['train_date'],
        'threshold': _model_artifact['threshold'],
    }
```

### Step 3: Integrate into Daily Update Pipeline

```python
# In bridge_daily_update.py, AFTER all engine outputs are computed:

from services.ml_service import predict_ml_confidence

# Add ML confidence to cached data
cached_data = predict_ml_confidence(cached_data)

# ml_confidence is now available alongside Apollo_Score, LS_Score, etc.
# No schema changes needed — it's just another field on each stock object.
```

### Step 4: Frontend — ML Confidence Column

Add `ml_confidence` as a column in the following tabs:

| Tab | Column Position | Sortable? | Filterable? |
|---|---|---|---|
| Screener | After FQS column | Yes | Yes (slider: 0–100%) |
| IPO Tab | After Quality column | Yes | Yes |
| Blue Sky Tab | After EL Status column | Yes | Yes |
| 52WH Tab | After FQS column | Yes | Yes |
| Scanner | Available as filter preset | Yes | Yes ("ML Confidence > 65%") |

```tsx
// ML Confidence column component (reusable across tabs)

interface MLConfidenceCellProps {
  value: number;  // 0.0 to 1.0
}

// Visual treatment:
//   >= 0.70  →  Green, bold, ▲▲▲  (High conviction)
//   0.55-0.69 →  Blue, ▲▲        (Moderate conviction)
//   0.45-0.54 →  Gray, ▲          (Neutral — model unsure)
//   < 0.45  →  Red, ▼           (Low conviction — model expects underperformance)

// Display format: "73%" with color and arrow indicators
// Tooltip: "ML Confidence: 73% probability of 5%+ gain in 20 days"
```

### Step 5: ML-Based Filter Presets in Scanner

```
New Scanner Presets:
  "ML High Conviction"  → ml_confidence >= 0.70 AND Apollo_Score >= 80
  "ML Contrarian"        → ml_confidence >= 0.65 AND Apollo_Score < 80 (model likes it, engine doesn't)
  "ML + EL1 Confluence"  → ml_confidence >= 0.65 AND ELStatus = 'EL1'
```

### Phase 4 Deliverable

- [ ] `backend/services/ml_service.py` — ML inference service
- [ ] `backend/api/routes/ml.py` — ML API endpoint
- [ ] `bridge_daily_update.py` — ML confidence integrated into daily pipeline
- [ ] ML Confidence column added to Screener, IPO, Blue Sky, 52WH tabs
- [ ] ML-based filter presets in Scanner
- [ ] Visual treatment: color-coded confidence levels with tooltips

---

## 11. Phase 5: Live Inference Pipeline

**Duration:** 1–2 days  
**Goal:** ML Confidence updates in real-time (or near real-time) when Kite Connect data refreshes.

### Kite Connect Integration

With Kite Connect providing live data, the ML inference pipeline becomes:

```
Kite Connect API
    │
    ├─→ Live price data → existing pipeline → Apollo/LS engine outputs
    │                                                           │
    └─→ Volume data (new!) ──────────────────────────────────────┘
                                                                │
                                                    ml_service.predict_ml_confidence()
                                                                │
                                                    ml_confidence updated in cache
                                                                │
                                                    Frontend polls/refreshes
```

### Model Retraining Schedule

| Trigger | Action |
|---|---|
| Weekly (Sunday) | Retrain model on latest data (includes past week's results as new training data) |
| Monthly | Full hyperparameter re-search, evaluate if AUC has degraded |
| AUC drops > 0.03 from baseline | Alert: model may be stale, investigate regime change |
| New engine version deployed | Retrain from scratch — engine outputs have changed |

### Retraining Script

```python
# File: ml/retrain.py

"""
Weekly retraining script.
Run via cron: 0 6 * * 0 (every Sunday at 6 AM)

Steps:
1. Rebuild ml_ready.parquet from latest backtest data
2. Purged time-series split (use last 2 months as test)
3. Train XGBoost with saved best params
4. Evaluate AUC
5. If AUC within 0.03 of baseline → save new model
6. If AUC dropped > 0.03 → alert, keep old model
"""

import subprocess
import sys

# Step 1: Rebuild dataset
subprocess.run([sys.executable, 'ml/build_dataset.py'], check=True)

# Step 2: Retrain (skip hyperparameter search — use saved params)
# Load saved params from last training
import pickle
with open('ml/models/xgboost_20d_v1.pkl', 'rb') as f:
    saved_params = pickle.load(f)['params']

# Use saved params, just retrain on new data
# (Full hyperparameter search script: ml/train_xgboost.py)
```

### Phase 5 Deliverable

- [ ] Kite Connect volume data flowing into ML feature pipeline
- [ ] ML Confidence updates with each data refresh cycle
- [ ] `ml/retrain.py` — Weekly retraining script
- [ ] Cron job configured for Sunday retraining
- [ ] AUC monitoring with alert on degradation

---

## 12. Acceptance Criteria

### Model Quality

| Metric | Minimum Threshold | Good Target | Excellent |
|---|---|---|---|
| AUC-ROC (20d) | >= 0.55 | >= 0.58 | >= 0.62 |
| Avg Precision | >= 0.30 | >= 0.35 | >= 0.40 |
| Precision @ 50% recall | >= 0.40 | >= 0.45 | >= 0.50 |

**Important context:** AUC of 0.55-0.60 is typical for stock return prediction with tabular features. This is not image classification where 0.95 is expected. Financial data has inherently low signal-to-noise ratio. Even a 3-5% edge over random compounds significantly over many trades.

### Integration Quality

- [ ] ML Confidence column visible in Screener, IPO, Blue Sky, 52WH, Scanner
- [ ] Column is sortable and filterable
- [ ] Color coding: green (>=70%), blue (55-69%), gray (45-54%), red (<45%)
- [ ] Tooltip shows: "ML Confidence: X% probability of 5%+ gain in 20 days"
- [ ] Model metadata endpoint returns target, AUC, train date
- [ ] No model loaded gracefully handled (column hidden or shows "N/A")

### Data Integrity

- [ ] Purged time-series split validated with zero overlap
- [ ] No look-ahead bias in feature computation
- [ ] Same preprocessing pipeline used for training AND inference
- [ ] Model retraining preserves feature column order

---

## 13. Risk Register

| # | Risk | Probability | Impact | Mitigation |
|---|---|---|---|---|
| R1 | No predictive signal (AUC < 0.52) | Medium | High | Phase 2 diagnostic catches this early. Cost: 1 day. If no signal, skip XGBoost and focus on engine improvement. |
| R2 | Model overfits to bull market period | Medium | Medium | Purged split + embargo prevents leakage. Monitor AUC degradation monthly. Retrain weekly to adapt to regime changes. |
| R3 | Feature set changes after engine update | Low | High | Retrain model from scratch whenever Apollo or LayerSignal engine is updated. Version the model files. |
| R4 | Class imbalance (few stocks gain 5%+) | Medium | Low | `class_weight='balanced'` in logistic regression, `scale_pos_weight` in XGBoost. Verified in Phase 2. |
| R5 | Model staleness over time | High | Medium | Weekly retraining via cron. AUC monitoring with alert on >0.03 drop. |
| R6 | Users over-rely on ML score | Low | Medium | Tooltip and docs clarify: "This is a probability, not a guarantee. Use alongside engine signals." |
| R7 | Kite Connect data gaps (missing days) | Low | Low | Preprocessing handles NaN with fillna(0). Missing days skipped in training. |
| R8 | 150K rows insufficient for XGBoost | Low | Low | XGBoost works well on 50K+ rows with proper regularization. Our shallow trees (depth 3-5) further reduce overfitting risk. |

---

## 14. Milestone Timeline

### Day-by-Day Plan

| Day | Phase | Task | Deliverable |
|---|---|---|---|
| **1** | Pre | Verify Gate 1 (parquet integrity) and Gate 2 (volume data) | Gate conditions documented, proceed/blocked decision |
| **1** | P1 | Create `ml/features.py` and `ml/targets.py` | Feature and target modules |
| **2** | P1 | Create `ml/build_dataset.py`, build `ml_ready.parquet` | Clean ML-ready dataset with stats |
| **2** | P1 | Create `ml/splits.py` (Gap 2 fix) | Purged time-series split module |
| **3** | P2 | Create `ml/diagnostic.py` (Gap 3 fix), create `ml/train_logistic.py` | Diagnostic modules |
| **3** | P2 | Run logistic regression, evaluate AUC, permutation importance | **GO/NO-GO decision point** |
| **4** | P3 | Create `ml/train_xgboost.py`, run hyperparameter search | Trained XGBoost model |
| **4-5** | P3 | Evaluate, compare against logistic baseline, save model | `xgboost_20d_v1.pkl` + importance chart |
| **6** | P4 | Create `backend/services/ml_service.py` | ML inference service |
| **6** | P4 | Create `backend/api/routes/ml.py` | ML API endpoint |
| **7** | P4 | Integrate into `bridge_daily_update.py` | ML confidence in daily pipeline |
| **7-8** | P4 | Add ML Confidence column to all 5 tabs | Column visible across dashboard |
| **8** | P4 | Add ML-based filter presets in Scanner | 3 new scanner presets |
| **9** | P5 | Wire Kite Connect volume data into ML features | Live volume in feature pipeline |
| **9** | P5 | Create `ml/retrain.py`, set up weekly cron | Automated retraining |
| **10** | P5 | End-to-end testing, AUC monitoring setup | Full ML meta-learner live |

### Total: ~10 working days

---

## Appendix A: File Structure

```
ml/
├── features.py              # Feature preprocessing (P1)
├── targets.py               # Forward return target computation (P1)
├── splits.py                # Purged time-series split — GAP 2 FIX (P2)
├── diagnostic.py            # Permutation importance — GAP 3 FIX (P2)
├── build_dataset.py         # Dataset construction script (P1)
├── train_logistic.py        # Logistic regression diagnostic (P2)
├── train_xgboost.py         # XGBoost training with hyperparameter search (P3)
├── retrain.py               # Weekly retraining script (P5)
├── models/
│   └── xgboost_20d_v1.pkl   # Serialized trained model (P3)
└── output/
    ├── feature_importance_logistic.png   # LR feature chart (P2)
    └── feature_importance_xgboost.png    # XGB feature chart (P3)

backend/services/
└── ml_service.py            # ML inference service (P4)

backend/api/routes/
└── ml.py                    # ML API endpoints (P4)
```

## Appendix B: Feature Importance Interpretation Guide

When you see the permutation importance output, here's how to act on it:

| Finding | Action |
|---|---|
| Apollo_Score is #1 or #2 | Confirms engine value. ML is leveraging the same signal. Good. |
| Gate_X has near-zero importance | Consider simplifying — that gate may not add predictive value. Investigate before removing. |
| Exit_Pressure has high NEGATIVE importance | Confirms your existing risk filter. High EP genuinely predicts poor outcomes. |
| RSI21/36/56 all have low importance | Raw indicators don't add much beyond what Apollo already captures. Expected. |
| Bucket_L3 has high positive importance | L3 stocks (large-cap quality) at highs have better follow-through. Aligns with engine logic. |
| Traded_Value (log) has high importance | Volume matters more than expected. Good that Kite Connect is coming. |
| All features have near-zero importance (AUC < 0.52) | No signal exists in current features. Do NOT proceed to XGBoost. |
