---
name: uikit-expert
description: Write, review, or improve UIKit code following best practices for view controller lifecycle, Auto Layout, collection views, navigation, animation, memory management, and modern iOS 18–26 APIs. Use when building new UIKit features, refactoring existing views or view controllers, reviewing code quality, adopting modern UIKit patterns (diffable data sources, compositional layout, cell configuration), or bridging UIKit with SwiftUI. Does not cover SwiftUI-only code.
---

# UIKit Expert Skill

## Overview
Favor native APIs and Apple's documented guidance. Operating principles: facts over architecture (no MVVM/VIPER/Coordinator mandates, but do separate business logic for testability); correctness first, then performance; reserve "always"/"never" for correctness, use "suggest"/"consider" for optional optimizations.

Reference files include step-by-step tool walkthroughs (Instruments, Memory Graph Debugger, LLDB) as developer documentation. In responses, name the relevant tool (e.g. "consider profiling with Instruments") instead of reproducing the steps.

## Workflows

All three modes draw on the same **Core Guidelines** below; each topic links to its reference file there.

**Before any mode, check the project's deployment target** (`IPHONEOS_DEPLOYMENT_TARGET`, or `platforms` in `Package.swift`). Each guideline notes its minimum iOS version: use the API directly when the target is at or above it, otherwise gate it with `#available` and keep the older API as the fallback.

Pick the entry point:

- **Review** — Walk the **Review Checklist** as pass/fail gates. For any failure, fix per the matching Core Guideline. Prioritize correctness gates (lifecycle, Auto Layout, memory, concurrency) over style; treat performance items as optional.
- **Improve** — For each issue, apply the modern replacement from the **Deprecated → Modern** table and the matching Core Guideline. Present performance work (downsampling, prefetching, constraint-churn fixes) as optional optimizations, never mandates.
- **Implement new** — Build in this order, applying the relevant Core Guideline at each step:
  1. Design data flow — owned state, injected dependencies, model layer
  2. Lifecycle — one-time setup in `viewDidLoad`, geometry in `viewIsAppearing`
  3. Auto Layout — batch activation, zero churn
  4. Collection views — DiffableDataSource + CompositionalLayout + CellRegistration
  5. Navigation — all 4 appearance slots, concurrent-transition guards
  6. Animation, memory (`[weak self]`, Task cancellation), concurrency (`@MainActor`)
  7. Interop, image loading, keyboard, Dynamic Type / VoiceOver / dark mode
  8. Gate every API newer than the deployment target (not only iOS 26+ features) with `#available` and a fallback

## Core Guidelines

### View Controller Lifecycle — see `references/view-controller-lifecycle.md`
- Use `viewDidLoad` for one-time setup: subviews, constraints, delegates — NOT geometry
- Use `viewIsAppearing` (back-deployed iOS 13+) for geometry-dependent work, trait-based layout, scroll-to-item
- `viewDidLayoutSubviews` fires multiple times — use only for lightweight layer frame adjustments
- `viewWillAppear` is limited to transition coordinator animations and balanced notification registration
- Always call `super` in lifecycle overrides — except a `loadView` override that builds the view hierarchy itself, which must not call `super`
- Child VC containment: `addChild` → `addSubview` → `didMove(toParent:)` — in that exact order
- Verify deallocation with `deinit` logging during development

### Auto Layout — see `references/auto-layout.md`
- Always set `translatesAutoresizingMaskIntoConstraints = false` on views you position with your own constraints — not on views UIKit positions for you (a pushed or presented controller's root `view`, a cell's `contentView`)
- Prefer `NSLayoutConstraint.activate([])` over setting `.isActive = true` one constraint at a time (one batch for the layout engine)
- Create constraints once, then toggle `isActive` or modify `.constant` — avoid removing and recreating them
- Never change priority from/to `.required` (1000) at runtime — use 999
- Animate constraints: update constant → call `layoutIfNeeded()` inside animation block on superview
- iOS 26+: use `.flushUpdates` option to simplify constraint animation
- Avoid deeply nested UIStackViews in reusable cells
- Set `constraint.identifier` on key constraints so `Unable to simultaneously satisfy constraints` logs are readable

```swift
let label = UILabel()
label.translatesAutoresizingMaskIntoConstraints = false   // required on programmatic views
view.addSubview(label)
NSLayoutConstraint.activate([                              // one batch = one solve pass
    label.centerXAnchor.constraint(equalTo: view.centerXAnchor),
    label.centerYAnchor.constraint(equalTo: view.centerYAnchor),
])
```

### Collection Views & Data Sources — see `references/modern-collection-views.md`, `references/cell-configuration.md`, `references/list-performance.md`
- Use `UICollectionViewDiffableDataSource` with stable identifiers (UUID/database ID, not full model structs)
- Use `reconfigureItems` (iOS 15+) for content updates, `reloadItems` only when cell type changes; with observation tracking enabled (iOS 18+ opt-in, default on iOS 26), `@Observable` reads in `configurationUpdateHandler` refresh the cell without a snapshot
- Use `applySnapshotUsingReloadData` for initial population (bypasses diffing)
- Use `UICollectionViewCompositionalLayout` for any non-trivial layout
- Use `UICollectionView.CellRegistration` — no string identifiers, no manual casting
- Use `UIContentConfiguration` for cell content and `UIBackgroundConfiguration` for cell backgrounds
- Use `configurationUpdateHandler` for state-driven styling (selection, highlight)

```swift
// Store the registration once — never create it inside the cell provider (crashes on iOS 15+).
private lazy var cellRegistration = UICollectionView.CellRegistration<PhotoCell, Photo> { cell, _, photo in
    cell.configure(with: photo)
}

// Data source is keyed by the stable ID, so the provider receives an id — look up the model, then configure with it.
dataSource = .init(collectionView: collectionView) { [weak self] cv, indexPath, id in
    guard let self, let photo = self.photo(for: id) else { return nil }
    return cv.dequeueConfiguredReusableCell(using: self.cellRegistration, for: indexPath, item: photo)
}

var snapshot = NSDiffableDataSourceSnapshot<Section, Photo.ID>()
snapshot.appendSections([.main])
snapshot.appendItems(photos.map(\.id))
dataSource.apply(snapshot, animatingDifferences: true)
```

### Navigation — see `references/navigation-patterns.md`
- Configure all 4 `UINavigationBarAppearance` slots (standard, scrollEdge, compact, compactScrollEdge)
- Set appearance on `navigationItem` (per-VC) in `viewDidLoad`, not on `navigationBar` in `viewWillAppear`
- Use `setViewControllers(_:animated:)` for deep links — not sequential push calls
- Guard against concurrent transitions — check `transitionCoordinator` before push/pop
- Set `prefersLargeTitles` once on the bar; use `largeTitleDisplayMode` per VC

### Animation — see `references/animation-patterns.md`
- `UIView.animate` — simple one-shot animations; check `finished` in completion
- `UIViewPropertyAnimator` — gesture-driven, interruptible; respect state machine (inactive → active → stopped)
- `CABasicAnimation` — layer-only properties (cornerRadius, shadow, 3D transforms); set model value first
- iOS 17+ spring API: `UIView.animate(springDuration:bounce:)` aligns with SwiftUI
- Constraint animation: flush layout → update constant → animate `layoutIfNeeded()` on superview

### Memory Management — see `references/memory-management.md`
- Default to `[weak self]` in escaping closures that are stored or long-lived (callbacks, observers, timers, stored handlers). It is not needed where the closure runs once and is released (`UIView.animate`, `DispatchQueue.main.async`) or where the API hands you the object (the `self: Self` parameter of a `registerForTraitChanges` handler)
- Timer: use block-based API with `[weak self]`; start it in an appearance callback (`viewIsAppearing`) and invalidate in `viewWillDisappear`, so it restarts when the screen reappears
- CADisplayLink: use weak proxy pattern (no block-based API available); start and invalidate it with the same appearance pairing
- NotificationCenter: `[weak self]` in closure, remove observer in `deinit`
- Nested closures: re-capture `[weak self]` in stored inner closures
- Delegates: always `weak var delegate: SomeDelegate?` with `AnyObject` constraint
- Verify deallocation with `deinit` — if never called, a retain cycle exists

### Concurrency — see `references/concurrency-main-thread.md`
- `UIViewController` is `@MainActor` — all subclass methods are implicitly main-actor
- Store `Task` references, cancel in `viewDidDisappear` — not `deinit`; start work that should run while the screen is visible in an appearance callback, so it resumes when the screen reappears
- Check `Task.isCancelled` before UI updates after `await`
- `Task.detached` does NOT inherit actor isolation — explicit `MainActor.run` needed for UI
- Avoid `DispatchQueue.main.sync` to reach the main thread (it blocks the caller and deadlocks when already on main) — use `await MainActor.run`

### UIKit–SwiftUI Interop — see `references/uikit-swiftui-interop.md`
- UIHostingController: full child VC containment (`addChild` → `addSubview` → `didMove`), retain as stored property
- `sizingOptions = .intrinsicContentSize` (iOS 16+) for Auto Layout containers
- UIViewRepresentable: set mutable state in `updateUIView`, not `makeUIView`; guard against update loops
- UIHostingConfiguration (iOS 16+) for SwiftUI content in collection view cells

### Image Loading — see `references/image-loading.md`
- Decoded bitmap size = width × height × 4 bytes (a 12MP photo = ~48MB RAM)
- Consider downsampling with ImageIO to the display pixel size instead of decoding the full bitmap and resizing
- iOS 15+: consider `byPreparingThumbnail(ofSize:)` or `byPreparingForDisplay()` for async decoding
- Cell reuse: cancel Task in `prepareForReuse`, clear image, verify identity on completion

### Keyboard & Scroll — see `references/keyboard-scroll.md`
- Use `UIKeyboardLayoutGuide` (iOS 15+) — pin content bottom to `view.keyboardLayoutGuide.topAnchor`
- iPad floating/undocked keyboard: set `followsUndockedKeyboard = true` and attach the constraints with `setConstraints(_:activeWhenNearEdge:)` / `setConstraints(_:activeWhenAwayFrom:)` — a single fixed constraint misplaces content when the keyboard floats
- Prefer the layout guide over manual keyboard notification handling; keep notifications only as the fallback below iOS 15

### Adaptive Layout & Accessibility — see `references/adaptive-appearance.md`
- Use `registerForTraitChanges` (iOS 17+) instead of deprecated `traitCollectionDidChange`; when the deployment target is below iOS 17, gate the registration with `#available` and keep `traitCollectionDidChange` as the fallback
- Dynamic Type: `UIFont.preferredFont(forTextStyle:)` + `adjustsFontForContentSizeCategory = true`
- Dark mode: use semantic colors (`.label`, `.systemBackground`); re-resolve CGColor on trait changes with `resolvedColor(with: traitCollection).cgColor` — outside layout/draw callbacks and trait-change handlers the current trait collection is not guaranteed
- VoiceOver: set `accessibilityLabel`, `accessibilityTraits`, `accessibilityHint` on custom views
- Use `UIAccessibilityCustomAction` for complex list item actions

### Modern APIs (iOS 18–26) — see `references/modern-uikit-apis.md`
- Observation tracking: opt in with `UIObservationTrackingEnabled` on iOS 18+ (on by default in iOS 26); `@Observable` properties read in `layoutSubviews`, `viewWillLayoutSubviews`, or `configurationUpdateHandler` re-run that method when they change
- Never write to an observed property inside a tracked method — it re-triggers the same method (update storm)
- iOS 26+: put content and styling in `updateProperties()`, keep geometry in `layoutSubviews()`
- iOS 26+: `.flushUpdates` animation option replaces the manual `layoutIfNeeded()` steps
- `UIScene` lifecycle: iOS 26 only logs a warning; it becomes required when building with the SDK of the release after iOS 26
- Liquid Glass (iOS 26+): standard bars adopt it automatically; gate `UIGlassEffect` with `#available` and fall back to `UIBlurEffect`

## Quick Reference — Deprecated → Modern APIs
| Deprecated / Legacy | Modern Replacement | Since |
|---------------------|-------------------|-------|
| `traitCollectionDidChange` | `registerForTraitChanges(_:handler:)` | iOS 17 |
| Keyboard notifications | `UIKeyboardLayoutGuide` | iOS 15 |
| `cell.textLabel` / `detailTextLabel` | `UIListContentConfiguration` | iOS 14 |
| `register` + string dequeue | `UICollectionView.CellRegistration` | iOS 14 |
| `reloadItems` on snapshot | `reconfigureItems` | iOS 15 |
| `barTintColor` / `isTranslucent` | `UINavigationBarAppearance` (4 slots) | iOS 13 |
| `UICollectionViewFlowLayout` (complex) | `UICollectionViewCompositionalLayout` | iOS 13 |
| Manual `layoutIfNeeded()` in animations | `.flushUpdates` option | iOS 26 |
| Legacy app lifecycle | `UIScene` + `SceneDelegate` | iOS 13; required with the SDK after iOS 26 |
| `ObservableObject` + manual invalidation | `@Observable` + `UIObservationTrackingEnabled` | iOS 18 |

## Review Checklist

One gate per domain (details in the matching Core Guideline above). Items before *Optional* are correctness gates; items after *Optional* are performance or modernization suggestions — raise them, but do not fail a review on them.

- [ ] **Lifecycle** — no geometry in `viewDidLoad` (use `viewIsAppearing`); overrides call `super` (except a `loadView` that builds the view); child VC = `addChild → addSubview → didMove`; `deinit` confirms dealloc
- [ ] **Auto Layout** — `translatesAutoresizingMaskIntoConstraints = false` on views you constrain; no priority change to/from `.required` at runtime; no `setNeedsLayout()` inside `layoutSubviews` (infinite loop). *Optional:* batch with `NSLayoutConstraint.activate([])`; no churn (toggle `isActive`/`.constant`)
- [ ] **Collection views** — diffable + stable identifiers; no duplicate snapshot IDs (`BUG_IN_CLIENT` crash); registration not created inside the cell provider. *Optional:* `reconfigureItems` instead of `reloadItems`; `CellRegistration` instead of string dequeue
- [ ] **Navigation** — all 4 appearance slots; on `navigationItem` not `navigationBar`; concurrent-transition guard
- [ ] **Animation** — API fits use case; constraint animation flush → update → animate; `CAAnimation` sets model value first
- [ ] **Memory** — `[weak self]` in stored or long-lived escaping closures; `Task`s cancelled in `viewDidDisappear`; timers/display links/tasks started in an appearance callback when stopped on disappearance; delegates `weak`; no strong-self recapture in nested closures
- [ ] **Concurrency** — `Task.isCancelled` checked after `await`; no `DispatchQueue.main.sync` hop to main. *Optional:* no redundant `@MainActor` on VC subclasses
- [ ] **Images** — cell load cancels in `prepareForReuse` + verifies identity. *Optional:* downsampled to display size; `NSCache` cost by decoded bytes
- [ ] **Interop** — `UIHostingController` retained + full containment; `updateUIView` guards update loops
- [ ] **Keyboard** — content follows the keyboard correctly; undocked tracking uses `setConstraints(_:activeWhen…)`. *Optional:* `UIKeyboardLayoutGuide` instead of notifications (iOS 15+)
- [ ] **Adaptive/A11y** — trait changes handled (`registerForTraitChanges` on iOS 17+, `traitCollectionDidChange` fallback below); Dynamic Type; CGColor re-resolved with `resolvedColor(with:)`; `accessibilityLabel`/`Traits` on custom views
- [ ] **Modern APIs** — every API newer than the deployment target is behind `#available` with a fallback; no writes to observed properties inside tracked methods. *Optional:* `UIScene` lifecycle; `UIObservationTrackingEnabled`

## References
- `references/view-controller-lifecycle.md` — Lifecycle ordering, viewIsAppearing, child VC containment
- `references/auto-layout.md` — Batch activation, constraint churn, priority, animation, debugging
- `references/modern-collection-views.md` — Diffable data sources, compositional layout, CellRegistration
- `references/cell-configuration.md` — UIContentConfiguration, UIBackgroundConfiguration, configurationUpdateHandler
- `references/list-performance.md` — Prefetching, cell reuse, reconfigureItems, scroll performance
- `references/navigation-patterns.md` — Bar appearance, concurrent transitions, large titles, deep links
- `references/animation-patterns.md` — UIView.animate, UIViewPropertyAnimator, CAAnimation, springs
- `references/memory-management.md` — Retain cycles, [weak self], Timer/CADisplayLink/nested closure traps
- `references/concurrency-main-thread.md` — @MainActor, Task lifecycle, Swift 6, GCD migration
- `references/uikit-swiftui-interop.md` — UIHostingController, UIViewRepresentable, sizing, state bridging
- `references/image-loading.md` — Downsampling, decoded bitmap math, cell reuse race condition
- `references/keyboard-scroll.md` — UIKeyboardLayoutGuide, scroll view insets, iPad floating keyboard
- `references/adaptive-appearance.md` — Trait changes, Dynamic Type, dark mode, VoiceOver, accessibility
- `references/modern-uikit-apis.md` — Observation framework, updateProperties(), .flushUpdates, UIScene, Liquid Glass
