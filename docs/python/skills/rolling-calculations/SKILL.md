---
name: rolling-calculations
description: Calculate rolling windows, cumulative values, and moving averages using update_by operators
---

Help the user create rolling calculations, moving averages, EMAs, and cumulative statistics.

## When to use this skill

- User mentions "rolling average", "moving average", "MA"
- User mentions "EMA", "exponential moving average", "weighted average"
- User asks for "cumulative sum", "running total", "YTD"
- User wants "trailing" calculations (trailing 20 days, last N rows)
- User needs "lag", "lead", "previous row", "delta"
- User mentions "VWAP", "rolling VWAP"

## Quick answers

### Simple moving average (tick-based)

```python syntax
from deephaven.updateby import rolling_avg_tick

# 20-period moving average
result = table.update_by(rolling_avg_tick("MA20=Price", rev_ticks=20), by="Symbol")
```

### Time-based rolling average

```python syntax
from deephaven.updateby import rolling_avg_time

# 5-minute rolling average
result = table.update_by(
    rolling_avg_time("Timestamp", "MA5Min=Price", rev_time="PT5M"), by="Symbol"
)
```

### Exponential moving average (EMA)

```python syntax
from deephaven.updateby import ema_tick, ema_time

# EMA with decay rate (ticks)
result = table.update_by(ema_tick(decay_ticks=20, cols="EMA20=Price"), by="Symbol")

# EMA with time decay
result = table.update_by(
    ema_time("Timestamp", decay_time="PT5M", cols="EMA5Min=Price"), by="Symbol"
)
```

### Cumulative sum

```python syntax
from deephaven.updateby import cum_sum

result = table.update_by(cum_sum("CumulativeVolume=Volume"), by="Symbol")
```

### Previous/next row (lag/lead)

```python syntax
from deephaven.updateby import delta

# Change from previous row
result = table.update_by(delta("PriceChange=Price"), by="Symbol")

# For explicit lag, use forward_fill with rolling_group_tick
```

## Core update_by operators

### Cumulative operators

```python syntax
from deephaven.updateby import cum_sum, cum_prod, cum_min, cum_max

result = table.update_by(
    [
        cum_sum("CumVolume=Volume"),
        cum_prod("CumReturn=DailyReturn"),  # For compound returns
        cum_min("RunningLow=Price"),
        cum_max("RunningHigh=Price"),
    ],
    by="Symbol",
)
```

### Rolling operators (tick-based)

```python syntax
from deephaven.updateby import (
    rolling_sum_tick,
    rolling_avg_tick,
    rolling_min_tick,
    rolling_max_tick,
    rolling_std_tick,
    rolling_count_tick,
)

result = table.update_by(
    [
        rolling_sum_tick("RollingVol20=Volume", rev_ticks=20),
        rolling_avg_tick("MA20=Price", rev_ticks=20),
        rolling_min_tick("Low20=Price", rev_ticks=20),
        rolling_max_tick("High20=Price", rev_ticks=20),
        rolling_std_tick("Volatility20=Price", rev_ticks=20),
    ],
    by="Symbol",
)
```

### Rolling operators (time-based)

```python syntax
from deephaven.updateby import (
    rolling_sum_time,
    rolling_avg_time,
    rolling_min_time,
    rolling_max_time,
    rolling_std_time,
    rolling_count_time,
)

result = table.update_by(
    [
        rolling_sum_time("Timestamp", "Vol5Min=Volume", rev_time="PT5M"),
        rolling_avg_time("Timestamp", "MA5Min=Price", rev_time="PT5M"),
        rolling_std_time("Timestamp", "Volatility5Min=Price", rev_time="PT5M"),
        rolling_count_time("Timestamp", "Ticks5Min=Price", rev_time="PT5M"),
    ],
    by="Symbol",
)
```

### EMA operators

```python syntax
from deephaven.updateby import ema_tick, ema_time, ems_tick, ems_time

# Exponential moving average
result = table.update_by(
    [
        ema_tick(decay_ticks=12, cols="EMA12=Price"),
        ema_tick(decay_ticks=26, cols="EMA26=Price"),
    ],
    by="Symbol",
)

# Then calculate MACD
result = result.update("MACD = EMA12 - EMA26")

# Exponential moving sum (for weighted volumes, etc.)
result = table.update_by(ems_tick(decay_ticks=20, cols="EMS20=Volume"), by="Symbol")
```

### Delta (change from previous)

```python syntax
from deephaven.updateby import delta

result = table.update_by(
    [delta("PriceChange=Price"), delta("VolumeChange=Volume")], by="Symbol"
)

# Percentage change
result = result.update("PctChange = PriceChange / (Price - PriceChange)")
```

## Common patterns

### VWAP (Volume-Weighted Average Price)

```python syntax
from deephaven.updateby import cum_sum

# Cumulative VWAP
result = table.update_by(
    [cum_sum("CumPriceVol=Price*Volume"), cum_sum("CumVolume=Volume")], by="Symbol"
).update("VWAP = CumPriceVol / CumVolume")

# Rolling VWAP
from deephaven.updateby import rolling_sum_tick

result = table.update_by(
    [
        rolling_sum_tick("RollingPriceVol=Price*Volume", rev_ticks=20),
        rolling_sum_tick("RollingVol=Volume", rev_ticks=20),
    ],
    by="Symbol",
).update("VWAP20 = RollingPriceVol / RollingVol")
```

### Bollinger Bands

```python syntax
from deephaven.updateby import rolling_avg_tick, rolling_std_tick

result = table.update_by(
    [
        rolling_avg_tick("MA20=Price", rev_ticks=20),
        rolling_std_tick("Std20=Price", rev_ticks=20),
    ],
    by="Symbol",
).update(["UpperBand = MA20 + 2 * Std20", "LowerBand = MA20 - 2 * Std20"])
```

### RSI (Relative Strength Index)

```python syntax
from deephaven.updateby import delta, ema_tick

result = (
    table.update_by(delta("Change=Price"), by="Symbol")
    .update(["Gain = Change > 0 ? Change : 0", "Loss = Change < 0 ? -Change : 0"])
    .update_by(
        [
            ema_tick(decay_ticks=14, cols="AvgGain=Gain"),
            ema_tick(decay_ticks=14, cols="AvgLoss=Loss"),
        ],
        by="Symbol",
    )
    .update(["RS = AvgGain / AvgLoss", "RSI = 100 - (100 / (1 + RS))"])
)
```

### MACD

```python syntax
from deephaven.updateby import ema_tick

result = (
    table.update_by(
        [
            ema_tick(decay_ticks=12, cols="EMA12=Price"),
            ema_tick(decay_ticks=26, cols="EMA26=Price"),
        ],
        by="Symbol",
    )
    .update("MACD = EMA12 - EMA26")
    .update_by(ema_tick(decay_ticks=9, cols="Signal=MACD"), by="Symbol")
    .update("Histogram = MACD - Signal")
)
```

### Rolling correlation

```python syntax
from deephaven.updateby import rolling_wavg_tick

# Approximation using rolling calculations
# For exact correlation, use rolling_group and calculate
```

### Multiple moving averages

```python syntax
from deephaven.updateby import rolling_avg_tick

result = table.update_by(
    [
        rolling_avg_tick("MA5=Price", rev_ticks=5),
        rolling_avg_tick("MA10=Price", rev_ticks=10),
        rolling_avg_tick("MA20=Price", rev_ticks=20),
        rolling_avg_tick("MA50=Price", rev_ticks=50),
        rolling_avg_tick("MA200=Price", rev_ticks=200),
    ],
    by="Symbol",
)
```

## Window options

### Forward and backward windows

```python syntax
from deephaven.updateby import rolling_avg_tick

# Trailing only (default)
result = table.update_by(
    rolling_avg_tick("TrailingMA=Price", rev_ticks=10), by="Symbol"
)

# Forward looking
result = table.update_by(rolling_avg_tick("ForwardMA=Price", fwd_ticks=10), by="Symbol")

# Centered window
result = table.update_by(
    rolling_avg_tick("CenteredMA=Price", rev_ticks=5, fwd_ticks=5), by="Symbol"
)
```

### Time-based window edges

```python syntax
from deephaven.updateby import rolling_avg_time

# Last 5 minutes
result = table.update_by(
    rolling_avg_time("Timestamp", "MA5Min=Price", rev_time="PT5M"), by="Symbol"
)

# Last 1 hour
result = table.update_by(
    rolling_avg_time("Timestamp", "MA1Hr=Price", rev_time="PT1H"), by="Symbol"
)

# Last 1 day
result = table.update_by(
    rolling_avg_time("Timestamp", "MA1Day=Price", rev_time="P1D"), by="Symbol"
)
```

## Common pitfalls

1. **Insufficient history** — Rolling calculations return null until enough data exists.

   ```python syntax
   # MA20 is null for first 19 rows per Symbol
   result = table.update_by(rolling_avg_tick("MA20=Price", rev_ticks=20), by="Symbol")
   ```

2. **Group by column** — Don't forget `by` parameter to calculate separately per symbol.

   ```python syntax
   # CORRECT - separate MA per symbol
   result = table.update_by(rolling_avg_tick("MA=Price", rev_ticks=20), by="Symbol")

   # WRONG - MA across all symbols mixed together
   result = table.update_by(rolling_avg_tick("MA=Price", rev_ticks=20))
   ```

3. **Tick vs time windows** — Tick-based uses row count, time-based uses timestamp column.

4. **EMA decay interpretation** — `decay_ticks=20` means ~63% weight on last 20 ticks (1-1/e).

5. **Sorted data for time-based** — Time-based rolling ops work best with time-sorted data.

## Related skills

- `work-with-time` — Parse timestamps for time-based windows
- `business-calendar` — Business-day rolling calculations
- `transform-data` — Combine with filtering and aggregation
- `time-series-join` — Use rolling calculations with temporal joins
