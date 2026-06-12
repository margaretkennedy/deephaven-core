---
name: visualize-data
description: Create charts and plots from Deephaven tables including line, scatter, bar, and OHLC charts
---

Help the user create visualizations from Deephaven tables.

## When to use this skill

- User wants to "plot", "chart", "graph", or "visualize" data
- User mentions specific chart types: line, scatter, bar, histogram, OHLC, candlestick
- User wants to see "trends", "distributions", or "correlations"
- User mentions "dashboard" or "real-time chart"

## Quick answers

### Line chart

```python syntax
from deephaven.plot.figure import Figure

# Single series
fig = Figure().plot_xy("Price", t=table, x="Timestamp", y="Price").show()

# Multiple series
fig = (
    Figure()
    .plot_xy("AAPL", t=aapl_data, x="Timestamp", y="Price")
    .plot_xy("GOOG", t=goog_data, x="Timestamp", y="Price")
    .show()
)
```

### Scatter plot

```python syntax
from deephaven.plot.figure import Figure

fig = Figure().plot_xy("Correlation", t=table, x="Price", y="Volume").show()
```

### Bar chart

```python syntax
from deephaven.plot.figure import Figure

fig = (
    Figure()
    .plot_cat("Volume by Symbol", t=summary, category="Symbol", y="TotalVolume")
    .show()
)
```

### OHLC / Candlestick

```python syntax
from deephaven.plot.figure import Figure

fig = (
    Figure()
    .plot_ohlc(
        "AAPL",
        t=ohlc_data,
        x="Date",
        open="Open",
        high="High",
        low="Low",
        close="Close",
    )
    .show()
)
```

## Complete examples

### Time series line chart

```python
from deephaven.plot.figure import Figure
from deephaven import empty_table

# Sample data
prices = empty_table(100).update(
    [
        "Timestamp = '2024-01-15T09:30:00 ET' + i * MINUTE",
        "Price = 150 + randomDouble(-5, 5)",
    ]
)

# Create line chart
fig = Figure().plot_xy("Price", t=prices, x="Timestamp", y="Price").show()
```

### Multi-series with legend

```python syntax
from deephaven.plot.figure import Figure

fig = (
    Figure()
    .plot_xy("AAPL", t=aapl, x="Timestamp", y="Price")
    .plot_xy("GOOG", t=goog, x="Timestamp", y="Price")
    .plot_xy("MSFT", t=msft, x="Timestamp", y="Price")
    .show()
)
```

### Plot by category (auto-series from column)

```python syntax
from deephaven.plot.figure import Figure

# Automatically create one series per symbol
fig = (
    Figure()
    .plot_xy_by("Prices", t=all_prices, x="Timestamp", y="Price", by="Symbol")
    .show()
)
```

### Histogram

```python syntax
from deephaven.plot.figure import Figure

fig = (
    Figure().plot_xy_hist("Return Distribution", t=returns, x="Return", nbins=50).show()
)
```

### Pie chart

```python syntax
from deephaven.plot.figure import Figure

fig = (
    Figure()
    .plot_pie("Market Share", t=summary, category="Symbol", y="MarketCap")
    .show()
)
```

## Customization

### Axis labels and titles

```python syntax
from deephaven.plot.figure import Figure

fig = (
    Figure()
    .x_label("Time")
    .y_label("Price ($)")
    .plot_xy("AAPL Price", t=data, x="Timestamp", y="Price")
    .show()
)
```

### Multiple Y axes (twinx)

```python syntax
from deephaven.plot.figure import Figure

fig = (
    Figure()
    .new_axes()
    .plot_xy("Price", t=data, x="Timestamp", y="Price")
    .twin_x()
    .plot_xy("Volume", t=data, x="Timestamp", y="Volume")
    .show()
)
```

### Subplots

```python syntax
from deephaven.plot.figure import Figure

fig = (
    Figure(rows=2, cols=1)
    .new_chart(row=0, col=0)
    .plot_xy("Price", t=data, x="Timestamp", y="Price")
    .new_chart(row=1, col=0)
    .plot_xy("Volume", t=data, x="Timestamp", y="Volume")
    .show()
)
```

### Line styles and colors

```python syntax
from deephaven.plot.figure import Figure
from deephaven.plot import PlotStyle, Color

fig = (
    Figure()
    .plot_xy("Price", t=data, x="Timestamp", y="Price")
    .line_color(Color.BLUE)
    .line_style(PlotStyle.LINE)
    .show()
)
```

### Point markers

```python syntax
from deephaven.plot.figure import Figure
from deephaven.plot import Shape

fig = (
    Figure()
    .plot_xy("Data", t=data, x="X", y="Y")
    .point_shape(Shape.CIRCLE)
    .point_size(5)
    .show()
)
```

## Common patterns

### Real-time updating chart

```python
from deephaven.plot.figure import Figure
from deephaven import time_table

# Ticking data
live_prices = time_table("PT1S").update(["Price = 100 + Math.sin(i / 10.0) * 10"])

# Chart updates automatically
fig = Figure().plot_xy("Live Price", t=live_prices, x="Timestamp", y="Price").show()
```

### Price with moving average overlay

```python syntax
from deephaven.plot.figure import Figure
from deephaven.updateby import rolling_avg_tick

# Add moving average
data_with_ma = prices.update_by(
    rolling_avg_tick("MA20=Price", rev_ticks=20), by="Symbol"
)

fig = (
    Figure()
    .plot_xy("Price", t=data_with_ma, x="Timestamp", y="Price")
    .plot_xy("MA20", t=data_with_ma, x="Timestamp", y="MA20")
    .show()
)
```

### Candlestick with volume

```python syntax
from deephaven.plot.figure import Figure

fig = (
    Figure(rows=2, cols=1)
    .new_chart(row=0, col=0)
    .plot_ohlc(
        "Price", t=ohlc, x="Date", open="Open", high="High", low="Low", close="Close"
    )
    .new_chart(row=1, col=0)
    .plot_cat_hist("Volume", t=ohlc, category="Date", y="Volume")
    .show()
)
```

### Scatter with trend line

```python syntax
from deephaven.plot.figure import Figure

# Calculate trend (simple linear regression)
with_trend = data.update(
    [
        "N = count(X)",
        "SumX = sum(X)",
        "SumY = sum(Y)",
        "SumXY = sum(X*Y)",
        "SumX2 = sum(X*X)",
        "Slope = (N * SumXY - SumX * SumY) / (N * SumX2 - SumX * SumX)",
        "Intercept = (SumY - Slope * SumX) / N",
        "Trend = Slope * X + Intercept",
    ]
)

fig = (
    Figure()
    .plot_xy("Data", t=with_trend, x="X", y="Y")
    .plot_xy("Trend", t=with_trend, x="X", y="Trend")
    .show()
)
```

## Common pitfalls

1. **Call .show()** — Charts aren't displayed until you call `.show()`.

   ```python syntax
   # WRONG - chart not shown
   fig = Figure().plot_xy("Price", t=data, x="X", y="Y"

   # CORRECT
   fig = Figure().plot_xy("Price", t=data, x="X", y="Y").show()
   ```

2. **Column names are strings** — Pass column names as strings, not column references.

   ```python syntax
   # CORRECT
   .plot_xy("Series", t=data, x="Timestamp", y="Price")

   # WRONG
   .plot_xy("Series", t=data, x=Timestamp, y=Price)
   ```

3. **Large datasets** — Very large tables may plot slowly. Consider downsampling first.

   ```python syntax
   # Downsample before plotting
   sampled = large_table.head(10000)
   fig = Figure().plot_xy("Price", t=sampled, x="X", y="Y").show()
   ```

4. **Null values** — Nulls create gaps in line charts. Filter or fill them if unwanted.

   ```python syntax
   clean_data = data.where("!isNull(Price)")
   ```

5. **Time axis formatting** — Deephaven auto-formats time axes, but ensure timestamps are proper `Instant` type.

## Chart types reference

| Method          | Chart type         | Required params           |
| --------------- | ------------------ | ------------------------- |
| `plot_xy`       | Line/scatter       | x, y                      |
| `plot_xy_by`    | Multi-series line  | x, y, by                  |
| `plot_cat`      | Bar chart          | category, y               |
| `plot_cat_hist` | Category histogram | category, y               |
| `plot_xy_hist`  | Histogram          | x, nbins                  |
| `plot_ohlc`     | Candlestick        | x, open, high, low, close |
| `plot_pie`      | Pie chart          | category, y               |

## Related skills

- `transform-data` — Prepare data for visualization
- `rolling-calculations` — Add moving averages to charts
- `export-and-share` — Share charts with others
- `work-with-time` — Format time axes
