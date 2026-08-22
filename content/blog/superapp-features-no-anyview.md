+++
title = "Superapp features without AnyView"
description = "Type-erasing feature roots resets SwiftUI identity. Identify screens with FeatureID, compose them with a typed ViewBuilder in each app, and link only the packages that app ships."
date = 2026-08-22

[taxonomies]
tags = ["swift", "swiftui", "architecture", "swiftpm"]
+++

Several superapps share a pool of feature packages. Each app ships its own subset: a full VMS client app might include cameras and events; a doorbell app might ship cameras and intercom.

The useful constraint is that **an app should only link the features it shows.** Camera UI must not pull in events or intercom. A shared host that observes the session should not import every screen just to pick one `rootView`.

The tempting way to keep that split is a registry of `(FeatureID) -> AnyView`: feature packages register closures at bootstrap, the parent looks them up in `body`. That lookup is where identity goes wrong.

## Why AnyView on `rootView` is a problem

A parent **observes** some model—connection state, ticks, a session object. In `body` it asks: give me `rootView` for this feature.

If the answer is wrapped in `AnyView`, every model update rebuilds `body` and produces a **new** `AnyView`. SwiftUI does not see “the same camera list.” It sees a new type-erased box. The old tree is thrown away (`@State` and all) and built again. The feature id did not change; the screen still resets.

`any View` is the same erasure. The problem is not the word `AnyView`. It is type-erasing the feature root.

Return a **concrete** view from `@ViewBuilder`—a `switch` on the feature id. The parent can still observe and refresh. The child keeps its type, so SwiftUI treats it as an update, not a replacement.

## Three ways to assemble the superapp

Shared feature packages; each app picks a subset.

1. **Registry of `AnyView`.** Packages stay decoupled. Identity does not.
2. **One `switch` in a shared UI module.** Identity is fine. Every app is forced to depend on every feature package, including ones it never shows.
3. **A composition type in the app.** That app depends only on the features it ships. `rootView` is a typed `@ViewBuilder`. The observing parent is generic over the composition and never imports camera or event or intercom UI.

Whether a feature is **on** for this account is a separate check. Availability does not build the view.

## FeatureID and FeatureComposition

`FeatureID` is a small string-backed type. One value for “which feature,” labels, and the `switch`.

`FeatureComposition` has one job: `rootView`.

```swift
protocol FeatureComposition {
    associatedtype Root: View

    @ViewBuilder
    static func rootView(for feature: FeatureID) -> Root
}
```

Each app owns one type that conforms. The observing parent is generic over that type and calls `rootView` **without** wrapping it.

## A complete sketch, split by package

Kernel package **Features**—no screens, only the id and the protocol.

```swift
import SwiftUI

struct FeatureID: Hashable, Identifiable, RawRepresentable, ExpressibleByStringLiteral {
    let rawValue: String
    var id: String { rawValue }
    init(rawValue: String) { self.rawValue = rawValue }
    init(stringLiteral value: String) { self.rawValue = value }

    static let cameras = FeatureID(rawValue: "cameras")
    static let events = FeatureID(rawValue: "events")
    static let intercom = FeatureID(rawValue: "intercom")
}

protocol FeatureComposition {
    associatedtype Root: View

    @ViewBuilder
    static func rootView(for feature: FeatureID) -> Root
}
```

**CameraUI**, **EventsUI**, and **IntercomUI** do not import each other. `@State` on the camera list is the piece that would reset if the host erased the root.

```swift
// CameraUI
import SwiftUI

struct CameraList: View {
    @State private var selected: String?

    var body: some View {
        List(["Gate", "Lobby"], id: \.self, selection: $selected) { name in
            Text(name)
        }
        .navigationTitle("Cameras")
        .safeAreaInset(edge: .bottom) {
            Text(selected.map { "Selected: \($0)" } ?? "Select a camera")
                .padding()
        }
    }
}
```

```swift
// EventsUI
import SwiftUI

struct EventList: View {
    var body: some View {
        List(["Motion at gate", "Door opened"], id: \.self) { Text($0) }
            .navigationTitle("Events")
    }
}
```

```swift
// IntercomUI
import SwiftUI

struct IntercomView: View {
    var body: some View {
        VStack {
            Text("Front door")
            Button("Talk") { }
        }
        .navigationTitle("Intercom")
    }
}
```

**AppHost** depends on Features only. It observes the session, shows the tick counter, and puts every feature the app asked for into a `TabView`. It does not import CameraUI, EventsUI, or IntercomUI.

```swift
import SwiftUI

@MainActor
final class SessionModel: ObservableObject {
    @Published var ticks = 0

    func tickForever() async {
        while !Task.isCancelled {
            try? await Task.sleep(for: .seconds(1))
            ticks += 1
        }
    }
}

struct ObservingHost<Composition: FeatureComposition>: View {
    @ObservedObject var model: SessionModel
    let features: [FeatureID]

    var body: some View {
        VStack {
            Text("session ticks: \(model.ticks)")
            TabView {
                ForEach(features) { feature in
                    Composition.rootView(for: feature)
                        .tabItem { Text(feature.rawValue) }
                }
            }
        }
        .task { await model.tickForever() }
    }
}
```

The timer rebuilds `ObservingHost.body` every second. Because `rootView` is a concrete `@ViewBuilder` tree, `CameraList` is the same type as last time. Select **Gate**: the highlight and “Selected: Gate” stay while ticks climb.

Wrap the same call in `AnyView` and that list **blinks**: each tick is a new type-erased box, `@State` is thrown away, selection jumps back to none.

```swift
Composition.rootView(for: feature)                 // selection survives ticks
AnyView(Composition.rootView(for: feature))        // selection resets every second
```

**VmsApp** is the full VMS: it depends on CameraUI and EventsUI and does not depend on IntercomUI.

```swift
import SwiftUI

enum VmsFeatures: FeatureComposition {
    @ViewBuilder
    static func rootView(for feature: FeatureID) -> some View {
        switch feature {
        case .cameras: CameraList()
        case .events: EventList()
        default: EmptyView()
        }
    }
}

struct VmsRoot: View {
    @StateObject private var session = SessionModel()

    var body: some View {
        ObservingHost<VmsFeatures>(model: session, features: [.cameras, .events])
    }
}
```

**DoorbellApp** is the smaller CCTV client. Same host, different composition, **CameraUI and IntercomUI** — EventsUI is not in the graph.

```swift
import SwiftUI

enum DoorbellFeatures: FeatureComposition {
    @ViewBuilder
    static func rootView(for feature: FeatureID) -> some View {
        switch feature {
        case .cameras: CameraList()
        case .intercom: IntercomView()
        default: EmptyView()
        }
    }
}

struct DoorbellRoot: View {
    @StateObject private var session = SessionModel()

    var body: some View {
        ObservingHost<DoorbellFeatures>(model: session, features: [.cameras, .intercom])
    }
}
```

The VMS links cameras and events. The doorbell app links cameras and intercom. The host never type-erases `rootView`.

Do not erase `rootView`. Put the `switch` in the app that owns the dependencies.
