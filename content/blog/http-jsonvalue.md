+++
title = "JSONValue: JSON-shaped values without Any"
description = "A dependency-free Swift enum for dynamic API fields, query rows, and test fixtures"
date = 2026-05-19

[taxonomies]
tags = ["swift", "http", "swiftpm", "codable", "json"]
+++

This note continues the small HTTP series — see [RequestResponse](@/blog/http-request-response-package.md) for request shape and decode, and [SafeEnum](@/blog/http-safeenum-package.md) when a **known** string enum drifts. Here the problem is different: the payload is JSON, but you cannot or should not freeze it in one `Codable` struct.

The published slice is [**JSONValue**](https://github.com/avgx/JSONValue).

## When a struct is the wrong tool

Sometimes the shape is intentionally open:

- a search or analytics API returns **rows** whose keys are metric names (`"cloud.domain"`, `"@count"`, `"rectangle.h"`) and change with the query
- a client builds a **filter** or chart config as nested JSON and posts it back
- a backend adds **metadata** next to a stable core, and you only care about a few keys today
- tests need to assert on a **whole JSON body** without maintaining a struct per fixture variant

`[String: Any]` and `AnyCodable` decode almost anything. The cost shows up later: casting, non-JSON values slipping through, numbers that are `Int` in one response and `Double` in another, and no reliable `==` for fixtures.

`JSONValue` is a closed `enum` — `null`, `bool`, `number`, `string`, `array`, `object` — that only represents what JSON can represent.

## JSONValue

Add the dependency:

```swift
.package(url: "https://github.com/avgx/JSONValue.git", from: "1.0.2")
```

```swift
import JSONValue
```

Numbers use a nested `JSONNumber` (`.int(Int64)` or `.double(Double)`) under `.number`, so large IDs stay exact and `42` compares equal to `42.0` when you diff responses. For everyday access, prefer accessors (`intValue`, `doubleValue`, `stringValue`, …) instead of matching on storage.

## Life examples

### Query rows with dotted keys

A monitoring or BI API often returns flat rows, not nested structs:

```json
{
  "cloud.domain": 1274,
  "@count": 42,
  "rectangle.h": 0.23851851851851846,
  "detector.type": "faceAppeared"
}
```

Model the row as a dictionary, read what you need, ignore the rest until the product cares:

```swift
typealias QueryRow = [String: JSONValue]

let row: QueryRow = [
    "cloud.domain": 1274,
    "@count": 42,
    "rectangle.h": 0.23851851851851846,
    "detector.type": "faceAppeared",
]

let domain = row["cloud.domain"]?.intValue
let count = row["@count"]?.intValue
let height = row["rectangle.h"]?.doubleValue
let detector = row["detector.type"]?.stringValue
```

No code generation per metric, no `as? Int` on `Any`.

### Filters and chart config you assemble in the app

The client owns a JSON document the server interprets:

```swift
let filter: JSONValue = [
    "version": 0,
    "table": "events",
    "period": ["type": "today"],
    "limit": 100,
]

// POST body: encode filter directly
let body = try JSONEncoder().encode(filter)
```

Nested literals and `@dynamicMemberLookup` keep nested access readable in UI code:

```swift
filter.period?["type"]?.stringValue  // "today"
```

### Preview arguments and heterogeneous arrays

Dashboards and widgets often take an argument list whose items differ in type:

```swift
let previewArgs: [JSONValue] = [
    "1 hour",
    "2026-05-18T00:00:00Z",
    1000,
]
```

Encode as a JSON array; decode the slots you understand with subscripts and accessors.

### Stable core, dynamic bag

When part of the response is fixed and part is not, decode into `JSONValue` first, then peel off the typed slice:

```swift
struct Row: Codable, Equatable {
    let cloudDomain: Int
    let detectorType: String

    enum CodingKeys: String, CodingKey {
        case cloudDomain = "cloud.domain"
        case detectorType = "detector.type"
    }
}

let value: JSONValue = [
    "cloud.domain": 1274,
    "detector.type": "faceAppeared",
    "extra.metric": 99,  // ignored until you need it
]

let row = try value.decode(Row.self)
```

The reverse path — `try JSONValue(row)` — is useful when building request bodies from existing `Encodable` models. Both directions go through `JSONEncoder` / `JSONDecoder`; that is intentional convenience, not a hot-path zero-cost bridge.

### Tests that compare JSON, not reference types

```swift
let expected: JSONValue = [
    "table": "events",
    "limit": 5,
    "filter": ["period": ["type": "forever"]],
]

let actual = try JSONDecoder().decode(JSONValue.self, from: responseData)
#expect(actual == expected)
```

`JSONValue` is honestly `Equatable`. That is awkward with `AnyCodable` and `Any` in general.

## JSONValue vs AnyCodable

[AnyCodable](https://github.com/Flight-School/AnyCodable) (and similar wrappers) store an `Any`. They are handy for a one-off decode into a strongly typed model. For **long-lived** dynamic JSON in public types, tests, or networking layers, the differences add up:

| | `JSONValue` | `AnyCodable` |
|---|---|---|
| Storage | closed `enum` | `Any` |
| What can live inside | JSON primitives only | whatever decoding produced (`Int`, `Double`, `NSNumber`, sometimes more) |
| `Equatable` | yes, for trees and fixtures | fragile or absent |
| `Sendable` | by design | usually `@unchecked Sendable` over `Any` |
| Access | `intValue`, `stringValue`, … | `value as? Int` (breaks when the runtime type is `Double`) |
| Round-trip encode | same cases as decode | can diverge for nested numbers and collections |

At the call site:

```swift
// AnyCodable-style
let count = (row["@count"]?.value as? Int)

// JSONValue
let count = row["@count"]?.intValue
```

The second form centralizes “is this a whole number?” across `JSONNumber.int` and `JSONNumber.double`.

`AnyCodable` is still fine when JSON is decoded once, converted immediately into your models, never exposed from a library API, and you do not need equality or predictable re-encoding.

If you publish something like `public typealias QueryRow = [String: JSONValue]`, consumers get a documented JSON contract. With `AnyCodable`, every caller inherits casting and runtime type guessing.

## What stays out of scope

`JSONValue` is not a query language, schema validator, or JSONPath implementation. It is a **transport and fixture** type: hold JSON, navigate it, compare it, and cross into `Codable` when a boundary is worth typing.

---

[RequestResponse](@/blog/http-request-response-package.md) is where you describe the call and decode next to status and raw data. **JSONValue** is for the responses (or bodies) where the JSON tree itself is the useful abstraction — open rows, filters, and bags — without falling back to `Any`.
