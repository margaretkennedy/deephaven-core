---
title: LivenessScope
---

`LivenessScope` is a Groovy class that manages the reference counting of tables and other query resources that are created within it.

## Syntax

```groovy syntax
scope = new LivenessScope()
```

## Parameters

None.

## Returns

An instance of a `LivenessScope` class.

## Examples

The following example creates a [function-generated](../table-operations/create/create.md) blink table inside a `LivenessScope`. After the try block exits, the scope remains open and the table continues updating. Calling `release` stops the update graph from refreshing the table immediately — you don't have to wait for garbage collection.

```groovy ticking-table order=null
import io.deephaven.engine.liveness.*
import io.deephaven.engine.context.ExecutionContext
import io.deephaven.util.SafeCloseable
import io.deephaven.engine.table.impl.util.FunctionGeneratedTableFactory

defaultCtx = ExecutionContext.getContext()

randomData = { ->
    try (SafeCloseable ignoredContext = defaultCtx.open()) {
        return emptyTable(5).update("X = randomInt(0, 10)", "Y = randomDouble(-50.0, 50.0)")
    }
}

scope = new LivenessScope()

try (SafeCloseable ignored = LivenessScopeStack.open(scope, false)) {
    // Create a blink table that generates 5 rows per second
    blinkTable = FunctionGeneratedTableFactory.create(randomData, 1000)
}

// Table continues ticking here (scope is open, not released)

// When done, stop the table from updating
scope.release()
```

`LivenessScope` is useful when you need explicit control over when tables stop updating — for example, when generating data for a temporary analysis or rotating through views in a dashboard.

## Related documentation

- [How to use Liveness Scopes](../../conceptual/liveness-scope-concept.md)
- [Execution Context](../../conceptual/execution-context.md)
- [Javadoc](/core/javadoc/io/deephaven/engine/liveness/LivenessScope.html)
