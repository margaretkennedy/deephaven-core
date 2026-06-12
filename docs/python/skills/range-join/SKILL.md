---
name: range-join
description: Join tables using range conditions with range_join for window-based lookups and aggregations
---

Help the user join tables where the match condition is a range rather than exact equality.

## When to use this skill

- User wants to match rows within a range or window
- User mentions "range join", "window join", or "between" conditions
- User wants to aggregate values within a time or value range
- User needs all matches within a window (not just closest like aj)
- Examples: "all trades within 5 minutes", "prices between bid and ask"

## Quick answer

```python syntax
from deephaven.experimental import range_join

# For each order, find all trades within the order's price range
result = range_join(
    left_table=orders,
    right_table=trades,
    on=["Symbol"],  # Exact match columns
    agg=agg.count_("NumTrades"),  # Aggregation for matches
    range_match="Price >= LowPrice AND Price <= HighPrice",
)
```

## How range_join works

Unlike exact joins, `range_join`:

1. Matches on exact key columns (`on` parameter)
2. Then matches rows where the range condition is true (`range_match`)
3. Aggregates all matching right rows into the result (`agg`)

## Complete example: Trades within order price bands

```python
from deephaven import empty_table
from deephaven.experimental import range_join
from deephaven import agg

# Orders with acceptable price ranges
orders = empty_table(3).update(
    [
        "Symbol = (i % 2 == 0) ? `AAPL` : `GOOG`",
        "OrderId = i",
        "LowPrice = 145.0 + i * 5",
        "HighPrice = 155.0 + i * 5",
    ]
)

# Trades
trades = empty_table(20).update(
    [
        "Symbol = (i % 2 == 0) ? `AAPL` : `GOOG`",
        "TradePrice = 140.0 + i * 2",
        "TradeQty = 100 + i * 10",
    ]
)

# For each order, count trades and sum quantity within price range
result = range_join(
    left_table=orders,
    right_table=trades,
    on=["Symbol"],
    agg=[
        agg.count_("NumTrades"),
        agg.sum_("TotalQty=TradeQty"),
        agg.avg("AvgPrice=TradePrice"),
    ],
    range_match="TradePrice >= LowPrice AND TradePrice <= HighPrice",
)
```

## Variations

### Time window range join

```python syntax
from deephaven.experimental import range_join
from deephaven import agg

# For each event, aggregate all data within a time window
result = range_join(
    left_table=events,
    right_table=market_data,
    on=["Symbol"],
    agg=[agg.avg("AvgPrice=Price"), agg.count_("NumTicks")],
    range_match="Timestamp >= WindowStart AND Timestamp < WindowEnd",
)
```

### Collect matching rows (group)

```python syntax
from deephaven.experimental import range_join
from deephaven import agg

# Collect all matching trade prices into an array
result = range_join(
    left_table=orders,
    right_table=trades,
    on=["Symbol"],
    agg=agg.group("MatchingPrices=TradePrice"),
    range_match="TradePrice >= LowPrice AND TradePrice <= HighPrice",
)
```

### One-sided range (greater than, less than)

```python syntax
# All trades at or above a threshold
result = range_join(
    left_table=thresholds,
    right_table=trades,
    on=["Symbol"],
    agg=agg.count_("NumAbove"),
    range_match="TradePrice >= ThresholdPrice",
)
```

### Multiple aggregations

```python syntax
from deephaven import agg

result = range_join(
    left_table=orders,
    right_table=trades,
    on=["Symbol"],
    agg=[
        agg.count_("NumTrades"),
        agg.sum_("TotalVolume=Qty"),
        agg.min_("MinPrice=Price"),
        agg.max_("MaxPrice=Price"),
        agg.avg("AvgPrice=Price"),
        agg.std("StdPrice=Price"),
    ],
    range_match="Price >= LowPrice AND Price <= HighPrice",
)
```

## Common patterns

### VWAP within price band

```python syntax
result = range_join(
    left_table=orders,
    right_table=trades,
    on=["Symbol"],
    agg=[agg.weighted_avg("VWAP=Price", "Qty"), agg.sum_("TotalQty=Qty")],
    range_match="Price >= LowPrice AND Price <= HighPrice",
)
```

### Event impact analysis

```python syntax
# Aggregate market activity in windows around events
events_with_windows = events.update(
    ["WindowStart = Timestamp - MINUTE * 5", "WindowEnd = Timestamp + MINUTE * 5"]
)

impact = range_join(
    left_table=events_with_windows,
    right_table=trades,
    on=["Symbol"],
    agg=[
        agg.count_("TradeCount"),
        agg.sum_("Volume=Qty"),
        agg.max_("HighPrice=Price"),
        agg.min_("LowPrice=Price"),
    ],
    range_match="TradeTime >= WindowStart AND TradeTime < WindowEnd",
)
```

### Binned statistics

```python syntax
# Create bins and aggregate data into them
bins = empty_table(10).update(["BinStart = i * 10.0", "BinEnd = (i + 1) * 10.0"])

histogram = range_join(
    left_table=bins,
    right_table=data,
    on=[],  # No exact match columns
    agg=agg.count_("Count"),
    range_match="Value >= BinStart AND Value < BinEnd",
)
```

## Common pitfalls

1. **Must use aggregation** — `range_join` requires an `agg` parameter. Unlike regular joins that produce one row per match, range_join aggregates matches.

   ```python syntax
   # WRONG - no aggregation
   result = range_join(orders, trades, on=["Symbol"], range_match="...")

   # CORRECT - include aggregation
   result = range_join(
       orders, trades, on=["Symbol"], agg=agg.count_("N"), range_match="..."
   )
   ```

2. **Range column names** — The range condition uses column names from both tables. Left table columns come from `left_table`, comparison operators, then right table column names.

3. **Empty matches** — If no right rows match the range, aggregations return appropriate defaults (0 for count, null for avg, etc.).

4. **Performance on large ranges** — Very wide ranges that match many rows can be slow. Consider filtering the right table first.

5. **Experimental API** — `range_join` is in `deephaven.experimental` and the API may evolve.

## range_join vs aj/raj

| Feature    | range_join                          | aj/raj               |
| ---------- | ----------------------------------- | -------------------- |
| Match type | Range condition (between, >=, etc.) | Single closest match |
| Result     | Aggregated values                   | Single row columns   |
| Use case   | Windows, bands, buckets             | Point-in-time lookup |

## Related skills

- `time-series-join` — For closest-match temporal joins
- `join-tables` — For exact key-based joins
- `rolling-calculations` — For time-based rolling windows with update_by
- `aggregate-data` — For standard group-by aggregations
