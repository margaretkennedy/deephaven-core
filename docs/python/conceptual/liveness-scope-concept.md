---
title: How to use liveness scopes
sidebar_label: Liveness scope
---

A liveness scope lets you control when Deephaven stops updating ticking tables. This is useful when you create temporary tables that you'll discard — without a liveness scope, those tables keep updating in the background until Java garbage collection eventually cleans them up.

> [!NOTE]
> Liveness scopes only matter for **ticking (live) tables**. Static tables don't update, so they don't benefit from liveness scope management.

## Why use a liveness scope?

Consider a dashboard that lets users filter data by different criteria. Each time the user changes the filter, you create a new ticking table. The old table is no longer displayed, but Deephaven's [update graph](./table-update-model.md) keeps refreshing it until garbage collection runs — which could be seconds, minutes, or longer.

This wastes CPU cycles and memory. With many filter changes, you could have dozens of "zombie" tables updating in the background.

A liveness scope solves this: when you call `release`, the update graph **immediately** stops refreshing all tables managed by that scope. You don't have to wait for garbage collection.

## Example: Seeing liveness scope in action

This example demonstrates the effect of releasing a liveness scope. We create a ticking time table with a listener that prints a message on each update. When the scope is released, the updates stop immediately.

```python test-set=liveness-demo order=1
from deephaven import time_table
from deephaven.liveness_scope import LivenessScope
from deephaven.table_listener import listen

# Create a liveness scope
scope = LivenessScope()

with scope.open():
    # Create a time table that ticks every second
    tt = time_table("PT1S")

    # Attach a listener that prints on each tick
    def on_update(update, is_replay):
        added_rows = update.added()
        if added_rows:
            print(f"Table updated: {len(added_rows['Timestamp'])} new row(s)")

    my_listener = listen(tt, on_update)
```

After the `with` block exits, the scope is still **open** (not released). The table continues ticking and the listener prints "Table updated..." every second — even though the code block has finished.

To stop the updates, release the scope:

```python test-set=liveness-demo order=2
# Release the scope - updates stop immediately
scope.release()
print("Scope released. The listener will no longer print.")
```

After `release()`, the update graph stops refreshing the table. The listener receives no more updates. This happens immediately — you don't have to wait for garbage collection.

## How liveness scopes work

Deephaven's liveness scopes allow the nodes in a refreshing query's update propagation graph to be updated proactively by assessing the nodes' "liveness" (whether or not they are active), rather than only via actions of the Java garbage collector. This does not replace garbage collection (GC), but it does allow cleanup to happen immediately when objects in the GUI are not needed. This is accomplished internally via reference counting, and works automatically for all users.

When a table is live in Deephaven, the update propagation graph accumulates parent nodes that produce data (data sources) at the top, and all the child nodes that stream down the graph. Deephaven's liveness scope system keeps track of these referents: when child objects cease to be referenced and the parents' liveness count goes down, they will be cleaned up immediately without waiting for GC.

## Demonstrating the problem

Before we demonstrate liveness scopes in action, let's demonstrate the problem that liveness scopes solve.

This query creates a simple tree table grouped by Sym. Two tables will open: `crypto` and `data`.

```python order=null
from deephaven.csv import read as read_csv
from deephaven import merge

crypto = (
    read_csv(
        "https://media.githubusercontent.com/media/deephaven/examples/main/CryptoCurrencyHistory/CSV/FakeCryptoTrades_20230209.csv"
    )
    .first_by("Instrument")
    .update(["ID = Instrument", "Parent = (String)null"])
    .update(
        [
            "Timestamp = (Instant)null",
            "Exchange = (String)null",
            "Price = (Double)null",
            "Size = (Double)null",
        ]
    )
)

data = read_csv(
    "https://media.githubusercontent.com/media/deephaven/examples/main/CryptoCurrencyHistory/CSV/FakeCryptoTrades_20230209.csv"
).update(["ID = String.valueOf(ii)", "Parent = Instrument"])

combo = merge([crypto, data])

combo_tree = combo.tree("ID", "Parent")

data = None
combo = None
```

The above example works, but the query creates many objects that are not needed after it is run. The [`LivenessScope`](https://docs.deephaven.io/core/pydoc/code/deephaven.liveness_scope.html#deephaven.liveness_scope.LivenessScope) can manage these objects and release them when they are no longer needed. This is best practice because it allows Deephaven to conserve memory.

In this example, we will run the same query, but enclose it in a [`LivenessScope`](https://docs.deephaven.io/core/pydoc/code/deephaven.liveness_scope.html#deephaven.liveness_scope.LivenessScope).

```python order=null
from deephaven.liveness_scope import LivenessScope
from deephaven.csv import read as read_csv
from deephaven import merge

scope = LivenessScope()

with scope.open():
    crypto = (
        read_csv(
            "https://media.githubusercontent.com/media/deephaven/examples/main/CryptoCurrencyHistory/CSV/FakeCryptoTrades_20230209.csv"
        )
        .first_by("Instrument")
        .update(["ID = Instrument", "Parent = (String)null"])
        .update(
            [
                "Timestamp = (Instant)null",
                "Exchange = (String)null",
                "Price = (Double)null",
                "Size = (Double)null",
            ]
        )
    )

    data = read_csv(
        "https://media.githubusercontent.com/media/deephaven/examples/main/CryptoCurrencyHistory/CSV/FakeCryptoTrades_20230209.csv"
    ).update(["ID = String.valueOf(ii)", "Parent = Instrument"])

    combo = merge([crypto, data])

    combo_tree = combo.tree("ID", "Parent")

data = None
combo = None
```

## How to create a liveness scope

A liveness scope can be created from the [`liveness_scope`](../reference/engine/liveness-scope.md) function or the [`LivenessScope`](../reference/engine/LivenessScope.md) class directly:

- **`liveness_scope()`** — use in a `with` block or as a function decorator. The scope is automatically released when the block exits.
- **`LivenessScope()`** — gives finer control. Can be opened multiple times and explicitly released with `release()`.

```python order=null
from deephaven.liveness_scope import liveness_scope, LivenessScope

scope_from_function = liveness_scope()
scope_from_class = LivenessScope()
```

## Liveness scope methods

A liveness scope has several methods:

- **`open()`** — opens the scope for use in a `with` statement. Only available on `LivenessScope`.
- **`release()`** — closes the scope and stops updating all managed resources. Only available on `LivenessScope`.
- **`manage(referent)`** — explicitly manages an object in this scope.
- **`preserve(referent)`** — keeps an object live in the outer scope when the current scope exits.
- **`unmanage(referent)`** — stops managing an object so it can be collected.

### Using the function (`liveness_scope`)

The `liveness_scope` function creates a scope that automatically releases when the `with` block or decorated function exits. All ticking tables created inside stop updating when the scope ends.

**Problem:** You need a static snapshot from a ticking table, but the intermediate ticking table keeps updating after you're done with it.

**Solution:** Use a liveness scope to stop the intermediate table, while preserving the snapshot:

```python skip-test
from deephaven import time_table
from deephaven.liveness_scope import liveness_scope

with liveness_scope() as scope:
    tt = time_table("PT1S").update("X = i")  # Ticking table
    snapshot = tt.snapshot()  # Static snapshot of current data
    scope.preserve(snapshot)  # Keep the snapshot alive after scope exits

# tt is no longer updating, but snapshot is still available
result = snapshot
```

Use `preserve` to keep specific objects alive after the scope exits. Everything else stops updating.

**Problem:** A function creates temporary ticking tables to compute a result, but those tables keep updating after the function returns.

**Solution:** Use `liveness_scope` as a decorator — all tables created inside stop updating when the function returns:

```python skip-test
import deephaven.numpy as dhnp
from deephaven.liveness_scope import liveness_scope


@liveness_scope
def get_current_values(source_table):
    # Filter to latest rows, convert to numpy, then let table stop updating
    latest = source_table.last_by("Sym")
    return dhnp.to_numpy(latest)
```

### Using the class (`LivenessScope`)

Use `LivenessScope` when you need to control **when** the scope is released, rather than tying it to a `with` block.

**Problem:** A dashboard creates filtered views based on user input. Each time the user changes the filter, a new ticking table is created — but the old one keeps updating in the background.

**Solution:** Return both the table and its scope. When the user changes the filter, release the old scope before creating a new one:

```python skip-test
from deephaven import time_table
from deephaven.liveness_scope import LivenessScope


def create_filtered_view(filter_value):
    """Create a ticking table that the caller controls."""
    scope = LivenessScope()
    with scope.open():
        tt = time_table("PT1S").where(f"i % 10 == {filter_value}")
        return tt, scope


# Create a view
table, scope = create_filtered_view(5)
# ... display table in UI, use it for a while ...

# User changes filter - stop old table, create new one
scope.release()
table, scope = create_filtered_view(3)
```

## When to use the function vs. the class

| Use case                                                      | Recommendation                       |
| ------------------------------------------------------------- | ------------------------------------ |
| Extract data from a ticking table, then discard it            | `liveness_scope()` with `with` block |
| Decorated function that creates temporary tables              | `liveness_scope` decorator           |
| Table lifetime controlled by user action (e.g., button click) | `LivenessScope` class                |
| Rotating through multiple views in a dashboard                | `LivenessScope` class                |

## Related documentation

- [`LivenessScope`](../reference/engine/LivenessScope.md)
- [`liveness_scope`](../reference/engine/liveness-scope.md)
- [Execution Context](./execution-context.md)
- [Table Update Model](./table-update-model.md)
- [Pydoc](/core/pydoc/code/deephaven.liveness_scope.html#module-deephaven.liveness_scope)
