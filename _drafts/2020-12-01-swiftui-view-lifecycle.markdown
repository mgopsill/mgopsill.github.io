---
title: "SwiftUI View Lifecycle"
layout: post
date: 2020-12-01 22:44
image: /assets/images/markdown.jpg
headerImage: false
tag:
- swiftui
star: true
category: blog
author: mikegopsill
description: SwiftUI View Lifecycle Description
---
Each view undergoes a series of events from its birth to its death, which is referred to as a *lifecycle*. Understanding it is essential when building apps in SwiftUI. In this article, we will explore the three phases of the SwiftUI view lifecycle.
Before we talk about lifecycle, we need to agree on what a view is.
# Understanding SwiftUI Views
To describe what’s displayed onscreen, we create a graph of *views*.
> **View** is a _definition_ of a piece of UI. [1](https://developer.apple.com/videos/play/wwdc2020/10040/)

SwiftUI uses this definition to create an appropriate rendering.
> **View** is a function of a state. [2](https://developer.apple.com/videos/play/wwdc2019/226/)

To update UI, we don’t mutate the view graph directly. Instead, we modify the state, and a new view graph is calculated from the state. Then SwiftUI performs rendering to reflect the changes.

> The identity and lifetime of SwiftUI views are separate from the lifetime of `struct`s that define them. [1](https://developer.apple.com/videos/play/wwdc2020/10040/)

`SwiftUI.ViewGraph` manages a rendering and non-rendering hierarchy of views. From what I’ve learned from reflecting the SwiftUI binary, their corresponding names are `DisplayList` and `ViewList`:

1. The *non-rendering hierarchy* is a definition of what needs to be displayed onscreen. SwiftUI creates it using `body`s of the view structs that we provide. On every UI update, SwiftUI traverses the previous and new snapshots of the non-rendering hierarchy to calculate changes. It relies on the private `AttributeGraph` framework to represent view attributes and perform diffing.
2. SwiftUI uses the *rendering hierarchy* to produce an actual drawing. Views from this hierarchy have their own identities, and their lifetime corresponds to how long they are displayed onscreen.

> The type of view `body` encodes the whole view subgraph.

If you inspect the type of your app’s root view `body`, you’ll notice that it contains the entire view hierarchy:
```swift
struct ContentView: View {
    var body: some View {
        Text("Hello, world!")
            .background(Color.yellow)
            .font(.title)
            .dump()
    }
}

extension View {
    func dump() -> Self {
        print(Mirror(reflecting: self))
        return self
    }
}
```

This will print:
```swift
Mirror for ModifiedContent<ModifiedContent<Text, _BackgroundModifier<Color>>, _EnvironmentKeyWritingModifier<Optional<Font>>>`
```

This includes not only visible views, but also all variations of views that can appear onscreen:

```swift
struct ContentView_: View {
    var body: some View {
        Group {
            if true {
                Color.yellow
            } else {
                Text("Impossible")
            }
        }
        .dump()
    }
}
```

This will print:

`Mirror for Group<_ConditionalContent<Color, Text>>`

Although `Text` can never become visible, it’s still present in `_ConditionContent<Color, Text`.

According to [Thinking in SwiftUI](https://www.objc.io/books/thinking-in-swiftui/), the benefit of encoding the entire view hierarchy instead of just currently visible one is that it allows for more efficient diffing.

> Views should be lightweight and cheap. [1](https://developer.apple.com/videos/play/wwdc2020/10040/)

The consequence of encoding the entire view hierarchy into the `body`’s type is that large parts of the view graph are constructed upfront. Furthermore, according to  [Data Essentials in SwiftUI](https://developer.apple.com/videos/play/wwdc2020/10040/) , views are copied very frequently. Therefore, we must keep two rules in mind when designing SwiftUI views:

* Make initialization cheap.
* Make `body` a  [pure function](https://www.vadimbulavin.com/pure-functions-higher-order-functions-and-first-class-functions-in-swift/) , free of side effects. Simply create your view and return, no dispatching extra work.

# SwiftUI View Lifecycle
> **Lifecycle** is the series of events that happen from the creation of a SwiftUI view to its destruction.

Each view in SwiftUI has a *lifecycle* that we can observe and manipulate during its three main phases. The three phases are *Appearing*, *Updating*, and *Disappearing*. They are summarized in the below diagram:

![swiftui view lifecycle](/assets/images/swiftui-view-lifecycle.svg)

Apart from the three lifecycle phases, there are two rendering phases: *Layout* and *Commit*.

In the *Layout* phase, SwiftUI initializes the non-rendering view hierarchy, computes frames, connects views to state, calculates diffs to be committed onscreen.
The *layout* phase must be pure. Any side effects will result in undefined behavior:

```swift
struct ContentView: View {
    @State var count = 0

    var body: some View {
        count += 1 // ❌ side effect
        ...
    }
}
```

The above code generates a runtime warning:

`Modifying state during view update, this will cause undefined behavior.`

In the **Commit** phase, SwiftUI updates the rendering view hierarchy, commits all changes onscreen, and destroys all views which are not needed anymore.

# Appearing
**Appearing** means inserting a view into a view graph. In this phase, a view is initialized, subscribed to the state, and rendered for the first time.

![appearing](/assets/images/appearing.svg)

1. At the time of initialization, a view is not connected to the state. This makes view construction cheap since the whole view hierarchy is built upfront.
2. After the initialization, and before the `body` is computed, a view gets connected to the state.
3. View `body` is called for the first time.
4. Update view graph and render changes.
5. The `onAppear()` method is called top-down: from parent to child view.

# Updating
**Updating** is performed in response to an external event or state mutation.
> *External event* means the Combine publisher which is a single abstraction to represent external changes to SwiftUI  [[2]](https://developer.apple.com/videos/play/wwdc2019/226/) .

![updating](/assets/images/updating.svg)

1. A user action causes state change or SwiftUI detects data emitted by a publisher observed via `View.onReceive()`.
2. A view, which owns a mutated state or which received an external event, and all of its children, is compared against its previous snapshot. At this point, we can provide our custom definition of view equality by conforming our view to the `Equatable` protocol and wrapping it into `EquatableView`.
3. SwiftUI invalidates the views that have changed.
4. Update view graph and render invalidated views. All the updates flow down through the view hierarchy.

> Note that only views that are connected to the state can provide custom equality implementation. Views not connected to the state are always re-rendered.
An important takeaway is that pushing the state down the view hierarchy reduces the number of views to be invalidated and re-rendered when the state changes.

# Disappearing
**Disappearing** means removing a view from the hierarchy.

![disappearing](/assets/images/disappearing.svg)

The `onDisappear()` method is called after a view has been removed from the hierarchy. Similarly to `onAppear()`, `onDisappear()` is called top-down: from parent to child view.
