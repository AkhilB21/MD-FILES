# Apollo Convergence & Profiling Architecture

> End-to-end implementation blueprint. Version 1.0 | Aug 2026
> 
> Synthesizes: Cross-Report Pipeline, Signal Profiling Engine, Profile Analytics, Stock Personality Profiler, Apollo Integration, and Intraday Engine (future).

---

## Table of Contents

1. [Project Structure](#1-project-structure)
2. [Phase 0: Data Fixes (Day 1)](#2-phase-0-data-fixes-day-1)
3. [Phase 1: Cross-Report Pipeline Fixes (Day 2-3)](#3-phase-1-cross-report-pipeline-fixes-day-2-3)
4. [Phase 2: Signal Profiling Engine (Day 4-7)](#4-phase-2-signal-profiling-engine-day-4-7)
5. [Phase 3: Profile Analytics (Day 30+)](#5-phase-3-profile-analytics-day-30)
6. [Phase 4: Apollo Backtest Integration (Day 60+)](#6-phase-4-apollo-backtest-integration-day-60)
7. [Phase 5: Stock Personality Profiler (Parallel, Week 3+)](#7-phase-5-stock-personality-profiler-parallel-week-3)
8. [Phase 6: Intraday Engine (Future)](#8-phase-6-intraday-engine-future)
9. [Implementation Timeline](#9-implementation-timeline)
10. [Key Design Principles](#10-key-design-principles)

---

## 1. Project Structure

```
apollo_core/
├── cross_report_pipeline.py      # [EXISTS - Phase 1] Main pipeline
├── report_parser.py             # [EXISTS] CSV parsing (refactor per Phase 1)
├── convergence_engine.py        # [EXISTS] Convergence scoring (fix per Phase 1)
├── conviction_pyramid.py        # [EXISTS] Watchlist builder (fix per Phase 1)
├── signal_profiler.py           # [NEW - Phase 2] Event/outcome recording
├── outcome_backfiller.py        # [NEW - Phase 2] Daily backfill script
├── profile_analytics.py         # [NEW - Phase 3] Aggregate analysis
├── stock_personality.py          # [NEW - Phase 5] Per-symbol behavioral fingerprint
├── sector_engine.py             # [NEW - Phase 5] Sector rotation tracking
└── schemas.py                    # [EXTEND] All dataclasses in one place

apollo_data/
├── profiling.db                  # [NEW - Phase 2] SQLite database
├── parquet/                       # [EXISTS] OHLCV parquets
└── daily_reports/                # [EXISTS] Chartink CSVs

outputs/
├── watchlist_core.csv           # L2+L3 only (actionable)
├── watchlist_extended.csv       # L1+L2+L3 (full)
└── profile_reports/              # Weekly analytics output
```

---

## 2. Phase 0: Data Fixes (Day 1)

### 2.1 Stop Dropping Delivery Data

**File**: `data_repo/repo.py` (or equivalent normalizer)

**Problem**: `dlv_qty` is in raw NSE data but dropped during sync.

**Fix**:
```python
# BEFORE
COLUMNS_TO_KEEP = ['Date', 'Open', 'High', 'Low', 'Close', 'Volume']

# AFTER
COLUMNS_TO_KEEP = ['Date', 'Open', 'High', 'Low', 'Close', 'Volume', 'dlv_qty']

# Derive delivery_pct at load time
df['delivery_pct'] = (df['dlv_qty'] / df['Volume']) * 100
# Where volume is 0, delivery_pct = NaN (already traded, no issue)
```

**Impact**: Unlocks delivery-based accumulation/distribution scoring. High delivery on up-days = real buying, not intraday churn. This is a NSE-specific signal with genuine edge.

### 2.2 Add NIFTY 50 Baseline Data

**Requirement**: Alpha-relative outcomes need a market benchmark.

**Approach**: Download or compute NIFTY 50 daily close series. Store as `parquet/NIFTY50.parquet` with columns `[Date, Open, High, Low, Close, Volume]`.

**Source options** (pick one):
- Yahoo Finance: `^NSEI`
- Existing Apollo data if NIFTY 50 ETF (NIFTYBEES) is in your universe
- Chartink daily screener (it includes index constituents)

**Usage**: 
```python
def load_nifty_returns(date: date, horizon_days: int) -> float:
    """Return NIFTY 50 return over horizon_days ending on date."""
    nifty = pd.read_parquet('parquet/NIFTY50.parquet')
    # Find the close on date and date - horizon_days
    idx = nifty.index.get_loc(date)
    past_close = nifty['Close'].iloc[idx - horizon_days]
    current_close = nifty['Close'].iloc[idx]
    return (current_close - past_close) / past_close * 100
```

### 2.3 Add India VIX Data (Optional but Recommended)

**Why**: Market regime context. A signal in low-VIX vs high-VIX has different expected behavior.

**Source**: NSE publishes daily VIX close. Download as `parquet/INDIA_VIX.parquet`.

**Usage**: `vix_pctile_100d = VIX_today's percentile rank vs trailing 100 days`.

---

## 3. Phase 1: Cross-Report Pipeline Fixes (Day 2-3)

### 3.1 Fix Within-Tier Scoring

**File**: `conviction_pyramid.py`

**Problem**: All L3 stocks score exactly 85. No internal differentiation.

**Fix**: Add within-tier tiebreakers.

```python
def compute_priority_score(
    conviction_level: ConvictionLevel,
    convergence: ConvergenceProfile,
    is_hot_sector: bool,
    delivery_pct: float | None = None
) -> float:
    """
    Revised scoring with within-tier differentiation.
    Range: 0-100 (approx)
    """
    # Base from conviction level
    base = conviction_level.value * 20  # 20/40/60
    
    # Convergence bonus (only for L2+ where convergence is computed)
    if conviction_level in (ConvictionLevel.L2_MOMENTUM_RSI, ConvictionLevel.L3_CONVERGENT):
        if convergence.regime == ConvergenceRegime.MATURE_CONVERGENT:
            conv_bonus = 25
        elif convergence.regime == ConvergenceRegime.DEVELOPING_CONVERGENT:
            conv_bonus = 15
        elif convergence.regime == ConvergenceRegime.DIVERGENT_BULLISH:
            conv_bonus = 0  # No bonus for divergent
        else:
            conv_bonus = 0
    else:
        conv_bonus = 0
    
    # Fade penalty (overrides convergence bonus)
    fade_penalty = 20 if convergence.regime == ConvergenceRegime.DIVERGENT_BEARISH else 0
    
    # --- WITHIN-TIER TIEBREAKERS (NEW) ---
    
    # Spread tiebreaker: tighter convergence = higher score
    # Normalize: spread 0 -> +10, spread 5 -> +0
    spread_bonus = max(0, (5.0 - convergence.rsi21_spread) / 5.0 * 10) \
        if conviction_level.value >= 2 else 0
    
    # RSI avg tiebreaker: higher avg = stronger trend
    # Normalize: avg 65 -> +0, avg 90 -> +5
    rsi_bonus = max(0, (convergence.rsi21_avg - 65) / 25 * 5) \
        if conviction_level.value >= 2 else 0
    
    # D-W gradient tiebreaker: slight preference for aligned (small gradient)
    # Penalize large negative gradient (daily fading vs weekly)
    grad_penalty = max(0, (-10 - convergence.daily_weekly_gradient) / 10 * 3) \
        if convergence.daily_weekly_gradient < -10 else 0
    
    # Sector bonus
    sector_bonus = 10 if is_hot_sector else 0
    
    # Delivery bonus (if available, for L3 stocks)
    delivery_bonus = 0
    if delivery_pct is not None and conviction_level.value >= 3 and delivery_pct > 60:
        delivery_bonus = 3
    
    score = (base + conv_bonus + spread_bonus + rsi_bonus 
             + sector_bonus + delivery_bonus - fade_penalty - grad_penalty)
    return round(max(0, min(100, score)), 1)
```

**Expected result**: L3 scores now range ~70-100 instead of flat 85. L2 scores range ~50-75.

### 3.2 Add Missing Columns to Watchlist Output

**File**: `cross_report_pipeline.py` (output section)

**Problem**: Architecture specified columns that aren't in the CSV.

**Fix**: Add all columns with NULL placeholders for Phase 2 data.

```python
# Complete output schema (add ALL columns, even if empty)
WATCHLIST_COLUMNS = [
    # Core (exists)
    'Symbol', 'Conviction Level', 'Priority Score', 
    'Momentum Regime', 'Convergence Regime',
    'RSI21 Spread (D/W)', 'RSI21 Avg', 'Daily-Weekly Gradient',
    'Signals',
    
    # NEW: Within-tier differentiators
    'Spread Bonus', 'RSI Bonus', 'Gradient Penalty',  # for transparency
    
    # NEW: Context columns (NULL in Phase 1, populated in Phase 2)
    'Sector',                          # from screener_stocks
    'Market Cap Bucket',               # LARGE/MID/SMALL/UNKNOWN
    'Is Hot Sector',                   # bool
    'Delivery Pct',                    # from dlv_qty fix
    '52W High Distance Pct',           # (52W_High - Last_Price) / 52W_High
    'ROE', 'D/E', 'P/E',              # from screener_stocks
    
    # NEW: Signal persistence (Phase 2)
    'Consecutive Watchlist Days',      # how many days in a row on list
    'Previous Conviction Level',       # yesterday's level
    'Conviction Change',               # UPGRADED/DOWNGRADED/NEW/UNCHANGED
    
    # NEW: Market context
    'Nifty 21D Return Pct',            # market regime context
    'VIX Percentile 100D',             # NULL until VIX data added
]
```

**Critical rule**: Schema stability. Downstream consumers (profiling engine, future intraday engine) depend on these columns existing. Never remove a column. Add new ones at the end.

### 3.3 Split Output into Two Files

**Problem**: 1259 stocks (70% L1) is too noisy for human review.

**Fix**: Generate two output files.

```python
# In cross_report_pipeline.py output section
watchlist_full = final_df  # all 1259
watchlist_core = final_df[final_df['Conviction Level'].isin(['L2_MOMENTUM_RSI', 'L3_CONVERGENT'])]

watchlist_full.to_csv('outputs/watchlist_extended.csv', index=False)
watchlist_core.to_csv('outputs/watchlist_core.csv', index=False)
```

- `watchlist_core.csv`: ~370 stocks. This is your daily action list.
- `watchlist_extended.csv`: Full universe. For analytics and profiling ingestion.

### 3.4 Handle DIVERGENT_BEARISH Safely Without Intraday Data

**File**: `convergence_engine.py`

**Problem**: When `id_daily_gradient` defaults to 0.0 (no intraday data), DIVERGENT_BEARISH cannot be reliably detected.

**Fix**: Guard the regime classification.

```python
def classify_regime(
    rsi21_spread: float,
    rsi21_avg: float,
    id_daily_gradient: float,
    daily_weekly_gradient: float,
    daily_rsi21: float,
    id_rsi21: float | None = None,  # None when intraday unavailable
    has_intraday_data: bool = False
) -> ConvergenceRegime:
    
    if not has_intraday_data:
        # Phase 1: Only classify using D/W gradient and spread
        if rsi21_spread < 3.0 and rsi21_avg > 70:
            return ConvergenceRegime.MATURE_CONVERGENT
        elif rsi21_spread < 5.0 and rsi21_avg > 65:
            return ConvergenceRegime.DEVELOPING_CONVERGENT
        elif rsi21_spread > 7.0 and rsi21_avg > 70:
            return ConvergenceRegime.DIVERGENT_BULLISH
        else:
            return ConvergenceRegime.NEUTRAL
    
    # Phase 2+: Full classification with intraday gradient
    if daily_rsi21 > 75 and id_rsi21 is not None and (daily_rsi21 - id_rsi21) > 15:
        return ConvergenceRegime.DIVERGENT_BEARISH
    # ... rest of full classification
```

---

## 4. Phase 2: Signal Profiling Engine (Day 4-7)

### 4.1 Database Schema

**File**: `signal_profiler.py` (creates and writes)

Two tables with strict no-lookahead separation.

```sql
-- Table 1: Signal Events (written at watchlist generation time)
CREATE TABLE IF NOT EXISTS signal_events (
    symbol           TEXT NOT NULL,
    entry_date       TEXT NOT NULL,          -- YYYY-MM-DD
    
    -- Signal state (known at entry time)
    conviction_level     TEXT,                -- L1/L2/L3
    convergence_regime   TEXT,                -- MATURE/DEVELOPING/DIVERGENT_BULLISH/NEUTRAL
    momentum_regime      TEXT,                -- SUSTAINED/ACCELERATING/DECELERATING/BOUNCE/WEAK
    priority_score       REAL,
    rsi21_avg            REAL,
    rsi21_spread         REAL,
    d_w_gradient         REAL,
    signals_matched      TEXT,                -- pipe-delimited
    
    -- Context
    sector               TEXT,
    market_cap_bucket    TEXT,                -- LARGE/MID/SMALL/UNKNOWN
    delivery_pct         REAL,
    dist_52w_high_pct    REAL,
    roe                  REAL,
    de_ratio             REAL,
    pe_ratio             REAL,
    is_hot_sector        INTEGER,             -- 0/1
    
    -- Signal persistence
    consecutive_days     INTEGER,             -- days in a row on watchlist
    prev_conviction      TEXT,                -- previous day's level
    conviction_change    TEXT,                -- UPGRADED/DOWNGRADED/NEW/UNCHANGED
    
    -- Market context
    nifty_21d_return      REAL,
    vix_pctile_100d      REAL,
    
    -- Metadata
    engine_version       TEXT,                -- git hash or version tag
    created_at           TEXT DEFAULT (datetime('now')),
    
    PRIMARY KEY (symbol, entry_date)
);

-- Table 2: Signal Outcomes (backfilled on future dates)
CREATE TABLE IF NOT EXISTS signal_outcomes (
    symbol           TEXT NOT NULL,
    entry_date       TEXT NOT NULL,
    horizon_days     INTEGER NOT NULL,        -- 1, 3, 5, 10, 21
    
    -- Outcomes (computed at entry_date + horizon_days)
    ret_pct          REAL,                    -- raw return %
    alpha_pct        REAL,                    -- ret_pct - nifty_return %
    mfe_pct          REAL,                    -- max favorable excursion %
    mae_pct          REAL,                    -- max adverse excursion %
    gap_up_pct       REAL,                    -- next-day opening gap %
    realized_vol     REAL,                    -- std of daily returns in window
    days_above_entry INTEGER,                 -- days closing above entry price
    days_below_5pct  INTEGER,                 -- days dropping >5% from entry
    
    -- Metadata
    recorded_at      TEXT DEFAULT (datetime('now')),
    
    PRIMARY KEY (symbol, entry_date, horizon_days),
    FOREIGN KEY (symbol, entry_date) REFERENCES signal_events(symbol, entry_date)
);

-- Indexes for common queries
CREATE INDEX IF NOT EXISTS idx_events_conviction ON signal_events(conviction_level);
CREATE INDEX IF NOT EXISTS idx_events_regime ON signal_events(convergence_regime);
CREATE INDEX IF NOT EXISTS idx_events_date ON signal_events(entry_date);
CREATE INDEX IF NOT EXISTS idx_outcomes_horizon ON signal_outcomes(horizon_days);
CREATE INDEX IF NOT EXISTS idx_outcomes_date ON signal_outcomes(entry_date);
```

### 4.2 Signal Event Writer

**File**: `signal_profiler.py`

**When**: Runs immediately after watchlist CSV generation (same script, appended step).

```python
def record_signal_events(watchlist_df: pd.DataFrame, date: str, db_path: str):
    """
    Write today's watchlist entries into signal_events table.
    One INSERT per row. Idempotent: uses INSERT OR REPLACE.
    """
    conn = sqlite3.connect(db_path)
    
    # Load yesterday's watchlist for persistence tracking
    yesterday_events = load_yesterday_events(conn, date)
    
    for _, row in watchlist_df.iterrows():
        symbol = row['Symbol']
        
        # Compute signal persistence
        prev = yesterday_events.get(symbol)
        if prev is None:
            consecutive_days = 1
            prev_conviction = None
            conviction_change = 'NEW'
        else:
            consecutive_days = prev['consecutive_days'] + 1
            prev_conviction = prev['conviction_level']
            if row['Conviction Level'] > prev_conviction:
                conviction_change = 'UPGRADED'
            elif row['Conviction Level'] < prev_conviction:
                conviction_change = 'DOWNGRADED'
            else:
                conviction_change = 'UNCHANGED'
        
        conn.execute('''
            INSERT OR REPLACE INTO signal_events (
                symbol, entry_date,
                conviction_level, convergence_regime, momentum_regime,
                priority_score, rsi21_avg, rsi21_spread, d_w_gradient,
                signals_matched, sector, market_cap_bucket, delivery_pct,
                dist_52w_high_pct, roe, de_ratio, pe_ratio,
                is_hot_sector, consecutive_days, prev_conviction, conviction_change,
                nifty_21d_return, vix_pctile_100d, engine_version
            ) VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?)
        ''', (
            symbol, date,
            row['Conviction Level'], row['Convergence Regime'], row['Momentum Regime'],
            row['Priority Score'], row['RSI21 Avg'], row['RSI21 Spread (D/W)'],
            row['Daily-Weekly Gradient'], row['Signals'],
            row.get('Sector'), row.get('Market Cap Bucket'),
            row.get('Delivery Pct'), row.get('52W High Distance Pct'),
            row.get('ROE'), row.get('D/E'), row.get('P/E'),
            int(row.get('Is Hot Sector', False)),
            consecutive_days, prev_conviction, conviction_change,
            row.get('Nifty 21D Return Pct'), row.get('VIX Percentile 100D'),
            get_engine_version()
        ))
    
    conn.commit()
    conn.close()
```

### 4.3 Outcome Backfiller

**File**: `outcome_backfiller.py`

**When**: Runs daily after market close, BEFORE watchlist generation.

**What it does**: For all signal_events where `entry_date + horizon_days == today`, compute and INSERT the outcome.

```python
HORIZONS = [1, 3, 5, 10, 21]

def run_daily_backfill(db_path: str, parquet_dir: str, nifty_path: str):
    """
    For every signal event whose outcome is now determinable,
    compute forward returns and INSERT into signal_outcomes.
    """
    conn = sqlite3.connect(db_path)
    today = get_last_trading_date()  # last date in parquet data
    
    # Find all events that need outcomes computed today
    for horizon in HORIZONS:
        target_date = offset_trading_date(today, -horizon)  # entry_date = today - horizon
        
        events = conn.execute('''
            SELECT symbol, entry_date 
            FROM signal_events 
            WHERE entry_date = ?
            AND symbol NOT IN (
                SELECT symbol FROM signal_outcomes 
                WHERE entry_date = ? AND horizon_days = ?
            )
        ''', (target_date, target_date, horizon)).fetchall()
        
        nifty_ret = compute_nifty_return(nifty_path, target_date, horizon)
        
        for symbol, entry_date in events:
            outcome = compute_outcome(
                symbol, entry_date, horizon, 
                parquet_dir, nifty_ret
            )
            
            conn.execute('''
                INSERT OR REPLACE INTO signal_outcomes (
                    symbol, entry_date, horizon_days,
                    ret_pct, alpha_pct, mfe_pct, mae_pct,
                    gap_up_pct, realized_vol, days_above_entry, days_below_5pct
                ) VALUES (?,?,?,?,?,?,?,?,?,?,?)
            ''', (
                symbol, entry_date, horizon,
                outcome.ret_pct, outcome.alpha_pct, 
                outcome.mfe_pct, outcome.mae_pct,
                outcome.gap_up_pct, outcome.realized_vol,
                outcome.days_above_entry, outcome.days_below_5pct
            ))
    
    conn.commit()
    conn.close()


def compute_outcome(
    symbol: str, entry_date: str, horizon: int,
    parquet_dir: str, nifty_ret: float
) -> OutcomeResult:
    """
    Load parquet, find entry_date, compute forward metrics.
    Strictly uses only data from entry_date to entry_date+horizon.
    No look-ahead.
    """
    df = pd.read_parquet(f'{parquet_dir}/{symbol}.parquet')
    entry_idx = df.index.get_loc(entry_date)
    
    entry_close = df['Close'].iloc[entry_idx]
    end_idx = min(entry_idx + horizon, len(df) - 1)
    end_close = df['Close'].iloc[end_idx]
    
    # Raw return
    ret_pct = (end_close - entry_close) / entry_close * 100
    
    # Alpha
    alpha_pct = ret_pct - nifty_ret
    
    # MFE/MAE over the window
    window = df.iloc[entry_idx:end_idx + 1]
    mfe_pct = ((window['High'].max() - entry_close) / entry_close) * 100
    mae_pct = ((window['Low'].min() - entry_close) / entry_close) * 100
    
    # Gap up next day
    if end_idx + 1 < len(df):
        next_open = df['Open'].iloc[entry_idx + 1]
        gap_up_pct = (next_open - entry_close) / entry_close * 100
    else:
        gap_up_pct = None
    
    # Realized volatility
    daily_returns = df['Close'].iloc[entry_idx + 1:end_idx + 1].pct_change().dropna()
    realized_vol = daily_returns.std() * 100 if len(daily_returns) > 1 else None
    
    # Days above/below entry
    closes = df['Close'].iloc[entry_idx + 1:end_idx + 1]
    days_above_entry = int((closes > entry_close).sum())
    days_below_5pct = int((closes < entry_close * 0.95).sum())
    
    return OutcomeResult(
        ret_pct=ret_pct, alpha_pct=alpha_pct,
        mfe_pct=mfe_pct, mae_pct=mae_pct,
        gap_up_pct=gap_up_pct, realized_vol=realized_vol,
        days_above_entry=days_above_entry, days_below_5pct=days_below_5pct
    )
```

### 4.4 Daily Runner Script

**File**: `daily_runner.sh` (or Python orchestrator)

**Execution order** (runs after market close, after parquet data is updated):

```bash
# Step 1: Backfill outcomes for events that have reached their horizon
python outcome_backfiller.py

# Step 2: Generate today's watchlist (existing pipeline)
python cross_report_pipeline.py

# Step 3: Record today's signal events into profiling DB
# (This step is now embedded in cross_report_pipeline.py as a final step)
```

---

## 5. Phase 3: Profile Analytics (Day 30+)

### 5.1 When to Start

Do NOT run analytics before you have at least 30 days of outcomes for horizon=5. That means the earliest useful analytics run is ~55 days after Phase 2 goes live (30 days of events + 5 days for T+5 outcomes to accumulate, with some buffer). For horizon=21, you need 50+ days.

### 5.2 Analytics Queries

**File**: `profile_analytics.py`

**Query 1: Conviction Level Performance**
```sql
-- Core question: Does conviction level separate forward alpha?
SELECT 
    e.conviction_level,
    COUNT(*) as n,
    ROUND(AVG(o.alpha_pct), 2) as mean_alpha_5d,
    ROUND(SUM(CASE WHEN o.alpha_pct > 0 THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 1) as hit_rate_5d,
    ROUND(AVG(o.mae_pct), 2) as mean_mae_5d,
    ROUND(AVG(o.mfe_pct), 2) as mean_mfe_5d
FROM signal_events e
JOIN signal_outcomes o ON e.symbol = o.symbol AND e.entry_date = o.entry_date
WHERE o.horizon_days = 5
  AND o.alpha_pct IS NOT NULL
GROUP BY e.conviction_level
HAVING n >= 50
ORDER BY mean_alpha_5d DESC;
```

**Query 2: Convergence Regime Performance**
```sql
-- Does convergence regime matter?
SELECT 
    e.convergence_regime,
    COUNT(*) as n,
    ROUND(AVG(o.alpha_pct), 2) as mean_alpha_5d,
    ROUND(SUM(CASE WHEN o.alpha_pct > 0 THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 1) as hit_rate_5d
FROM signal_events e
JOIN signal_outcomes o ON e.symbol = o.symbol AND e.entry_date = o.entry_date
WHERE o.horizon_days = 5
  AND e.conviction_level = 'L3_CONVERGENT'  -- Focus on L3 only
  AND o.alpha_pct IS NOT NULL
GROUP BY e.convergence_regime
HAVING n >= 30
ORDER BY mean_alpha_5d DESC;
```

**Query 3: Natural Stop-Loss Discovery**
```sql
-- What's the natural pain threshold per conviction level?
-- Percentile distribution of MAE
SELECT 
    e.conviction_level,
    COUNT(*) as n,
    -- Percentiles of worst intrawindow drawdown
    MIN(o.mae_pct) as worst_mae,
    ROUND(MAX(CASE WHEN rn <= 0.50 THEN o.mae_pct END), 2) as p50_mae,
    ROUND(MAX(CASE WHEN rn <= 0.75 THEN o.mae_pct END), 2) as p75_mae,
    ROUND(MAX(CASE WHEN rn <= 0.90 THEN o.mae_pct END), 2) as p90_mae,
    ROUND(MAX(CASE WHEN rn <= 0.95 THEN o.mae_pct END), 2) as p95_mae
FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY e.conviction_level ORDER BY o.mae_pct) as rn,
           COUNT(*) OVER (PARTITION BY e.conviction_level) as cnt
    FROM signal_events e
    JOIN signal_outcomes o ON e.symbol = o.symbol AND e.entry_date = o.entry_date
    WHERE o.horizon_days = 21 AND o.mae_pct IS NOT NULL
) sub
GROUP BY e.conviction_level
HAVING n >= 50;
```

**Query 4: Optimal Holding Period**
```sql
-- Which horizon maximizes alpha per conviction level?
SELECT 
    e.conviction_level,
    o.horizon_days,
    COUNT(*) as n,
    ROUND(AVG(o.alpha_pct), 2) as mean_alpha,
    ROUND(SUM(CASE WHEN o.alpha_pct > 0 THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 1) as hit_rate,
    ROUND(AVG(o.alpha_pct) / NULLIF(AVG(o.mae_pct), 0), 2) as alpha_per_mae  -- risk-adjusted
FROM signal_events e
JOIN signal_outcomes o ON e.symbol = o.symbol AND e.entry_date = o.entry_date
WHERE o.alpha_pct IS NOT NULL
GROUP BY e.conviction_level, o.horizon_days
HAVING n >= 50
ORDER BY e.conviction_level, alpha_per_mae DESC;
```

**Query 5: Conviction Upgrades — Do They Signal?**
```sql
-- Is a UPGRADE to L3 predictive?
SELECT 
    e.conviction_change,
    e.conviction_level,
    COUNT(*) as n,
    ROUND(AVG(o.alpha_pct), 2) as mean_alpha_5d,
    ROUND(SUM(CASE WHEN o.alpha_pct > 0 THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 1) as hit_rate_5d
FROM signal_events e
JOIN signal_outcomes o ON e.symbol = o.symbol AND e.entry_date = o.entry_date
WHERE o.horizon_days = 5
  AND o.alpha_pct IS NOT NULL
  AND e.conviction_change IN ('UPGRADED', 'NEW')
GROUP BY e.conviction_change, e.conviction_level
HAVING n >= 20
ORDER BY mean_alpha_5d DESC;
```

**Query 6: Market Regime Interaction**
```sql
-- Does the signal work better in low-VIX or high-VIX?
SELECT 
    CASE 
        WHEN e.vix_pctile_100d < 33 THEN 'LOW_VIX'
        WHEN e.vix_pctile_100d < 66 THEN 'MID_VIX'
        ELSE 'HIGH_VIX'
    END as vix_regime,
    e.conviction_level,
    COUNT(*) as n,
    ROUND(AVG(o.alpha_pct), 2) as mean_alpha_5d
FROM signal_events e
JOIN signal_outcomes o ON e.symbol = o.symbol AND e.entry_date = o.entry_date
WHERE o.horizon_days = 5
  AND o.alpha_pct IS NOT NULL
  AND e.vix_pctile_100d IS NOT NULL
GROUP BY vix_regime, e.conviction_level
HAVING n >= 20
ORDER BY vix_regime, mean_alpha_5d DESC;
```

### 5.3 Analytics Output

**File**: `outputs/profile_reports/weekly_report_YYYY-MM-DD.csv`

Generated weekly. One row per (conviction_level, convergence_regime, momentum_regime) combination with:
- Sample size
- Mean alpha at each horizon
- Hit rate at each horizon
- Mean MFE/MAE
- Rolling 60-event window metrics (vs all-time)

**Minimum sample size rule**: Any aggregate statistic with `n < 50` is flagged as `INSUFFICIENT_SAMPLE` and should not be used for decision-making.

---

## 6. Phase 4: Apollo Backtest Integration (Day 60+)

### 6.1 Prerequisite

**Do NOT integrate with Apollo's backtest until profile analytics has produced statistically significant results.** Specifically:

1. You have 30+ days of T+5 outcomes for L3 stocks (n >= 200 L3 events)
2. Query 1 shows L3 alpha_5d is positive and statistically different from L1 (not just higher, but significantly)
3. Query 2 shows MATURE_CONVERGENT outperforms DEVELOPING_CONVERGENT (or you know they're equivalent)

If L3 doesn't separate from L1 on alpha, the conviction pyramid adds no edge and should not be integrated as a gate. It can still be used as a manual watchlist.

### 6.2 Integration Point

**File**: `trade_engine.py` → `extract_trades()`

```python
def extract_trades(df, config, watchlist=None):
    # ... existing technical signal logic ...
    
    if technical_signal_fires and watchlist is not None:
        entry = watchlist.get(symbol)
        
        if entry is None:
            # Stock not on watchlist — apply higher threshold
            effective_threshold = base_threshold + 5
            if apollo_score < effective_threshold:
                continue
        else:
            # Stock is on watchlist — apply conviction-based modifier
            if entry['convergence_regime'] == 'DIVERGENT_BEARISH':
                continue  # Skip — intraday fading detected
            
            if entry['conviction_level'] == 'L3_CONVERGENT':
                effective_threshold = base_threshold - 2  # Slightly easier entry
            elif entry['conviction_level'] == 'L2_MOMENTUM_RSI':
                effective_threshold = base_threshold + 1
            else:
                effective_threshold = base_threshold + 3
            
            if apollo_score < effective_threshold:
                continue
    
    # ... continue with trade execution ...
```

### 6.3 Integration as Overlay, Not Gate

Consistent with Apollo's v3.4.1 philosophy (BUCKET is reference-only, NEVER used as a gate):

- The watchlist modifies the **threshold** for Apollo's existing score, it does not add or remove trades on its own
- A stock NOT on the watchlist can still be traded if Apollo's score is high enough (+5 threshold penalty)
- A stock ON the watchlist with DIVERGENT_BEARISH is the only hard skip
- The conviction level feeds into `TradeRecord` as a recorded-for-transparency field, exactly like the existing `bucket_classifier` output

---

## 7. Phase 5: Stock Personality Profiler (Parallel, Week 3+)

### 7.1 What This Is (Separate from Signal Profiling)

System B from the behavioral analytics discussion. Per-symbol behavioral fingerprint stored in individual Parquet files. This is NOT the same as signal profiling.

- **Signal profiling** answers: "Given my signal fired, what happens next?"
- **Stock personality** answers: "What kind of stock is this in general?"

### 7.2 Storage: Per-Symbol Parquet with Two Panels

```python
# Each stock gets: parquet/personality/SYMBOL_personality.parquet
# With two tables merged by date:

# Panel 1: Daily State (one row per date)
# Columns:
#   date, trend_stage, dist_from_swing_low_pct, dist_from_swing_high_pct,
#   adx, atr_pctile_100d, delivery_pct, range_pctile_100d,
#   rsi21, rsi36, rsi56, rsi_stack_intact

# Panel 2: Event Log (one row per discrete event)
# Columns:
#   event_date, event_type, event_value,
#   ret_5d, ret_10d, ret_21d, alpha_5d, alpha_10d, alpha_21d,
#   mfe_21d, mae_21d
# event_type values:
#   SWING_HIGH, SWING_LOW, SUPPORT_TEST, RESISTANCE_TEST,
#   BREAKOUT_ABOVE_RESISTANCE, BREAKDOWN_BELOW_SUPPORT,
#   RSI_OVERBOUGHT, RSI_OVERSOLD, RSI_DIVERGENCE,
#   ACCUMULATION_START, DISTRIBUTION_START,
#   VOLUME_CLIMAX, VOLUME_DRY_UP,
#   CIRCUIT_HIT_UPPER, CIRCUIT_HIT_LOWER
```

### 7.3 Feature List (Prioritized by Data Availability)

**Compute Now (OHLCV + Screener data):**

| Feature | Formula | Source |
|---|---|---|
| `rally_pct_from_swing_low` | `(swing_high - swing_low) / swing_low * 100` | OHLCV |
| `decline_pct_from_swing_high` | `(swing_high - swing_low) / swing_high * 100` | OHLCV |
| `retrace_pct_of_prior_leg` | `prior_leg_range - current_leg_range / prior_leg_range` | OHLCV |
| `swing_leg_bars` | sessions between swing points | OHLCV |
| `support_undercut_pct` | `min(low) - support_level / support_level` | OHLCV |
| `atr_regime` | `ATR_today / ATR_100d_avg` (contraction < 0.7, expansion > 1.3) | OHLCV |
| `gap_fill_pct` | `count(gaps filled within 5 days) / count(gaps)` | OHLCV |
| `range_pctile_100d` | `(high-low) percentile vs trailing 100 days` | OHLCV |
| `rsi_extreme_outcome` | `forward 10D return after RSI14 crosses 70/30` | OHLCV |
| `pullback_depth_pct_in_trend` | `max drawdown from peak during uptrend` | OHLCV |
| `delivery_pct` | `(dlv_qty / volume) * 100` | OHLCV (after fix) |
| `circuit_hit_followthrough` | `next-day return after circuit hit` | OHLCV + screener |
| `dist_52w_high_pct` | `(52W_High - close) / 52W_High * 100` | OHLCV |

**Add a Column, Not a Feed (from screener_stocks):**

| Feature | Formula | Source |
|---|---|---|
| `rel_volume` | `today_volume / screener_avg_volume` | OHLCV + screener |
| `trade_value_rank` | percentile of Trade Value (Cr) | screener |

**Defer (Requires New Data Feed):**

| Feature | Source | Why Defer |
|---|---|---|
| `oi_buildup_type` | NSE F&O bhavcopy | New feed needed |
| `rollover_pct` | NSE F&O bhavcopy | New feed needed |
| `earnings_gap_pct` | Earnings calendar | New feed needed |

### 7.4 Swing Point Detection

```python
def detect_swing_points(df: pd.DataFrame, n: int = 3) -> pd.DataFrame:
    """
    Fractal swing detection: N bars of lower highs/lows on each side.
    Returns DataFrame of (date, type, price) for each swing point.
    """
    swings = []
    for i in range(n, len(df) - n):
        window = df.iloc[i-n:i+n+1]
        mid = df.iloc[i]
        
        # Swing High: middle bar's high is highest in window
        if mid['High'] == window['High'].max():
            swings.append({'date': df.index[i], 'type': 'SWING_HIGH', 'price': mid['High']})
        
        # Swing Low: middle bar's low is lowest in window
        if mid['Low'] == window['Low'].min():
            swings.append({'date': df.index[i], 'type': 'SWING_LOW', 'price': mid['Low']})
    
    return pd.DataFrame(swings)
```

### 7.5 Integration with Signal Profiling

The stock personality profiler feeds INTO the signal profiling engine as context columns. Specifically:

```python
# In signal_profiler.py, when recording a signal event:
personality = load_personality(symbol, date)  # from Parquet

# These become context columns in signal_events:
event.typical_pullback_depth_60evt = personality['pullback_depth_pct_60evt']
event.false_break_rate_60evt = personality['false_break_rate_60evt']
event.avg_rally_from_swing_low = personality['rally_pct_from_swing_low_60evt']
```

This allows the analytics engine to ask: "Among L3 CONVERGENT stocks, do those with low typical pullback depth (strong trends) outperform those with high pullback depth (choppy trends)?" That's a more refined question than L3 alone.

---

## 8. Phase 6: Intraday Engine (Future)

### 8.1 What Migrates From Daily to Real-Time

| Component | Current (Daily) | Intraday Engine |
|---|---|---|
| RSI Dashboard | 6 static CSVs from Chartink | Live RSI computation as candles close |
| Convergence Engine | Computed once EOD from 6 CSVs | Recomputed every 15M candle close |
| `id_daily_gradient` | Static (15M from CSV) | Real-time (15M from live feed) |
| Convergence Regime | Once per day | Updated every 15M candle |
| Sector Heatmap | Daily snapshot | Updated every 1H candle |
| Watchlist | Static CSV for next day | Live prioritized list with alerts |

### 8.2 New Intraday-Specific Features

**Gradient Reversal Detector**
```python
class GradientReversalDetector:
    """
    Monitors 15M RSI21 vs Daily RSI21 in real-time.
    If gradient drops from positive to negative within a session,
    that's live intraday distribution.
    """
    def __init__(self, symbol: str):
        self.symbol = symbol
        self.session_gradients = []  # (timestamp, gradient)
        self.alerted = False
    
    def update(self, rsi_15m: float, rsi_daily: float, timestamp: datetime):
        gradient = rsi_15m - rsi_daily
        self.session_gradients.append((timestamp, gradient))
        
        # Check for reversal: was positive, now negative
        if len(self.session_gradients) >= 2:
            prev_grad = self.session_gradients[-2][1]
            if prev_grad > 5 and gradient < -3 and not self.alerted:
                self.alerted = True
                return GradientReversalAlert(
                    symbol=self.symbol,
                    timestamp=timestamp,
                    previous_gradient=prev_grad,
                    current_gradient=gradient,
                    rsi_15m=rsi_15m,
                    rsi_daily=rsi_daily
                )
        return None
```

**Convergence Breakdown Alert**
```python
def detect_convergence_breakdown(
    current_spread: float,
    morning_spread: float,  # spread at market open
    threshold: float = 4.0   # spread widening by this much = breakdown
) -> bool:
    """
    If a stock entered the day as MATURE_CONVERGENT (spread < 3)
    but spread widens significantly intraday, alert.
    """
    return (morning_spread < 3.0) and (current_spread - morning_spread > threshold)
```

**Real-Time Sector Rotation**
```python
# Every 1H, recompute sector participation
# If a sector goes from <30% to >60% RSI-intact within a session,
# that's live institutional rotation into the sector
def detect_live_sector_rotation(
    sector_rsi_pct_now: dict[str, float],
    sector_rsi_pct_morning: dict[str, float],
    threshold: float = 30.0
) -> list[SectorRotationAlert]:
    alerts = []
    for sector, pct_now in sector_rsi_pct_now.items():
        pct_morning = sector_rsi_pct_morning.get(sector, 0)
        if pct_morning < 30 and pct_now > 60:
            alerts.append(SectorRotationAlert(
                sector=sector,
                morning_pct=pct_morning,
                current_pct=pct_now,
                rotation_type='SURGE'
            ))
        elif pct_morning > 60 and pct_now < 30:
            alerts.append(SectorRotationAlert(
                sector=sector,
                morning_pct=pct_morning,
                current_pct=pct_now,
                rotation_type='EXIT'
            ))
    return alerts
```

### 8.3 Intraday to Daily Handoff

At EOD, the intraday engine produces a "session summary" that supplements the daily watchlist:

```python
@dataclass
class IntradaySessionSummary:
    symbol: str
    date: str
    open_convergence_regime: str       # what it was at market open
    close_convergence_regime: str      # what it was at market close
    regime_changed: bool               # did it flip during the day?
    max_gradient: float                # peak id-daily gradient during session
    min_gradient: float                # trough id-daily gradient during session
    gradient_reversal: bool            # did positive->negative flip happen?
    alerts_fired: list[str]             # what alerts triggered
```

This summary gets stored in the profiling DB alongside signal_events, giving you intraday granularity on how the stock behaved between signal generation and the next day's open.

---

## 9. Implementation Timeline

```
Week 1:  Phase 0 (Data Fixes) + Phase 1 (Pipeline Fixes)
          - Fix dlv_qty drop in repo.py
          - Add NIFTY50.parquet + INDIA_VIX.parquet
          - Fix within-tier scoring
          - Add missing columns to watchlist output
          - Split into watchlist_core + watchlist_extended
          - Guard DIVERGENT_BEARISH without intraday data
          - START SAVING DAILY WATCHLISTS (date-stamped)

Week 2:  Phase 2 (Signal Profiling Engine)
          - Build signal_events + signal_outcomes SQLite schema
          - Build signal_profiler.py (event writer)
          - Build outcome_backfiller.py (daily backfill)
          - Wire into daily_runner.sh
          - Begin accumulating data

Week 3:  Phase 5 (Stock Personality Profiler - starts in parallel)
          - Build swing point detection
          - Compute personality features for top 50 stocks
          - Generate per-symbol personality Parquets
          - (Lower priority than profiling — can be deferred)

Week 4-8: DATA ACCUMULATION
          - Just run the pipeline daily
          - No code changes
          - Manual review of watchlist_core.csv daily
          - Build intuition about which conviction levels / regimes produce good trades

Week 8+:  Phase 3 (Profile Analytics)
          - Run first analytics queries (need 30+ days of T+5 outcomes)
          - Generate first weekly_report
          - Validate hypotheses: Does L3 outperform L1? Does MATURE outperform DEVELOPING?
          - Discover natural SL levels from MAE distribution
          - Discover optimal holding period from horizon comparison

Week 12+: Phase 4 (Apollo Integration - CONDITIONAL)
          - ONLY if analytics show L3 alpha > L1 alpha with statistical significance
          - Integrate as threshold modifier in extract_trades()
          - Record conviction level in TradeRecord for transparency
          - Run walk-forward validation

Future:    Phase 6 (Intraday Engine)
          - Requires live data vendor
          - Promote convergence_engine to real-time
          - Build gradient reversal detector
          - Build convergence breakdown alerts
          - Build live sector rotation monitor
```

---

## 10. Key Design Principles

### 10.1 Profiling Over Parameterizing

Observe forward behavior distributions first. Derive SL, TSL, and holding periods from the data's natural characteristics (percentile MAE, day-of-max-gain distribution) rather than imposing fixed parameters. This is the lesson from Apollo's indicator exit collapse: overfitted parameters are fragile; empirical distributions are robust.

### 10.2 Alpha-Relative, Not Raw Returns

Every return metric must be computed relative to NIFTY. A 5% return when NIFTY is up 6% is negative alpha. Without this adjustment, you cannot distinguish signal quality from market beta.

### 10.3 No Look-Ahead by Schema Design

Signal events and outcomes are in separate tables. Events are written at signal time; outcomes are backfilled N days later. It is structurally impossible to look ahead because the outcome rows don't exist yet when the event is recorded.

### 10.4 Minimum Sample Size Enforcement

No aggregate statistic is actionable with n < 50. Intersection statistics (conviction × regime × momentum) require n < 100 to be meaningful. The analytics engine must enforce these floors and flag insufficient samples.

### 10.5 Rolling Windows, Not Lifetime Averages

All behavioral metrics use rolling windows (60 events, 100 days, etc.). Stock character changes. A 3-year average pullback depth is misleading if the stock's behavior shifted 6 months ago. Track the delta between rolling windows — the delta itself is a signal.

### 10.6 Overlay, Not Gate

Consistent with Apollo v3.4.1. The conviction pyramid modifies Apollo's existing thresholds but never adds or removes trades on its own (except DIVERGENT_BEARISH hard skip). The watchlist is recorded in TradeRecord for transparency. If the profiling data shows the pyramid adds no alpha over Apollo's existing score, it becomes reference-only — same fate as bucket_classifier.

### 10.7 Schema Stability

Output CSVs and database schemas are append-only. New columns are added at the end. Existing columns are never renamed or removed. Downstream consumers (profiling engine, intraday engine, analytics) depend on column existence.

### 10.8 Two Separate Systems

- **System A (Signal Profiling)**: Central SQLite. Event-driven. Answers "is my signal good?" Consumed by watchlist and Apollo integration.
- **System B (Stock Personality)**: Per-symbol Parquet. Continuous panel. Answers "what kind of stock is this?" Consumed by SL calibration and position sizing.

Build System A first (produces actionable insight in 60-90 days). Build System B in parallel (produces immediate value from historical data). They feed each other but are independently useful.
