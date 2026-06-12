---
name: work-with-time
description: Parse, format, and manipulate timestamps and time values in Deephaven
---

Help the user work with timestamps, dates, and time operations.

## When to use this skill

- User mentions "timestamp", "datetime", "date", "time"
- User wants to "parse", "format", or "convert" time values
- User asks about "time zones", "UTC", "local time"
- User needs "date parts" like year, month, day, hour
- User wants to do "time arithmetic" (add days, subtract hours)

## Deephaven time types

| Type            | Description            | Example                   |
| --------------- | ---------------------- | ------------------------- |
| `Instant`       | Point in time (UTC)    | `2024-01-15T14:30:00Z`    |
| `LocalDate`     | Date without time      | `2024-01-15`              |
| `LocalTime`     | Time without date      | `14:30:00`                |
| `LocalDateTime` | Date and time, no zone | `2024-01-15T14:30:00`     |
| `ZonedDateTime` | Date, time, and zone   | `2024-01-15T14:30:00 ET`  |
| `Duration`      | Time span              | `PT1H30M` (1 hour 30 min) |
| `Period`        | Date span              | `P1M` (1 month)           |

## Quick answers

### Get current time

```python
from deephaven.time import dh_now, dh_today

current_instant = dh_now()  # Current Instant (UTC)
current_date = dh_today()  # Current LocalDate
```

### Parse strings to time

```python
from deephaven.time import to_j_instant, to_j_local_date, to_j_local_time

# ISO format strings
instant = to_j_instant("2024-01-15T14:30:00Z")
instant_tz = to_j_instant("2024-01-15T14:30:00 ET")

local_date = to_j_local_date("2024-01-15")
local_time = to_j_local_time("14:30:00")
```

### Parse in table formulas

```python syntax
# In update/select formulas
table = source.update(
    [
        "Timestamp = parseInstant(DateString + `T` + TimeString + `Z`)",
        "TradeDate = parseLocalDate(DateString)",
        "TradeTime = parseLocalTime(TimeString)",
    ]
)
```

### Format time to strings

```python syntax
# In table formulas
table = source.update(
    [
        "DateStr = formatDate(Timestamp, 'America/New_York')",
        "TimeStr = formatTime(Timestamp, 'America/New_York')",
        "FullStr = formatDateTime(Timestamp, 'America/New_York')",
    ]
)
```

### Extract date/time parts

```python syntax
table = source.update(
    [
        "Year = year(Timestamp, 'ET')",
        "Month = monthOfYear(Timestamp, 'ET')",
        "Day = dayOfMonth(Timestamp, 'ET')",
        "Hour = hourOfDay(Timestamp, 'ET')",
        "Minute = minuteOfHour(Timestamp, 'ET')",
        "Second = secondOfMinute(Timestamp, 'ET')",
        "DayOfWeek = dayOfWeek(Timestamp, 'ET')",  # 1=Monday, 7=Sunday
    ]
)
```

### Time arithmetic

```python syntax
table = source.update(
    [
        # Add/subtract time
        "Tomorrow = plusDays(Timestamp, 1)",
        "NextHour = plusTime(Timestamp, HOUR)",
        "OneHourAgo = minusTime(Timestamp, HOUR)",
        # Difference between times
        "Elapsed = diffNanos(StartTime, EndTime)",
        "ElapsedSeconds = diffNanos(StartTime, EndTime) / 1_000_000_000",
    ]
)
```

## Time constants

```python syntax
# Duration constants (use in formulas)
SECOND = "PT1S"  # 1 second
MINUTE = "PT1M"  # 1 minute
HOUR = "PT1H"  # 1 hour
DAY = "P1D"  # 1 day

# Examples in formulas
table = source.update(
    [
        "FiveMinLater = plusTime(Timestamp, 5 * MINUTE)",
        "ThreeHoursAgo = minusTime(Timestamp, 3 * HOUR)",
    ]
)
```

## Time zones

```python syntax
# Common timezone identifiers
"UTC"  # UTC

"ET"  # US Eastern (auto DST)
"America/New_York"  # US Eastern (explicit)
"America/Chicago"  # US Central
"America/Los_Angeles"  # US Pacific
"Europe/London"  # UK
"Asia/Tokyo"  # Japan

# Convert between zones
table = source.update(
    [
        "NYTime = toLocalTime(Timestamp, 'America/New_York')",
        "LondonTime = toLocalTime(Timestamp, 'Europe/London')",
    ]
)
```

## Common patterns

### Filter by date

```python syntax
# Today's data
today_data = table.where("Date = today()")

# Specific date
jan15 = table.where("Date = parseLocalDate(`2024-01-15`)")

# Date range
week_data = table.where(
    ["Date >= parseLocalDate(`2024-01-08`)", "Date <= parseLocalDate(`2024-01-14`)"]
)
```

### Filter by time of day

```python syntax
# Market hours (9:30 AM - 4:00 PM ET)
market_hours = table.where(
    [
        "hourOfDay(Timestamp, 'ET') >= 9",
        "(hourOfDay(Timestamp, 'ET') > 9 || minuteOfHour(Timestamp, 'ET') >= 30)",
        "hourOfDay(Timestamp, 'ET') < 16",
    ]
)

# Simpler: use business calendar skill for this
```

### Group by time period

```python syntax
# Group by date
daily = table.agg_by(agg.sum_("Volume"), by="Date")

# Group by hour
hourly = table.update("Hour = hourOfDay(Timestamp, 'ET')").agg_by(
    agg.sum_("Volume"), by=["Date", "Hour"]
)

# Group by minute (floor to minute)
by_minute = table.update("MinuteTs = lowerBin(Timestamp, MINUTE)").agg_by(
    agg.sum_("Volume"), by="MinuteTs"
)
```

### Bin timestamps

```python syntax
# Round down to 5-minute bins
table = source.update("Bin5Min = lowerBin(Timestamp, 5 * MINUTE)")

# Round down to hour
table = source.update("HourBin = lowerBin(Timestamp, HOUR)")

# Round down to day
table = source.update("DayBin = lowerBin(Timestamp, DAY)")
```

### Create time series

```python
from deephaven import time_table, empty_table

# Ticking table (live)
ticking = time_table("PT1S")  # Every second

# Static time series
times = empty_table(100).update(["Timestamp = '2024-01-15T09:30:00 ET' + i * MINUTE"])
```

## Common pitfalls

1. **Instant vs LocalDateTime** — `Instant` is always UTC. Use `LocalDateTime` or `ZonedDateTime` for local times.

2. **String format matters** — `parseInstant` expects ISO format. For custom formats, build the string first.

   ```python syntax
   # Custom format
   table = source.update(
       "Timestamp = parseInstant(Year + `-` + Month + `-` + Day + `T00:00:00Z`)"
   )
   ```

3. **Time zone in extraction** — Always specify timezone when extracting parts.

   ```python syntax
   # CORRECT
   "Hour = hourOfDay(Timestamp, 'ET')"

   # May give UTC hour
   "Hour = hourOfDay(Timestamp)"
   ```

4. **Epoch conversion** — Use `epochNanos`, `epochMicros`, `epochMillis`, `epochSeconds` for numeric timestamps.

   ```python syntax
   table = source.update(
       ["EpochMs = epochMillis(Timestamp)", "FromEpoch = epochNanosToInstant(EpochNanos)"]
   )
   ```

5. **Null timestamps** — Time functions return null when input is null. Handle explicitly if needed.

## Related skills

- `business-calendar` — Market hours, holidays, business day math
- `time-series-join` — Join tables by timestamp
- `rolling-calculations` — Time-windowed calculations
- `transform-data` — Filter by time conditions
