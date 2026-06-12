---
name: transform-data
description: Filter, select, sort, update, and aggregate table data using core table operations
---

Help the user transform tables using Deephaven's core operations.

## When to use this skill

- User wants to "filter", "where", "subset", or "select rows"
- User wants to "select columns", "add columns", or "remove columns"
- User wants to "sort", "order by", or "rank"
- User wants to "group by", "aggregate", "sum", "average", or "count"
- User wants to "update", "calculate", or "add new columns"

## Quick answers by operation

### Filter rows

```python syntax
# Single condition
filtered = table.where("Price > 100")

# Multiple conditions (AND)
filtered = table.where(["Symbol = `AAPL`", "Price > 100"])

# OR conditions
filtered = table.where("Symbol = `AAPL` || Symbol = `GOOG`")

# Using in for multiple values
filtered = table.where("Symbol in `AAPL`, `GOOG`, `MSFT`")

# Null handling
filtered = table.where("!isNull(Price)")
```

### Select/remove columns

```python syntax
# Keep only specific columns
selected = table.select(["Symbol", "Price", "Volume"])

# Rename columns
renamed = table.rename_columns(["Ticker=Symbol", "Cost=Price"])

# Drop columns (select all except)
dropped = table.drop_columns(["UnwantedCol1", "UnwantedCol2"])

# Reorder columns
reordered = table.move_columns(0, ["Symbol", "Price"])  # Move to front
```

### Add/update columns

```python syntax
# Add new calculated columns
updated = table.update(
    ["Mid = (Bid + Ask) / 2", "Spread = Ask - Bid", "SpreadBps = Spread / Mid * 10000"]
)

# Conditional update
updated = table.update("Category = Price > 100 ? `High` : `Low`")

# Update with null handling
updated = table.update("CleanPrice = replaceIfNull(Price, 0.0)")
```

### Sort

```python syntax
# Single column ascending
sorted_table = table.sort("Price")

# Descending
sorted_table = table.sort_descending("Price")

# Multiple columns
sorted_table = table.sort(["Symbol", "Timestamp"])

# Mixed directions
from deephaven import SortDirection

sorted_table = table.sort(
    ["Symbol", "Price"], [SortDirection.ASCENDING, SortDirection.DESCENDING]
)
```

### Aggregate (group by)

```python syntax
from deephaven import agg

# Single aggregation
totals = table.agg_by(agg.sum_("TotalVolume=Volume"), by="Symbol")

# Multiple aggregations
summary = table.agg_by(
    [
        agg.count_("NumTrades"),
        agg.sum_("TotalVolume=Volume"),
        agg.avg("AvgPrice=Price"),
        agg.min_("LowPrice=Price"),
        agg.max_("HighPrice=Price"),
        agg.first("FirstTime=Timestamp"),
        agg.last("LastTime=Timestamp"),
    ],
    by="Symbol",
)

# Multiple group-by columns
by_sym_date = table.agg_by(agg.sum_("Volume"), by=["Symbol", "Date"])

# No group-by (entire table)
grand_total = table.agg_all_by(agg.sum_("Total=Volume"))
```

## select vs view vs update vs update_view

| Method        | Creates new columns?   | Memory            | Use when                              |
| ------------- | ---------------------- | ----------------- | ------------------------------------- |
| `select`      | Yes, only specified    | New copy          | Need subset of columns                |
| `view`        | Yes, only specified    | Formulas, no copy | Memory efficiency, simple formulas    |
| `update`      | Yes, keeps all columns | New + original    | Adding columns to existing            |
| `update_view` | Yes, keeps all columns | Formulas          | Adding calculated columns efficiently |

```python syntax
# select - only these columns, copied
result = table.select(["Symbol", "NewCol = Price * 2"])

# view - only these columns, computed on demand
result = table.view(["Symbol", "NewCol = Price * 2"])

# update - all original columns plus new, copied
result = table.update("NewCol = Price * 2")

# update_view - all columns plus new, computed on demand
result = table.update_view("NewCol = Price * 2")
```

## Common patterns

### Chained transformations

```python syntax
result = (
    source.where("Date = today()")
    .update("Mid = (Bid + Ask) / 2")
    .sort_descending("Volume")
    .head(100)
)
```

### Top N per group

```python syntax
# Top 5 by volume for each symbol
top_per_symbol = table.sort_descending("Volume").head_by(5, by="Symbol")
```

### Running calculations

```python syntax
# Cumulative sum (use update_by for this)
from deephaven.updateby import cum_sum

result = table.update_by(cum_sum("CumVolume=Volume"), by="Symbol")
```

### Conditional columns

```python syntax
result = table.update(
    [
        "Side = Price > PrevClose ? `Up` : (Price < PrevClose ? `Down` : `Flat`)",
        "Magnitude = abs(Price - PrevClose)",
        "IsSignificant = Magnitude > 1.0",
    ]
)
```

### Dedup / distinct

```python syntax
# First occurrence of each key
deduped = table.first_by("Symbol")

# Last occurrence of each key
latest = table.last_by("Symbol")

# Distinct values
distinct_symbols = table.select_distinct("Symbol")
```

## Aggregation reference

```python syntax
from deephaven import agg

# Count
agg.count_("NumRows")

# Sum, Avg, Min, Max, Std, Var
agg.sum_("TotalVolume=Volume")
agg.avg("AvgPrice=Price")
agg.min_("MinPrice=Price")
agg.max_("MaxPrice=Price")
agg.std("StdPrice=Price")
agg.var("VarPrice=Price")

# First, Last
agg.first("FirstValue=Value")
agg.last("LastValue=Value")

# Weighted average
agg.weighted_avg("VWAP=Price", "Volume")

# Percentiles
agg.percentile(0.5, "MedianPrice=Price")
agg.percentile(0.95, "P95Price=Price")

# Group into arrays
agg.group("Prices=Price")

# Unique count
agg.count_distinct("NumUnique=Symbol")
```

## Common pitfalls

1. **String literals need backticks** — In `where` clauses, string values use backticks, not quotes.

   ```python syntax
   # CORRECT
   table.where("Symbol = `AAPL`")

   # WRONG
   table.where("Symbol = 'AAPL'")  # Won't work
   ```

2. **Column names are case-sensitive** — `Price` and `price` are different columns.

3. **Aggregation requires agg import** — Don't forget to import agg functions.

   ```python syntax
   from deephaven import agg

   table.agg_by(agg.sum_("Total=Value"), by="Key")
   ```

4. **update vs update_view memory** — `update` copies data, `update_view` computes on read. For large tables with complex formulas, choose wisely.

5. **where with nulls** — `where("Col > 5")` excludes null values. Use `isNull(Col)` explicitly if needed.

## Related skills

- `join-tables` — Combine filtered/aggregated tables
- `rolling-calculations` — For running/rolling aggregations
- `work-with-time` — Time-based filtering and calculations
- `import-data` — Get data to transform
