---
title: LivenessScope
---

`LivenessScope` is a Python class that manages the reference counting of tables and other query resources that are created within it.

## Syntax

```python syntax
scope = LivenessScope()
```

## Parameters

None.

## Returns

An instance of a `LivenessScope` class.

## Examples

The following example creates a [function-generated](../table-operations/create/function_generated_table.md) blink table inside a `LivenessScope`. After the `with` block exits, the scope remains open and the table continues updating. Calling `release` stops the update graph from refreshing the table immediately — you don't have to wait for garbage collection.

```python ticking-table order=null
from deephaven.liveness_scope import LivenessScope
from deephaven import function_generated_table, empty_table


def random_data():
    return empty_table(5).update(
        ["X = randomInt(0, 10)", "Y = randomDouble(-50.0, 50.0)"]
    )


scope = LivenessScope()

with scope.open():
    # Create a blink table that generates 5 rows per second
    blink_table = function_generated_table(
        table_generator=random_data, refresh_interval_ms=1000
    )

# Table continues ticking here (scope is open, not released)

# When done, stop the table from updating
scope.release()
```

`LivenessScope` is useful when you need explicit control over when tables stop updating — for example, when generating data for a temporary analysis or rotating through views in a dashboard.

## Related documentation

- [How to use Liveness Scopes](../../conceptual/liveness-scope-concept.md)
- [`liveness_scope`](./liveness-scope.md)
- [Pydoc](/core/pydoc/code/deephaven.liveness_scope.html#deephaven.liveness_scope.LivenessScope)
