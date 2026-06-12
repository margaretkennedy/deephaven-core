---
name: join-tables
description: Join two or more tables by key columns using exact match joins (natural_join, exact_join, join)
---

Help the user combine tables using key-based joins.

## When to use this skill

- User wants to "join", "merge", "combine", or "lookup" data between tables
- User mentions matching on keys, IDs, or symbols
- User wants to add columns from one table to another
- NOT for time-based joins (see `time-series-join` skill)
- NOT for range-based joins (see `range-join` skill)

## Decision guide: Which join to use?

| Scenario                                | Join type      | Behavior on no match   |
| --------------------------------------- | -------------- | ---------------------- |
| Add columns, expect all keys to match   | `exact_join`   | Error                  |
| Add columns, allow missing keys         | `natural_join` | Null values            |
| Allow duplicate keys in right table     | `join`         | Duplicate rows         |
| Keep all left rows, group right matches | `join`         | Multiple rows per left |

## Quick answers

### natural_join — Most common, null on no match

```python syntax
# Add price columns to trades using Symbol as key
result = trades.natural_join(prices, on="Symbol", joins="Bid, Ask")

# Multiple key columns
result = trades.natural_join(prices, on=["Symbol", "Exchange"], joins="Bid, Ask")

# Rename columns during join
result = trades.natural_join(
    prices, on="Symbol", joins="CurrentBid=Bid, CurrentAsk=Ask"
)
```

**When to use**: You want to enrich a table with lookup data, and missing matches should become null.

### exact_join — Strict, error on no match

```python syntax
# Every trade MUST have a matching price
result = trades.exact_join(prices, on="Symbol", joins="Bid, Ask")
```

**When to use**: Data integrity is critical — you want an error if any key is missing.

### join — Allow duplicates, Cartesian on match

```python syntax
# Each trade gets ALL matching orders (may create multiple rows)
result = trades.join(orders, on="Symbol", joins="OrderId, OrderQty")
```

**When to use**: Right table may have multiple matches per key, and you want all combinations.

## Variations

### Join all columns from right table

```python syntax
# Omit joins parameter to get all non-key columns
result = trades.natural_join(reference_data, on="Symbol")
```

### Multiple key columns

```python syntax
# Match on both Symbol AND Exchange
result = trades.natural_join(prices, on=["Symbol", "Exchange"], joins="Bid, Ask")
```

### Key columns with different names

```python syntax
# Left table has "Sym", right table has "Symbol"
result = trades.natural_join(prices, on="Sym=Symbol", joins="Bid, Ask")
```

### Self-join

```python syntax
# Join a table to itself (e.g., compare current vs previous)
from deephaven.updateby import rolling_group_tick

# Alternative: use update_by for row comparisons
```

### Left join equivalent

```python syntax
# natural_join IS a left join - keeps all left rows, nulls for no match
result = left_table.natural_join(right_table, on="Key", joins="Value")
```

### Inner join equivalent

```python syntax
# Filter out nulls after natural_join
result = left_table.natural_join(right_table, on="Key", joins="Value").where(
    "!isNull(Value)"
)
```

## Common patterns

### Enrich trades with reference data

```python syntax
# Add company name and sector to trade data
enriched = trades.natural_join(
    companies, on="Symbol", joins="CompanyName, Sector, Industry"
)
```

### Lookup latest prices

```python syntax
# Get current bid/ask for each position
positions_with_prices = positions.natural_join(
    latest_quotes, on="Symbol", joins="Bid, Ask, Mid=(Bid+Ask)/2"
)
```

### Combine data from multiple sources

```python syntax
# Chain joins
result = trades.natural_join(quotes, on="Symbol", joins="Bid, Ask").natural_join(
    reference, on="Symbol", joins="Sector"
)
```

## Common pitfalls

1. **Duplicate keys in right table** — `natural_join` and `exact_join` require unique keys in the right table. Use `last_by` or `first_by` to dedupe first, or use `join` if you want all matches.

   ```python syntax
   # Dedupe right table first
   unique_prices = prices.last_by("Symbol")
   result = trades.natural_join(unique_prices, on="Symbol", joins="Price")
   ```

2. **Column name conflicts** — If both tables have a column with the same name (not a key), you must rename it in the `joins` parameter.

   ```python syntax
   # Both have "Timestamp" - rename right table's version
   result = trades.natural_join(quotes, on="Symbol", joins="QuoteTime=Timestamp")
   ```

3. **Null handling** — With `natural_join`, unmatched rows get null values. Filter them or handle with `replaceIfNull`.

   ```python syntax
   result = trades.natural_join(prices, on="Symbol", joins="Bid").update(
       "Bid = replaceIfNull(Bid, 0.0)"
   )
   ```

4. **Key type mismatch** — Key columns must have compatible types. Convert if needed.

   ```python syntax
   # If one table has int keys and another has strings
   trades_fixed = trades.update("Symbol = (String)Symbol")
   ```

## Related skills

- `time-series-join` — Join by timestamp with aj/raj
- `range-join` — Join on value ranges
- `transform-data` — Filter and prepare data before joining
- `aggregate-data` — Aggregate after joining
