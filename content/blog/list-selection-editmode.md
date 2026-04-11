+++
title = "SwiftUI List: selection, editMode, and NavigationLink without the foot-guns"
description = "Stable multi-select with value-based navigation: explicit editMode, constant empty selection, and hit testing."
date = 2026-04-10

[taxonomies]
tags = ["swift", "swiftui", "list", "navigation"]
+++

Combining `List(selection:)`, `editMode`, and `NavigationLink` on one screen often leads to:

1. **Edit mode resetting** when list data changes (for example after **Add**), even though the user never tapped **Done**.
2. **Taps navigating** instead of toggling selection while the list appears to be in selection mode.
3. **Visual flicker** if you replace the entire row subtree when switching modes.

## What usually goes wrong

- Passing `selection: nil` when not editing switches `List` between different configurations. When the list reloads, environment-driven `editMode` can be disturbed.
- `NavigationLink` participates in hit testing and can consume gestures meant for row selection.

## A stable pattern

1. **Own `EditMode`** with `@State` and inject it using `.environment(\.editMode, $editMode)` so the list is not the implicit owner of that state.
2. **Always pass a `Binding<Set<ID>>` to `List`**: use `$selection` while editing, and **`.constant([])`** when not editing—avoid `nil` so the list does not flip initializer shapes.
3. **Keep a single row tree**: keep `NavigationLink`, and use **`.allowsHitTesting(editMode != .active)`** so selection receives taps while editing.
4. Show **`EditButton()`** only when already in edit mode (it becomes **Done**). Enter edit mode with a separate control (menu, button, etc.).

## Inline reference (split across sections)

The runnable sample lives in **one file** at the end of this article: copy it into `ListSelectionEditModeDemo.swift` (or any name) in a SwiftUI app target and open **Preview**.

---

## Appendix — full listing

Paste the block below into a single `.swift` file in your project. It has no external dependencies beyond SwiftUI.

```swift
import SwiftUI

// MARK: - Demo model

private struct Item: Identifiable, Hashable {
    let id: UUID
    var title: String
}

// MARK: - Screen

private struct SelectionListScreen: View {
    @State private var items: [Item] = [
        Item(id: UUID(), title: "Alpha"),
        Item(id: UUID(), title: "Beta"),
    ]
    @State private var selection: Set<UUID> = []
    @State private var editMode: EditMode = .inactive

    var body: some View {
        NavigationStack {
            List(selection: editMode == .active ? $selection : .constant([])) {
                ForEach(items) { item in
                    NavigationLink(value: item.id) {
                        VStack(alignment: .leading) {
                            Text(item.title)
                            Text(item.id.uuidString.prefix(8) + "…")
                                .font(.caption)
                                .foregroundStyle(.secondary)
                        }
                        .frame(maxWidth: .infinity, alignment: .leading)
                        .contentShape(Rectangle())
                    }
                    .allowsHitTesting(editMode != .active)
                }
            }
            .animation(editMode == .active ? nil : .default, value: editMode)
            .navigationTitle("Items")
            .navigationDestination(for: UUID.self) { id in
                if let item = items.first(where: { $0.id == id }) {
                    Text("Detail: \(item.title)")
                        .padding()
                        .navigationTitle("Detail")
                } else {
                    Text("Missing")
                }
            }
            .toolbar {
                ToolbarItem(placement: .topBarTrailing) {
                    if editMode == .active {
                        EditButton()
                    } else {
                        Menu {
                            Button("Edit list") { editMode = .active }
                        } label: {
                            Image(systemName: "ellipsis.circle")
                        }
                    }
                }
                ToolbarItem(placement: .topBarLeading) {
                    if editMode == .active, !selection.isEmpty {
                        Button(role: .destructive) {
                            items.removeAll { selection.contains($0.id) }
                            selection.removeAll()
                        } label: {
                            Label("Delete (\(selection.count))", systemImage: "trash")
                        }
                    } else if editMode != .active {
                        Button {
                            items.append(Item(id: UUID(), title: "New \(items.count)"))
                        } label: {
                            Label("Add", systemImage: "plus")
                        }
                    }
                }
            }
            .environment(\.editMode, $editMode)
        }
    }
}

#Preview("List + selection + editMode") {
    SelectionListScreen()
}
```

### Checklist

| Symptom | Mitigation |
|--------|------------|
| Edit or selection resets on data reload | `@State editMode` + `.environment(\.editMode, $editMode)` |
| Selection taps trigger navigation | `.allowsHitTesting(false)` on `NavigationLink` while `editMode == .active` |
| Janky mode transitions | Same row hierarchy; disable link instead of swapping row type |
