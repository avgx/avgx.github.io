+++
title = "List, row, and detail: Named, typealiases, and extensions"
description = "Keep list rows thin with a small protocol, constraint aliases, and conformance on the types you already have."
date = 2026-04-11

[taxonomies]
tags = ["swift", "swiftui", "protocols", "architecture"]
+++

When every row and header takes the full domain model, previews get noisy and diffs get wide. You do not need a second “list DTO” struct for every screen. You can keep navigation and editing on the real types and still share one row layout.

This post is a small pattern in three parts: a **`Named`** contract for what the list actually shows, **`typealias`** for repeated protocol stacks, and **`extension YourType: Named`** so classes and structs opt in without new stored properties.

## A tiny protocol and two aliases

List UI only needs a title line and an optional subtitle. Everything else—`Codable`, `ObservableObject`, persistence—can stay on the model.

```swift
protocol Named {
    var name: String { get }
    var details: String? { get }
}

typealias NamedObject = Identifiable & Named & ObservableObject
typealias NamedValue = Identifiable & Named & Hashable
```

`NamedObject` and `NamedValue` are compile-time constraints, not new runtime types. Pick `NamedObject` for reference models you observe, and `NamedValue` for value types you pass by ID in navigation.

## One generic row

```swift
struct NamedRow<Model: Named>: View {
    let model: Model

    var body: some View {
        VStack(alignment: .leading) {
            Text(model.name)
            if let details = model.details {
                Text(details)
                    .font(.caption)
                    .foregroundStyle(.secondary)
            }
        }
    }
}
```

Any type that conforms to `Named` can use `NamedRow` with no further generics at the call site.

## Reference types: conform in an extension

Say you already have a document class with `Identifiable` and `@Published` fields. You do not rename `title` to `name` for the UI layer—you map in an extension.

```swift
final class Document: ObservableObject, Identifiable {
    let id: UUID
    @Published var title: String
    @Published var body: String
    // …
}

extension Document: Named {
    var name: String { title }

    var details: String? {
        let t = body.trimmingCharacters(in: .whitespacesAndNewlines)
        guard !t.isEmpty else { return nil }
        return String(t.prefix(56)) + (t.count > 56 ? "…" : "")
    }
}
```

After that, `Document` satisfies `NamedObject`. Use `@ObservedObject` (or a store-owned instance) so rows refresh when `@Published` properties change.

## Value types: same idea

```swift
struct Article: Identifiable, Codable, Hashable {
    let id: UUID
    var headline: String
    var markdown: String
}

extension Article: Named {
    var name: String { headline }

    var details: String? {
        let t = markdown.trimmingCharacters(in: .whitespacesAndNewlines)
        guard !t.isEmpty else { return nil }
        return String(t.prefix(48)) + (t.count > 48 ? "…" : "")
    }
}
```

`Article` is now a `NamedValue`: good for `NavigationLink(value: article.id)` and a detail screen that loads or edits the full value by ID. The list row stays dumb; detail owns the heavy payload.

## Why extensions instead of parallel models

You avoid a shadow struct that must track every field rename. Preview strings and truncation live next to other UI-facing concerns, while generics stay readable as `T: NamedObject` or `T: NamedValue`. If `Named` already exists in your module, add only the conformances and the row.

## Wiring a list and detail (sketch)

Value-based navigation keeps the list stable: rows show `Named`, destinations resolve by `ID`. A minimal shape looks like this—adapt the store to your app.

```swift
List {
    ForEach(shelf.articles) { article in
        NavigationLink(value: article.id) {
            NamedRow(model: article)
        }
    }
}
.navigationDestination(for: UUID.self) { id in
    if let article = shelf.articles.first(where: { $0.id == id }) {
        // full editor or reader
    }
}
```

For `NamedObject`, the row usually wraps `@ObservedObject var document: Document` and passes `document` into `NamedRow(model:)` so updates flow from the shared instance.

If you combine selection and edit mode with `NavigationLink`, see [SwiftUI List: selection, editMode, and NavigationLink](@/blog/list-selection-editmode.md) for a pattern that avoids edit mode and taps fighting each other.
