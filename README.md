# Smart Combo Bot v8.1 — Safe Mode
**Production-grade algorithmic trading bot for Bybit USDT Perpetual Futures**
**Language:** Rust (async/await) | **Exchange:** Bybit Mainnet | **Live since:** March 8, 2026

---

## Overview

Smart Combo Bot v8.1 is a fully autonomous 24/7 trading bot that combines four independent strategies into a single pipeline with cross-market confirmation, multi-timeframe confluence, and Kelly-based position sizing. Written in Rust for maximum performance and reliability.

The **Safe Mode** configuration limits maximum leverage to 3×, caps open positions at 2, and requires 70% cross-market consensus before any entry — making it suitable for live capital from day one with controlled drawdown (max 10%).

---

## Architecture

```
MAIN LOOP (every 30 seconds)
│
├─ 1. Wallet + Position Sync      (every 3 min)
│       └─ Fetch equity, uPnL, open positions from Bybit
│
├─ 2. Drawdown Guard              (every tick)
│       └─ Stop all trading if equity drops > 10% from peak
│
├─ 3. Symbol Rotation             (every 1h)
│       └─ Scan 39 whitelisted symbols, pick top 7 by momentum
│
├─ 4. Aggregated Market State     (every 60s)
│       ├─ CVD (Cumulative Volume Delta) across 6 coins
│       ├─ Funding Rate mean/median across 10 coins
│       ├─ Open Interest growth across 7 coins
│       ├─ Liquidation detection across 5 coins
│       └─ Momentum Score (net long vs short pressure)
│
├─ 5. Market Regime Detection     (every 2h)
│       └─ Bull / Sideways / Bear based on aggregated state
│
├─ 6. EXIT Pipeline               (every tick)
│       ├─ ATR Trailing Stop      (check every 60s, activate after +1.5% profit)
│       ├─ Partial TP             (close 50% @ entry ± ATR×2.5)
│       └─ Max Hold Time          (force-close after 6h)
│
└─ 7. ENTRY Pipeline              (every tick)
        ├─ Aggregated Gate        (≥70% of scanned coins must agree)
        ├─ Multi-TF Confluence    (1m + 5m + 15m + 1h — all 4 must agree)
        ├─ Setup Score            (EMA200 + VWAP + BB + Vol + RSI, min 60/100)
        ├─ 5m Breakout/Breakdown  (0.4% above/below period high/low)
        ├─ OI Triple Confirm      (FR + OI growth + price action)
        └─ Kelly Sizing           (adaptive, 10–40% of equity, ATR-based SL/TP)
```

---

## Strategies

### 1. Momentum Long
Long entry when all 4 timeframes show bullish confluence, OI is growing, CVD is positive, and setup score ≥ 60/100. Uses 5m breakout as trigger candle (closed, no repainting).

### 2. Momentum Short
Mirror of Momentum Long. Short entry on bearish confluence with 5m breakdown trigger.

### 3. Funding Rate Harvest Long
Enters long position 20 minutes before funding payment when funding rate is strongly negative (< -0.02%). Exits after collecting funding. Max hold 8h, leverage 2×.

### 4. Funding Rate Harvest Short
Enters short position 20 minutes before funding payment when funding rate is strongly positive (> 0.08%). Same exit logic.

### 5. Combo Filter
Master gate that requires all individual strategy conditions PLUS aggregated consensus ≥ 70%. Final confidence score < 60 → skip.

---

## Technical Indicators

| Indicator | Parameters | Purpose |
|-----------|-----------|---------|
| RSI | Period 14 | Overbought (65) / Oversold (35) detection |
| ATR | Period 14 | Volatility measurement, SL/TP sizing |
| EMA | 9, 21, 50, 200 | Trend direction across timeframes |
| VWAP | 48 bars (4h window) | Fair value reference |
| Bollinger Bands | Period 20, 2σ | Breakout confirmation |
| CVD | All candles | Buy vs sell pressure cumulative |
| OI | Cache 20 snapshots | Open Interest growth detection |
| RSI Divergence | 14 bars | Reversal confirmation |
| Volume Spike | 48-bar rolling avg | Breakout volume filter |
| Liquidation Detection | Wick analysis | Stop-hunt detection |

---

## Safe Mode Parameters (v8.1)

### Risk Management
| Parameter | Value | Description |
|-----------|-------|-------------|
| `MAX_EQUITY_PCT` | 8% | Max equity per position |
| `MAX_OPEN_POSITIONS` | 2 | Max simultaneous positions |
| `MAX_DRAWDOWN_PCT` | 10% | Circuit breaker threshold |
| `MIN_EQUITY_USDT` | $50 | Minimum equity to trade |

### Leverage (ATR-adaptive)
| Volatility | Leverage | ATR% Range |
|-----------|---------|-----------|
| Low | 3× | ATR% < 1.0% |
| Mid | 2× | ATR% 1.0–2.5% |
| High | 2× | ATR% > 2.5% |
| Extreme | 1× | ATR% > 5.0% |
| FR Harvest | 2× | Fixed |

### Kelly Sizing
| Confidence | Kelly Multiplier |
|-----------|----------------|
| ≥ 80% | 40% of equity |
| ≥ 65% | 25% of equity |
| < 65% | 10% of equity |
| Perfect setup (≥85/100) | +10% bonus |

### Entry Filters
| Parameter | Value | Description |
|-----------|-------|-------------|
| `AGG_CONSENSUS` | 70% | Cross-market agreement |
| `TF_MIN_AGREE` | 4/4 | All timeframes must agree |
| `SETUP_MIN` | 60/100 | Minimum setup quality |
| `SETUP_PERFECT` | 85/100 | Perfect setup (Kelly bonus) |
| `BREAKOUT_MARGIN` | 0.4% | Above/below period high/low |
| `OI_GROW_PCT` | 1.0% | Minimum OI growth |

### Exit Parameters
| Parameter | Value | Description |
|-----------|-------|-------------|
| `ATR_SL` | 2.0× ATR | Stop-loss distance |
| `ATR_TP1` | 2.5× ATR | Partial TP (closes 50%) |
| `ATR_TRAIL` | 3.0× ATR | Trailing stop distance |
| `ATR_TRAIL_ACT` | +1.5% profit | Trailing stop activation |
| `MOM_HOLD_HOURS` | 6h | Maximum hold time |
| `FR_HOLD_HOURS` | 8h | FR harvest max hold |

---

## Symbol Whitelist — 39 Coins

### Tier 1 — High Liquidity (19)
`BTCUSDT` `ETHUSDT` `BNBUSDT` `XRPUSDT` `SOLUSDT` `DOGEUSDT` `AVAXUSDT` `LINKUSDT`
`ADAUSDT` `MATICUSDT` `TRXUSDT` `LTCUSDT` `DOTUSDT` `NEARUSDT` `SUIUSDT` `APTUSDT`
`ARBUSDT` `INJUSDT` `BCHUSDT`

### Tier 2 — Mid Cap (12)
`ATOMUSDT` `UNIUSDT` `OPUSDT` `XLMUSDT` `LDOUSDT` `FTMUSDT` `RUNEUSDT` `AAVEUSDT`
`ICPUSDT` `STXUSDT` `CRVUSDT` `FILUSDT`

### Tier 3 — Volatile / Niche (8)
`MKRUSDT` `DYDXUSDT` `GMXUSDT` `VETUSDT` `EGLDUSDT` `ALGOUSDT` `SNXUSDT` `COMPUSDT`

### Aggregation Symbols
- **CVD**: BTC, ETH, SOL, BNB, XRP, DOGE
- **Funding Rate**: BTC, ETH, SOL, BNB, XRP, DOGE, AVAX, LINK, ADA, ARB
- **Open Interest**: BTC, ETH, SOL, BNB, XRP, DOGE, LINK
- **Liquidations**: BTC, ETH, SOL, BNB, XRP
- **Fallback**: BTC, ETH, SOL

---

## Core Functions Reference

### API & Auth
| Function | Description |
|----------|-------------|
| `sign()` | HMAC-SHA256 signature for Bybit API |
| `auth_headers()` | Build authenticated request headers |
| `rate_limit()` | Rate limiter (1 request per 1200ms) |
| `with_retry()` | Retry wrapper (3 attempts, 2s base delay) |

### Market Data
| Function | Description |
|----------|-------------|
| `fetch_wallet()` | Equity + uPnL from Bybit Unified |
| `fetch_ticker()` | Funding rate + next payment time |
| `fetch_mark_price()` | Current mark price |
| `fetch_klines()` | OHLCV candles (1m/5m/15m/1h) |
| `fetch_oi()` | Open Interest snapshot |
| `fetch_positions()` | Active positions |
| `fetch_inst_info()` | Instrument specs (qty step, min qty) |

### Indicators
| Function | Description |
|----------|-------------|
| `calc_rsi()` | RSI(14) |
| `calc_atr()` | ATR(14) — True Range based |
| `calc_ema()` | Exponential Moving Average |
| `calc_vwap()` | Volume Weighted Average Price |
| `calc_bb()` | Bollinger Bands (period, mult) |
| `calc_cvd()` | Cumulative Volume Delta + slope |
| `is_vol_spike()` | Volume spike vs rolling average |
| `detect_liq()` | Liquidation wick detection |
| `rsi_divergence()` | Bullish/bearish divergence |
| `setup_score()` | Composite quality score 0–100 |

### Strategy Engine
| Function | Description |
|----------|-------------|
| `multi_tf_signal()` | 4-timeframe confluence (1m/5m/15m/1h) |
| `build_snapshot()` | Per-symbol market state snapshot |
| `compute_agg()` | Cross-market aggregated state |
| `oi_triple_confirm()` | FR + OI + price confirmation |
| `detect_regime()` | Bull/Sideways/Bear detection |
| `run_combo()` | Master entry filter with all gates |

### Execution
| Function | Description |
|----------|-------------|
| `open_and_log()` | Open position + log to CSV |
| `close_qty()` | Close partial or full position |
| `set_sl()` | Set stop-loss order |
| `verify_sl()` | Verify SL was accepted by exchange |
| `set_leverage()` | Set adaptive leverage |
| `check_exits()` | ATR trailing + partial TP + max hold |
| `update_trailing()` | Update trailing stop level |
| `sync_positions()` | Reconcile local state with exchange |
| `check_drawdown()` | Circuit breaker check |
| `rotate_symbols()` | Hourly symbol selection |

---

## Technology Stack

| Component | Library | Version |
|-----------|---------|---------|
| Async runtime | tokio | 1.x (rt-multi-thread) |
| HTTP client | reqwest | 0.12 (rustls-tls) |
| Serialization | serde + serde_json | 1.x |
| Logging | tracing + tracing-subscriber | 0.1/0.3 |
| HMAC signing | hmac + sha2 | 0.12/0.10 |
| Time | chrono + tokio-util | 0.4/0.7 |
| CSV output | csv | 1.x |
| Config | dotenvy | 0.15 |

**Build:** `opt-level=3`, `lto=true`, `strip=true` → ~4MB binary

---

## Performance & Resource Usage

| Metric | Value |
|--------|-------|
| Binary size | ~4 MB |
| RAM at runtime | ~15 MB |
| CPU usage | < 1% (idle between ticks) |
| API calls per tick | 8–15 (rate limited) |
| Disk (logs, 14 days) | ~10–50 MB |
| VPS requirement | 2 cores, 3GB RAM, Ubuntu 22.04 |

---

## File Structure

```
funding_bot/
├── src/
│   └── main.rs          # Full source — 2,199 lines
├── Cargo.toml           # Dependencies manifest
├── Cargo.lock           # Locked dependency versions
├── .env                 # API keys (NOT included — create yourself)
└── target/
    └── release/
        └── funding_bot  # Compiled binary (~4MB)
```

**Log output:**
```
~/bots/logs/bot.log           # Runtime logs (INFO level)
~/funding_bot/trades_YYYYMMDD.csv  # Trade history CSV
```

---

## Quick Start

```bash
# 1. Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# 2. Create .env
echo "BYBIT_API_KEY=your_key" > .env
echo "BYBIT_API_SECRET=your_secret" >> .env
echo "RUST_LOG=info" >> .env

# 3. Compile
cargo build --release

# 4. Run
nohup ./target/release/funding_bot >> logs/bot.log 2>&1 &
```

See **SETUP.pdf** for full installation guide including systemd, log rotation, and auto-restart.

---

## Live Trading Stats

| Metric | Value |
|--------|-------|
| Live since | March 8, 2026 |
| Starting equity | $163.44 USDT |
| Exchange | Bybit Mainnet (USDT Perpetual) |
| Mode | Safe Mode v8.1 |

---

## License

© 2026 JerzyDev — **Exclusive License.**
Sold under exclusive rights transfer. The buyer is the sole licensee.
The seller retains no right to resell, redistribute, or sublicense this software.
Redistribution prohibited. All rights reserved.

> **Disclaimer:** Cryptocurrency trading involves substantial risk of loss.
> The author provides this software as-is without warranty.
> The buyer assumes full responsibility for all trading activity and financial results.
## Live Performance — Day 0 (March 8, 2026)

Starting equity: **$163.48 USDT**

![Day 0](Przechwycenie_obrazu_ekranu_2026-03-08_17-27-27.png)
