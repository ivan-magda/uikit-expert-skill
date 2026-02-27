# UIViewController lifecycle: the definitive correctness guide

**`viewIsAppearing(_:)` has changed the rules.** Introduced at WWDC 2023 and back-deployed to iOS 13, this single callback fills the longstanding gap between "too early" (`viewWillAppear`) and "too often" (`viewDidLayoutSubviews`), giving you accurate geometry and traits exactly once per appearance — before the user sees anything. This guide covers every lifecycle method from `init` through `deinit` with ✅ correct and ❌ incorrect Swift code for each pattern, reflecting Apple's documentation and community best practices through 2024–2026.

---

## The exact callback sequence from birth to death

UIKit calls lifecycle methods in a strict, documented order. Understanding this sequence is foundational — every mistake in later sections traces back to a developer assuming a method fires at a different point than it actually does.

**Phase 1 — Creation and loading (once per lifetime):**

1. `init(coder:)` or `init(nibName:bundle:)` — object allocation
2. `loadView()` — creates or loads the view hierarchy
3. `viewDidLoad()` — view is in memory; outlets connected

**Phase 2 — Appearance transition (every time the view appears):**

4. `viewWillAppear(_:)` — transition begins; view is **not** yet in the hierarchy
5. *(System adds view to hierarchy)*
6. *(System updates trait collections)*
7. *(System updates geometry — size, safe area insets, margins)*
8. `viewIsAppearing(_:)` — traits and geometry are **accurate**; view is not yet visible on screen
9. `viewWillLayoutSubviews()` — about to run layout
10. *(Auto Layout pass)*
11. `viewDidLayoutSubviews()` — layout complete
12. *(System composites frame to display)*
13. `viewDidAppear(_:)` — transition animation finished; view is on screen

**Phase 3 — Disappearance (every time the view leaves):**

14. `viewWillDisappear(_:)` — still visible
15. `viewDidDisappear(_:)` — removed from screen

**Phase 4 — Deallocation (once):**

16. `deinit` — object destroyed

Steps 4–8 happen inside the **same `CATransaction`**, so UI changes made in any of those callbacks become visible to the user simultaneously. Step 13 (`viewDidAppear`) runs in a **separate** transaction — changes there produce a visible flash. Steps 9–11 can fire **multiple times** during a single appearance and whenever layout is invalidated while the view is visible.

```swift
// ✅ Lifecycle logger — paste into any VC for debugging
final class LifecycleVC: UIViewController {
    override func loadView()                           { super.loadView(); print("1 ─ loadView") }
    override func viewDidLoad()                        { super.viewDidLoad(); print("2 ─ viewDidLoad") }
    override func viewWillAppear(_ a: Bool)            { super.viewWillAppear(a); print("3 ─ viewWillAppear") }
    override func viewIsAppearing(_ a: Bool)           { super.viewIsAppearing(a); print("4 ─ viewIsAppearing") }
    override func viewWillLayoutSubviews()             { super.viewWillLayoutSubviews(); print("5 ─ viewWillLayoutSubviews") }
    override func viewDidLayoutSubviews()              { super.viewDidLayoutSubviews(); print("6 ─ viewDidLayoutSubviews") }
    override func viewDidAppear(_ a: Bool)             { super.viewDidAppear(a); print("7 ─ viewDidAppear") }
    override func viewWillDisappear(_ a: Bool)         { super.viewWillDisappear(a); print("8 ─ viewWillDisappear") }
    override func viewDidDisappear(_ a: Bool)          { super.viewDidDisappear(a); print("9 ─ viewDidDisappear") }
    deinit                                             { print("10 ─ deinit") }
}
```

---

## viewIsAppearing fills the gap that existed for 15 years

Before iOS 17 / WWDC 2023, developers faced an impossible choice. `viewWillAppear` fires before the view enters the hierarchy, so **safe area insets are all zeros** and **trait collections may be stale**. `viewDidLayoutSubviews` has correct geometry but fires many times — even while the view is already on screen — forcing developers to use boolean flags for one-shot work. `viewDidAppear` has everything but fires after the view is visible, causing flashes.

`viewIsAppearing(_:)` solves this cleanly. It fires **once per appearance**, after the view is in the hierarchy with **updated traits and accurate geometry**, but before the view is composited to the screen.

| State at callback time | `viewWillAppear` | `viewIsAppearing` |
|---|---|---|
| View added to hierarchy | ❌ | ✅ |
| Trait collections updated | ❌ | ✅ |
| Geometry (size, safe area) accurate | ❌ | ✅ |
| Transition coordinator available | ✅ | ❌ |
| Fires once per appearance | ✅ | ✅ |

The method is declared with `@available(iOS 13.0, *)` — Apple back-deployed it because the implementation existed internally since iOS 13. **No availability check is needed** for any app targeting iOS 13+.

```swift
// ❌ INCORRECT — safe area insets are (0,0,0,0) in viewWillAppear
override func viewWillAppear(_ animated: Bool) {
    super.viewWillAppear(animated)
    // safeAreaInsets = UIEdgeInsets(top: 0, left: 0, bottom: 0, right: 0) ← WRONG
    let usableHeight = view.bounds.height - view.safeAreaInsets.top - view.safeAreaInsets.bottom
    headerView.frame.size.height = usableHeight * 0.3  // Calculated from zero insets!
}

// ❌ INCORRECT — trait collections may be stale from previous appearance
override func viewWillAppear(_ animated: Bool) {
    super.viewWillAppear(animated)
    // After iPad Split View resize, this still reports .regular
    // from the PREVIOUS appearance — not the current compact width
    if traitCollection.horizontalSizeClass == .compact {
        enableSingleColumnLayout()  // May never execute!
    }
}
```

```swift
// ✅ CORRECT — viewIsAppearing has accurate geometry and traits
override func viewIsAppearing(_ animated: Bool) {
    super.viewIsAppearing(animated)

    // Safe area insets are real values: e.g. (top: 59, left: 0, bottom: 34, right: 0)
    let usableHeight = view.bounds.height - view.safeAreaInsets.top - view.safeAreaInsets.bottom
    headerView.frame.size.height = usableHeight * 0.3

    // Trait collections are current — reflects iPad Split View state
    if traitCollection.horizontalSizeClass == .compact {
        enableSingleColumnLayout()
    }

    // Scroll to a specific position (requires accurate content size)
    if let selected = tableView.indexPathForSelectedRow {
        tableView.scrollToRow(at: selected, at: .middle, animated: false)
    }
}
```

**When to still use `viewWillAppear`:** Only for two cases — (1) alongside transition animations via the `transitionCoordinator`, and (2) balanced notification registration paired with `viewDidDisappear`. For everything else, **`viewIsAppearing` is the default choice** per Apple's own guidance.

---

## What work belongs in each lifecycle method

Each callback has a precise role. Placing work in the wrong method is the root cause of most UIViewController bugs.

### loadView — create views programmatically (no Storyboard)

```swift
// ✅ CORRECT — programmatic view creation; do NOT call super
override func loadView() {
    let root = CustomRootView()
    root.backgroundColor = .systemBackground
    self.view = root  // Required: must assign self.view
}
```

```swift
// ❌ INCORRECT — calling super.loadView() when overriding
override func loadView() {
    super.loadView()          // Creates a default UIView you immediately replace — wasteful
    self.view = CustomRootView()
}

// ❌ INCORRECT — overriding loadView when using Storyboard
override func loadView() {
    view = UIView()           // Prevents Storyboard view from loading!
}

// ❌ INCORRECT — accessing self.view inside loadView
override func loadView() {
    let v = UIView()
    v.addSubview(self.view)   // Triggers loadView again → infinite recursion!
    self.view = v
}

// ❌ INCORRECT — calling loadView() directly
let vc = MyViewController()
vc.loadView()                 // Never do this; use loadViewIfNeeded() if you must
```

### viewDidLoad — one-time, geometry-independent setup

```swift
// ✅ CORRECT — add subviews, constraints, data sources; no geometry math
override func viewDidLoad() {
    super.viewDidLoad()
    view.addSubview(tableView)
    NSLayoutConstraint.activate([
        tableView.topAnchor.constraint(equalTo: view.topAnchor),
        tableView.leadingAnchor.constraint(equalTo: view.leadingAnchor),
        tableView.trailingAnchor.constraint(equalTo: view.trailingAnchor),
        tableView.bottomAnchor.constraint(equalTo: view.bottomAnchor)
    ])
    tableView.dataSource = self
    tableView.register(Cell.self, forCellReuseIdentifier: "cell")
}
```

```swift
// ❌ INCORRECT — geometry is not final in viewDidLoad
override func viewDidLoad() {
    super.viewDidLoad()
    // view.bounds is often (0, 0, 320, 480) — storyboard placeholder, not real device size
    let circle = UIView(frame: CGRect(x: 0, y: 0,
        width: view.bounds.width / 2,       // Wrong width!
        height: view.bounds.width / 2))
    circle.layer.cornerRadius = view.bounds.width / 4  // Wrong radius!
    view.addSubview(circle)
}
```

### viewWillAppear — transition animations and notification registration only

```swift
// ✅ CORRECT — alongside animation (requires transitionCoordinator)
override func viewWillAppear(_ animated: Bool) {
    super.viewWillAppear(animated)

    // Register notifications (balanced with viewDidDisappear)
    NotificationCenter.default.addObserver(
        self, selector: #selector(dataChanged),
        name: .dataDidUpdate, object: nil
    )

    // Animate alongside the push/present transition
    transitionCoordinator?.animate(alongsideTransition: { _ in
        self.navigationController?.navigationBar.tintColor = .systemBlue
    })
}
```

```swift
// ❌ INCORRECT — expensive network call fires on every tab switch
override func viewWillAppear(_ animated: Bool) {
    super.viewWillAppear(animated)
    // This fires every time user switches tabs, pops back, or dismisses a modal!
    NetworkService.shared.fetchAllUsers { [weak self] users in
        self?.users = users
        self?.tableView.reloadData()
    }
}
```

### viewIsAppearing — the default per-appearance callback

Use for all UI updates that need traits or geometry. Called once per appearance.

### viewWillLayoutSubviews / viewDidLayoutSubviews — ongoing layout adjustments

```swift
// ✅ CORRECT — corner radius that updates whenever bounds change
override func viewDidLayoutSubviews() {
    super.viewDidLayoutSubviews()
    avatarView.layer.cornerRadius = avatarView.bounds.height / 2
    avatarView.layer.shadowPath = UIBezierPath(
        ovalIn: avatarView.bounds
    ).cgPath
}
```

```swift
// ❌ INCORRECT — one-time setup in a method called dozens of times
override func viewDidLayoutSubviews() {
    super.viewDidLayoutSubviews()
    // This creates a new gradient layer on EVERY layout pass!
    let gradient = CAGradientLayer()
    gradient.frame = view.bounds
    gradient.colors = [UIColor.red.cgColor, UIColor.blue.cgColor]
    view.layer.insertSublayer(gradient, at: 0)  // Stacking layers endlessly
}
```

### viewDidAppear — post-transition work the user should see start

```swift
// ✅ CORRECT — start visible animations after the view is on screen
override func viewDidAppear(_ animated: Bool) {
    super.viewDidAppear(animated)
    startPulseAnimation()
    analyticsService.trackScreenView("HomeScreen")
}
```

```swift
// ❌ INCORRECT — UI configuration in viewDidAppear causes visible flash
override func viewDidAppear(_ animated: Bool) {
    super.viewDidAppear(animated)
    // User sees the OLD background first, then it snaps to green
    view.backgroundColor = .systemGreen
}
```

### viewWillDisappear / viewDidDisappear — teardown and cleanup

```swift
// ✅ CORRECT — balanced cleanup
override func viewWillDisappear(_ animated: Bool) {
    super.viewWillDisappear(animated)
    view.endEditing(true)               // Resign first responder
    observationTask?.cancel()           // Cancel async work
}

override func viewDidDisappear(_ animated: Bool) {
    super.viewDidDisappear(animated)
    NotificationCenter.default.removeObserver(self)  // Balanced with viewWillAppear
}
```

---

## Child view controller containment demands exact ordering

The containment API has strict rules about which calls UIKit handles automatically and which you must make yourself. Getting the order wrong silently breaks appearance callbacks, trait propagation, and event routing.

### The two automatic calls you must not duplicate

- **`addChild(_:)` automatically calls `willMove(toParent: self)`** on the child. You do not call it yourself when adding.
- **`removeFromParent()` automatically calls `didMove(toParent: nil)`** on the child. You do not call it yourself when removing.

You **must** manually call `didMove(toParent: self)` after adding the view, and `willMove(toParent: nil)` before removing it.

```swift
// ✅ CORRECT — adding a child view controller
func addContent(_ child: UIViewController) {
    addChild(child)                        // Step 1: parent-child link (auto-calls willMove)
    child.view.frame = view.bounds         // Step 2: size the view
    child.view.autoresizingMask = [.flexibleWidth, .flexibleHeight]
    view.addSubview(child.view)            // Step 3: view hierarchy
    child.didMove(toParent: self)          // Step 4: notify completion ← YOU must call this
}

// ✅ CORRECT — removing a child view controller
func removeContent(_ child: UIViewController) {
    child.willMove(toParent: nil)          // Step 1: notify impending removal ← YOU must call
    child.view.removeFromSuperview()       // Step 2: view hierarchy
    child.removeFromParent()               // Step 3: parent-child link (auto-calls didMove)
}
```

```swift
// ❌ INCORRECT — skipping addChild entirely
func addContentBroken(_ child: UIViewController) {
    view.addSubview(child.view)            // Just adding the view
    // Result: child never receives viewWillAppear/viewDidAppear,
    // trait collection changes don't propagate,
    // rotation callbacks are missed,
    // Xcode may throw UIViewControllerHierarchyInconsistency in debug
}

// ❌ INCORRECT — forgetting didMove(toParent:)
func addContentIncomplete(_ child: UIViewController) {
    addChild(child)
    view.addSubview(child.view)
    // Missing: child.didMove(toParent: self)
    // The child's didMove(toParent:) override never fires — any setup there is skipped
}

// ❌ INCORRECT — reversed removal order
func removeContentBackwards(_ child: UIViewController) {
    child.removeFromParent()               // Too early! Calls didMove(toParent: nil) prematurely
    child.view.removeFromSuperview()
    child.willMove(toParent: nil)          // Too late! Child already detached from parent
}

// ❌ INCORRECT — redundant willMove call when adding
func addContentRedundant(_ child: UIViewController) {
    child.willMove(toParent: self)         // Unnecessary — addChild does this automatically
    addChild(child)                        // Calls willMove again — child gets notified twice
    view.addSubview(child.view)
    child.didMove(toParent: self)
}
```

### Animated transitions between children

```swift
// ✅ CORRECT — transitioning from one child to another
func swap(from oldVC: UIViewController, to newVC: UIViewController) {
    oldVC.willMove(toParent: nil)
    addChild(newVC)

    transition(from: oldVC, to: newVC,
               duration: 0.3,
               options: .transitionCrossDissolve,
               animations: nil) { _ in
        oldVC.removeFromParent()           // Auto-calls didMove(toParent: nil)
        newVC.didMove(toParent: self)
    }
}
```

---

## Five common mistakes that cause subtle, hard-to-trace bugs

**Mistake 1: Geometry in viewDidLoad.** The view's bounds are storyboard placeholders — often **320 × 480** regardless of device. Any frame-based calculation will be wrong. Move it to `viewIsAppearing` or `viewDidLayoutSubviews`.

**Mistake 2: Expensive work in viewWillAppear.** This method fires on every tab switch, every navigation pop, and every full-screen modal dismissal. A network request here means redundant calls dozens of times per session. One-time fetches belong in `viewDidLoad`; lightweight refresh or UI sync belongs in `viewIsAppearing`.

**Mistake 3: Forgetting `super` calls.** Every lifecycle method except `loadView` (when overriding programmatically) **requires** calling `super`. Missing it breaks UIKit's internal state machine, prevents child forwarding, and can silently swallow appearance callbacks for child view controllers.

```swift
// ❌ INCORRECT — missing super
override func viewWillAppear(_ animated: Bool) {
    // No super call → children never receive viewWillAppear
    updateUI()
}

// ✅ CORRECT — always call super
override func viewWillAppear(_ animated: Bool) {
    super.viewWillAppear(animated)
    updateUI()
}
```

**Mistake 4: Expecting viewWillAppear after dismissing a sheet.** With `.pageSheet` or `.formSheet` presentation (the default since iOS 13), the presenting view controller's view **stays in the hierarchy**. When the sheet is dismissed, `viewWillAppear` is **not called** because the view never disappeared. Only `.fullScreen` and `.overFullScreen` styles trigger the full appearance cycle.

**Mistake 5: Presenting a modal from viewDidLoad or viewWillAppear.**

```swift
// ❌ INCORRECT — view is not in the window hierarchy yet
override func viewDidLoad() {
    super.viewDidLoad()
    present(loginVC, animated: true)
    // Console: "whose view is not in the window hierarchy"
}

// ✅ CORRECT — wait until the view is on screen
override func viewDidAppear(_ animated: Bool) {
    super.viewDidAppear(animated)
    if needsLogin {
        present(loginVC, animated: true)
    }
}
```

---

## Deallocation verification catches retain cycles early

If a view controller's `deinit` never fires after it leaves the screen, you have a memory leak. The simplest check:

```swift
// ✅ Add to every view controller during development
deinit {
    print("✅ \(type(of: self)) deallocated")
}
```

Pop or dismiss the controller. If nothing prints, you have a retain cycle. Use Xcode's **Memory Graph Debugger** (three-circles icon in the debug bar) to identify the specific object and property causing the cycle.

For the complete deallocation verification toolkit (delayed assertions, symbolic breakpoints, unit test helpers, community libraries, and the full Memory Graph Debugger workflow), see `references/memory-management.md` § "Five patterns for verifying UIViewController deallocation".

### The most common leak patterns

```swift
// ❌ LEAK — strong self in stored closure
class ProfileVC: UIViewController {
    var onUpdate: (() -> Void)?

    override func viewDidLoad() {
        super.viewDidLoad()
        onUpdate = {
            self.refresh()     // self → onUpdate → closure → self
        }
    }
}

// ✅ FIX — weak capture
onUpdate = { [weak self] in
    self?.refresh()
}
```

```swift
// ❌ LEAK — async for-loop never terminates
override func viewWillAppear(_ animated: Bool) {
    super.viewWillAppear(animated)
    Task {
        for await value in stream.values {
            updateUI(value)    // Holds self alive forever
        }
    }
}

// ✅ FIX — store and cancel the task
private var streamTask: Task<Void, Never>?

override func viewWillAppear(_ animated: Bool) {
    super.viewWillAppear(animated)
    streamTask = Task { [weak self] in
        for await value in stream.values {
            self?.updateUI(value)
        }
    }
}

override func viewWillDisappear(_ animated: Bool) {
    super.viewWillDisappear(animated)
    streamTask?.cancel()       // Breaks the infinite await
}
```

---

## TransitionCoordinator synchronizes rotation and multitasking animations

When the device rotates or the user resizes an iPad Split View window, UIKit calls `viewWillTransition(to:with:)` with a coordinator that lets you animate custom changes in lockstep with the system's bounds animation.

```swift
// ✅ CORRECT — animate alongside rotation/resize
override func viewWillTransition(
    to size: CGSize,
    with coordinator: UIViewControllerTransitionCoordinator
) {
    super.viewWillTransition(to: size, with: coordinator)  // Required — forwards to children

    let isLandscape = size.width > size.height

    coordinator.animate(alongsideTransition: { [weak self] _ in
        self?.sidebarWidthConstraint.constant = isLandscape ? 320 : 0
        self?.view.layoutIfNeeded()
    }, completion: { [weak self] context in
        if !context.isCancelled {
            self?.updateCellSizes(for: size)
        }
    })
}
```

```swift
// ❌ INCORRECT — missing super; children never learn about size change
override func viewWillTransition(
    to size: CGSize,
    with coordinator: UIViewControllerTransitionCoordinator
) {
    // No super call! Child VCs will not receive the size change notification
    updateLayout(for: size)
}

// ❌ INCORRECT — raw UIView.animate instead of coordinator
override func viewWillTransition(
    to size: CGSize,
    with coordinator: UIViewControllerTransitionCoordinator
) {
    super.viewWillTransition(to: size, with: coordinator)
    // This animation runs on its OWN timeline, out of sync with the rotation
    UIView.animate(withDuration: 0.3) {
        self.sidebarWidthConstraint.constant = size.width > size.height ? 320 : 0
        self.view.layoutIfNeeded()
    }
}
```

The `transitionCoordinator` is also available during view controller transitions in `viewWillAppear`. This is the **only** reason to still use `viewWillAppear` for animation work — `viewIsAppearing` does not expose the coordinator.

```swift
// ✅ CORRECT — alongside animation during a push/present
override func viewWillAppear(_ animated: Bool) {
    super.viewWillAppear(animated)

    transitionCoordinator?.animate(alongsideTransition: { _ in
        self.dimmingView.alpha = 0.5
    }, completion: { context in
        if context.isCancelled {
            self.dimmingView.alpha = 0.0   // Revert if interactive pop cancelled
        }
    })
}
```

---

## Conclusion

The UIViewController lifecycle in 2024–2026 has a clear decision framework. **`viewIsAppearing` is the new default** for per-appearance UI work — it gives you real geometry and current traits in a callback that fires exactly once, eliminating years of workarounds with boolean flags in `viewDidLayoutSubviews`. Reserve `viewDidLoad` for one-time, size-independent setup. Reserve `viewWillAppear` exclusively for transition coordinator animations and balanced notification registration. Use `viewDidLayoutSubviews` only for continuously-updating layer calculations like corner radii that must respond to every layout pass.

For child containment, the asymmetry is the key mental model: **`addChild` auto-calls `willMove`; `removeFromParent` auto-calls `didMove`**. You manually supply the other half. Reversing the order doesn't crash — it silently breaks appearance forwarding, which is worse.

Every view controller should ship with a `deinit` print during development. Combine it with the Memory Graph Debugger for retain cycle forensics, and use `[weak self]` in all stored closures and long-lived async tasks. These patterns are simple, mechanical, and catch the majority of UIKit memory leaks before they reach production.
---

## Summary Checklist

- [ ] `viewDidLoad` contains only one-time, geometry-independent setup (subviews, constraints, delegates)
- [ ] Geometry-dependent work (layer frames, scroll-to-item, trait-based layout) is in `viewIsAppearing`, not `viewWillAppear` or `viewDidLoad`
- [ ] `viewWillAppear` is used only for transition coordinator animations or balanced notification registration
- [ ] `viewDidLayoutSubviews` does NOT add subviews, activate constraints, or call `setNeedsLayout()`
- [ ] Every lifecycle override calls `super` (forgetting `super.viewWillAppear(animated)` silently breaks UIKit state)
- [ ] Child VC containment follows exact order: `addChild` → `addSubview` → `didMove(toParent:)`
- [ ] Child VC removal follows: `willMove(toParent: nil)` → `removeFromSuperview` → `removeFromParent`
- [ ] `deinit` contains a debug print/log for leak verification during development
- [ ] No expensive work (network, heavy computation) in `viewWillAppear` — it fires on every tab switch and navigation pop
- [ ] `loadView` override does NOT call `super.loadView()`
