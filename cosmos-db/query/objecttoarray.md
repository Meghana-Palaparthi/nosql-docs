---
title: OBJECTTOARRAY
description: The `OBJECTTOARRAY` function converts field/value pairs in a JSON object to a JSON array.
ai-usage: ai-assisted
ms.date: 05/13/2026
---

# `OBJECTTOARRAY` - Query language in Cosmos DB (in Azure and Fabric)

The `OBJECTTOARRAY` function converts field/value pairs in a JSON object to a JSON array.

## Syntax

```nosql
OBJECTTOARRAY(<object_expr> [, <string_expr_1>, <string_expr_2>])
```

## Arguments

| | Description |
| --- | --- |
| **`object_expr`** | An object expression with properties in field/value pairs. |
| **`string_expr_1`** | A string expression with a name for the field representing the *field* portion of the original field/value pair. |
| **`string_expr_2`** | A string expression with a name for the field representing the *value* portion of the original field/value pair. |

## Return types

Returns an array of elements with two fields, either `k` and `v` or custom-named fields.

## Examples

This section contains examples of how to use this query language construct.

### Convert object to array

In this example, the `OBJECTTOARRAY` function is used to convert a JSON object to an array.

```nosql
SELECT VALUE
  OBJECTTOARRAY({
    "a": "12345",
    "b": "67890"
  })
```

```json
[
  [
    {
      "k": "a",
      "v": "12345"
    },
    {
      "k": "b",
      "v": "67890"
    }
  ]
]
```

## Dynamic property access

Azure Cosmos DB query language doesn't provide a built-in function to enumerate object property names (like `OBJECT_KEYS`). To enumerate unknown property names, use `OBJECTTOARRAY` and `JOIN` over the resulting key-value pairs.

```json
{
  "id": "1",
  "metadata": {
    "color": "blue",
    "size": "small"
  }
}
```

```nosql
SELECT p.id, kv["key"] AS propertyName, kv["value"] AS propertyValue
FROM p
JOIN kv IN OBJECTTOARRAY(p.metadata, "key", "value")
```

This pattern returns one row per property in `p.metadata` for each item.

```json
[
  { "id": "1", "propertyName": "color", "propertyValue": "blue" },
  { "id": "1", "propertyName": "size", "propertyValue": "small" }
]
```

If you only need to check whether a specific property exists, use [`IS_DEFINED`](is-defined.md).

```nosql
SELECT p.id, IS_DEFINED(p.metadata["region"]) AS hasRegion
FROM p
```

For better query performance and simpler filtering, model variable attributes as arrays when possible instead of using dynamic object keys. Arrays are easier to index and filter directly than dynamically enumerated object properties.

## Related content

- [JOIN](join.md)
- [IS_DEFINED](is-defined.md)
- [Keywords](keywords.md)
- [Subquery](subquery.md)
