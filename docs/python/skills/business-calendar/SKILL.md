---
name: business-calendar
description: Filter data to business hours, handle holidays, and perform business day calculations using Deephaven calendars
---

Help the user work with business hours, market schedules, and holiday calendars.

## When to use this skill

- User mentions "business hours", "market hours", "trading hours"
- User asks about "holidays", "business days", "trading days"
- User wants to filter to "NYSE hours", "market open", "pre-market"
- User needs "previous business day", "next trading day"
- User asks about "business day count" or "days between" excluding holidays

## Quick answers

### Get a business calendar

```python syntax
from deephaven.calendar import calendar

# Built-in market calendars
nyse = calendar("NYSE")  # New York Stock Exchange
nasdaq = calendar("NASDAQ")  # NASDAQ
```

> [!NOTE]
> Calendars must be configured in your Deephaven deployment. See [business calendars documentation](../../how-to-guides/business-calendar.md).

### Filter to business hours

```python syntax
from deephaven.calendar import calendar

nyse = calendar("NYSE")

# Only rows during market hours
market_hours = table.where("nyse.isBusinessTime(Timestamp)")

# Only rows on trading days
trading_days = table.where("nyse.isBusinessDay(Date)")
```

### Filter to specific schedule periods

```python syntax
from deephaven.calendar import calendar

nyse = calendar("NYSE")

# Standard market hours only (9:30 AM - 4:00 PM ET)
standard_hours = table.where("nyse.isBusinessTime(Timestamp)")

# Rows on business days (excludes weekends and holidays)
business_only = table.where("nyse.isBusinessDay(Timestamp)")
```

## Calendar methods reference

### Day checks

```python syntax
cal = calendar("NYSE")

table = source.update(
    [
        # Is this a business day?
        "IsTradingDay = cal.isBusinessDay(Date)",
        # Is this a holiday?
        "IsHoliday = cal.isLastBusinessDayOfWeek(Date)",
        # Is this the last business day of the week?
        "IsLastBizDayOfWeek = cal.isLastBusinessDayOfWeek(Date)",
        # Is this the last business day of the month?
        "IsLastBizDayOfMonth = cal.isLastBusinessDayOfMonth(Date)",
    ]
)
```

### Time checks

```python syntax
cal = calendar("NYSE")

table = source.update(
    [
        # Is this during business hours?
        "IsMarketOpen = cal.isBusinessTime(Timestamp)",
    ]
)
```

### Navigation

```python syntax
cal = calendar("NYSE")

table = source.update(
    [
        # Previous/next business day
        "PrevBizDay = cal.previousBusinessDay(Date)",
        "NextBizDay = cal.nextBusinessDay(Date)",
        # N business days forward/back
        "Plus5BizDays = cal.plusBusinessDays(Date, 5)",
        "Minus5BizDays = cal.minusBusinessDays(Date, 5)",
    ]
)
```

### Business day math

```python syntax
cal = calendar("NYSE")

table = source.update(
    [
        # Count business days between dates
        "BizDaysBetween = cal.numberBusinessDates(StartDate, EndDate)",
        # Days until next business day
        "DaysToNextBiz = cal.numberBusinessDates(Date, cal.nextBusinessDay(Date))",
    ]
)
```

### Schedule periods

```python syntax
cal = calendar("NYSE")

table = source.update(
    [
        # Get business schedule for a day
        "MarketOpen = cal.businessDate(Timestamp).businessSchedule().businessStart()",
        "MarketClose = cal.businessDate(Timestamp).businessSchedule().businessEnd()",
    ]
)
```

## Common patterns

### Filter to market hours only

```python syntax
from deephaven.calendar import calendar

nyse = calendar("NYSE")

# Standard market hours
market_hours_data = trades.where("nyse.isBusinessTime(Timestamp)")
```

### Aggregate by trading day

```python syntax
from deephaven.calendar import calendar
from deephaven import agg

nyse = calendar("NYSE")

daily_stats = trades.where("nyse.isBusinessDay(Date)").agg_by(
    [
        agg.first("Open=Price"),
        agg.max_("High=Price"),
        agg.min_("Low=Price"),
        agg.last("Close=Price"),
        agg.sum_("Volume"),
    ],
    by=["Symbol", "Date"],
)
```

### Identify pre-market / after-hours

```python syntax
from deephaven.calendar import calendar

nyse = calendar("NYSE")

categorized = trades.update(
    [
        "Session = nyse.isBusinessTime(Timestamp) ? `Regular` : "
        + "(hourOfDay(Timestamp, 'ET') < 9 || "
        + "(hourOfDay(Timestamp, 'ET') == 9 && minuteOfHour(Timestamp, 'ET') < 30)) ? "
        + "`PreMarket` : `AfterHours`"
    ]
)
```

### Calculate business day returns

```python syntax
from deephaven.calendar import calendar

nyse = calendar("NYSE")

returns = (
    prices.update(
        [
            "PrevBizDay = nyse.previousBusinessDay(Date)",
        ]
    )
    .natural_join(
        prices.view(["Symbol", "PrevDate=Date", "PrevClose=Close"]),
        on=["Symbol", "PrevBizDay=PrevDate"],
        joins="PrevClose",
    )
    .update("Return = (Close - PrevClose) / PrevClose")
)
```

### Filter to specific trading session

```python syntax
# Morning session only (9:30 AM - 12:00 PM ET)
morning = trades.where(
    ["nyse.isBusinessTime(Timestamp)", "hourOfDay(Timestamp, 'ET') < 12"]
)

# Afternoon session (12:00 PM - 4:00 PM ET)
afternoon = trades.where(
    ["nyse.isBusinessTime(Timestamp)", "hourOfDay(Timestamp, 'ET') >= 12"]
)
```

### End of day snapshot

```python syntax
from deephaven.calendar import calendar

nyse = calendar("NYSE")

# Get data from last close
as_of_close = prices.where(
    [
        "Date = nyse.previousBusinessDay(today())",
    ]
).last_by("Symbol")
```

## Available calendars

```python syntax
from deephaven.calendar import calendar_names

# List all available calendars
print(calendar_names())
```

## Common pitfalls

1. **Calendar object scope** — Create the calendar object once and reference it in formulas.

   ```python syntax
   nyse = calendar("NYSE")  # Create once

   table = source.where("nyse.isBusinessTime(Timestamp)")  # Reference by name
   ```

2. **Date vs Instant** — Some calendar methods expect `LocalDate`, others accept `Instant`. Check the method signature.

3. **Time zone alignment** — Calendars use their own timezone (NYSE = ET). Ensure your timestamps are compatible.

4. **Holiday handling** — `isBusinessDay` returns false for holidays. If you need holiday details, check the calendar's holiday schedule.

5. **Half days** — Some markets have early closes (e.g., day before holidays). These are still business days but with shorter hours.

## Related skills

- `work-with-time` — General time parsing and manipulation
- `time-series-join` — Combine with business calendar filtering
- `transform-data` — Filter data using calendar conditions
- `rolling-calculations` — Business-day rolling calculations
