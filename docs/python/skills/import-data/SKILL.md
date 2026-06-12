---
name: import-data
description: Import data into Deephaven from files (Parquet, CSV), streaming sources (Kafka), Python objects, or databases
---

Help the user load data into Deephaven tables from various sources.

## When to use this skill

- User wants to "load", "import", "read", or "ingest" data
- User mentions file formats: Parquet, CSV, JSON
- User mentions streaming: Kafka, real-time, live data
- User wants to push data from Python (pandas, lists, dictionaries)
- User mentions database or JDBC connections

## Quick answers by source

### Parquet files

```python syntax
from deephaven.parquet import read

# Single file
table = read("/path/to/data.parquet")

# Directory of Parquet files
table = read("/path/to/directory/")

# S3 (requires configuration)
table = read("s3://bucket/path/to/data.parquet")
```

### CSV files

```python syntax
from deephaven import read_csv

# Basic CSV
table = read_csv("/path/to/data.csv")

# With options
table = read_csv(
    "/path/to/data.csv",
    header_row=0,  # Row containing headers (0-indexed)
    skip_rows=0,  # Rows to skip before header
    allow_missing_columns=False,
    ignore_empty_lines=True,
)
```

### From Python objects

```python
from deephaven import new_table
from deephaven.column import string_col, int_col, double_col

# From explicit columns
table = new_table(
    [
        string_col("Symbol", ["AAPL", "GOOG", "MSFT"]),
        double_col("Price", [150.0, 140.0, 380.0]),
        int_col("Volume", [1000, 2000, 1500]),
    ]
)
```

```python
from deephaven import pandas as dhpd
import pandas as pd

# From pandas DataFrame
df = pd.DataFrame({"Symbol": ["AAPL", "GOOG", "MSFT"], "Price": [150.0, 140.0, 380.0]})
pd_table = dhpd.to_table(df)
```

### Kafka streaming

```python syntax
from deephaven import kafka_consumer as kc
from deephaven.stream.kafka.consumer import TableType, KeyValueSpec

# JSON messages
table = kc.consume(
    {"bootstrap.servers": "localhost:9092"},
    topic="market-data",
    key_spec=KeyValueSpec.IGNORE,
    value_spec=kc.json_spec(
        [("Symbol", dht.string), ("Price", dht.double), ("Timestamp", dht.Instant)]
    ),
    table_type=TableType.append(),
)
```

### Push data dynamically (for real-time updates)

```python syntax
from deephaven import DynamicTableWriter
import deephaven.dtypes as dht

# Create a writer
table_writer = DynamicTableWriter(
    {"Symbol": dht.string, "Price": dht.double, "Timestamp": dht.Instant}
)

# Get the live table
live_table = table_writer.table

# Push rows (call from your data source)
from deephaven.time import dh_now

table_writer.write_row("AAPL", 150.25, dh_now())
```

### Input tables (editable by users)

```python syntax
from deephaven import input_table
from deephaven import dtypes as dht

# Create empty input table
input_t = input_table(
    {"Symbol": dht.string, "TargetPrice": dht.double}, key_cols=["Symbol"]
)

# Add rows programmatically
from deephaven import new_table
from deephaven.column import string_col, double_col

new_rows = new_table(
    [string_col("Symbol", ["AAPL", "GOOG"]), double_col("TargetPrice", [200.0, 180.0])]
)
input_t.add(new_rows)
```

## Variations

### Reading multiple files with a pattern

```python syntax
from deephaven.parquet import read

# All Parquet files in directory
table = read("/data/trades/")

# Parquet files maintain partitioning structure
# /data/trades/date=2024-01-01/data.parquet
# /data/trades/date=2024-01-02/data.parquet
table = read("/data/trades/")  # date column auto-detected
```

### Specifying column types for CSV

```python syntax
from deephaven import read_csv
from deephaven import dtypes as dht

# Override inferred types
table = read_csv(
    "/path/to/data.csv", col_types={"Timestamp": dht.Instant, "Price": dht.double}
)
```

### Kafka with Avro or Protobuf

```python syntax
# Avro (requires schema registry)
table = kc.consume(
    {
        "bootstrap.servers": "localhost:9092",
        "schema.registry.url": "http://localhost:8081",
    },
    topic="trades",
    key_spec=KeyValueSpec.IGNORE,
    value_spec=kc.avro_spec("trades-value"),
    table_type=TableType.append(),
)
```

## Common pitfalls

1. **CSV type inference** — Large files may have incorrect type inference. Specify `col_types` explicitly for critical columns.

2. **Kafka table types** — Use `TableType.append()` for append-only streams, `TableType.blink()` for blink tables (rows exist for one cycle), or `TableType.ring(capacity)` for ring buffers.

3. **Parquet timestamp handling** — Parquet timestamps are read as `Instant` by default. Use `to_j_local_date_time` if you need local time.

4. **Memory with large CSVs** — CSVs are fully loaded into memory. For very large files, consider converting to Parquet first.

5. **DynamicTableWriter thread safety** — `write_row` is thread-safe, but batch writes with `write_rows` are more efficient for high throughput.

## Related skills

- `join-tables` — After importing, join multiple data sources
- `time-series-join` — Join imported data by timestamp
- `transform-data` — Filter and transform imported data
- `export-and-share` — Export processed data back out
