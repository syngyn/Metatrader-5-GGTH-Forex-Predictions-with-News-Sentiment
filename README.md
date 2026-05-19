# GGTH Predictor — ML Forex Trading System

> **EA v1.19 · Predictor v9.5 · GUI v2.3 · Sentiment v2.3**

A two-process algorithmic trading system for EURUSD on MetaTrader 5. Python runs a five-model deep-learning ensemble and a news-sentiment pipeline. A compiled MQL5 Expert Advisor reads the results from shared JSON files and manages live trades in real time. Neither process blocks the other — Python can be restarted, retrained, or updated without affecting open positions.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  PYTHON SIDE                                                    │
│                                                                 │
│  unified_predictor_v9.py          main.py                      │
│  (5-model ML ensemble)            (news sentiment pipeline)    │
│                                                                 │
│  Writes every N minutes:          Writes every 10 minutes:     │
│  ea_signal_EURUSD_M5.json         forex_sentiment.json         │
└────────────────┬──────────────────────────┬────────────────────┘
                 │       MQL5\Files          │
┌────────────────▼──────────────────────────▼────────────────────┐
│  METATRADER 5                                                   │
│                                                                 │
│  GGTH_2026_v19.mq5  (Expert Advisor)                          │
│  • Reads signal + sentiment on every tick                       │
│  • Manages entries, averaging, TP/SL, journal                  │
│  • Renders live panel on chart                                  │
└─────────────────────────────────────────────────────────────────┘
```

Communication is **one-way through files**. Python writes; the EA reads. If the signal file goes stale, the built-in watchdog suspends new entries and shows `[WARN]` on the chart panel. The sentiment layer is optional — if `forex_sentiment.json` is missing or stale, all ML signals pass through unmodified.

---

## Features

### Machine Learning
- Five-model ensemble: **LSTM · GRU · Transformer · TCN · LightGBM**
- Three prediction timeframes: **1H / 4H / 1D** trained and published simultaneously
- Per-timeframe **Kalman filter** smoothing on ensemble output
- **Adaptive ensemble weights** — softmax weights updated each cycle based on each model's recent directional accuracy
- **Per-model health tracking** — consecutive-failure counter excludes unhealthy models until manually reset
- **Persisted ensemble state** — weights, prediction history, and health counters survive process restarts
- **Beta-distribution adaptive confidence floor** — updated from live EA trade outcomes
- **HMM regime detection** — classifies market into trending / ranging / volatile; regime bias applied to ensemble weights
- Hyperparameter tuning via **Keras-Tuner**

### Sentiment Analysis
- Async news fetch from **6 free RSS feeds** (no API keys required)
- Optional paid feeds: **NewsAPI · Finnhub · MarketAux**
- Three-model scoring ensemble: **FinBERT (50%) · VADER (30%) · TextBlob (20%)**
- **Time-decay weighting** — article weight halves every 6 hours; articles older than 48 hours discarded
- **Source-quality multipliers** — Reuters 1.0 down to generic RSS 0.6
- Keyword-regex **currency detection** across 8 currencies (USD / EUR / GBP / JPY / CHF / AUD / CAD / NZD)
- Per-pair sentiment derived from base-minus-quote currency scores
- **Atomic file writes** — EA never reads a partially written JSON

### Expert Advisor
- **Sentiment veto** — blocks trades directly opposing confirmed news sentiment (auto-bypassed in Strategy Tester)
- **Three-level averaging down** — each level independently configurable for lots and pip trigger
- **Dollar-amount profit-protection close** — closes entire campaign when combined floating P&L hits target
- **Configurable trailing stop** with activation-pip threshold
- **MA trend filter** (EMA-83 default) and **RSI filter** (14-period)
- **Multi-session trading windows** — up to three independent time windows per day
- **Day-of-week filter** — individually enable/disable each day
- **Adaptive learning** — EA monitors closed-trade outcomes and adjusts signal threshold and lot multiplier within configurable bounds
- **FIFO-compliant** position management (NFA/CFTC)
- **Trade journal CSV** — per-trade decision context written to MQL5\Files for post-session analysis
- **Stale-prediction watchdog** — heartbeat timestamp validated every tick
- **Live on-chart panel** — shows predictions, regime, adaptive stats, and sentiment in real time

---

## Requirements

### Python
| Package | Version | Notes |
|---------|---------|-------|
| Python | 3.9 – 3.11 | **3.12+ not supported** — TF 2.15 requires ≤ 3.11 |
| tensorflow | ==2.15.0 | Pinned — saved models are not cross-version portable |
| keras | ==2.15.0 | Must match TensorFlow exactly |
| protobuf | >=3.19,<4.0 | TF 2.15 requirement; pip will break this silently without the pin |
| lightgbm | >=3.3,<5.0 | |
| keras-tuner | >=1.1,<1.5 | |
| MetaTrader5 | >=5.0.37 | Windows only |
| hmmlearn | >=0.3,<0.4 | Regime detection |
| scipy | >=1.7,<2.0 | hmmlearn dependency; explicit pin prevents silent mismatch |
| numpy | >=1.23.5,<2.0 | |
| pandas | >=1.3,<3.0 | |
| scikit-learn | >=1.0,<2.0 | |
| aiohttp | >=3.9 | Sentiment pipeline async HTTP |
| feedparser | >=6.0 | RSS parsing |
| vaderSentiment | >=3.3 | Sentiment model |
| textblob | >=0.18 | Sentiment model |
| transformers | >=4.40 | FinBERT |
| torch | >=2.2 | FinBERT backend |

> **tkinter** is included with Python on Windows. On Linux: `sudo apt-get install python3-tk`

### MetaTrader 5
- MetaTrader 5 terminal (any broker)
- MetaEditor (included with MT5, press F4)
- EURUSD symbol available in Market Watch

---

## Quick Start

### 1 — Install Python

Download Python **3.10 or 3.11** from [python.org](https://www.python.org/downloads/).

> ⚠ Check **"Add Python to PATH"** during installation. Without this, all batch files fail.

### 2 — Run the Setup Wizard

```bat
setup_wizard.bat
```

The wizard scans `%APPDATA%\MetaQuotes\Terminal\*` for MT5 installations, lets you confirm the correct one, and writes `config.json` atomically. If your MT5 is in a non-standard location, use the manual entry option.

**To find your MT5 Files path manually:**  
MetaTrader 5 → File → Open Data Folder → navigate to `MQL5\Files` → copy the address bar.

### 3 — Compile the EA

1. Press **F4** inside MetaTrader 5 to open MetaEditor
2. Open `GGTH_2026_v19.mq5`
3. Press **F7** to compile — zero errors and zero warnings expected
4. The EA appears in MT5 → Navigator → Expert Advisors → `GGTH_2026_v19`

### 4 — Launch the GUI

```bat
run_ggth_gui.bat
```

On first run this creates a `.venv`, installs all dependencies from `requirements.txt` (5–15 min), starts the sentiment pipeline in a separate console window, and opens the GGTH GUI.

### 5 — Train Models

In the GUI: select `train-multitf`, ensure all five model types are ticked (LSTM, GRU, Transformer, TCN, LightGBM), and click **Start**.

First training run: **20–90 minutes** depending on hardware.

When complete, `ea_signal_EURUSD_M5.json` appears in your `MQL5\Files` folder.

### 6 — Attach the EA

1. Open an **EURUSD M5** chart in MT5
2. Drag `GGTH_2026_v19` from Navigator onto the chart
3. Configure inputs (see [EA Input Settings](#ea-input-settings))
4. Common tab → tick **Allow algorithmic trading**
5. Click OK — a smiley face in the chart corner confirms the EA is active

---

## Repository Layout

```
GGTH/
│
├── setup_wizard.bat           # ★ First-time installer entry point
├── setup_wizard.py            # GUI installer (launched by wizard.bat)
├── run_ggth_gui.bat           # Daily launcher
│
├── ggth_gui.py                # Main GUI — tkinter control panel (v2.3)
├── unified_predictor_v9.py    # ML prediction engine (v9.5)
├── config_manager.py          # config.json read/write with schema validation (v2.2)
├── model_builders.py          # LSTM / GRU / Transformer / TCN builders (v2.2)
├── logger.py                  # Thread-safe rotating file + console logger (v2.2)
│
├── main.py                    # Sentiment pipeline entry point (async loop)
├── forex_sentiment.py         # News → ensemble → per-currency → per-pair aggregation
├── news_fetcher.py            # Async RSS + optional API fetcher (aiohttp)
├── sentiment_models.py        # FinBERT / VADER / TextBlob ensemble
├── sentiment_reader_py.py     # Reads forex_sentiment.json; veto/bias logic
├── sentiment_writer.py        # Atomic JSON writer (temp-file + os.replace)
├── currency_mapper.py         # Keyword regex → currency detection
│
├── config.py                  # Sentiment pipeline settings (edit directly)
├── config.json                # Written by setup wizard — do not hand-edit
├── requirements.txt           # Pinned package versions
│
├── ggth_predictor.log         # Auto-created rotating log (5 MB × 3 backups)
├── .venv/                     # Auto-created virtual environment
│
└── MQL5/ (MetaTrader 5)
    └── Experts/
        └── GGTH_2026_v19.mq5  # Expert Advisor source
```

**Files written to `MQL5\Files` at runtime:**

| File | Written by | Read by |
|------|-----------|---------|
| `ea_signal_EURUSD_M5.json` | `unified_predictor_v9.py` | EA every tick |
| `ea_signal_EURUSD_H1.json` | `unified_predictor_v9.py` | EA every tick |
| `ea_signal_EURUSD_H4.json` | `unified_predictor_v9.py` | EA every tick |
| `forex_sentiment.json` | `main.py` | EA every tick |
| `EURUSD_trade_outcomes.json` | EA | `unified_predictor_v9.py` |
| `EURUSD_trade_journal.csv` | EA | User (post-session review) |
| `ensemble_state_EURUSD.json` | `unified_predictor_v9.py` | `unified_predictor_v9.py` on restart |

---

## ML Engine

`unified_predictor_v9.py` — the core prediction engine.

### Model Ensemble

| Model | Type | Specialization |
|-------|------|---------------|
| LSTM | Bidirectional LSTM + Attention | Long-range sequential dependencies |
| GRU | Bidirectional GRU (2 layers) | Shorter-range sequences; faster training |
| Transformer | Multi-head self-attention (TransformerBlock) | Non-local patterns; multi-scale correlations |
| TCN | Temporal Convolutional Network with residual skip | Trend regimes; efficient parallel training |
| LightGBM | Gradient boosted trees | Non-linear feature interactions; fast inference |

### Prediction Cycle

1. Fetch OHLCV bars from MT5 via the MetaTrader5 Python package
2. Build feature matrix (price transforms, technical indicators, regime flags)
3. Run inference on all five models independently per timeframe
4. Combine predictions using per-timeframe softmax ensemble weights
5. Apply per-timeframe 1-D Kalman filter to smooth output
6. Write flat `ea_signal_*.json` files atomically to `MQL5\Files`
7. Read back EA trade outcomes; update Beta-distribution adaptive confidence floor
8. Update ensemble weights based on directional accuracy; persist state

### Ensemble Weight Learning

Weights are updated using a **softmax-over-inverse-error** scheme. Models with lower recent directional error receive higher weight. The update is identity-aligned — a slot per model exists in every cycle regardless of failures, so a failing model cannot accidentally inherit another model's error metric. Models exceeding 5 consecutive failures are excluded from the cycle until the health file is reset.

### Regime Detection

A **Gaussian HMM** is fitted at the end of training and persisted alongside model weights. Each prediction cycle classifies the current market into one of three regimes (trending / ranging / volatile) and applies a bias to ensemble model weights accordingly. Falls back to a heuristic ATR/volatility classifier if the HMM file is absent.

### GUI Modes

| Mode | Description |
|------|-------------|
| `train-multitf` | Train all five models across 1H / 4H / 1D in one pass (**recommended**) |
| `train` | Train on the primary timeframe only |
| `predict-mtf-cont` | Continuous multi-timeframe prediction at configured interval |
| `predict-mtf-once` | Single prediction cycle then exit |
| `backtest` | Generate historical signals over a date range |
| `safe-backtest` | Backtest using only training-time scalers (no data leakage) |
| `tune` | Keras-Tuner hyperparameter search (GPU recommended) |
| `report` | Print metrics summary without running a cycle |

---

## Sentiment Pipeline

`main.py` + `forex_sentiment.py` — runs as a separate background process.

### Cycle (every 10 minutes)

```
RSS feeds + optional APIs
        │
        ▼
  news_fetcher.py  ──────────────────────────────────► up to ~300 articles/cycle
        │
        ▼
  currency_mapper.py  ──────────────────────────────► which currencies does each article mention?
        │
        ▼
  sentiment_models.py  (Ensemble.score)
    ├── FinBERT   50%  (finance-domain BERT, ~400 MB download on first run)
    ├── VADER     30%  (rule-based lexicon, no download)
    └── TextBlob  20%  (general NLP, no download)
        │
        ▼
  forex_sentiment.py  (aggregate + derive_pairs)
    • time-decay weighting  (half-life 6 h, max-age 48 h)
    • source-quality multipliers
    • per-currency weighted average
    • per-pair score = (base_score − quote_score) / 2
        │
        ▼
  sentiment_writer.py  ──────────────────────────────► forex_sentiment.json  (atomic write)
```

### Supported News Sources

| Source | Type | Quality weight |
|--------|------|---------------|
| FXStreet | Free RSS | 0.85 |
| ForexLive | Free RSS | 0.85 |
| MarketWatch | Free RSS | 0.85 |
| DailyFX | Free RSS | 0.80 |
| Investing.com | Free RSS | 0.75 |
| Finnhub | API (free tier) | 0.80 |
| NewsAPI | API (free tier) | 0.70 |
| Reuters (via NewsAPI) | API | 1.00 |

No API keys are required for basic operation.

### Sentiment Modes (EA-side)

| Mode | Behaviour |
|------|-----------|
| `veto` | Block trades when `\|score\| > InpSentimentVetoBand` and signal opposes sentiment |
| `bias` | Blend sentiment into signal strength (configured in `sentiment_reader_py.py`) |
| `off` | Display sentiment panel only; no trades blocked |

Fail-open design: missing file, stale snapshot (> `InpSentimentMaxAgeSec`), or low confidence (< `InpSentimentMinConf`) all result in all ML signals passing through unmodified.

---

## Expert Advisor

`GGTH_2026_v19.mq5` — compiled and attached to an EURUSD M5 chart.

### Trade Lifecycle

```
OnTick()
  │
  ├── ReadSentimentFile()         parse forex_sentiment.json (mtime-guarded)
  ├── ReadSignalFile()            parse ea_signal_EURUSD_M5.json
  ├── Watchdog check              stale? → suspend new entries, show [WARN]
  │
  ├── IsTradingAllowed()
  │     ├── Day-of-week filter
  │     ├── Session window filter  (up to 3 independent windows)
  │     ├── Spread limit
  │     └── Margin utilization limit
  │
  ├── Signal gates (applied in order)
  │     ├── InpMinPredictionPips  (minimum predicted move)
  │     ├── MA trend filter       (EMA-83 default)
  │     ├── RSI filter            (14-period, 75/35 thresholds)
  │     └── Sentiment veto        (auto-bypassed in Strategy Tester)
  │
  ├── Open initial position       (if no campaign open and signal passes all gates)
  │
  ├── Manage open campaign
  │     ├── Averaging down        (Level 1 @ 15 pip, Level 2 @ 35 pip, Level 3 disabled)
  │     ├── Profit-protection close  (all legs when combined P&L ≥ InpProfitTargetAmount)
  │     ├── Trailing stop         (if enabled, engages after InpTrailingActivationPips)
  │     ├── Stop-loss / TP        (shared SL from blended average entry)
  │     └── Max hold time         (force-close if campaign age > InpMaxHoldHours)
  │
  ├── Adaptive learning           (every InpAdaptEveryN closed campaigns)
  │     ├── Compute rolling win rate and profit factor over last N campaigns
  │     ├── Blend-adjust InpMinPredictionPips and lot multiplier
  │     └── Write outcome to ea_signal file for Python to consume
  │
  └── DrawPanel()                 update on-chart display
```

### FIFO Compliance

`InpFIFOCompliant = true` enforces NFA/CFTC first-in-first-out position closing. Required for US-regulated brokers. Disable only for non-US brokers where hedging is permitted.

### Strategy Tester

Set `InpStrategyTesterMode = true` when backtesting. The EA reads predictions from pre-exported CSV files rather than live JSON, and the sentiment veto is automatically disabled (historical sentiment data is not available in the tester).

---

## Configuration Reference

### config.json

Written by the setup wizard. Contains exactly two keys. Do not add others.

```json
{
  "mt5_files_path": "C:\\Users\\YourName\\AppData\\Roaming\\MetaQuotes\\Terminal\\<hash>\\MQL5\\Files",
  "version": "2.3"
}
```

| Key | Description |
|-----|-------------|
| `mt5_files_path` | Full path to `MQL5\Files` for the terminal your EA is attached to |
| `version` | Schema version gate — must be `"2.3"`. Do not edit. |

To update the path: re-run `setup_wizard.bat`, or edit the MT5 Files field in the GUI.

### config.py (Sentiment Pipeline)

Edit directly in a text editor. Restart the sentiment pipeline window for changes to take effect.

```python
mt5_files_path          = r"C:\Users\...\MQL5\Files"  # must match config.json
output_filename         = "forex_sentiment.json"
update_interval_seconds = 600          # how often the pipeline re-runs (10 min)
half_life_hours         = 6.0          # article weight halves every 6 hours
max_age_hours           = 48.0         # discard articles older than this
newsapi_key             = ""           # optional — newsapi.org free tier (100/day)
finnhub_key             = ""           # optional — finnhub.io free tier (60/min)
marketaux_key           = ""           # optional — marketaux.com free tier (100/day)

ensemble_weights = {
    "finbert":  0.50,
    "vader":    0.30,
    "textblob": 0.20,
}
```

---

## EA Input Settings

Full reference for all inputs in the MT5 EA dialog. Grouped as they appear in the Inputs tab.

<details>
<summary><strong>Testing Mode</strong></summary>

| Parameter | Default | Description |
|-----------|---------|-------------|
| `InpStrategyTesterMode` | `false` | Read predictions from CSV files instead of live JSON. Required for Strategy Tester backtests. |

</details>

<details>
<summary><strong>Trading Setup</strong></summary>

| Parameter | Default | Description |
|-----------|---------|-------------|
| `InpSymbol` | `EURUSD` | Trading symbol. Must match Market Watch name exactly. |
| `InpTradingTimeframe` | `PERIOD_H1` | Prediction timeframe the EA selects from the multi-timeframe signal file. |
| `InpEnableTrading` | `true` | Master switch. `false` = monitor only, no trades opened. |
| `InpLotMode` | `LOT_MODE_FIXED` | `FIXED`: use `InpFixedLot`. `RISK`: calculate lots from `InpRiskPercent`. |
| `InpFixedLot` | `0.10` | Base lot size (FIXED mode). Adaptive multiplier scales this up or down. |
| `InpRiskPercent` | `1.0` | Percent of balance risked per trade (RISK mode). |
| `InpMagic` | `20260522` | Magic number stamped on all EA orders. |
| `InpFIFOCompliant` | `true` | Enforce FIFO closing order. Required for NFA/US brokers. |

</details>

<details>
<summary><strong>Institutional Filters</strong></summary>

| Parameter | Default | Description |
|-----------|---------|-------------|
| `InpMaxMarginUsagePct` | `60.0` | Block new entries when used margin exceeds this % of equity. |

</details>

<details>
<summary><strong>Strategy & Signal</strong></summary>

| Parameter | Default | Description |
|-----------|---------|-------------|
| `InpMinPredictionPips` | `17.0` | Minimum predicted price move (pips) to open a trade. Adapted dynamically by the adaptive learning system. |

</details>

<details>
<summary><strong>Averaging Down</strong></summary>

| Parameter | Default | Description |
|-----------|---------|-------------|
| `InpUseAveragingDown` | `true` | Enable multi-level averaging. |
| `InpAvgLevel1Lots` | `0.20` | Lot size for averaging entry Level 1. |
| `InpAvgLevel1Pips` | `15` | Adverse pips from initial entry before Level 1 fires. |
| `InpAvgLevel2Lots` | `0.20` | Lot size for Level 2. |
| `InpAvgLevel2Pips` | `35` | Adverse pips before Level 2 fires. |
| `InpAvgLevel3Lots` | `0.30` | Lot size for Level 3. |
| `InpAvgLevel3Pips` | `10000` | Set to `10000` to disable Level 3 (default). |

> ⚠ All averaging levels share one stop-loss from the blended average entry. Large Level 3 lots can expose significantly more capital than the initial trade.

</details>

<details>
<summary><strong>Profit Protection</strong></summary>

| Parameter | Default | Description |
|-----------|---------|-------------|
| `InpUseProfitProtection` | `true` | Close all campaign legs when combined floating P&L hits target. |
| `InpMinPositionsForProtection` | `2` | Minimum open legs before the profit target is checked. |
| `InpProfitTargetAmount` | `27.00` | Combined floating profit (USD) to trigger full-campaign close. |

</details>

<details>
<summary><strong>Max Hold Time</strong></summary>

| Parameter | Default | Description |
|-----------|---------|-------------|
| `InpUseMaxHoldTime` | `true` | Enable time-based force-close. |
| `InpMaxHoldHours` | `10` | Force-close any campaign open longer than this many hours. |

</details>

<details>
<summary><strong>Market Context Veto</strong></summary>

| Parameter | Default | Description |
|-----------|---------|-------------|
| `InpUseMarketContextVeto` | `false` | Block entries during ATR spikes or abnormal candle moves. |
| `InpVolatilitySpikeMultiplier` | `2.5` | Block when current ATR exceeds N× the recent average. |
| `InpMaxCandleChangePercent` | `0.5` | Block when current candle move exceeds this % of price. |
| `InpVolatilityLookback` | `20` | Bars used to compute average ATR for comparison. |

</details>

<details>
<summary><strong>Take Profit & Stop Loss</strong></summary>

| Parameter | Default | Description |
|-----------|---------|-------------|
| `InpUsePredictedPrice` | `true` | Use ML predicted target price as TP (recommended). |
| `InpStopLossPips` | `60` | Stop-loss distance in pips from blended average entry. |
| `InpTakeProfitPips` | `200` | Fixed TP in pips when `InpUsePredictedPrice = false`. |
| `InpTPMultiplier` | `1.0` | Multiplier applied to predicted TP distance. |
| `InpMinTPPips` | `17` | Minimum TP distance (clamped up if prediction is smaller). |
| `InpMaxTPPips` | `70` | Maximum TP distance (clamped down if prediction is larger). |

</details>

<details>
<summary><strong>Trend Filter</strong></summary>

| Parameter | Default | Description |
|-----------|---------|-------------|
| `InpUseTrendFilter` | `true` | BUY only above MA; SELL only below MA. |
| `InpTrendMAPeriod` | `83` | MA period. |
| `InpTrendMAMethod` | `MODE_EMA` | MA type: EMA / SMA / SMMA / LWMA. |
| `InpTrendMAPrice` | `PRICE_CLOSE` | Applied price for MA calculation. |

</details>

<details>
<summary><strong>RSI Filter</strong></summary>

| Parameter | Default | Description |
|-----------|---------|-------------|
| `InpUseRSIFilter` | `true` | Block overbought BUY and oversold SELL signals. |
| `InpRSIPeriod` | `14` | RSI calculation period. |
| `InpRSIOverbought` | `75.0` | RSI level above which BUY signals are blocked. |
| `InpRSIOversold` | `35.0` | RSI level below which SELL signals are blocked. |

</details>

<details>
<summary><strong>Trailing Stop</strong></summary>

| Parameter | Default | Description |
|-----------|---------|-------------|
| `InpUseTrailingStop` | `false` | Enable trailing stop. |
| `InpTrailingStopPips` | `12` | Distance behind price the stop follows (pips). |
| `InpTrailingStepPips` | `5` | Minimum price movement before stop is moved. |
| `InpTrailingActivationPips` | `25` | Minimum profit in pips before trailing engages. Set `0` to engage immediately. |

</details>

<details>
<summary><strong>Trading Days</strong></summary>

| Parameter | Default |
|-----------|---------|
| `InpTradeMonday` | `true` |
| `InpTradeTuesday` | `true` |
| `InpTradeWednesday` | `true` |
| `InpTradeThursday` | `true` |
| `InpTradeFriday` | `true` |
| `InpTradeSaturday` | `false` |
| `InpTradeSunday` | `false` |

</details>

<details>
<summary><strong>Trading Sessions (up to 3 windows)</strong></summary>

| Parameter | Default | Description |
|-----------|---------|-------------|
| `InpUseSession1` | `true` | Enable Session 1. |
| `InpSession1StartHour` | `0` | Session 1 start (server time, 24h). |
| `InpSession1EndHour` | `17` | Session 1 end. Default covers London + NY overlap. |
| `InpUseSession2` | `false` | Enable Session 2 (e.g. Asian session). |
| `InpUseSession3` | `false` | Enable Session 3. |

All times are broker **server time** as shown in MT5's bottom-right clock.

</details>

<details>
<summary><strong>Display Settings</strong></summary>

| Parameter | Default | Description |
|-----------|---------|-------------|
| `InpFontSize` | `14` | Panel font size in points. |
| `InpLineSpacing` | `12` | Extra vertical spacing (px) between rows. |
| `InpPanelPadding` | `10` | Inner padding (px) between panel border and content. |
| `InpTextColor` | `clrWhite` | Standard text color. |
| `InpUpColor` | `clrLimeGreen` | Color for UP predictions and bullish labels. |
| `InpDownColor` | `clrRed` | Color for DOWN predictions and bearish labels. |
| `InpXOffset` | `20` | Horizontal pixel offset from left edge. |
| `InpYOffset` | `30` | Vertical pixel offset from top. |
| `InpShowDebug` | `true` | Show diagnostic rows (file age, veto reasons). Disable for cleaner production display. |

</details>

<details>
<summary><strong>Adaptive Learning</strong></summary>

| Parameter | Default | Description |
|-----------|---------|-------------|
| `InpEnableAdaptiveLearning` | `true` | Enable online adaptive parameter adjustment. |
| `InpAdaptLookback` | `20` | Campaigns in the rolling evaluation window. |
| `InpAdaptEveryN` | `5` | Trigger an adaptation step every N closed campaigns. |
| `InpAdaptRate` | `0.15` | Blend rate for threshold/multiplier updates (0.05=slow, 0.30=fast). |
| `InpAdaptMinPredFloor` | `1.0` | Hard lower bound on adapted signal threshold (pips). |
| `InpAdaptMinPredCeil` | `60.0` | Hard upper bound on adapted signal threshold (pips). |
| `InpAdaptLotMultFloor` | `0.25` | Minimum adaptive lot multiplier. |
| `InpAdaptLotMultCeil` | `1.50` | Maximum adaptive lot multiplier. |
| `InpAdaptResetOnInit` | `false` | Reset all learned params on EA init. Use after changing base lot size or SL. |
| `InpShowAdaptiveDebug` | `true` | Show adaptive stats row in panel. |

</details>

<details>
<summary><strong>Stale Prediction Watchdog</strong></summary>

| Parameter | Default | Description |
|-----------|---------|-------------|
| `InpEnableStaleWatchdog` | `true` | Suspend new entries if signal file is stale. |
| `InpStalePredictionMaxMinutes` | `90` | Maximum signal file age before watchdog fires. |
| `InpFailClosedOnMissingHeartbeat` | `false` | Block trading if no heartbeat received since EA init. |
| `InpShowWatchdogStatusOnChart` | `true` | Show `[OK]` / `[WARN]` banner on chart. |

</details>

<details>
<summary><strong>Trade Journal</strong></summary>

| Parameter | Default | Description |
|-----------|---------|-------------|
| `InpEnableTradeJournal` | `true` | Write per-trade decision context to CSV. |
| `InpJournalAveragingTrades` | `true` | Also journal averaging-down legs. |
| `InpShowJournalDebug` | `false` | Print journal write events to MT5 Experts log. |

</details>

<details>
<summary><strong>Sentiment Filter</strong></summary>

| Parameter | Default | Description |
|-----------|---------|-------------|
| `InpUseSentiment` | `true` | Enable sentiment veto (auto-disabled in Strategy Tester). |
| `InpSentimentFile` | `forex_sentiment.json` | Sentiment JSON filename in MT5 Common Files. |
| `InpSentimentMinConf` | `0.20` | Minimum confidence before sentiment is acted on. |
| `InpSentimentMaxAgeSec` | `1800` | Maximum snapshot age (seconds) before it is ignored (30 min). |
| `InpSentimentVetoBand` | `0.10` | Minimum `\|score\|` required to veto. Scores in (−0.10, +0.10) never block. |
| `InpShowSentimentPanel` | `true` | Show sentiment section in on-chart panel. |

</details>

---

## GUI Reference

`ggth_gui.py` is a tkinter control panel that builds and runs the predictor CLI command with the selected options.

### Key Controls

| Control | Description |
|---------|-------------|
| Symbol | Trading symbol to train/predict (default EURUSD) |
| Action | Predictor mode (see [GUI Modes](#gui-modes)) |
| Force Retrain | Overwrite existing saved models |
| Interval (min) | Prediction cycle interval for continuous modes |
| Models | Toggle each of the five model types independently |
| Kalman Filter | Apply per-timeframe Kalman smoothing to predictions |
| Python exe | Path to Python interpreter (defaults to active venv) |
| Predictor | Path to `unified_predictor_v9.py` |
| MT5 Files | Path to `MQL5\Files` — saved to `config.json` on click of **Save MT5 Path** |
| Train Start / End | Optional date window for training data |
| Predict Start / End | Optional date window for prediction/backtest |
| **Start** | Build CLI command and launch subprocess |
| **Stop** | Graceful shutdown: CTRL_BREAK_EVENT → 5s wait → terminate |

Output streams to the scrollable log area in real time. **Save Log** writes the current log area content to a timestamped `.txt` file.

---

## On-Chart Panel

```
GGTH PREDICTOR v1.19  │  EURUSD
Price: 1.08432    Regime: trending
────────────────────────────────────────────────────────
[OK] Watchdog  age=2m / limit=90m
────────────────────────────────────────────────────────
PREDICTIONS
1H  UP    1.08500  (+0.06%)     Acc: 45/72  (62.5%)
4H  DOWN  1.08100  (-0.31%)     Acc: 12/20  (60.0%)
1D  UP    1.09200  (+0.71%)     Acc:  5/8   (62.5%)
────────────────────────────────────────────────────────
ADAPTIVE LEARNING
WR: 58.3%  PF: 1.42  Kelly: 16.2%  LotMult: 0.95x
AdaptPred: 18.4 pips  #adapt: 5  Trades: 47
────────────────────────────────────────────────────────
NEWS SENTIMENT
BULLISH  score: +0.241  conf: 68%
Base: +0.312  Quote: -0.071  Age: 4m
SELL signals may be vetoed
```

| Row | Description |
|-----|-------------|
| Header | EA version, symbol, current price, detected regime |
| Watchdog | `[OK]` or `[WARN]` with current file age vs configured maximum |
| Predictions | Per-timeframe: direction, predicted target price, % move, running accuracy |
| Adaptive Learning | Win rate, profit factor, Kelly %, lot multiplier, adapted pip threshold, trade count |
| News Sentiment | Direction label, score (−1 to +1), confidence %, base/quote scores, snapshot age, active veto message |

---

## Troubleshooting

**Panel shows `[STALE]` or `[WARN]`**
- Python predictor has stopped writing fresh signal files
- Restart via `run_ggth_gui.bat`
- Check `ggth_predictor.log` for import errors or MT5 connection failures
- Verify MT5 is open and logged in — the predictor requires MT5 to fetch price data

**Sentiment section shows "No data"**
- `forex_sentiment.json` has not been written yet or is in the wrong folder
- Confirm `main.py` is running in the separate console opened by `run_ggth_gui.bat`
- On first run, FinBERT downloads ~400 MB — wait for completion
- Confirm `output_filename` in `config.py` matches `InpSentimentFile` in EA settings

**EA opens no trades despite an active signal**
- Check `InpEnableTrading = true`
- Predicted pip move may be below `InpMinPredictionPips`
- Confirm current server time is within an enabled session
- Check `InpMaxMarginUsagePct` — margin may be over the limit
- Check the sentiment panel for an active veto message
- Verify the Common tab → **Allow algorithmic trading** is ticked

**Models fail to load after a package upgrade**
- Keras 2.15 saved models cannot be loaded with a different Keras major version
- Restore: `.venv\Scripts\activate` then `pip install -r requirements.txt`
- If you intentionally upgraded, delete saved models and retrain via `train-multitf`

**`config.json` unknown-key warning in log**
- An older setup wizard wrote keys that v2.3 no longer reads — harmless
- To clean up: delete `config.json` and re-run `setup_wizard.bat`

**MT5 connection errors during training**
- Ensure MT5 is open and logged in
- The MetaTrader5 Python package requires 64-bit Python matching your 64-bit MT5 terminal
- MT5: Tools → Options → Expert Advisors → Allow algorithmic trading

---

## License

Copyright 2026 Jason Rusk. All rights reserved.

---

## Author

**Jason Rusk** — jason.w.rusk@gmail.com
