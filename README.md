Thanks for the thorough review and for flagging the mismatch between the published backtest results and what you’re seeing in the Docker environment. I’ve gone through each of your questions and tested the live setup locally. Below is a detailed breakdown of what’s happening and how I plan to bring the live runs back in line with the backtests.

---

## 1. What i understood

- The same code ships to both the backtester and the Docker containers; there’s no hidden logic gap.  
- The drift comes from runtime inputs—today’s market structure, the parameters being used inside the container, and whether the bot receives enough history to start.  
- 1st strategy stayed flat because the enhanced parameter set i used for the +20.64% backtest only triggers when the market is cleanly trending. In the current sideways regime, those filters block every entry.  
- 2nd Strategy is stuck on “Insufficient history” because the container isn’t finishing the preload of candles (either the fetch times out or the configured history depth is shorter than the 288-bar requirement), so it never leaves the warm-up guard.

---

### 2.1 Data Source Differences

| What i am comparing | Backtest | Live container | Why it matters |
| --- | --- | --- | --- |
| Market feed | Hourly bars from Yahoo Finance (`yfinance`) | Live Coinbase data via the template’s `PaperExchange` | Slight differences in sampling/adjustment. Yahoo is smoothed; live data reflects every intraday wiggle. |
| Period | Jan–Jun 2024 (strong uptrend) | November 2025 (currently sideways) | Entry gates tuned for a trending tape rarely fire in a range-bound tape. |
| Warm-up | It load the whole dataset before the first signal | It need ~300 bars on startup for the 288 SMA. If preload fails, the bot waits indefinitely | Explains why Swing Grid keeps reporting “Insufficient history.” |

### 2.2 Configuration Differences

- **Adaptive Momentum backtest** – ran with `configs/enhanced_params.json`, a very aggressive profile (RSI buy ≥ 60, min trend slope 0.3%, max volatility 7%). That profile isn’t applied automatically in the container unless we set `BOT_STRATEGY_PARAMS` to that JSON.  
- **Adaptive Momentum live** – if the container runs with defaults, those are still strict (RSI 55, min trend slope 0.6%) and will block trades in a sideways regime. My smoke test using today’s tape showed repeated holds with `reason = trend_slope` or `volatility_ceiling`.  
- **Swing Grid** – needs `BOT_HISTORY` ≥ 450 and matching `preload_history_bars` so the 288-period SMA has enough data. If runtime sticks with 200, the guard never clears.

### 2.3 Implementation Differences

- There aren’t any. The code packaged in the Docker image matches the repo files you reviewed:  
  - `adaptive-momentum-submission/your-strategy-template/your_strategy.py`  
  - `swing-grid-submission/your-strategy-template/your_strategy.py`
- I validated this by rebuilding the Docker image and comparing checksums.

### 2.4 Environment/Dependency Issues

- Dependencies inside the containers match the backtest environment. No missing packages turned up in logs.  
- The only external dependency that can block it is the history fetch during `prepare()`. If the container launches before networking is ready or the API throttles it, the preload fails and the strategy never leaves the warm-up guard.  
- Cooldown/state handling is already fixed: `_scaled_out` and `_position_opened_at` reset properly after trades.

---

## 3. What’s Really Happening

### (Strategy 1)

1. **Aggressive gates:** The numbers that delivered +20.64% require clean momentum: `min_trend_slope = 0.003`, `rsi_buy = 60`, `max_volatility = 0.07`. In a sideways market those gates never pass, so the bot never buys. Logs show hold reasons like `trend_slope` or `volatility_ceiling`.
2. **Cooldown/time-stop:** Even if we got in, the bot sits in a 60-minute cooldown plus a 60-hour time stop. That’s fine in a trend, but when no entry happens the bot looks dead.
3. **Market mismatch:** January–June 2024 was trending; November 2025 isn’t. The backtest profile doesn’t match the live regime.

### (Strategy 2)

1. **History preload failed:** It need at least 300–450 bars before the SMA/ATR calculations make sense. If `fetch_market_snapshot` returns fewer candles or times out, `generate_signal` keeps saying “Insufficient history.”
2. **Runtime history length:** The template default is 200 bars. Our backtest uses ≥ 420. If Ops launches the container without overriding `BOT_HISTORY`, it never clears the warm-up.
3. **Logs confirm it:** In my runs, a successful preload logs `preload_bars=450`. Your Docker logs don’t show that, so the warm-up never finished.

---

## 4. Fix Plan

### (Strategy 1)

1. **Align parameters now:** In the container, set `BOT_STRATEGY_PARAMS` to the contents of `configs/enhanced_params.json` if you want the same aggressive profile. If you prefer to adapt to the current market, lower the gates a bit (for example `min_trend_slope: 0.0015`, `rsi_buy: 57`, `momentum_threshold: 0.7`, `max_volatility: 0.09`) so it still trade without blowing up risk.
2. **Watch the logs:** `last_signal_data` already prints the hold reasons. If you still see `trend_slope` or `volatility_ceiling` constantly, it need to loosen those gates for the present regime.
3. **Roadmap:** I’ll add a two-profile system (trending vs sideways) so the bot toggles automatically based on ADX or slope variance.

### (Strategy 2)

1. **Make sure enough history loads:** Launch with `BOT_HISTORY = 450` and `BOT_STRATEGY_PARAMS = '{"preload_history_bars": 450, "min_history_bars": 288}'`. That mirrors the backtest.
2. **Verify preload:** Wait for the log line reporting `preload_bars`. If it’s missing, the fetch failed; restart after verifying connectivity or I can add retry logic to harden this.
3. **Operator checklist:** README now calls out the warm-up requirement. Let’s share a short pre-flight list (see next section) so Ops can confirm everything on launch.
