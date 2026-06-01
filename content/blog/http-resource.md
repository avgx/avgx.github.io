+++
title = "Resource: one enum for async data state"
description = "A dependency-free Swift enum for idle, loading, cached, refreshing, failed, and stale data — Equatable and Sendable for managers and SwiftUI."
date = 2026-05-31

[taxonomies]
tags = ["swift", "http", "swiftpm", "architecture", "swiftui"]
+++

This note continues the small HTTP series.
Here the focus shifts to **what the app shows while data moves**: manager and UI state after the network call, not the wire format itself.

The published slice is [**Resource**](https://github.com/avgx/Resource).

## Code

```swift
public enum Resource<Value, Failure>
where Value: Equatable & Sendable, Failure: Error & Equatable & Sendable {
    case idle
    case loading
    case available(Value)
    case refreshing(Value)
    case failed(Failure)
    case stale(Value, Failure)
}
```

## What problem it solves

Most screens need to answer more than “do we have a value?”:

- nothing loaded yet;
- first load in flight;
- data on screen;
- refresh in flight while old data stays visible;
- hard failure with nothing to show;
- stale cache after a failed refresh.

The usual workaround is a bundle of properties:

```swift
var value: [Item]?
var isLoading = false
var error: Error?
```

That model quietly allows impossible states — value and error together, 
loading with no distinction between first fetch and refresh — and `Error` itself is not `Equatable`, 
so the whole snapshot is awkward in `@Observable` models and tests.

`Resource` is one enum. Exactly one case is active. Associated data lives in the case, not in parallel flags.

## States

No cached data:

```
.idle
.loading
.failed(error)
```

Cached data:

```
.available(value)
.refreshing(value)
.stale(value, error)
```

## Transitions

Typical lifecycle a manager drives:

```
.idle
↓
.loading
↓
.available(value)
↓
.refreshing(value)
↓
.available(updatedValue) or .stale(value, error)
```

First load can fail: `.loading → .failed(error)`. 
Retry from empty state: `.failed → .loading`. 
After stale data, retry keeps the cache: `.stale → .refreshing`.

The enum does not enforce transitions — your manager assigns the next case — but naming the cases this way keeps refresh and stale paths explicit instead of overloading `isLoading`.

## Why `Failure`, not `Error`

`Resource` is `Equatable`. A type-erased `Error` is not, so the failure associated value must be something you can compare and show — a small app error enum, a message struct, `String`. 
Map `URLError`, decoding failures, or backend codes into `Failure` once, at the repository or manager boundary.

## Computed properties

For code that does not want a full `switch`:

- `value` — cached payload in `.available`, `.refreshing`, and `.stale`;
- `isAvailable` — `value != nil` (true for all three cached cases);
- `isLoading` — `.loading` or `.refreshing`;
- `failure` — error in `.failed` or `.stale`;
- `isFailed` — `failure != nil`.

## Usage

```swift
@Observable
final class ItemsManager {
    private(set) var items: Resource<[Item], AppError> = .idle
}
```
Or for iOS15
```swift
@MainActor
final class ItemsManager: ObservableObject {
        @Published public private(set) var items: Resource<[Item], AppError> = .idle
}
```

SwiftUI maps one case to one view — no extra flags:

```swift
switch manager.items {
case .idle:
    ContentUnavailableView("No Data")
case .loading:
    ProgressView()
case .available(let items), .refreshing(let items):
    ItemsList(items, isRefreshing: manager.items.isLoading)
case .failed(let error):
    ErrorView(error)
case .stale(let items, let error):
    ItemsList(items, banner: error)
}
```

