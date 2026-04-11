+++
title = "List, row, and detail: typealiases, Named, and extensions on existing types"
description = "Thin generic SwiftUI over NamedObject and NamedValue; adopt Named with one extension."
date = 2026-04-11

[taxonomies]
tags = ["swift", "swiftui", "protocols", "architecture"]
+++

Passing large models through every view parameter hurts readability and diffing. A practical split:

1. **`Named`** — minimal text contract for list rows (and light headers).
2. **`typealias`** — name repeated protocol compositions once.
3. **`extension ExistingType: Named`** — your type may already be `Identifiable` (and `Codable`, or `ObservableObject`). Add `Named` in an extension; generic rows and lists compile without changing stored properties.

## Protocols and typealiases

```swift
protocol Named {
    var name: String { get }
    var details: String? { get }
}

typealias NamedObject = Identifiable & Named & ObservableObject
typealias NamedValue = Identifiable & Named & Hashable
```

`NamedObject` and `NamedValue` are constraint aliases, not new runtime types.

## Generic row

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

## ObservableObject: class + one extension

A `Document` can start without `Named`. After `extension Document: Named`, it satisfies `NamedObject` and fits the same generic UI.

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

Use `@ObservedObject` (or hold the instance from a store) so list rows update when `@Published` fields change.

## Codable struct: same extension trick

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

`Article` is now a `NamedValue` for generic constraints. Navigation stays **ID-based**; detail loads or edits the full value separately.

## Why extensions instead of parallel DTO types

- No second struct to keep in sync with the domain model.
- UI-specific preview strings stay in the UI layer.
- Generics stay short: `T: NamedObject` or `T: NamedValue`.

---

## Appendix — full listing

One Swift file: two previews (reference flow and value-type flow). Copy into your app target and run **Preview**.

```swift
import SwiftUI
import Combine

// If this target already defines `Named` / `NamedObject` (e.g. Named.swift),
// delete the "Protocols & typealiases" section below and keep the rest.

// MARK: - Protocols & typealiases

protocol Named {
    var name: String { get }
    var details: String? { get }
}

typealias NamedObject = Identifiable & Named & ObservableObject
typealias NamedValue = Identifiable & Named & Hashable

// MARK: - Generic row

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

// MARK: - ObservableObject example

final class Document: ObservableObject, Identifiable {
    let id: UUID
    @Published var title: String
    @Published var body: String

    init(id: UUID = UUID(), title: String, body: String) {
        self.id = id
        self.title = title
        self.body = body
    }
}

extension Document: Named {
    var name: String { title }

    var details: String? {
        let t = body.trimmingCharacters(in: .whitespacesAndNewlines)
        guard !t.isEmpty else { return nil }
        return String(t.prefix(56)) + (t.count > 56 ? "…" : "")
    }
}

@MainActor
final class DocumentLibrary: ObservableObject {
    @Published var documents: [Document.ID: Document] = [:]

    var sortedDocuments: [Document] {
        documents.values.sorted { $0.name.localizedCaseInsensitiveCompare($1.name) == .orderedAscending }
    }

    func seedIfEmpty() {
        guard documents.isEmpty else { return }
        let doc = Document(title: "README", body: String(repeating: "Section.\n", count: 12))
        documents[doc.id] = doc
    }
}

private struct DocumentRowView: View {
    @ObservedObject var document: Document

    var body: some View {
        NamedRow(model: document)
    }
}

private struct DocumentListScreen: View {
    @StateObject private var library = DocumentLibrary()

    var body: some View {
        NavigationStack {
            List {
                ForEach(library.sortedDocuments) { doc in
                    NavigationLink(value: doc.id) {
                        DocumentRowView(document: doc)
                    }
                }
            }
            .navigationTitle("Documents (class)")
            .navigationDestination(for: UUID.self) { id in
                if let doc = library.documents[id] {
                    DocumentDetailView(document: doc)
                } else {
                    Text("Missing")
                }
            }
            .onAppear { library.seedIfEmpty() }
        }
    }
}

private struct DocumentDetailView: View {
    @ObservedObject var document: Document

    var body: some View {
        Form {
            Section("Named (via extension)") {
                NamedRow(model: document)
            }
            TextField("Title", text: $document.title)
            TextEditor(text: $document.body)
                .frame(minHeight: 160)
        }
        .navigationTitle("Detail")
    }
}

// MARK: - Codable value example

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

@MainActor
final class ArticleShelf: ObservableObject {
    @Published private(set) var articles: [Article] = []

    func seedIfEmpty() {
        guard articles.isEmpty else { return }
        articles = [
            Article(id: UUID(), headline: "Protocols", markdown: String(repeating: "# Section\n", count: 8)),
        ]
    }
}

private struct ArticleListScreen: View {
    @StateObject private var shelf = ArticleShelf()

    var body: some View {
        NavigationStack {
            List {
                ForEach(shelf.articles) { article in
                    NavigationLink(value: article.id) {
                        NamedRow(model: article)
                    }
                }
            }
            .navigationTitle("Articles (struct)")
            .navigationDestination(for: UUID.self) { id in
                if let article = shelf.articles.first(where: { $0.id == id }) {
                    ArticleDetailView(article: article)
                } else {
                    Text("Missing")
                }
            }
            .onAppear { shelf.seedIfEmpty() }
        }
    }
}

private struct ArticleDetailView: View {
    let article: Article

    var body: some View {
        Form {
            Section("Named (via extension)") {
                NamedRow(model: article)
            }
            Text(article.markdown)
                .font(.body)
        }
        .navigationTitle(article.headline)
    }
}

// MARK: - Previews

#Preview("NamedObject — Document") {
    DocumentListScreen()
}

#Preview("NamedValue — Article") {
    ArticleListScreen()
}
```

**Same module as an existing `Named` protocol:** delete the protocol + `typealias` block from the appendix and rely on your project’s definitions.

**Two `#Preview` entries in one file:** supported in recent Xcode; if needed, split into two files or keep a single `#Preview`.
