---
name: time-series-join
description: Join tables by timestamp using as-of joins (aj/raj) for point-in-time lookups
---

Help the user perform time-aligned joins where exact timestamp matches aren't required.

## When to use this skill

- User wants "as-of join", "point-in-time", "temporal join", or "time-series join"
- User wants the "latest value as of" a timestamp
- User wants to match trades with quotes at trade time
- User mentions "most recent", "closest", or "prior" timestamp matching
- User is working with financial data (trades, quotes, prices)

## Decision guide: aj vs raj

| Join                       | Finds                                            | Use case                                       |
| -------------------------- | ------------------------------------------------ | ---------------------------------------------- |
| `aj` (as-of join)          | Latest right row WHERE right_time <= left_time   | "What was the quote when this trade happened?" |
| `raj` (reverse as-of join) | Earliest right row WHERE right_time >= left_time | "What was the next quote after this trade?"    |

## Quick answers

### aj — Get latest value as of each timestamp

```python syntax
# Match each trade with the most recent quote (quote_time <= trade_time)
result = trades.aj(quotes, on=["Symbol", "TradeTime >= QuoteTime"], joins="Bid, Ask")
```

**Explanation**: For each trade, find the quote with the largest `QuoteTime` that is still <= `TradeTime`, matching on `Symbol`.

### raj — Get next value after each timestamp

```python syntax
# Match each trade with the next quote (quote_time >= trade_time)
result = trades.raj(quotes, on=["Symbol", "TradeTime <= QuoteTime"], joins="Bid, Ask")
```

**Explanation**: For each trade, find the quote with the smallest `QuoteTime` that is >= `TradeTime`.

## Complete example: Trade-quote alignment

```python
from deephaven import empty_table

# Simulated trades
trades = empty_table(5).update(
    [
        "Symbol = `AAPL`",
        "TradeTime = '2024-01-15T09:30:00 ET' + i * MINUTE",
        "TradePrice = 150.0 + randomDouble(-1, 1)",
    ]
)

# Simulated quotes (more frequent)
quotes = empty_table(20).update(
    [
        "Symbol = `AAPL`",
        "QuoteTime = '2024-01-15T09:30:00 ET' + i * 15 * SECOND",
        "Bid = 149.5 + randomDouble(-0.5, 0.5)",
        "Ask = 150.5 + randomDouble(-0.5, 0.5)",
    ]
)

# For each trade, get the quote that was active at trade time
trades_with_quotes = trades.aj(
    quotes, on=["Symbol", "TradeTime >= QuoteTime"], joins="Bid, Ask, QuoteTime"
)

# Calculate trade vs quote spread
analysis = trades_with_quotes.update(
    ["Mid = (Bid + Ask) / 2", "TradeVsMid = TradePrice - Mid"]
)
```

## Variations

### Multiple match columns

```python syntax
# Match on Symbol AND Exchange, then by time
result = trades.aj(
    quotes, on=["Symbol", "Exchange", "TradeTime >= QuoteTime"], joins="Bid, Ask"
)
```

### Get all columns from right table

```python syntax
# Omit joins to get all non-key columns
result = trades.aj(quotes, on=["Symbol", "TradeTime >= QuoteTime"])
```

### Rename columns during join

```python syntax
result = trades.aj(
    quotes,
    on=["Symbol", "TradeTime >= QuoteTime"],
    joins="TradeBid=Bid, TradeAsk=Ask, QuoteTimestamp=QuoteTime",
)
```

### Time column with same name

```python syntax
# Both tables have "Timestamp"
result = trades.aj(
    quotes.rename_columns("QuoteTime=Timestamp"),
    on=["Symbol", "Timestamp >= QuoteTime"],
    joins="Bid, Ask",
)
```

## Common patterns

### Point-in-time portfolio valuation

```python syntax
# Value positions using prices as of each valuation time
valuations = valuation_times.aj(
    prices, on=["Symbol", "ValuationTime >= PriceTime"], joins="Price"
).update("Value = Shares * Price")
```

### Forward-fill missing data

```python syntax
# Use aj to forward-fill sparse data
from deephaven import time_table

# Create regular time grid
time_grid = time_table("PT1M").update("Symbol = `AAPL`")

# Fill with last known value
filled = time_grid.aj(sparse_data, on=["Symbol", "Timestamp >= DataTime"], joins="Value")
```

### Bid-ask spread at trade time

```python syntax
result = trades.aj(
    quotes, on=["Symbol", "TradeTime >= QuoteTime"], joins="Bid, Ask"
).update(["Spread = Ask - Bid", "TradeVsBid = Price - Bid", "TradeVsAsk = Price - Ask"])
```

### Look ahead (next event)

```python syntax
# Get the next trade after each quote
quotes_with_next_trade = quotes.raj(
    trades,
    on=["Symbol", "QuoteTime <= TradeTime"],
    joins="NextTradePrice=Price, NextTradeTime=TradeTime",
)
```

## Common pitfalls

1. **Time column must be last in `on`** — The timestamp column for the as-of comparison must be the last column in the `on` list.

   ```python syntax
   # CORRECT - time comparison last
   on = ["Symbol", "TradeTime >= QuoteTime"]

   # WRONG - time comparison not last
   on = ["TradeTime >= QuoteTime", "Symbol"]  # Won't work as expected
   ```

2. **Right table should be sorted by time** — For best performance, ensure the right table is sorted by the timestamp column.

   ```python syntax
   quotes_sorted = quotes.sort("QuoteTime")
   result = trades.aj(quotes_sorted, on=["Symbol", "TradeTime >= QuoteTime"], joins="Bid")
   ```

3. **Null on no prior match** — If no right row has a timestamp <= left timestamp, the joined columns are null.

   ```python syntax
   # Handle nulls
   result = trades.aj(quotes, on=["Symbol", "TradeTime >= QuoteTime"], joins="Bid").where(
       "!isNull(Bid)"
   )  # Filter out trades before any quotes
   ```

4. **Time zone awareness** — Ensure both timestamp columns use the same time zone or convert them.

   ```python syntax
   # Convert to same zone if needed
   trades_utc = trades.update("TradeTime = toInstant(TradeTime, 'UTC')")
   ```

5. **aj vs natural_join** — Use `natural_join` only when timestamps match exactly. Use `aj` when you need "closest prior" matching.

## Related skills

- `join-tables` — For exact key-based joins
- `range-join` — For range-based window joins
- `work-with-time` — Parse and manipulate timestamps
- `rolling-calculations` — Alternative for time-windowed calculations
