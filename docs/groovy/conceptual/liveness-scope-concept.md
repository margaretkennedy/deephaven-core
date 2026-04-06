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

```groovy test-set=liveness-demo order=1
import io.deephaven.engine.liveness.*
import io.deephaven.util.SafeCloseable
import io.deephaven.engine.table.impl.InstrumentedTableUpdateListener

// Create a liveness scope
scope = new LivenessScope()

SafeCloseable scopeCloseable = LivenessScopeStack.open(scope, false)

// Create a time table that ticks every second
tt = timeTable("PT1S")

// Attach a listener that prints on each tick
listener = new InstrumentedTableUpdateListener("demo-listener") {
    @Override
    void onUpdate(io.deephaven.engine.table.TableUpdate upstream) {
        println "Table updated: ${upstream.added().size()} new row(s)"
    }
}
tt.addUpdateListener(listener)

scopeCloseable.close()
```

After the try block exits, the scope is still **open** (not released). The table continues ticking and the listener prints "Table updated..." every second — even though the code block has finished.

To stop the updates, release the scope:

```groovy test-set=liveness-demo order=2
// Release the scope - updates stop immediately
scope.release()
println "Scope released. The listener will no longer print."
```

After `release`, the update graph stops refreshing the table. The listener receives no more updates. This happens immediately — you don't have to wait for garbage collection.

## How to create a liveness scope

A [`LivenessScope`](/core/javadoc/io/deephaven/engine/liveness/LivenessScope.html) is used with the [`LivenessScopeStack`](/core/javadoc/io/deephaven/engine/liveness/LivenessScopeStack.html) to manage ticking tables:

- **`LivenessScopeStack.open()`** — creates an anonymous scope that automatically releases when the try-with-resources block exits.
- **`LivenessScopeStack.open(scope, true)`** — uses a named scope that automatically releases when the block exits.
- **`LivenessScopeStack.open(scope, false)`** — uses a named scope that stays open after the block exits, allowing you to call `release` later.

### Using an anonymous scope (auto-release)

**Problem:** You need to create intermediate ticking tables to compute a result, but those tables keep updating after you're done with them.

**Solution:** Use an anonymous scope with `LivenessScopeStack.open()`. All tables created inside stop updating when the block exits:

```groovy skip-test
import io.deephaven.engine.liveness.*
import io.deephaven.util.SafeCloseable

try (SafeCloseable ignored = LivenessScopeStack.open()) {
    // Create intermediate ticking tables
    tt = timeTable("PT1S").update("X = i")
    filtered = tt.where("X % 2 == 0")

    // Extract static result
    result = filtered.snapshot()

    // Use scope.manage() or transferTo() to preserve result if needed
}
// tt and filtered stop updating here; result is still available if preserved
```

### Using a named scope (manual release)

**Problem:** A dashboard creates filtered views based on user input. Each time the user changes the filter, a new ticking table is created — but the old one keeps updating in the background.

**Solution:** Create a named scope and release it when the user changes the filter:

```groovy skip-test
import io.deephaven.engine.liveness.*
import io.deephaven.util.SafeCloseable

def createFilteredView(filterValue) {
    scope = new LivenessScope()
    SafeCloseable ignored = LivenessScopeStack.open(scope, false)
    try {
        tt = timeTable("PT1S").where("i % 10 == ${filterValue}")
        return [tt, scope]
    } finally {
        ignored.close()
    }
}

// Create a view
(table, scope) = createFilteredView(5)
// ... display table in UI, use it for a while ...

// User changes filter - stop old table, create new one
scope.release()
(table, scope) = createFilteredView(3)
```

## Liveness scope methods

The [`LivenessScope`](/core/javadoc/io/deephaven/engine/liveness/LivenessScope.html) class provides these key methods:

- **`release`** — closes the scope and stops updating all managed resources.
- **`manage(referent)`** — explicitly manages an object in this scope.
- **`unmanage(referent)`** — stops managing an object so it can be collected.
- **`transferTo(other)`** — transfers all managed referents to another scope.

The [`LivenessScopeStack`](/core/javadoc/io/deephaven/engine/liveness/LivenessScopeStack.html) class provides:

- **`open()`** — creates an anonymous scope that auto-releases when closed.
- **`open(scope, autoRelease)`** — pushes a scope onto the stack; if `autoRelease` is true, releases when closed.
- **`push(scope)`** / **`pop(scope)`** — manual stack management (prefer try-with-resources instead).
- **`peek()`** — returns the current active scope.

## When to use auto-release vs. manual release

| Use case                                           | Recommendation                                               |
| -------------------------------------------------- | ------------------------------------------------------------ |
| Extract data from a ticking table, then discard it | `LivenessScopeStack.open()` (anonymous, auto-release)        |
| Compute intermediate results within a method       | `LivenessScopeStack.open(scope, true)` (named, auto-release) |
| Table lifetime controlled by user action           | `LivenessScopeStack.open(scope, false)` + `scope.release()`  |
| Rotating through multiple views in a dashboard     | Named scope with manual `release`                            |

## Related documentation

- [`LivenessScope`](../reference/engine/LivenessScope.md)
- [`LivenessScopeStack`](/core/javadoc/io/deephaven/engine/liveness/LivenessScopeStack.html)
- [Execution Context](./execution-context.md)
- [Table Update Model](./table-update-model.md)
