---
name: export-and-share
description: Export Deephaven tables to files (Parquet, CSV) and share data with collaborators
---

Help the user export data and share their work with others.

## When to use this skill

- User wants to "export", "save", "download", or "write" data
- User mentions output formats: Parquet, CSV
- User wants to "share" data with others
- User asks about "dashboards", "publishing", or "collaboration"
- User wants to send data to another system or tool

## Quick answers

### Export to Parquet

```python syntax
from deephaven.parquet import write

# Single file
write(table, "/path/to/output.parquet")

# With compression
write(table, "/path/to/output.parquet", compression_codec_name="SNAPPY")

# To S3 (requires configuration)
write(table, "s3://bucket/path/output.parquet")
```

### Export to CSV

```python syntax
from deephaven import write_csv

write_csv(table, "/path/to/output.csv")
```

### Convert to pandas

```python syntax
from deephaven import pandas as dhpd

# Full table to pandas DataFrame
df = dhpd.to_pandas(table)

# Then use pandas methods to export
df.to_csv("output.csv")
df.to_excel("output.xlsx")
df.to_json("output.json")
```

## Complete examples

### Export query results to Parquet

```python syntax
from deephaven.parquet import write
from deephaven import agg

# Process data
summary = trades.where("Date = today()").agg_by(
    [
        agg.sum_("TotalVolume=Volume"),
        agg.avg("AvgPrice=Price"),
        agg.count_("TradeCount"),
    ],
    by="Symbol",
)

# Export
write(summary, "/data/exports/daily_summary.parquet")
```

### Export to partitioned Parquet

```python syntax
from deephaven.parquet import write

# Data with Date column becomes partitioned
write(
    table,
    "/data/exports/trades",
    table_definition=table.definition,
    partition_cols=["Date"],
)

# Creates: /data/exports/trades/Date=2024-01-15/data.parquet
#          /data/exports/trades/Date=2024-01-16/data.parquet
```

### Batch export multiple tables

```python syntax
from deephaven.parquet import write

tables_to_export = {
    "trades": trades_summary,
    "quotes": quotes_summary,
    "positions": positions_snapshot,
}

for name, table in tables_to_export.items():
    write(table, f"/data/exports/{name}.parquet")
```

## Parquet options

### Compression

```python syntax
from deephaven.parquet import write

# SNAPPY (default) - good balance of speed and compression
write(table, "output.parquet", compression_codec_name="SNAPPY")

# GZIP - better compression, slower
write(table, "output.parquet", compression_codec_name="GZIP")

# ZSTD - best compression
write(table, "output.parquet", compression_codec_name="ZSTD")

# LZ4 - fastest
write(table, "output.parquet", compression_codec_name="LZ4")

# Uncompressed
write(table, "output.parquet", compression_codec_name="UNCOMPRESSED")
```

### Max dictionary size

```python syntax
from deephaven.parquet import write

# Increase dictionary size for columns with many repeated values
write(table, "output.parquet", max_dictionary_size=1048576)
```

## Sharing with the UI

### Tables automatically appear in UI

```python syntax
# Any table assigned to a variable in the global scope
# appears in the Deephaven web UI automatically

# This table will show in the UI as "daily_summary"
daily_summary = trades.agg_by(agg.sum_("Volume"), by="Symbol")

# Users with access to the session can see and interact with it
```

### Organizing tables in panels

```python syntax
# Tables appear in the UI panels automatically
# Name them descriptively
raw_trades = db.t("trades")
filtered_trades = raw_trades.where("Price > 100")
trade_summary = filtered_trades.agg_by(agg.count_("N"), by="Symbol")

# All three tables visible: raw_trades, filtered_trades, trade_summary
```

### Sharing figures/charts

```python syntax
from deephaven.plot.figure import Figure

# Figures also appear in the UI
price_chart = Figure().plot_xy("Price", t=prices, x="Timestamp", y="Price").show()

# "price_chart" appears in the UI and updates live
```

## Converting to other formats

### To pandas then anywhere

```python syntax
from deephaven import pandas as dhpd
import pandas as pd

# To pandas
df = dhpd.to_pandas(table)

# Then export to any pandas-supported format
df.to_csv("output.csv", index=False)
df.to_excel("output.xlsx", index=False)
df.to_json("output.json", orient="records")
df.to_html("output.html")

# Or use with other Python tools
# df.to_sql("table_name", connection)
# df.to_gbq("dataset.table", project_id="my-project")
```

### To Arrow for interop

```python syntax
import pyarrow as pa
from deephaven import arrow as dharrow

# Convert to PyArrow table
arrow_table = dharrow.to_arrow(table)

# Then use Arrow for further processing
# arrow_table.to_pandas()
# arrow_table.to_pydict()
```

## Common patterns

### Scheduled exports

```python syntax
from deephaven.parquet import write
from deephaven.time import now, formatDate


# Generate dated filename
def export_daily_snapshot(table, base_path):
    date_str = formatDate(now(), "America/New_York")
    path = f"{base_path}/snapshot_{date_str}.parquet"
    write(table, path)
    return path


# Call this from a scheduled task or at end of day
path = export_daily_snapshot(positions, "/data/snapshots")
```

### Export with metadata

```python syntax
from deephaven.parquet import write
from deephaven.time import now

# Add metadata columns before export
export_table = table.update(
    ["ExportTime = now()", "ExportVersion = `1.0`", "Source = `production`"]
)

write(export_table, "output.parquet")
```

### Incremental export (append)

```python syntax
from deephaven.parquet import write

# For incremental exports, use partitioning or unique filenames
# Deephaven doesn't append to existing Parquet files

# Option 1: Partition by date
write(new_data, "/data/exports/trades", partition_cols=["Date"])

# Option 2: Unique filenames with timestamp
from deephaven.time import now

timestamp = str(epochMillis(now()))
write(new_data, f"/data/exports/trades_{timestamp}.parquet")
```

### Export subset of columns

```python syntax
from deephaven.parquet import write

# Select only needed columns first
export_cols = table.select(["Symbol", "Price", "Volume", "Timestamp"])
write(export_cols, "output.parquet")
```

## Common pitfalls

1. **Static snapshot only** — Exports capture data at a point in time. Live/ticking tables export their current state.

   ```python syntax
   # This captures current state
   write(live_table, "snapshot.parquet")

   # For continuous export, schedule periodic writes
   ```

2. **Large table memory** — `to_pandas()` loads entire table into memory. For large tables, export to Parquet directly or sample first.

   ```python syntax
   # For large tables, avoid pandas conversion
   write(large_table, "output.parquet")  # Streams, doesn't load all to memory

   # Or sample first
   sample = large_table.head(100000)
   df = dhpd.to_pandas(sample)
   ```

3. **File path permissions** — Ensure the Deephaven server has write access to the target path.

4. **Overwrite behavior** — `write` overwrites existing files without warning.

5. **Time zone in exports** — Timestamps export as UTC in Parquet. Note this when importing elsewhere.

## Related skills

- `import-data` — Read exported data back in
- `transform-data` — Prepare data before export
- `visualize-data` — Create charts to share
- `join-tables` — Combine data before export
