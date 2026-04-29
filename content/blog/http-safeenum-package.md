+++
title = "SafeEnum: resilient enum decoding for Swift"
description = "A small Swift package for decoding backend-driven enums without breaking on unknown values."
date = 2026-04-29

[taxonomies]
tags = ["swift", "http", "swiftpm", "codable", "json"]
+++

This note continues the small HTTP series — see [RequestResponse](@/blog/http-request-response-package.md) for request shape and decode. Here the focus is **after** you have `Data`: contracts that drift — especially **string-backed enums** the backend can extend without your app shipping the same day.

The published slice is [**SafeEnum**](https://github.com/avgx/SafeEnum).

## Why this shows up in real apps

Backend contracts evolve. Mobile apps often ship out of step with the backend, and users do not update on every release—some keep an old build until it simply stops working.

That creates a familiar failure mode: the backend adds a new enum case, and an older app **crashes while decoding** JSON it could otherwise have handled gracefully.

Suppose you have a plain raw-value enum:

```swift
enum Status: String, Codable {
    case active
    case inactive
}
```

And a DTO:

```swift
struct Response: Codable {
    let status: Status
}
```

Everything works until the backend sends:

```json
{
  "status": "paused"
}
```

Decoding fails. From Swift’s point of view, that is correct: the raw value does not map to a case. For a **network** DTO, though, it is often the wrong outcome — you wanted a value you could branch on or log, not a thrown `DecodingError` that tears down the response.

## Why an optional enum does not solve this

It is tempting to write:

```swift
struct Response: Codable {
    let status: Status?
}
```

That does not help here. `Optional` only covers a **missing key** or **`null`**. It does **not** cover an invalid raw value, so this payload still throws:

```json
{
  "status": "paused"
}
```

That distinction matters. An unknown enum string is not “no value”; it is a **value Swift cannot map** to your cases.

## The usual workaround

A common fix is to fold unknowns into the enum:

```swift
enum Status: Codable {
    case active
    case inactive
    case unknown(String)
}
```

…and hand-write `init(from:)` / encoding. That works, but every enum repeats the same ceremony: custom decoding, raw representation, and wider `switch` surfaces across the codebase—especially noisy inside shared DTO packages.

## SafeEnum

`SafeEnum` is a thin wrapper so your **domain enum stays normal** and only the DTO boundary opts into resilience.

Add the dependency:

```swift
.package(url: "https://github.com/avgx/SafeEnum.git", from: "1.0.0")
```

```swift
import SafeEnum
```

## Usage

Keep the enum standard:

```swift
enum Status: String, Codable {
    case active
    case inactive
}
```

Use the wrapper only where JSON arrives:

```swift
struct Response: Codable {
    let status: SafeEnum<Status>
}
```

That is the whole idea at the call site.

## Known values

The backend sends:

```json
{
  "status": "active"
}
```

You get:

```swift
response.status.value
// .active

response.status.rawValue
// "active"
```

## Unknown values

The backend sends:

```json
{
  "status": "paused"
}
```

You get:

```swift
response.status.value
// nil

response.status.rawValue
// "paused"
```

No crash, and you still retain the string for logging or telemetry.

## Why the distinction still matters

Two different situations:

Missing field:

```json
{}
```

Unknown string for a field that **is** present:

```json
{
  "status": "future_status"
}
```

Those mean different things. `SafeEnum` keeps “could not map to a case” separate from “key absent / null,” which `Status?` alone would conflate once you start special-casing.

## Ergonomics

Fallback when you need a non-optional for business logic:

```swift
let status = response.status.unwrap(or: .inactive)
```

Logging when you want visibility into schema drift:

```swift
if response.status.value == nil {
    logger.warning(
        "Unknown status: \(response.status.rawValue)"
    )
}
```

When the backend adds a case before you ship, you still see the raw string in dashboards instead of only a decode failure.

## Wrapper vs `unknown(String)` on the enum

Both are legitimate. The wrapper keeps `enum Status: String, Codable` unchanged — no per-enum `Codable` boilerplate, no custom mapping tables—so DTO modules stay smaller and your feature code keeps switching on plain `Status` where you have already collapsed unknowns.

---

[RequestResponse](@/blog/http-request-response-package.md) is about request shape and decoding next to the raw response. **SafeEnum** is the same layer — types you decode into — when a string enum on the wire can move ahead of the cases in your build.
