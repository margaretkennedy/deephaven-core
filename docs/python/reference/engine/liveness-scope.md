---
title: liveness_scope
---

`liveness_scope` is a Python method for creating and opening a [`LivenessScope`](./LivenessScope.md) for running a block of code. Use this function to wrap a block of code using a `with` statement. For the duration of the `with` block, the liveness scope will be open and any liveness referents created will be automatically managed by it.

## Syntax

```python syntax
scope = liveness_scope()
```

## Parameters

None.

## Returns

A [`LivenessScope`](./LivenessScope.md).

## Examples

Use `liveness_scope` in a `with` block when you need to create intermediate ticking tables that should stop updating after the block exits. In this example, a function-generated blink table is created to compute a snapshot, then stops updating when the block exits:

```python skip-test
from deephaven.liveness_scope import liveness_scope
from deephaven import function_generated_table, empty_table


def random_data():
    return empty_table(5).update(
        ["X = randomInt(0, 10)", "Y = randomDouble(-50.0, 50.0)"]
    )


with liveness_scope() as scope:
    blink_table = function_generated_table(
        table_generator=random_data, refresh_interval_ms=1000
    )
    snapshot = blink_table.snapshot()  # Static snapshot
    scope.preserve(snapshot)  # Keep snapshot alive after scope exits

# blink_table stops updating here; snapshot is still available
result = snapshot
```

Use `preserve` to keep specific objects alive after the scope exits. Everything else stops updating.

As a decorator, the scope wraps the entire function — all ticking tables created inside stop updating when the function returns:

```python skip-test
import deephaven.numpy as dhnp
from deephaven.liveness_scope import liveness_scope


@liveness_scope
def get_current_values(source_table):
    latest = source_table.last_by("Sym")
    return dhnp.to_numpy(latest)  # Table stops updating after return
```

## Related documentation

- [How to use Liveness Scopes](../../conceptual/liveness-scope-concept.md)
- [`LivenessScope`](./LivenessScope.md)
- [Pydoc](/core/pydoc/code/deephaven.liveness_scope.html#deephaven.liveness_scope.liveness_scope)
