# 52-Week High Tab — End-to-End Implementation Roadmap (P0–P3)

**Document Type:** Implementation Plan  
**Scope:** Frontend (React/Next.js), Backend (Python API + SQLite), Data Pipeline (Google Sheet CSV enrichment)  
**Architecture Reference:** Apollo + LayerSignal Engine, 5-Gate System, IPO Tab, Scanner Tab, Watchlist, Alerts  
**Last Updated:** 2026-08-16

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Architecture Philosophy](#2-architecture-philosophy)
3. [Data Inventory — What Exists Today](#3-data-inventory--what-exists-today)
4. [Priority Matrix Overview](#4-priority-matrix-overview)
5. [P0 — Foundation Layer](#5-p0--foundation-layer)
   - 5.1 [52WH Proximity Detection + NH-NL Breadth Bar](#51-52wh-proximity-detection--nh-nl-breadth-bar)
   - 5.2 [Cross-Filter by Apollo Score, Gates, Quality, FQS](#52-cross-filter-by-apollo-score-gates-quality-fqs)
   - 5.3 [Proximity Tier Grading (At / Near / Approaching / Extended)](#53-proximity-tier-grading-at--near--approaching--extended)
6. [P1 — Signal Enrichment Layer](#6-p1--signal-enrichment-layer)
   - 6.1 [Exhaustion Flags via RSI Divergence](#61-exhaustion-flags-via-rsi-divergence)
   - 6.2 [Rate-of-Approach — Momentum Profile to the High](#62-rate-of-approach--momentum-profile-to-the-high)
   - 6.3 [Sector Cluster Detection](#63-sector-cluster-detection)
7. [P2 — Differentiation Layer](#7-p2--differentiation-layer)
   - 7.1 [Fresh vs. Repeat High Tracking](#71-fresh-vs-repeat-high-tracking)
   - 7.2 [ATH Sub-View (All-Time Highs)](#72-ath-sub-view-all-time-highs)
   - 7.3 [IPO + 52WH Confluence Badges](#73-ipo--52wh-confluence-badges)
   - 7.4 [Watchlist & Alert Cross-Reference](#74-watchlist--alert-cross-reference)
8. [P3 — Intelligence Layer](#8-p3--intelligence-layer)
   - 8.1 [Base-Rate Forward Return Scores](#81-base-rate-forward-return-scores)
   - 8.2 [Consistency Score — "Breaking Away" vs. "Choppy High"](#82-consistency-score--breaking-away-vs-choppy-high)
   - 8.3 [EL Status Confluence Integration](#83-el-status-confluence-integration)
9. [Frontend Component Architecture](#9-frontend-component-architecture)
10. [Backend & Data Pipeline Changes](#10-backend--data-pipeline-changes)
11. [Database Schema Additions](#11-database-schema-additions)
12. [Testing Strategy](#12-testing-strategy)
13. [Risk Register & Mitigation](#13-risk-register--mitigation)
14. [Milestone Timeline](#14-milestone-timeline)

---

## 1. Executive Summary

The 52-Week High (52WH) Tab is a new discovery-focused tab that surfaces stocks approaching, at, or exceeding their 52-week highs — but with a critical differentiator: every stock is cross-referenced against the existing Apollo + LayerSignal scoring engine. This is not a standalone highs tracker; it is a **lens on top of the existing engine**. A stock at a 52WH with Apollo Score 132, all 5 Gates passed, and LayerSignal ENTRY is structurally different from one at a 52WH with Score 68, 2 failed Gates, and LayerSignal HOLD. The tab's value lies in this intersection.

The implementation is organized into four priority tiers (P0–P3) calibrated to existing data availability:

| Priority | Focus | Data Needed | Estimated Effort | Value Delivered |
|----------|-------|-------------|------------------|-----------------|
| **P0** | Proximity detection, breadth bar, engine cross-filter, proximity tiers | High52W, CMP, Apollo fields (all existing) | Low — mostly filtering/sorting logic on cached data | High — immediately actionable watchlist generator |
| **P1** | Exhaustion flags, rate-of-approach, sector clustering | RSI21/36/56 (existing), Close history, Sector field (needs adding) | Medium — some new computed fields | High — adds risk filtering and context |
| **P2** | Fresh vs. repeat tracking, ATH sub-view, IPO confluence, watchlist alerts | Daily snapshots (new), all_time_high column, isIPO flag | Medium-High — new data infrastructure | Medium-High — unique differentiators |
| **P3** | Base-rate forward returns, consistency score, EL confluence | Backtest data, 20-day close history, ELStatus | High — requires backtest joins and historical data | High — transforms from reactive to predictive |

**Recommended approach:** Ship P0 as a single sprint (3–5 days), then P1 as a follow-up sprint (5–7 days). P2 and P3 are subsequent phases that can be decoupled from the initial launch.

---

## 2. Architecture Philosophy

### 2.1 Not a Standalone System

The 52WH Tab does **not** introduce new scoring logic, new signals, or new data sources. It is a **view layer** that re-uses the existing Apollo + LayerSignal pipeline outputs and applies 52WH-specific filtering, sorting, and annotation on top. Every stock in the tab already has:

- Apollo Score (0–200 composite score)
- Gate 1–5 pass/fail status
- Quality badge (STRONG / GOOD / MODERATE / WEAK)
- FQS grade (A / B / C / D)
- LayerSignal status (ENTRY / HOLD / EXIT / EL1–EL4)
- RSI21, RSI36, RSI56, ADX, Exit_Pressure, Volume data
- Bucket classification (L1 / L2 / L3)

The tab's job is to **slice this existing dataset** through the 52WH lens and surface the most actionable setups.

### 2.2 Core Differentiator vs. Generic Highs Trackers

Most market tools show a flat list of stocks at 52-week highs, sortable by pct-from-high or volume. The 52WH Tab's edge is the **engine conviction overlay**:

1. A 52WH stock with FQS 'A' and Quality 'STRONG' = institutional accumulation at highs (high-conviction setup)
2. A 52WH stock with FQS 'D' and Quality 'WEAK' = speculative blow-off (high-risk chase)
3. A 52WH stock with ELStatus 'EL1' = pattern-confirmed breakout (dual-signal confluence)
4. A 52WH stock with Exit_Pressure > 70 = distribution despite new high (exhaustion warning)

No other dashboard combines 52WH proximity with this depth of pre-computed engine context. This is the feature that should be emphasized in every UI decision.

### 2.3 Design Principle: Progressive Disclosure

The tab should work at three levels of depth:

| Level | User Sees | Interaction |
|-------|-----------|-------------|
| **Glance** | NH-NL breadth bar + count of stocks by proximity tier | No action needed — regime read at a glance |
| **Scan** | Filtered/sorted table of 52WH stocks with engine scores | Sort, filter by tier/Apollo/Gates/Quality |
| **Deep-Dive** | Individual stock card with exhaustion flags, momentum profile, confluence badges | Click into stock detail row expansion |

---

## 3. Data Inventory — What Exists Today

### 3.1 Fields Already Available Per Stock (in cachedResponse.data)

| Field | Source | Current Usage | 52WH Tab Usage |
|-------|--------|---------------|---------------|
| `CMP` | CSV pipeline | Display price | Compute proximity % to 52WH |
| `High52W` | CSV pipeline | Display in Screener | Proximity calculation, tier assignment |
| `Low52W` | CSV pipeline | Display in Screener | Not directly (future: 52W range context) |
| `Apollo_Score` | Apollo engine | Screener sort | Cross-filter, sort, conviction ranking |
| `Gate_1` through `Gate_5` | 5-Gate system | Screener filter | Gate count filter, individual gate pass/fail |
| `Quality` | Quality engine | Badge display | Sort, filter by conviction tier |
| `FQS` | FQS engine | Badge display | Sort, filter by fundamental quality |
| `LayerSignal` | LayerSignal engine | Scanner entry signal | EL Status confluence check |
| `ELStatus` | LayerSignal engine | Pattern classification | EL1-EL4 confluence badges (P3) |
| `RSI21`, `RSI36`, `RSI56` | RSI calculations | Screener display | Exhaustion divergence detection (P1) |
| `ADX` | Trend strength | Screener display | Confirmation of trend quality |
| `Exit_Pressure` | Distribution metric | Scanner/screener | Exhaustion warning flag |
| `Volume` | CSV pipeline | Volume display | Volume surge detection near highs |
| `Bucket` | Bucket system | L1/L2/L3 classification | Filter — L2/L3 more likely to have 52WH setups |
| `isIPO` | IPO detection | IPO Tab routing | IPO confluence badge (P2) |
| `Sector` | CSV pipeline | (partially available) | Sector cluster detection (P1) |
| `Sparkline` | Sparkline data | Mini chart display | Rate-of-approach calculation (P1) |

### 3.2 Fields That Need to Be Added

| Field | Priority | Source | Effort |
|-------|----------|--------|--------|
| `Sector` (for main universe, not just IPO) | P1 | CSV enrichment or NSE API | Low — add to existing CSV parse |
| `all_time_high` | P2 | Historical data enrichment | Medium — need price history per stock |
| `52wh_date` (date when current 52WH was set) | P2 | Daily snapshot tracking | High — new infrastructure |
| `prev_52wh` and `prev_52wh_date` | P2 | Daily snapshot tracking | High — new infrastructure |
| `high_consolidation_days` | P3 | 20-day close history analysis | Medium — compute from sparkline/close data |

---

## 4. Priority Matrix Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        52WH TAB — PRIORITY LAYERS                          │
├─────────────┬───────────────────────────────────────────────────────────────┤
│  P0 (Ship)  │  Proximity Detection  │  NH-NL Breadth  │  Engine Cross-     │
│  3-5 days   │  + Tier Grading       │  Bar            │  Filter + Sort     │
├─────────────┼───────────────────────────────────────────────────────────────┤
│  P1 (Next)  │  RSI Exhaustion      │  Rate-of-       │  Sector Cluster    │
│  5-7 days   │  Flags               │  Approach       │  Detection         │
├─────────────┼───────────────────────────────────────────────────────────────┤
│  P2 (Phase) │  Fresh vs. Repeat    │  ATH Sub-View   │  IPO + Watchlist   │
│  7-10 days  │  Tracking             │                 │  Confluence        │
├─────────────┼───────────────────────────────────────────────────────────────┤
│  P3 (Future)│  Base-Rate Scores    │  Consistency    │  EL Status         │
│  10-14 days │                       │  Score          │  Confluence        │
└─────────────┴───────────────────────────────────────────────────────────────┘
```

---

## 5. P0 — Foundation Layer

**Sprint Duration:** 3–5 days  
**Goal:** A fully functional 52WH Tab that users can open, see market breadth, and filter/sort highs by engine conviction. This is the minimum viable tab.

---

### 5.1 52WH Proximity Detection + NH-NL Breadth Bar

#### What It Does

Computes how close each stock's current price (CMP) is to its 52-week high (High52W), classifies it into a proximity tier, and aggregates the results into a market-wide breadth indicator (New Highs vs. New Lows thermometer).

#### Step-by-Step Implementation

**Step 1: Compute proximity percentage for every stock**

```python
# File: backend/services/highs_service.py (new file)
# This runs during the daily data update cycle

def compute_52wh_proximity(stock_data: list[dict]) -> list[dict]:
    """
    For each stock, compute:
      - proximity_pct: (CMP / High52W) * 100
      - distance_pct: (High52W - CMP) / High52W * 100
      - tier: 'AT' | 'NEAR' | 'APPROACHING' | 'EXTENDED' | 'FAR'
    """
    enriched = []
    for stock in stock_data:
        cmp = stock.get('CMP', 0)
        high52w = stock.get('High52W', 0)
        
        if high52w <= 0 or cmp <= 0:
            stock['proximity_pct'] = 0
            stock['distance_pct'] = 100
            stock['proximity_tier'] = 'FAR'
            enriched.append(stock)
            continue
        
        proximity_pct = (cmp / high52w) * 100
        distance_pct = ((high52w - cmp) / high52w) * 100
        
        # Tier assignment
        if proximity_pct >= 102:
            tier = 'EXTENDED'    # Above 52WH — already extended, chase risk
        elif proximity_pct >= 99:
            tier = 'AT'           # Within 1% — active breakout
        elif proximity_pct >= 95:
            tier = 'NEAR'         # 95-99% — setting up, watch for trigger
        elif proximity_pct >= 90:
            tier = 'APPROACHING'  # 90-95% — in the strike zone
        else:
            tier = 'FAR'          # Below 90% — not relevant for this tab
        
        stock['proximity_pct'] = round(proximity_pct, 2)
        stock['distance_pct'] = round(distance_pct, 2)
        stock['proximity_tier'] = tier
        enriched.append(stock)
    
    return enriched
```

**Step 2: NH-NL breadth calculation**

```python
def compute_nh_nl_breadth(stock_data: list[dict]) -> dict:
    """
    New Highs: stocks with proximity_pct >= 100 (CMP >= High52W)
    New Lows: stocks with CMP <= Low52W * 1.01 (within 1% of 52-week low)
    
    Returns breadth data for the thermometer bar.
    """
    total = len(stock_data)
    new_highs = sum(1 for s in stock_data if s.get('proximity_pct', 0) >= 100)
    new_lows = sum(1 for s in stock_data 
                   if s.get('CMP', 0) <= s.get('Low52W', float('inf')) * 1.01)
    
    return {
        'total_stocks': total,
        'new_highs': new_highs,
        'new_lows': new_lows,
        'nh_pct': round((new_highs / total) * 100, 1) if total > 0 else 0,
        'nl_pct': round((new_lows / total) * 100, 1) if total > 0 else 0,
        'net_hl': new_highs - new_lows,  # Positive = bullish breadth
        # Historical context (P2 fresh tracking will populate this)
        'nh_pct_5d_ago': None,
        'nh_pct_20d_ago': None,
    }
```

**Step 3: API endpoint**

```python
# File: backend/api/routes/highs.py (new file)

@router.get('/highs/breadth')
async def get_breadth():
    """Returns NH-NL breadth data for the thermometer bar."""
    data = get_cached_stock_data()  # existing function
    enriched = compute_52wh_proximity(data)
    breadth = compute_nh_nl_breadth(enriched)
    return breadth

@router.get('/highs/stocks')
async def get_highs_stocks(
    tier: str = None,          # Filter: AT, NEAR, APPROACHING, EXTENDED
    min_apollo: int = None,    # Filter: minimum Apollo Score
    min_gates: int = None,     # Filter: minimum Gate pass count (0-5)
    quality: str = None,       # Filter: STRONG, GOOD, MODERATE, WEAK
    fqs: str = None,           # Filter: A, B, C, D
    sector: str = None,        # Filter: sector name (P1)
    sort_by: str = 'proximity_pct',  # Sort field
    sort_dir: str = 'desc',    # asc or desc
    limit: int = 50,           # Pagination
    offset: int = 0,
):
    """Returns filtered, sorted list of stocks near their 52WH."""
    data = get_cached_stock_data()
    enriched = compute_52wh_proximity(data)
    
    # Apply tier filter — default to AT + NEAR + APPROACHING (exclude FAR and EXTENDED)
    if tier:
        enriched = [s for s in enriched if s['proximity_tier'] == tier]
    else:
        enriched = [s for s in enriched if s['proximity_tier'] in ('AT', 'NEAR', 'APPROACHING')]
    
    # Apply engine cross-filters
    if min_apollo is not None:
        enriched = [s for s in enriched if s.get('Apollo_Score', 0) >= min_apollo]
    if min_gates is not None:
        enriched = [s for s in enriched 
                    if sum(1 for g in ['Gate_1','Gate_2','Gate_3','Gate_4','Gate_5'] 
                          if s.get(g) == 'PASS') >= min_gates]
    if quality:
        enriched = [s for s in enriched if s.get('Quality', '').upper() == quality.upper()]
    if fqs:
        enriched = [s for s in enriched if s.get('FQS', '').upper() == fqs.upper()]
    
    # Sort
    reverse = sort_dir == 'desc'
    enriched.sort(key=lambda s: s.get(sort_by, 0), reverse=reverse)
    
    # Paginate
    total = len(enriched)
    page = enriched[offset:offset + limit]
    
    return {'total': total, 'data': page}
```

**Step 4: Frontend component — NH-NL Breadth Bar**

```tsx
// File: components/highs/BreadthBar.tsx (new file)

interface BreadthData {
  total_stocks: number;
  new_highs: number;
  new_lows: number;
  nh_pct: number;
  nl_pct: number;
  net_hl: number;
}

// Visual: Horizontal thermometer bar
// Left side: New Lows (red)  |  Neutral zone (gray)  |  Right side: New Highs (green)
// Labels: "New Highs: 47 (15.8%)" and "New Lows: 12 (4.0%)"
// Center: Net H-L: +35
// 
// Color coding for the net reading:
//   net_hl > +50: Strong bullish breadth (deep green)
//   net_hl +20 to +50: Moderately bullish (light green)
//   net_hl -20 to +20: Neutral (gray)
//   net_hl -50 to -20: Moderately bearish (light red)
//   net_hl < -50: Strong bearish breadth (deep red)
```

#### Acceptance Criteria

- [ ] Proximity percentage computed for all ~297 stocks on every data refresh
- [ ] Tier classification correct: AT (>=99%), NEAR (95-99%), APPROACHING (90-95%), EXTENDED (>102%)
- [ ] NH-NL breadth bar shows real-time counts and percentages
- [ ] API response time < 200ms (computed from cached data, no external calls)
- [ ] Stocks in EXTENDED tier are excluded from default view but available via filter toggle
- [ ] Breadth bar updates when daily data refreshes

---

### 5.2 Cross-Filter by Apollo Score, Gates, Quality, FQS

#### What It Does

Adds filter controls to the 52WH Tab that let users narrow down the highs list by existing engine metrics. This is the **primary differentiator** — it transforms a flat highs list into a conviction-ranked discovery tool.

#### Step-by-Step Implementation

**Step 1: Define filter state model**

```tsx
// File: components/highs/HighsFilters.tsx (new file)

interface HighsFilterState {
  // Proximity
  tiers: ('AT' | 'NEAR' | 'APPROACHING' | 'EXTENDED')[];
  
  // Engine conviction
  minApolloScore: number | null;       // Slider: 0-200, default null (no filter)
  minGateCount: number | null;          // Dropdown: 0/1/2/3/4/5, default null
  qualityFilter: string[];               // Multi-select: STRONG, GOOD, MODERATE, WEAK
  fqsFilter: string[];                   // Multi-select: A, B, C, D
  
  // LayerSignal (P3, but scaffold now)
  elStatusFilter: string[];              // Multi-select: EL1, EL2, EL3, EL4, NONE
  
  // Sorting
  sortBy: string;                        // proximity_pct | apollo_score | gate_count | distance_pct
  sortDir: 'asc' | 'desc';
}
```

**Step 2: Build filter bar UI**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ FILTERS                                                                      │
│                                                                              │
│ Tier:  [AT] [NEAR] [APPROACHING] [EXTENDED]     ← Toggle chips, default AT+NEAR+APPR  │
│                                                                              │
│ Apollo Score:  [====|==========] 100+             ← Range slider with min threshold  │
│ Gates:  [Any] [3+] [4+] [5/5]                   ← Quick-select buttons             │
│ Quality:  [STRONG] [GOOD] [MODERATE] [WEAK]     ← Multi-select chips               │
│ FQS:  [A] [B] [C] [D]                          ← Multi-select chips               │
│                                                                              │
│ Sort by:  [Proximity ▼]  [Apollo ▼]  [Gates ▼]  [Distance ▼]                 │
│                                                                              │
│ Active Filters: Apollo 100+ ×  |  Quality: STRONG ×  |  Clear All             │
│                                                                              │
│ Results: 23 stocks matching filters                                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Step 3: Computed column — Gate Count**

The API should return a `gate_count` field (integer 0-5) computed from Gate_1 through Gate_5. This is more useful for filtering/sorting than checking 5 individual gates.

```python
# In compute_52wh_proximity():
stock['gate_count'] = sum(
    1 for g in ['Gate_1', 'Gate_2', 'Gate_3', 'Gate_4', 'Gate_5'] 
    if stock.get(g) == 'PASS'
)
```

**Step 4: Default sort recommendation**

The default sort should be **proximity_pct DESC** (closest to high first), but the UI should prominently offer **Apollo Score DESC** as the primary alternative sort. The recommended workflow for users is:

1. Start with proximity sort to see what's at/near highs
2. Switch to Apollo Score sort to see which highs have the strongest engine conviction
3. Apply Quality/FQS filters to remove low-conviction names
4. Apply Gate count filter to find stocks with clean structural profiles

#### Acceptance Criteria

- [ ] Filter bar renders with all controls (tier toggles, Apollo slider, Gate buttons, Quality/FQS chips)
- [ ] Filters apply in real-time (debounced, no page reload)
- [ ] Active filters shown as removable chips below the filter bar
- [ ] "Clear All" button resets all filters to defaults
- [ ] Result count updates live as filters change
- [ ] Gate count computed and returned for every stock
- [ ] Default sort: proximity_pct DESC; Apollo Score sort easily accessible
- [ ] Filter state persists in URL query params (shareable links)

---

### 5.3 Proximity Tier Grading (At / Near / Approaching / Extended)

#### What It Does

Transforms the raw proximity percentage into actionable tier labels with distinct visual treatment and behavioral meaning. This turns a single data point into a **watchlist generator**.

#### Tier Definitions

| Tier | Condition | Signal Meaning | Color | UI Treatment |
|------|-----------|----------------|-------|--------------|
| **AT** | CMP >= 99% of 52WH | Active breakout — stock is at or within 1% of its high | Green badge | Prominent row highlight, "BREAKOUT" label |
| **NEAR** | 95% <= CMP < 99% of 52WH | Setting up — watch for trigger to AT | Yellow/Amber badge | Standard row, "SETUP" label |
| **APPROACHING** | 90% <= CMP < 95% of 52WH | In the strike zone — actionable watchlist candidates | Blue badge | Standard row, "WATCH" label |
| **EXTENDED** | CMP > 102% of 52WH | Already extended — chase risk, reduced position size | Red badge | Dimmed row, "EXTENDED" label with warning icon |
| **FAR** | CMP < 90% of 52WH | Not relevant — excluded from default view | No badge | Not shown unless user toggles "Show All" |

#### Step-by-Step Implementation

**Step 1: Tier badge component**

```tsx
// File: components/highs/TierBadge.tsx

const TIER_CONFIG = {
  AT:          { label: 'BREAKOUT',     color: '#10B981', bg: '#ECFDF5' },  // Green
  NEAR:        { label: 'SETUP',        color: '#F59E0B', bg: '#FFFBEB' },  // Amber
  APPROACHING: { label: 'WATCH',        color: '#3B82F6', bg: '#EFF6FF' },  // Blue
  EXTENDED:    { label: 'EXTENDED',     color: '#EF4444', bg: '#FEF2F2' },  // Red
  FAR:         { label: 'FAR',          color: '#6B7280', bg: '#F9FAFB' },  // Gray
};
```

**Step 2: Tier summary cards at top of tab**

```
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│  AT      │  │  NEAR    │  │ APPROACH │  │ EXTENDED │
│  12      │  │  34      │  │  67      │  │   8      │
│ stocks   │  │ stocks   │  │ stocks   │  │ stocks   │
│          │  │          │  │          │  │          │
│ ▲ 3 vs   │  │ ▼ 5 vs   │  │ ▲ 12 vs  │  │ ▲ 2 vs   │
│   prev   │  │   prev   │  │   prev   │  │   prev   │
└──────────┘  └──────────┘  └──────────┘  └──────────┘
```

Each card is clickable to filter the table to that tier. The "vs prev" delta (P2 feature, scaffold with `null` for now) shows change from previous trading day.

**Step 3: Tier column in the data table**

The main data table includes a "Tier" column showing the badge. Clicking the badge filters to that tier. The table should also support a compact view that groups rows by tier with collapsible sections.

#### Acceptance Criteria

- [ ] Five tiers correctly computed from proximity_pct
- [ ] Tier badges render with correct colors and labels
- [ ] Summary cards at top show count per tier (clickable to filter)
- [ ] Default view excludes FAR tier; EXTENDED shown but visually de-emphasized
- [ ] Tier column in data table with clickable badges
- [ ] "vs prev day" delta shows `—` placeholder for P0 (populated in P2)

---

## 6. P1 — Signal Enrichment Layer

**Sprint Duration:** 5–7 days  
**Goal:** Add risk filtering (exhaustion detection), momentum context (how did the stock get to the high?), and sector intelligence (is this a sector-wide move or stock-specific?).

---

### 6.1 Exhaustion Flags via RSI Divergence

#### What It Does

Detects bearish divergence across the three RSI timeframes (RSI21, RSI36, RSI56) to flag stocks that may be at 52WH but showing exhaustion signals. This is a **risk filter** — it tells the user "this stock is at its high, but the momentum indicators suggest it may be running out of steam."

#### Logic

Bearish divergence = Price makes a new 52WH (or is near it) but RSI makes a **lower high** compared to its recent peak. If all three RSI timeframes show divergence simultaneously, the exhaustion signal is strong.

```python
# File: backend/services/highs_service.py (extend existing)

def detect_rsi_divergence(stock: dict) -> dict:
    """
    Detects bearish RSI divergence for a stock near its 52WH.
    
    Returns:
      exhaustion_flags: list of str — e.g., ['RSI21_DIVERGENCE', 'RSI36_DIVERGENCE']
      exhaustion_severity: 'NONE' | 'MILD' | 'MODERATE' | 'STRONG'
    """
    flags = []
    
    rsi21 = stock.get('RSI21', 50)
    rsi36 = stock.get('RSI36', 50)
    rsi56 = stock.get('RSI56', 50)
    proximity = stock.get('proximity_pct', 0)
    exit_pressure = stock.get('Exit_Pressure', 0)
    
    # Only check divergence for stocks near/at their highs
    if proximity < 95:
        return {'exhaustion_flags': [], 'exhaustion_severity': 'NONE'}
    
    # RSI overbought threshold check
    # If RSI21 > 80 AND price at 52WH → overbought + at high = potential exhaustion
    if rsi21 > 80:
        flags.append('RSI21_OVERBOUGHT')
    
    # RSI21 > RSI36 > RSI56 is normal alignment (short-term > medium > long)
    # If RSI21 < RSI36 (short-term rolling over) while at 52WH → bearish crossover
    if rsi21 < rsi36 and proximity >= 99:
        flags.append('RSI21_36_CROSSUNDER')
    
    # All three RSIs declining from overbought while at 52WH
    if rsi21 > 70 and rsi36 > 65 and rsi56 > 60:
        # Stock is overbought on all timeframes at 52WH — exhaustion zone
        if rsi21 < rsi36 < rsi56:  # Inverted — long-term RSI higher than short-term
            flags.append('RSI_INVSTACK')
    
    # Exit Pressure confirmation
    if exit_pressure > 70 and proximity >= 99:
        flags.append('HIGH_EP_AT_HIGH')
    
    # Severity classification
    if len(flags) >= 3:
        severity = 'STRONG'
    elif len(flags) >= 2:
        severity = 'MODERATE'
    elif len(flags) >= 1:
        severity = 'MILD'
    else:
        severity = 'NONE'
    
    return {
        'exhaustion_flags': flags,
        'exhaustion_severity': severity,
    }
```

#### Exhaustion Flag Reference

| Flag | Trigger Condition | Meaning | Severity Contribution |
|------|-------------------|---------|----------------------|
| `RSI21_OVERBOUGHT` | RSI21 > 80 and proximity >= 95% | Short-term overbought at highs | 1x |
| `RSI21_36_CROSSUNDER` | RSI21 < RSI36 and proximity >= 99% | Short-term momentum rolling over | 1x |
| `RSI_INVSTACK` | RSI21 < RSI36 < RSI56 and all overbought | Full timeframe inversion — strong exhaustion | 2x |
| `HIGH_EP_AT_HIGH` | Exit_Pressure > 70 and proximity >= 99% | Distribution pressure at new high | 1x |

#### UI Treatment

```
Exhaustion Column in Table:
  NONE      → (no icon, clean)
  MILD     → ⚠️ yellow triangle, 1 flag
  MODERATE → ⚠️ orange triangle, 2 flags
  STRONG   → 🔴 red circle, 3+ flags

Row Expansion (click to see detail):
  ┌─────────────────────────────────────────────┐
  │ Exhaustion Analysis                          │
  │                                               │
  │ RSI21: 82.3 (Overbought)  ───┐               │
  │ RSI36: 71.5                  │ Bearish       │
  │ RSI56: 65.2                  │ Crossunder    │
  │                              ───┘            │
  │ Exit Pressure: 74 (High Distribution)       │
  │                                               │
  │ Flags: RSI21_OVERBOUGHT, RSI21_36_CROSSUNDER │
  │ Severity: MODERATE                           │
  └─────────────────────────────────────────────┘
```

#### Acceptance Criteria

- [ ] Exhaustion detection runs for all stocks with proximity >= 95%
- [ ] Four severity levels: NONE, MILD, MODERATE, STRONG
- [ ] Flags stored per stock and returned in API response
- [ ] Exhaustion column in table with severity icons
- [ ] Row expansion shows detailed RSI values and flag explanations
- [ ] User can filter by exhaustion severity ("Show only NONE and MILD")

---

### 6.2 Rate-of-Approach — Momentum Profile to the High

#### What It Does

Computes **how** the stock got to its 52WH — was it a slow grind or a sudden spike? The rate-of-approach provides critical context for risk/reward assessment. A stock that ground 15% higher over 30 days with low volatility is structurally different from one that gapped up 8% in a single day.

#### Required Data

This feature needs **Close price history** — at minimum the last 20 trading days. Two options:

1. **Sparkline data** (if available in the current pipeline): Extract from existing sparkline arrays
2. **New `close_history` field**: Store last 20 closes as a JSON array in the daily snapshot

Recommended: Use the sparkline data if it contains actual close values. If sparklines are pre-rendered (image data only), add a `close_20d` array to the daily enrichment.

#### Computed Metrics

```python
def compute_rate_of_approach(stock: dict, close_history: list[float]) -> dict:
    """
    Compute momentum profile metrics for a stock near its 52WH.
    
    close_history: list of last 20 daily closes, oldest first
    """
    if len(close_history) < 5:
        return {
            'return_5d': None, 'return_20d': None,
            'gap_up_count_10d': None, 'momentum_profile': 'INSUFFICIENT_DATA'
        }
    
    current = close_history[-1]
    
    # 5-day return %
    return_5d = ((current - close_history[-5]) / close_history[-5]) * 100 if len(close_history) >= 5 else None
    
    # 20-day return %
    return_20d = ((current - close_history[0]) / close_history[0]) * 100 if len(close_history) >= 20 else None
    
    # Gap-up count in last 10 days
    # Gap-up = today's open > yesterday's high by > 1%
    # (Simplified: if we don't have OHLC, use close-to-close jumps > 3% as proxy)
    gap_up_count = 0
    recent = close_history[-10:] if len(close_history) >= 10 else close_history
    for i in range(1, len(recent)):
        if recent[i] > recent[i-1] * 1.03:  # >3% daily jump as gap proxy
            gap_up_count += 1
    
    # Momentum profile classification
    if return_5d is not None and return_20d is not None:
        if return_5d > 10 and gap_up_count >= 2:
            profile = 'SPIKE'         # Sudden, gap-heavy — high chase risk
        elif return_5d > 5 and return_20d < 15:
            profile = 'ACCELERATING'  # Recent acceleration — moderate risk
        elif return_20d > 10 and return_5d < 3:
            profile = 'GRINDING'      # Steady grind — favorable risk/reward
        elif return_20d < 5:
            profile = 'CONSOLIDATING' # Tight range near high — coiling
        else:
            profile = 'MODERATE'      # Normal trending
    else:
        profile = 'INSUFFICIENT_DATA'
    
    return {
        'return_5d': round(return_5d, 2) if return_5d is not None else None,
        'return_20d': round(return_20d, 2) if return_20d is not None else None,
        'gap_up_count_10d': gap_up_count,
        'momentum_profile': profile,
    }
```

#### Momentum Profile Reference

| Profile | Conditions | Risk Assessment | Entry Recommendation |
|---------|-----------|-----------------|---------------------|
| `GRINDING` | 20d return > 10%, 5d return < 3%, few gaps | Lowest risk — orderly advance | Best entry on pullback to 20 EMA |
| `CONSOLIDATING` | 20d return < 5%, low volatility near high | Low risk — coiling breakout | Entry on volume breakout from consolidation |
| `MODERATE` | Normal trending, no extremes | Medium risk | Standard breakout entry with stop below recent swing low |
| `ACCELERATING` | 5d return > 5% but 20d < 15% | Medium-high risk — momentum picking up | Wait for pullback, do not chase |
| `SPIKE` | 5d return > 10% AND 2+ gap-ups | High risk — parabolic move | Avoid entry, or very small position with wide stop |

#### UI Treatment

```
Momentum Column in Table:
  GRINDING      → 🟢 Green bar, steady icon
  CONSOLIDATING → 🔵 Blue bar, coil icon
  MODERATE      → ⚪ Gray bar
  ACCELERATING  → 🟡 Yellow bar, speed icon
  SPIKE         → 🔴 Red bar, lightning icon

Row Expansion:
  ┌─────────────────────────────────────────────┐
  │ Momentum Profile: GRINDING                   │
  │                                               │
  │ 5-Day Return:    +2.3%                        │
  │ 20-Day Return:   +14.7%                       │
  │ Gap-Ups (10d):   0                            │
  │                                               │
  │ Interpretation: Steady orderly advance to     │
  │ the 52WH. Low chase risk. Favorable for       │
  │ pullback entry near 20 EMA support.           │
  └─────────────────────────────────────────────┘
```

#### Acceptance Criteria

- [ ] 5d return, 20d return, gap-up count computed for all stocks with sufficient history
- [ ] Five momentum profiles classified correctly
- [ ] Momentum profile column in table with color-coded icons
- [ ] Row expansion shows detailed metrics and interpretation text
- [ ] User can filter by momentum profile ("Show only GRINDING and CONSOLIDATING")

---

### 6.3 Sector Cluster Detection

#### What It Does

Identifies sectors where **multiple stocks** are simultaneously at/near their 52WH, indicating a sector-wide momentum move rather than isolated stock-specific action. This is the "breadth-as-regime" concept applied at the sector level.

#### Prerequisite

The `Sector` field must be available for the main universe (not just IPO stocks). This may require adding `Sector` to the CSV enrichment pipeline or fetching it from the NSE API during the daily update.

```python
# If Sector is not already in the main universe data:
# During bridge_daily_update.py, add a step to fetch sector for each stock
# Source: NSE API or Google Sheet CSV (if already there but not parsed)
```

#### Computation

```python
def detect_sector_clusters(stock_data: list[dict]) -> list[dict]:
    """
    Groups stocks near 52WH by sector and identifies clusters.
    
    A sector "cluster" = 3+ stocks from the same sector in the AT or NEAR tier.
    Returns sorted by cluster strength (number of stocks * avg proximity).
    """
    from collections import defaultdict
    
    # Filter to stocks near highs
    near_highs = [s for s in stock_data 
                  if s.get('proximity_tier') in ('AT', 'NEAR', 'APPROACHING')]
    
    # Group by sector
    sector_groups = defaultdict(list)
    for stock in near_highs:
        sector = stock.get('Sector', 'Unknown')
        if sector and sector != 'Unknown':
            sector_groups[sector].append(stock)
    
    # Find clusters (3+ stocks)
    clusters = []
    for sector, stocks in sector_groups.items():
        if len(stocks) >= 3:
            avg_proximity = sum(s['proximity_pct'] for s in stocks) / len(stocks)
            avg_apollo = sum(s.get('Apollo_Score', 0) for s in stocks) / len(stocks)
            at_count = sum(1 for s in stocks if s['proximity_tier'] == 'AT')
            
            clusters.append({
                'sector': sector,
                'stock_count': len(stocks),
                'at_count': at_count,
                'near_count': len(stocks) - at_count,
                'avg_proximity_pct': round(avg_proximity, 2),
                'avg_apollo_score': round(avg_apollo, 1),
                'top_stocks': sorted(stocks, key=lambda s: s.get('Apollo_Score', 0), reverse=True)[:5],
                'strength': len(stocks) * avg_proximity,  # Sort key
            })
    
    # Sort by strength descending
    clusters.sort(key=lambda c: c['strength'], reverse=True)
    return clusters
```

#### UI Treatment

```
┌─────────────────────────────────────────────────────────────────────┐
│ SECTOR CLUSTERS                                    3 clusters found │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  🏭 Capital Goods  (8 stocks, 3 at high)     Avg Apollo: 118        │
│  ████████████████████████████░░░  Avg Proximity: 98.2%                │
│  Top: L&T, HAL, Siemens, ABB, BHE  →  View All (8)                  │
│                                                                       │
│  💊 Healthcare     (5 stocks, 2 at high)     Avg Apollo: 105        │
│  ████████████████████░░░░░░░░  Avg Proximity: 96.8%                  │
│  Top: Sun Pharma, Dr Reddys, Cipla  →  View All (5)                 │
│                                                                       │
│  💻 IT Services     (4 stocks, 1 at high)     Avg Apollo: 112       │
│  ██████████████████░░░░░░░░░░  Avg Proximity: 95.4%                  │
│  Top: TCS, Infosys, Wipro, HCL  →  View All (4)                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

Clicking a cluster filters the main table to that sector. Clicking "View All" shows all stocks in the cluster.

#### Acceptance Criteria

- [ ] Sector field available for all stocks in the main universe
- [ ] Sector clusters detected when 3+ stocks from same sector are near 52WH
- [ ] Clusters sorted by strength (stock count * avg proximity)
- [ ] Cluster cards show: sector name, stock count, at-high count, avg Apollo, top 5 stocks
- [ ] Clicking cluster card filters main table to that sector
- [ ] Clusters section hidden when no clusters exist (with "No sector clusters detected" message)

---

## 7. P2 — Differentiation Layer

**Sprint Duration:** 7–10 days  
**Goal:** Add features that no other highs tracker offers — fresh vs. repeat tracking, ATH sub-view, IPO confluence, and watchlist/alert integration. These require new data infrastructure (daily snapshots).

---

### 7.1 Fresh vs. Repeat High Tracking

#### What It Does

Distinguishes between stocks hitting a **new** 52-week high today (fresh) versus stocks that have been hovering near their high for days or weeks (repeat). Fresh highs have statistically different follow-through characteristics than repeat highs, making this distinction actionable.

#### Data Infrastructure Required

**Daily snapshot table** — stores per-stock proximity data for each trading day. This is the biggest infrastructure investment in P2.

```sql
-- File: database/migrations/add_daily_highs_snapshot.sql

CREATE TABLE IF NOT EXISTS daily_highs_snapshot (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    date            TEXT NOT NULL,           -- 'YYYY-MM-DD'
    symbol          TEXT NOT NULL,           -- 'RELIANCE'
    high_52w        REAL,                   -- 52WH value on that day
    cmp             REAL,                   -- Close price on that day
    proximity_pct   REAL,                   -- (CMP / High52W) * 100
    proximity_tier  TEXT,                   -- AT, NEAR, APPROACHING, EXTENDED, FAR
    apollo_score    INTEGER,
    gate_count      INTEGER,
    quality         TEXT,
    fqs             TEXT,
    
    UNIQUE(date, symbol)
);

CREATE INDEX idx_dhs_date ON daily_highs_snapshot(date);
CREATE INDEX idx_dhs_symbol_date ON daily_highs_snapshot(symbol, date);
CREATE INDEX idx_dhs_tier_date ON daily_highs_snapshot(proximity_tier, date);
```

#### Computation

```python
def classify_fresh_vs_repeat(symbol: str, current_date: str, current_tier: str) -> dict:
    """
    Classify a stock's 52WH status as fresh or repeat.
    
    FRESH: Stock is AT tier today, and was NOT in AT tier in the last 5 trading days.
    REPEAT: Stock has been in AT or NEAR tier for 3+ consecutive trading days.
    RECLAIMED: Stock was EXTENDED or NEAR, dropped back, and is now AT again.
    """
    # Query last 10 trading days of snapshots for this symbol
    history = query_snapshot_history(symbol, current_date, lookback_days=10)
    
    if not history or len(history) < 3:
        return {'high_type': 'INSUFFICIENT_HISTORY', 'days_near_high': 0}
    
    # Count consecutive days in AT or NEAR tier (ending today)
    consecutive_near = 0
    for day in reversed(history[:-1]):  # Exclude today
        if day['proximity_tier'] in ('AT', 'NEAR'):
            consecutive_near += 1
        else:
            break
    
    # Was the stock recently at high, dropped, and now back?
    recent_tiers = [d['proximity_tier'] for d in history[-5:]]
    had_drop = any(t in ('APPROACHING', 'FAR') for t in recent_tiers[:-1])
    
    if current_tier == 'AT' and consecutive_near == 0 and not had_drop:
        high_type = 'FRESH'
    elif current_tier == 'AT' and had_drop:
        high_type = 'RECLAIMED'
    elif consecutive_near >= 5:
        high_type = 'EXTENDED_STAY'  # Been near high for 5+ days
    elif consecutive_near >= 3:
        high_type = 'REPEAT'
    else:
        high_type = 'EARLY_REPEAT'
    
    return {
        'high_type': high_type,
        'days_near_high': consecutive_near + 1,  # Include today
        'prev_52wh_date': get_previous_52wh_date(symbol, current_date),
    }
```

#### High Type Reference

| Type | Condition | Follow-Through Expectation | Actionability |
|------|-----------|---------------------------|---------------|
| `FRESH` | First day at 52WH, not near high recently | Highest — fresh breakouts have best 20-day forward returns | Highest priority for watchlist entry |
| `RECLAIMED` | Was at high, dropped back, now reclaiming | High — reclaim after pullback is strong pattern | Good entry on reclaim confirmation |
| `EARLY_REPEAT` | 2-3 days near high | Moderate — initial momentum still intact | Hold if owned, cautious new entry |
| `REPEAT` | 3-5 days near high | Lower — momentum may be cooling | Only enter on pullback |
| `EXTENDED_STAY` | 5+ days near high | Lowest — likely consolidating or exhausting | Avoid new entries, watch for breakout or failure |

#### Acceptance Criteria

- [ ] Daily snapshot table created and populated during each data update cycle
- [ ] Fresh vs. repeat classification computed for all AT-tier stocks
- [ ] Five high types classified correctly
- [ ] High type shown as badge in table and in row expansion
- [ ] "Days near high" counter displayed
- [ ] Tier summary cards show "vs prev day" delta (now populated with real data)

---

### 7.2 ATH Sub-View (All-Time Highs)

#### What It Does

Adds a toggle to switch between 52-Week Highs and All-Time Highs. These are structurally different signals:

- **52WH**: Stock could still be 40% below its actual ATH from 3 years ago. Overhead supply zones may exist.
- **ATH**: Zero historical overhead — pure blue-sky territory. No supply zones to resist further upside.

The risk profiles and follow-through statistics are completely different, warranting separate views.

#### Data Requirement

An `all_time_high` column for each stock in the main universe. Currently, ATH is tracked for IPO stocks in the `ipo_stocks` table. Extending to the full universe requires:

**Option A (Recommended): Historical price enrichment**
- During daily update, compare current CMP against the maximum price ever seen for that symbol across all historical data
- Store in a new `stock_ath` table:

```sql
CREATE TABLE IF NOT EXISTS stock_ath (
    symbol          TEXT PRIMARY KEY,
    all_time_high   REAL NOT NULL,
    ath_date        TEXT,                   -- When the ATH was set
    ath_cmp_pct     REAL,                   -- Current CMP as % of ATH
    updated_at      TEXT DEFAULT CURRENT_TIMESTAMP
);
```

**Option B: Simplified approximation**
- Use `High52W * 1.20` as a rough ATH proxy for stocks listed > 2 years
- Less accurate but zero new data infrastructure

Recommended: Option A for accuracy. The `stock_ath` table is a one-time historical scan plus daily incremental update.

#### UI Treatment

```
View Toggle (at top of tab):

  [52-Week Highs]  [All-Time Highs]  [Both]
       ↑ active          ↑ inactive       ↑ shows both

When ATH view is active:
  - Replace High52W column with ATH column
  - Recompute proximity_pct against ATH instead of 52WH
  - Recompute tiers against ATH
  - Show "Days Since ATH" column (how long ago the ATH was set)
  - Filter: "Stocks within 5% of ATH" is the default ATH view
```

#### Acceptance Criteria

- [ ] ATH table created and populated for all stocks in the universe
- [ ] View toggle switches between 52WH, ATH, and Both
- [ ] Proximity, tiers, and all filters recomputed against ATH when ATH view is active
- [ ] "Days since ATH" column shown in ATH view
- [ ] ATH data updates incrementally during daily data refresh

---

### 7.3 IPO + 52WH Confluence Badges

#### What It Does

Flags stocks that appear in **both** the IPO Tab's NEW_HIGH zone and the 52WH Tab. This dual-signal confluence is unique to the system — no other dashboard combines IPO status with 52WH proximity. An IPO stock at a 52WH/ATH is in pure blue-sky territory with no historical overhead.

#### Data Source

The `isIPO` flag and `ipoData` on each stock in the cachedResponse already exist. No new data needed.

#### Logic

```python
def check_ipo_confluence(stock: dict) -> dict:
    """
    Check if a 52WH stock is also an IPO in NEW_HIGH zone.
    """
    is_ipo = stock.get('isIPO', False)
    ipo_data = stock.get('ipoData', {})
    proximity_tier = stock.get('proximity_tier', '')
    
    if not is_ipo:
        return {'ipo_confluence': False, 'ipo_zone': None}
    
    # Check if stock is in IPO Tab's NEW_HIGH zone
    # (This depends on your IPO Tab zone logic — adapt as needed)
    # Typically: stock is within X% of its listing high or 52WH
    ipo_zone = ipo_data.get('zone', '')  # e.g., 'NEW_HIGH', 'ACCUMULATION', etc.
    
    confluence = is_ipo and proximity_tier in ('AT', 'NEAR') and ipo_zone == 'NEW_HIGH'
    
    return {
        'ipo_confluence': confluence,
        'ipo_zone': ipo_zone,
        'days_since_listing': ipo_data.get('days_since_listing', None),
        'listing_high': ipo_data.get('listing_high', None),
    }
```

#### UI Treatment

```
Badge:  "IPO + 52WH" badge — purple/violet color, diamond icon

Shown in:  
  - Table: dedicated column or overlay badge on stock name
  - Row expansion: "IPO Details" section with listing date, days since listing, listing high
  - Filter: "Show IPO Confluence Only" toggle

Tooltip: "This IPO stock is at/near its 52-week high in the NEW_HIGH zone. 
           Pure blue-sky territory with no historical overhead supply."
```

#### Acceptance Criteria

- [ ] IPO confluence check runs for all stocks with isIPO = true
- [ ] Purple "IPO + 52WH" badge shown in table when confluence detected
- [ ] Badge links/cross-references the IPO Tab for that stock
- [ ] Filter toggle to show only IPO confluence stocks
- [ ] Row expansion shows IPO-specific details (listing date, days since listing, listing high)

---

### 7.4 Watchlist & Alert Cross-Reference

#### What It Does

Cross-references the 52WH list against the user's Watchlist and active Scanner presets. Two use cases:

1. **Watchlist 52WH Alert**: If a stock on the user's Watchlist hits a 52WH, that's a trigger event — the user is already tracking it, and it's now breaking out.
2. **Scanner Confluence**: If a stock in the Scanner with an active preset (e.g., "L3 Ready") also appears in the 52WH list, that's a dual-signal confluence worth surfacing.

#### Implementation

```python
# Backend: Check watchlist stocks against 52WH data
@router.get('/highs/watchlist-alerts')
async def get_watchlist_52wh_alerts():
    """
    Returns stocks that are BOTH on the user's watchlist AND at/near 52WH.
    Generates alert_logs entries for newly triggered alerts.
    """
    watchlist = get_user_watchlist()  # existing function
    all_highs = get_52wh_stocks(tiers=['AT', 'NEAR'])
    
    watchlist_symbols = {w['symbol'] for w in watchlist}
    
    matches = [s for s in all_highs if s['symbol'] in watchlist_symbols]
    
    # Generate alerts for new matches (not already alerted today)
    for stock in matches:
        existing_alert = check_alert_exists(
            symbol=stock['symbol'],
            alert_type='WATCHLIST_52WH',
            date=today()
        )
        if not existing_alert:
            create_alert(
                symbol=stock['symbol'],
                alert_type='WATCHLIST_52WH',
                message=f"{stock['symbol']} is at 52-week high (Tier: {stock['proximity_tier']}, Apollo: {stock['Apollo_Score']})",
                severity='HIGH' if stock['proximity_tier'] == 'AT' else 'MEDIUM'
            )
    
    return {'alerts': matches, 'count': len(matches)}
```

#### UI Treatment

```
Alert Banner (when watchlist stocks hit 52WH):
┌─────────────────────────────────────────────────────────────────┐
│ ⚡ WATCHLIST ALERTS                              3 stocks       │
│                                                                   │
│ RELIANCE at 52WH (BREAKOUT)     Apollo: 142  Quality: STRONG    │
│ TCS near 52WH (SETUP)          Apollo: 118  Quality: GOOD       │
│ HAL at 52WH (BREAKOUT)         Apollo: 131  Quality: STRONG    │
│                                                                   │
│ [View All in 52WH Tab]  [Dismiss]                                 │
└─────────────────────────────────────────────────────────────────┘

Scanner Confluence Badge (in 52WH table):
  Stock row shows: "Scanner: L3 Ready" badge when the stock also 
  matches an active Scanner preset.
```

#### Acceptance Criteria

- [ ] Watchlist stocks at/near 52WH surfaced as alerts in the 52WH Tab
- [ ] Alert generated in alert_logs when a watchlist stock newly enters AT tier
- [ ] No duplicate alerts for same stock on same day
- [ ] Alert banner shows at top of 52WH Tab when alerts exist
- [ ] Scanner confluence badge shown when stock matches active preset
- [ ] User can dismiss alerts (persisted in session)

---

## 8. P3 — Intelligence Layer

**Sprint Duration:** 10–14 days  
**Goal:** Transform the tab from a reactive filter into a predictive intelligence tool using base-rate statistics, consistency scoring, and EL Status confluence.

---

### 8.1 Base-Rate Forward Return Scores

#### What It Does

Answers the question: "Historically, when a stock with similar characteristics was at its 52WH, what happened next?" This provides **evidence-based probability estimates** for forward returns, turning the tab from descriptive ("this stock is at its high") to predictive ("stocks like this have historically returned X% over the next 20 days").

#### Data Requirement

Historical backtest data with forward returns. This requires either:

1. **Running the Apollo/LayerSignal pipeline on historical data** to generate engine scores for past dates, then computing actual forward returns
2. **Joining the daily_highs_snapshot table** (P2) with actual forward price data

This is the most data-intensive feature in the roadmap.

#### Computation

```python
def compute_base_rates(
    tier: str,
    apollo_range: tuple,       # e.g., (100, 200)
    gate_count: int,
    quality: str,
    momentum_profile: str,
    lookforward_days: int = 20,
) -> dict:
    """
    Query historical snapshots for stocks matching the given profile,
    then compute actual forward returns for each occurrence.
    
    Returns base-rate statistics: win rate, avg return, median return,
    best/worst case, and distribution buckets.
    """
    # Step 1: Find all historical occurrences matching the profile
    historical_matches = query_historical_matches(
        tier=tier,
        min_apollo=apollo_range[0],
        max_apollo=apollo_range[1],
        min_gates=gate_count,
        quality=quality,
        momentum_profile=momentum_profile,
    )
    
    if len(historical_matches) < 10:
        return {
            'sample_size': len(historical_matches),
            'status': 'INSUFFICIENT_DATA',
            'message': f'Only {len(historical_matches)} historical matches. Need 10+ for reliable statistics.'
        }
    
    # Step 2: Compute forward returns for each match
    forward_returns = []
    for match in historical_matches:
        actual_return = compute_actual_forward_return(
            symbol=match['symbol'],
            start_date=match['date'],
            days=lookforward_days
        )
        if actual_return is not None:
            forward_returns.append(actual_return)
    
    # Step 3: Compute statistics
    returns = sorted(forward_returns)
    wins = [r for r in returns if r > 0]
    
    return {
        'sample_size': len(returns),
        'status': 'READY',
        'win_rate': round(len(wins) / len(returns) * 100, 1) if returns else 0,
        'avg_return': round(sum(returns) / len(returns), 2) if returns else 0,
        'median_return': round(returns[len(returns) // 2], 2) if returns else 0,
        'best_case': round(max(returns), 2) if returns else 0,
        'worst_case': round(min(returns), 2) if returns else 0,
        'p25': round(returns[len(returns) // 4], 2) if len(returns) >= 4 else None,
        'p75': round(returns[3 * len(returns) // 4], 2) if len(returns) >= 4 else None,
        # Distribution buckets
        'distribution': {
            'strong_gain': len([r for r in returns if r > 10]),     # > 10%
            'moderate_gain': len([r for r in returns if 2 < r <= 10]),  # 2-10%
            'flat': len([r for r in returns if -2 <= r <= 2]),       # -2% to 2%
            'moderate_loss': len([r for r in returns if -10 <= r < -2]), # -10% to -2%
            'strong_loss': len([r for r in returns if r < -10]),     # < -10%
        }
    }
```

#### UI Treatment

```
Base-Rate Panel (shown in row expansion or as a side panel):

┌───────────────────────────────────────────────────────────┐
│ Historical Base Rates (20-Day Forward Return)              │
│ Sample: 47 similar occurrences                            │
│                                                            │
│ Win Rate:  63.8%          Avg Return:  +4.2%              │
│ Median:    +3.1%          Best:  +18.7%   Worst: -12.3%   │
│                                                            │
│ Distribution:                                               │
│ ████████████████████████░░░░░░░░░░░  Strong Gain  12 (26%)│
│ ████████████████████░░░░░░░░░░░░░░  Moderate   16 (34%) │
│ ████████░░░░░░░░░░░░░░░░░░░░░░░░░  Flat        9 (19%)  │
│ ██████░░░░░░░░░░░░░░░░░░░░░░░░░░░  Mod. Loss  7 (15%)  │
│ ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  Str. Loss   3 (6%)  │
│                                                            │
│ Profile: AT tier, Apollo 120-160, 4+ Gates, GRINDING      │
└───────────────────────────────────────────────────────────┘
```

#### Acceptance Criteria

- [ ] Historical backtest data available for base-rate computation
- [ ] Base-rate statistics computed for each stock's profile (tier + Apollo + Gates + quality + momentum)
- [ ] Win rate, avg/median return, best/worst case, and distribution shown
- [ ] Minimum sample size threshold (10) enforced; "insufficient data" shown for rare profiles
- [ ] Distribution visualized as horizontal bar chart
- [ ] Base-rate panel accessible from row expansion and stock detail view

---

### 8.2 Consistency Score — "Breaking Away" vs. "Choppy High"

#### What It Does

Measures **how consistently** a stock has been trading near its current 52WH. A stock that has been consolidating near its high for 15+ days (high consolidation) is building a launchpad. A stock that touched its high once and dropped back is a "one-touch wonder" with less reliable follow-through.

#### Computation

```python
def compute_consistency_score(close_history: list[float], current_high: float) -> dict:
    """
    Track how many of the last 20 closes were within 3% of the current 52WH.
    
    Returns:
      high_close_count: number of closes within 3% of 52WH in last 20 days
      consistency_pct: (high_close_count / 20) * 100
      pattern: 'LAUNCHPAD' | 'BUILDING' | 'CHOPPY' | 'ONE_TOUCH'
    """
    if len(close_history) < 10:
        return {'consistency_score': None, 'pattern': 'INSUFFICIENT_DATA'}
    
    threshold = current_high * 0.97  # 3% below high
    recent_20 = close_history[-20:] if len(close_history) >= 20 else close_history
    
    high_close_count = sum(1 for c in recent_20 if c >= threshold)
    consistency_pct = (high_close_count / len(recent_20)) * 100
    
    # Also compute: max drawdown from high in the last 20 days
    max_price = max(recent_20)
    min_since_max = min(recent_20[recent_20.index(max_price):])
    max_dd_from_high = ((max_price - min_since_max) / max_price) * 100
    
    if consistency_pct >= 75 and max_dd_from_high < 3:
        pattern = 'LAUNCHPAD'     # Tight consolidation at highs
    elif consistency_pct >= 50:
        pattern = 'BUILDING'      # Moderate consistency, building base
    elif consistency_pct >= 25:
        pattern = 'CHOPPY'        # Some time near high but volatile
    else:
        pattern = 'ONE_TOUCH'     # Brief touch, dropped back quickly
    
    return {
        'high_close_count': high_close_count,
        'total_days': len(recent_20),
        'consistency_pct': round(consistency_pct, 1),
        'max_dd_from_high': round(max_dd_from_high, 2),
        'pattern': pattern,
    }
```

#### Pattern Reference

| Pattern | Consistency % | Max DD from High | Interpretation | Follow-Through |
|---------|--------------|-------------------|----------------|----------------|
| `LAUNCHPAD` | >= 75% | < 3% | Tight consolidation at highs — supply absorbed | Best — high probability of continuation breakout |
| `BUILDING` | 50-75% | < 5% | Moderately consistent — base forming | Good — watch for volume breakout |
| `CHOPPY` | 25-50% | Variable | Volatile near highs — no clear base | Caution — enter only on clear breakout confirmation |
| `ONE_TOUCH` | < 25% | > 5% | Brief touch then dropped — no conviction | Avoid — low probability of sustained move |

#### UI Treatment

```
Consistency Column:
  LAUNCHPAD  → 🟢 Green bar, "tight coil" icon
  BUILDING   → 🔵 Blue bar
  CHOPPY     → 🟡 Yellow bar
  ONE_TOUCH  → 🔴 Red bar

Row Expansion:
  "15 of 20 days (75%) within 3% of 52WH"
  Visual: mini bar chart showing last 20 days, colored by proximity
  Pattern: LAUNCHPAD — tight consolidation, supply absorbed
```

#### Acceptance Criteria

- [ ] Consistency score computed from 20-day close history
- [ ] Four patterns classified correctly
- [ ] Consistency column in table with color-coded icons
- [ ] Mini visualization in row expansion showing daily proximity over 20 days
- [ ] User can filter by consistency pattern

---

### 8.3 EL Status Confluence Integration

#### What It Does

Cross-references every 52WH stock with the LayerSignal engine's ELStatus (EL1 through EL4). A stock at 52WH with ELStatus 'EL1' (Primary Breakout) represents the **ideal confluence** — the breakout is confirmed by both price AND pattern engine. A stock at 52WH with ELStatus 'NONE' means the LayerSignal engine doesn't recognize the move as a valid pattern — much weaker signal.

This is a **zero-new-data, pure-logic cross-filter**. All data exists today.

#### Confluence Matrix

| 52WH Tier | EL1 (Primary Breakout) | EL2 (Continuation) | EL3 (Pullback Buy) | EL4 (Re-entry) | NONE |
|-----------|----------------------|--------------------|--------------------|-----------------|-------|
| **AT** | **Strongest signal** — dual confirmation (price + pattern) | Strong — breakout with continuation pattern | Unusual — at high but LS sees pullback | Good — re-entry at highs | Weakest — price at high but no pattern confirmation |
| **NEAR** | Setup forming — LS sees breakout before price confirms | Good — trending with continuation | Mixed — LS sees pullback near highs | Watch — re-entry pattern forming | No pattern confirmation |
| **APPROACHING** | Early signal — LS breakout before price at high | Good — trend continuation developing | Divergent — LS pullback vs. approaching high | Watch — re-entry setting up | No signal |

#### Implementation

```python
def compute_el_confluence(stock: dict) -> dict:
    """
    Cross-reference 52WH proximity with LayerSignal ELStatus.
    Returns confluence strength and recommendation.
    """
    el_status = stock.get('ELStatus', 'NONE')
    tier = stock.get('proximity_tier', '')
    
    # Confluence strength scoring
    confluence_scores = {
        ('AT', 'EL1'): 5,    # Strongest
        ('AT', 'EL2'): 4,
        ('AT', 'EL4'): 4,
        ('AT', 'EL3'): 2,
        ('AT', 'NONE'): 1,   # Weakest at high
        ('NEAR', 'EL1'): 4,
        ('NEAR', 'EL2'): 3,
        ('NEAR', 'EL4'): 3,
        ('NEAR', 'EL3'): 2,
        ('NEAR', 'NONE'): 1,
        ('APPROACHING', 'EL1'): 3,
        ('APPROACHING', 'EL2'): 2,
        ('APPROACHING', 'EL4'): 2,
        ('APPROACHING', 'EL3'): 1,
        ('APPROACHING', 'NONE'): 1,
    }
    
    score = confluence_scores.get((tier, el_status), 0)
    
    if score >= 4:
        strength = 'STRONG'
    elif score >= 3:
        strength = 'MODERATE'
    elif score >= 2:
        strength = 'WEAK'
    else:
        strength = 'NONE'
    
    return {
        'el_status': el_status,
        'confluence_score': score,
        'confluence_strength': strength,
        'is_dual_signal': tier in ('AT', 'NEAR') and el_status in ('EL1', 'EL2'),
    }
```

#### UI Treatment

```
EL Confluence Column:
  STRONG   → ✅ Green checkmark, "EL1 + AT" badge
  MODERATE → 🔵 Blue circle, "EL2 + NEAR" badge
  WEAK     → ⚪ Gray circle, status text
  NONE     → (dimmed)

Dual Signal Badge (special treatment for strongest confluence):
  When is_dual_signal = true:
    Stock name gets a golden border/glow effect
    Row background gets subtle green tint
    "DUAL SIGNAL" badge shown prominently
```

#### Acceptance Criteria

- [ ] EL Status cross-referenced for all 52WH stocks
- [ ] Confluence score (1-5) computed and returned per stock
- [ ] Four confluence strength levels classified
- [ ] EL confluence column in table with strength indicators
- [ ] "Dual Signal" badge for AT/NEAR + EL1/EL2 combinations
- [ ] User can filter by confluence strength and EL Status
- [ ] Sort by confluence score available

---

## 9. Frontend Component Architecture

### 9.1 File Structure

```
components/highs/
├── HighsTab.tsx              # Main tab container — orchestrates all sub-components
├── BreadthBar.tsx            # NH-NL thermometer bar (P0)
├── TierSummaryCards.tsx      # AT/NEAR/APPROACHING/EXTENDED count cards (P0)
├── HighsFilters.tsx          # Filter bar with all controls (P0, extended in P1/P2)
├── HighsTable.tsx            # Main data table (P0)
├── HighsTableRow.tsx         # Individual row with expansion (P0, extended in P1/P2/P3)
├── TierBadge.tsx             # Proximity tier badge component (P0)
├── ExhaustionPanel.tsx       # Exhaustion analysis in row expansion (P1)
├── MomentumPanel.tsx         # Rate-of-approach in row expansion (P1)
├── SectorClusters.tsx        # Sector cluster cards (P1)
├── FreshRepeatBadge.tsx      # Fresh vs. repeat high type badge (P2)
├── IPOConfluenceBadge.tsx    # IPO + 52WH confluence badge (P2)
├── WatchlistAlertBanner.tsx  # Watchlist 52WH alert banner (P2)
├── ViewToggle.tsx            # 52WH / ATH / Both toggle (P2)
├── BaseRatePanel.tsx         # Historical base-rate statistics panel (P3)
├── ConsistencyPanel.tsx      # Consistency score mini-chart (P3)
├── ELConfluenceBadge.tsx     # EL Status confluence badge (P3)
└── DualSignalBadge.tsx       # Dual signal highlight badge (P3)
```

### 9.2 Main Tab Layout

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ 52-Week Highs                                              [52WH] [ATH] [Both] │
│                                                                               │
│ ┌─────────────────────────────────────────────────────────────────────────┐   │
│ │ NH-NL BREADTH BAR (P0)                                                  │   │
│ │ New Lows: 12 (4.0%) ███████████████████░░░░░░░░░░░░░ New Highs: 47 (15.8%)│   │
│ │ Net H-L: +35 (Bullish)                                                  │   │
│ └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
│ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐                           │
│ │ AT: 12   │ │ NEAR: 34 │ │ APPR: 67 │ │ EXT: 8   │  ← Tier Summary Cards  │
│ └──────────┘ └──────────┘ └──────────┘ └──────────┘                           │
│                                                                               │
│ ⚡ WATCHLIST ALERTS: 3 stocks at 52WH  [View] [Dismiss]  ← P2               │
│                                                                               │
│ ┌─────────────────────────────────────────────────────────────────────────┐   │
│ │ SECTOR CLUSTERS (P1)                                                    │   │
│ │ Capital Goods (8) | Healthcare (5) | IT Services (4)                     │   │
│ └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
│ ┌─────────────────────────────────────────────────────────────────────────┐   │
│ │ FILTERS (P0)                                                            │   │
│ │ Tier: [AT][NEAR][APPR]  Apollo: [100+]  Gates: [3+]  Quality: [STRONG]  │   │
│ │ Sort: [Proximity ▼]  Active: Apollo 100+ × | Clear All  Results: 23     │   │
│ └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
│ ┌─────────────────────────────────────────────────────────────────────────┐   │
│ │ DATA TABLE (P0, expanded in P1-P3)                                       │   │
│ │                                                                          │   │
│ │ Symbol  │ Tier  │ Apollo │ Gates │ Quality │ FQS │ Exhaust │ Momentum │ EL  │   │
│ │ ────────┼───────┼────────┼───────┼─────────┼─────┼─────────┼──────────┼─────│   │
│ │ L&T     │ AT    │ 142    │ 5/5   │ STRONG  │ A   │ NONE    │ GRINDING │ EL1 │   │
│ │ HAL     │ AT    │ 131    │ 4/5   │ STRONG  │ A   │ MILD    │ MODERATE │ EL1 │   │
│ │ Reliance│ NEAR  │ 128    │ 5/5   │ GOOD    │ B   │ NONE    │ CONSOL   │ EL2 │   │
│ │ TCS     │ APPR  │ 118    │ 4/5   │ GOOD    │ A   │ NONE    │ GRINDING │ NONE│   │
│ │ ...     │       │        │       │         │     │         │          │     │   │
│ └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
│ Showing 23 of 121 stocks near 52WH                           Page 1 of 3     │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 9.3 Tab Placement in Navigation

Based on the earlier architecture decision for tab ordering (grouped by user intent):

```
Discover → Act → Review → Support

Screener → IPO Tab → Blue Sky → 52WH Tab → Watchlist → Scanner → Analytics → Alerts → Guidance → System
                ↑ Discover             ↑ Discover (also Act via alerts)
```

The 52WH Tab sits in the **Discover** group, after Blue Sky and before the transition to Act-group tabs. It's a natural companion to the Blue Sky tab (which focuses on ATH/IPO blue-sky breakout setups).

---

## 10. Backend & Data Pipeline Changes

### 10.1 New Python Service

```
backend/services/
├── highs_service.py          # NEW — all 52WH computation logic
│   ├── compute_52wh_proximity()
│   ├── compute_nh_nl_breadth()
│   ├── detect_rsi_divergence()         # P1
│   ├── compute_rate_of_approach()      # P1
│   ├── detect_sector_clusters()        # P1
│   ├── classify_fresh_vs_repeat()      # P2
│   ├── check_ipo_confluence()          # P2
│   ├── compute_base_rates()            # P3
│   ├── compute_consistency_score()     # P3
│   └── compute_el_confluence()         # P3
```

### 10.2 New API Routes

```
backend/api/routes/
├── highs.py                  # NEW — all 52WH endpoints
│   ├── GET /highs/breadth               # NH-NL breadth data
│   ├── GET /highs/stocks                # Filtered, sorted 52WH stock list
│   ├── GET /highs/sectors               # Sector cluster data (P1)
│   ├── GET /highs/watchlist-alerts      # Watchlist 52WH alerts (P2)
│   └── GET /highs/base-rates            # Base-rate statistics (P3)
```

### 10.3 Daily Update Integration

The `bridge_daily_update.py` script needs to be extended to:

1. **P0**: Call `compute_52wh_proximity()` during the daily enrichment step and cache results
2. **P2**: Insert a row into `daily_highs_snapshot` for each stock after each daily update
3. **P2**: Update `stock_ath` table if any stock sets a new all-time high
4. **P1**: Ensure `Sector` field is populated for all stocks (not just IPO)
5. **P3**: After sufficient snapshot history accumulates, pre-compute base-rate aggregates

```python
# In bridge_daily_update.py, after the main enrichment step:

# P0: Compute 52WH proximity for all stocks
from services.highs_service import compute_52wh_proximity, compute_nh_nl_breadth

enriched_data = compute_52wh_proximity(enriched_data)
breadth_data = compute_nh_nl_breadth(enriched_data)

# Cache for API consumption
cache.set('highs_stocks', enriched_data, ttl=86400)
cache.set('highs_breadth', breadth_data, ttl=86400)

# P2: Store daily snapshot
# (Only after daily_highs_snapshot table is created)
# for stock in enriched_data:
#     insert_daily_snapshot(date=today, **stock)
```

---

## 11. Database Schema Additions

### 11.1 New Tables

```sql
-- P2: Daily snapshot of 52WH proximity data
CREATE TABLE IF NOT EXISTS daily_highs_snapshot (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    date            TEXT NOT NULL,
    symbol          TEXT NOT NULL,
    high_52w        REAL,
    cmp             REAL,
    proximity_pct   REAL,
    proximity_tier  TEXT,
    apollo_score    INTEGER,
    gate_count      INTEGER,
    quality         TEXT,
    fqs             TEXT,
    UNIQUE(date, symbol)
);

-- P2: All-time high tracking
CREATE TABLE IF NOT EXISTS stock_ath (
    symbol          TEXT PRIMARY KEY,
    all_time_high   REAL NOT NULL,
    ath_date        TEXT,
    ath_cmp_pct     REAL,
    updated_at      TEXT DEFAULT CURRENT_TIMESTAMP
);
```\n### 11.2 Modified Tables

```sql
-- No existing tables need modification for P0 or P1.
-- The 52WH Tab reads from existing cached data and computes derived fields in memory.

-- P2 adds the two new tables above.
-- P3 may add:
--   base_rate_aggregates table (pre-computed statistics for common profiles)
--   alert_logs table already exists — just add new alert_type values
```

### 11.3 Index Strategy

```sql
-- Performance-critical indexes for P2 daily snapshot queries
CREATE INDEX idx_dhs_date ON daily_highs_snapshot(date);
CREATE INDEX idx_dhs_symbol_date ON daily_highs_snapshot(symbol, date);
CREATE INDEX idx_dhs_tier_date ON daily_highs_snapshot(proximity_tier, date);
CREATE INDEX idx_dhs_composite ON daily_highs_snapshot(symbol, date, proximity_tier);
```

---

## 12. Testing Strategy

### 12.1 Unit Tests

| Feature | Test Cases | Priority |
|---------|-----------|----------|
| `compute_52wh_proximity()` | Edge cases: CMP = High52W, CMP = 0, High52W = 0, negative values | P0 |
| Tier classification | Boundary values: 89.99%, 90%, 94.99%, 95%, 98.99%, 99%, 101.99%, 102% | P0 |
| NH-NL breadth | Empty data, all stocks at high, all stocks at low, mixed | P0 |
| `detect_rsi_divergence()` | All RSI combinations, proximity < 95% (should skip), Exit_Pressure edge | P1 |
| `compute_rate_of_approach()` | Insufficient history (< 5 days), exact boundary profiles | P1 |
| `detect_sector_clusters()` | No clusters, one cluster, multiple clusters, missing Sector field | P1 |
| `classify_fresh_vs_repeat()` | First day, repeat, reclaimed, insufficient history | P2 |
| `compute_consistency_score()` | All 20 days near high, none near high, mixed pattern | P3 |
| `compute_el_confluence()` | All (tier, ELStatus) combinations, unknown ELStatus | P3 |

### 12.2 Integration Tests

| Test | Description |
|------|-------------|
| API `/highs/stocks` | Verify filters, sorts, pagination work correctly together |
| API `/highs/breadth` | Verify breadth data matches manual calculation on test data |
| Daily update integration | Verify proximity computed and cached during daily cycle |
| Watchlist alert generation | Verify alert created for watchlist stock entering AT tier, no duplicate same day |
| Snapshot persistence | Verify daily_highs_snapshot populated correctly after update |

### 12.3 Frontend Tests

| Test | Description |
|------|-------------|
| Filter interaction | Applying Apollo >= 100 filter reduces result count correctly |
| Tier toggle | Clicking tier summary card filters table to that tier |
| Sort switching | Changing sort from proximity to Apollo reorders table correctly |
| Row expansion | Clicking row shows detailed panels (exhaustion, momentum, etc.) |
| URL state | Filter state reflected in URL params, page reload preserves filters |

---

## 13. Risk Register & Mitigation

| # | Risk | Probability | Impact | Mitigation |
|---|------|------------|--------|------------|
| R1 | `Sector` field missing for main universe stocks | High | Medium (P1 blocked) | Add Sector to CSV parse in `bridge_daily_update.py` before P1 sprint. Fallback: use `Industry` field if available. |
| R2 | Close price history not available for rate-of-approach | Medium | Medium (P1 partial) | Extract from sparkline data. If sparklines are image-only, add `close_20d` array to daily enrichment. |
| R3 | Daily snapshot table grows large over time | Low | Low | SQLite handles millions of rows. Add quarterly archival for snapshots older than 1 year. |
| R4 | ATH data requires historical price data not in current pipeline | Medium | Medium (P2 ATH blocked) | Start with Option B approximation (High52W * 1.20) for MVP, migrate to Option A when historical data available. |
| R5 | Base-rate statistics have insufficient sample sizes | High | Medium (P3 degraded) | Set minimum sample threshold (10 occurrences). Show "Insufficient Data" gracefully. Accumulate data over time. |
| R6 | Performance: computing proximity for 297 stocks on every API call | Low | Low | P0 computes from cached data (already in memory). No external API calls. < 50ms expected. |
| R7 | RSI divergence logic produces false positives | Medium | Low (P1 UX confusion) | Conservative thresholds. User can filter by severity. Show raw RSI values for user judgment. |
| R8 | Tab navigation reordering requires app-level changes | Low | Low | 52WH Tab is a new tab insertion. Update sidebar navigation config. No impact on existing tabs. |

---

## 14. Milestone Timeline

### Phase 1: P0 Foundation (Days 1–5)

| Day | Task | Deliverable |
|-----|------|-------------|
| 1 | Create `highs_service.py` with `compute_52wh_proximity()` and `compute_nh_nl_breadth()` | Backend computation logic |
| 1 | Create `highs.py` API routes with `/highs/breadth` and `/highs/stocks` endpoints | API endpoints |
| 2 | Build `BreadthBar.tsx` component | NH-NL thermometer bar renders with test data |
| 2 | Build `TierBadge.tsx` and `TierSummaryCards.tsx` | Tier visual components |
| 3 | Build `HighsFilters.tsx` with tier toggles, Apollo slider, Gate buttons, Quality/FQS chips | Filter bar UI |
| 3 | Integrate `gate_count` computed field into API response | Gate count available for filtering |
| 4 | Build `HighsTable.tsx` and `HighsTableRow.tsx` with all P0 columns | Data table with sorting |
| 4 | Build `HighsTab.tsx` main container, wire all components together | Functional tab |
| 5 | Connect to live API, test with real cached data | P0 complete — functional 52WH Tab |
| 5 | Add tab to navigation sidebar | 52WH Tab accessible from sidebar |

### Phase 2: P1 Enrichment (Days 6–12)

| Day | Task | Deliverable |
|-----|------|-------------|
| 6 | Ensure `Sector` field populated for all stocks | Sector data available |
| 6 | Add `close_20d` to daily enrichment (if needed for rate-of-approach) | Historical close data available |
| 7 | Implement `detect_rsi_divergence()` in `highs_service.py` | Exhaustion detection logic |
| 7 | Build `ExhaustionPanel.tsx` for row expansion | Exhaustion UI with flags and severity |
| 8 | Implement `compute_rate_of_approach()` in `highs_service.py` | Momentum profile computation |
| 8 | Build `MomentumPanel.tsx` for row expansion | Momentum UI with profile badge |
| 9 | Implement `detect_sector_clusters()` in `highs_service.py` | Cluster detection logic |
| 9 | Build `SectorClusters.tsx` component | Sector cluster cards with drill-down |
| 10 | Add exhaustion and momentum columns to table | New columns visible in main table |
| 10 | Add sector cluster filter ("Show cluster stocks only") | Filter integration |
| 11 | Integration testing — all P1 features working together | P1 complete |
| 12 | Buffer day for bug fixes and polish | P1 polished and ready |

### Phase 3: P2 Differentiation (Days 13–22)

| Day | Task | Deliverable |
|-----|------|-------------|
| 13 | Create `daily_highs_snapshot` table and migration script | Database schema ready |
| 13 | Create `stock_ath` table, run initial ATH population | ATH data available |
| 14 | Integrate snapshot insertion into `bridge_daily_update.py` | Snapshots auto-populated daily |
| 14 | Implement `classify_fresh_vs_repeat()` | Fresh/repeat classification logic |
| 15 | Build `FreshRepeatBadge.tsx` and integrate into table | Fresh/repeat badges visible |
| 15 | Populate "vs prev day" delta in tier summary cards | Live delta counts |
| 16 | Build `ViewToggle.tsx` for 52WH/ATH/Both switching | View toggle functional |
| 16 | Implement ATH proximity re-computation in API | ATH view shows correct data |
| 17 | Implement `check_ipo_confluence()` | IPO confluence detection logic |
| 17 | Build `IPOConfluenceBadge.tsx` and `WatchlistAlertBanner.tsx` | Badges and alert banner |
| 18 | Integrate watchlist alert generation with `alert_logs` | Auto-alerts for watchlist 52WH |
| 18 | Add Scanner confluence check and badge | Scanner dual-signal visible |
| 19-20 | Integration testing — all P2 features working together | P2 complete |
| 21-22 | Buffer days for bug fixes, edge cases, and polish | P2 polished and ready |

### Phase 4: P3 Intelligence (Days 23–36)

| Day | Task | Deliverable |
|-----|------|-------------|
| 23-25 | Build historical backtest pipeline — run engine on historical data | Historical data with engine scores |
| 25-27 | Implement `compute_base_rates()` with distribution statistics | Base-rate computation logic |
| 27-28 | Build `BaseRatePanel.tsx` with distribution visualization | Base-rate panel in row expansion |
| 29-30 | Implement `compute_consistency_score()` | Consistency score logic |
| 30-31 | Build `ConsistencyPanel.tsx` with mini 20-day chart | Consistency panel in row expansion |
| 32-33 | Implement `compute_el_confluence()` | EL confluence logic |
| 33-34 | Build `ELConfluenceBadge.tsx` and `DualSignalBadge.tsx` | EL badges and dual-signal highlight |
| 34-35 | Integration testing — all P3 features | P3 complete |
| 36 | Final end-to-end testing, performance optimization, documentation | Full 52WH Tab complete |

---

## Appendix A: Feature Dependency Graph

```
P0 ────────────────────────────────────────────────────────────
│  52WH proximity + tiers + breadth bar + engine cross-filter  │
│  (No new data infrastructure needed)                          │
└──────────┬────────────────────────────────────────────────────┘
           │
           ├─── P1: RSI exhaustion (uses existing RSI21/36/56)
           ├─── P1: Rate-of-approach (needs close_20d data)
           ├─── P1: Sector clusters (needs Sector field)
           │
           └─── P2: Fresh vs. repeat (needs daily_highs_snapshot table)
           ├─── P2: ATH sub-view (needs stock_ath table)
           ├─── P2: IPO confluence (uses existing isIPO flag)
           └─── P2: Watchlist alerts (uses existing watchlist + alert_logs)
               │
               └─── P3: Base-rates (needs historical snapshot data)
               └─── P3: Consistency score (needs close_20d + snapshots)
               └─── P3: EL confluence (uses existing ELStatus — no new data)
```

## Appendix B: API Response Schema (P0 Complete)

```json
{
  "breadth": {
    "total_stocks": 297,
    "new_highs": 47,
    "new_lows": 12,
    "nh_pct": 15.8,
    "nl_pct": 4.0,
    "net_hl": 35
  },
  "tier_summary": {
    "AT": { "count": 12, "delta_vs_prev": null },
    "NEAR": { "count": 34, "delta_vs_prev": null },
    "APPROACHING": { "count": 67, "delta_vs_prev": null },
    "EXTENDED": { "count": 8, "delta_vs_prev": null }
  },
  "stocks": {
    "total": 113,
    "page": 1,
    "limit": 50,
    "data": [
      {
        "symbol": "LT",
        "name": "Larsen & Toubro",
        "CMP": 3456.70,
        "High52W": 3480.00,
        "proximity_pct": 99.33,
        "distance_pct": 0.67,
        "proximity_tier": "AT",
        "Apollo_Score": 142,
        "gate_count": 5,
        "Quality": "STRONG",
        "FQS": "A",
        "ELStatus": "EL1",
        "Sector": "Capital Goods",
        "isIPO": false,
        "Bucket": "L3"
      }
    ]
  }
}
```

## Appendix C: Quick Reference — All New Fields Per Priority

| Field | Type | Priority | Computed From | Stored In |
|-------|------|----------|--------------|-----------|
| `proximity_pct` | REAL | P0 | CMP / High52W * 100 | In-memory (cached) |
| `distance_pct` | REAL | P0 | (High52W - CMP) / High52W * 100 | In-memory (cached) |
| `proximity_tier` | TEXT | P0 | proximity_pct thresholds | In-memory (cached) |
| `gate_count` | INT | P0 | COUNT(Gate_1..5 = PASS) | In-memory (cached) |
| `exhaustion_flags` | JSON | P1 | RSI21/36/56 + Exit_Pressure | In-memory (cached) |
| `exhaustion_severity` | TEXT | P1 | COUNT(flags) | In-memory (cached) |
| `return_5d` | REAL | P1 | Close[-1] / Close[-5] | In-memory (cached) |
| `return_20d` | REAL | P1 | Close[-1] / Close[-20] | In-memory (cached) |
| `gap_up_count_10d` | INT | P1 | Count(daily jumps > 3%) | In-memory (cached) |
| `momentum_profile` | TEXT | P1 | return_5d + return_20d + gaps | In-memory (cached) |
| `high_type` | TEXT | P2 | daily_highs_snapshot history | In-memory (cached) |
| `days_near_high` | INT | P2 | daily_highs_snapshot history | In-memory (cached) |
| `all_time_high` | REAL | P2 | Historical max price | stock_ath table |
| `ath_cmp_pct` | REAL | P2 | CMP / all_time_high * 100 | In-memory (cached) |
| `consistency_pct` | REAL | P3 | 20-day close distribution | In-memory (cached) |
| `consistency_pattern` | TEXT | P3 | consistency_pct + max_dd | In-memory (cached) |
| `confluence_score` | INT | P3 | (tier, ELStatus) matrix | In-memory (cached) |
| `confluence_strength` | TEXT | P3 | confluence_score thresholds | In-memory (cached) |
