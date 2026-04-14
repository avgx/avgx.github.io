+++
title = "SwiftUI List: selection, editMode, and NavigationLink together"
description = "Own edit mode, keep a non-nil selection binding, and turn off link hit testing while editing so multi-select stays predictable."
date = 2026-04-10

[taxonomies]
tags = ["swift", "swiftui", "list", "navigation"]
+++

`List(selection:)`, `.environment(\.editMode, ...)`, and `NavigationLink` in one screen is a combination that works until it does not. In practice you often see three symptoms: edit mode drops back to inactive after data changes (for example right after **Add**), taps navigate when the user meant to toggle selection, or the row subtree swaps between two different layouts and the transition feels rough.

None of that is you “using SwiftUI wrong.” It is mostly configuration and gesture ownership. The fix is a small set of rules you can apply once and reuse.

## What is actually going wrong

`List` has more than one initializer family. If you pass `selection: nil` when the user is not editing, you switch between different list configurations; when the array reloads, environment-driven edit mode can reset in surprising ways.

Separately, `NavigationLink` is still a control. It participates in hit testing, so in selection mode it can eat the tap that the list wants for the checkmark.

## A pattern that stays stable

Hold **`EditMode` in `@State`** and inject it with `.environment(\.editMode, $editMode)`. That makes the screen the source of truth instead of letting the list implicitly own mode.

**Always pass a `Binding<Set<ID>>` to `List`:** bind `$selection` while `editMode == .active`, and use `.constant([])` when inactive. Avoid `nil` for selection so you do not change which `List` initializer you are using across toggles.

**Keep one row hierarchy.** Keep the `NavigationLink` wrapper, and set `.allowsHitTesting(editMode != .active)` on it while editing. Selection receives taps; outside edit mode, navigation behaves as usual. If you instead swap the entire row type between modes, you get more layout churn than you need.

**Toolbar:** show `EditButton()` only while editing (where it acts as **Done**). Enter edit mode from a separate control—a menu or a dedicated button—so “edit” and “done” are not fighting the same placement.

## Example you can paste into a preview

Drop this into a single file in a SwiftUI target. It has no dependencies beyond SwiftUI.

```swift
import SwiftUI

private struct Item: Identifiable, Hashable {
    let id: UUID
    var title: String
}

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

## When something still feels off

If edit or selection resets after inserts or reloads, double-check that you are not passing `selection: nil` in inactive mode and that `editMode` is the same `@State` you inject into the environment.

If taps still navigate while editing, confirm hit testing is disabled on the link (not only on an inner label) for the whole row.

If the mode transition animates in a way you dislike, prefer the same row tree and only change hit testing and selection binding, rather than replacing `NavigationLink` with a plain row in edit mode.

For keeping list rows small while sharing a common layout, [List, row, and detail: Named, typealiases, and extensions](@/blog/list-row-detail-protocol-extensions.md) walks through a `Named` protocol and extensions on existing types.
