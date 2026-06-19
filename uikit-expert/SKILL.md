---
name: uikit-expert
description: Write, review, or improve UIKit code following best practices for view controller lifecycle, Auto Layout, collection views, navigation, animation, memory management, and modern iOS 18–26 APIs. Use when building new UIKit features, refactoring existing views or view controllers, reviewing code quality, adopting modern UIKit patterns (diffable data sources, compositional layout, cell configuration), or bridging UIKit with SwiftUI. Does not cover SwiftUI-only code.
---

# UIKit Expert Skill

## Overview
Favor native APIs and Apple's documented guidance. Operating principles: facts over architecture (no MVVM/VIPER/Coordinator mandates, but do separate business logic for testability); correctness first, then performance; reserve "always"/"never" for correctness, use "suggest"/"consider" for optional optimizations.

## Workflows

All three modes draw on the same **Core Guidelines** below; each topic links to its reference file there. Pick the entry point:

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
  8. Gate iOS 26+ features with `#available` and sensible fallbacks

## Core Guidelines

### View Controller Lifecycle — see `references/view-controller-lifecycle.md`
- Use `viewDidLoad` for one-time setup: subviews, constraints, delegates — NOT geometry
- Use `viewIsAppearing` (back-deployed iOS 13+) for geometry-dependent work, trait-based layout, scroll-to-item
- `viewDidLayoutSubviews` fires multiple times — use only for lightweight layer frame adjustments
- `viewWillAppear` is limited to transition coordinator animations and balanced notification registration
- Always call `super` in every lifecycle override
- Child VC containment: `addChild` → `addSubview` → `didMove(toParent:)` — in that exact order
- Verify deallocation with `deinit` logging during development

### Auto Layout — see `references/auto-layout.md`
- Always set `translatesAutoresizingMaskIntoConstraints = false` on programmatic views
- Use `NSLayoutConstraint.activate([])` — never individual `.isActive = true`
- Create constraints once, toggle `isActive` or modify `.constant` — never remove and recreate
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
- Use `reconfigureItems` for content updates, `reloadItems` only when cell type changes
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
- Default to `[weak self]` in all escaping closures
- Timer: use block-based API with `[weak self]`, invalidate in `viewWillDisappear`
- CADisplayLink: use weak proxy pattern (no block-based API available)
- NotificationCenter: `[weak self]` in closure, remove observer in `deinit`
- Nested closures: re-capture `[weak self]` in stored inner closures
- Delegates: always `weak var delegate: SomeDelegate?` with `AnyObject` constraint
- Verify deallocation with `deinit` — if never called, a retain cycle exists

### Concurrency — see `references/concurrency-main-thread.md`
- `UIViewController` is `@MainActor` — all subclass methods are implicitly main-actor
- Store `Task` references, cancel in `viewDidDisappear` — not `deinit`
- Check `Task.isCancelled` before UI updates after `await`
- `Task.detached` does NOT inherit actor isolation — explicit `MainActor.run` needed for UI
- Never call `DispatchQueue.main.sync` from background — use `await MainActor.run`

### UIKit–SwiftUI Interop — see `references/uikit-swiftui-interop.md`
- UIHostingController: full child VC containment (`addChild` → `addSubview` → `didMove`), retain as stored property
- `sizingOptions = .intrinsicContentSize` (iOS 16+) for Auto Layout containers
- UIViewRepresentable: set mutable state in `updateUIView`, not `makeUIView`; guard against update loops
- UIHostingConfiguration (iOS 16+) for SwiftUI content in collection view cells

### Image Loading — see `references/image-loading.md`
- Decoded bitmap size = width × height × 4 bytes (a 12MP photo = ~48MB RAM)
- Downsample with ImageIO at display size — never load full bitmap and resize
- iOS 15+: use `byPreparingThumbnail(of:)` or `prepareForDisplay()` for async decoding
- Cell reuse: cancel Task in `prepareForReuse`, clear image, verify identity on completion

### Keyboard & Scroll — see `references/keyboard-scroll.md`
- Use `UIKeyboardLayoutGuide` (iOS 15+) — pin content bottom to `view.keyboardLayoutGuide.topAnchor`
- iPad: set `followsUndockedKeyboard = true` for floating keyboards
- Replace all manual keyboard notification handling with the layout guide

### Adaptive Layout & Accessibility — see `references/adaptive-appearance.md`
- Use `registerForTraitChanges` (iOS 17+) instead of deprecated `traitCollectionDidChange`
- Dynamic Type: `UIFont.preferredFont(forTextStyle:)` + `adjustsFontForContentSizeCategory = true`
- Dark mode: use semantic colors (`.label`, `.systemBackground`); re-resolve CGColor on trait changes
- VoiceOver: set `accessibilityLabel`, `accessibilityTraits`, `accessibilityHint` on custom views
- Use `UIAccessibilityCustomAction` for complex list item actions

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
| Legacy app lifecycle | `UIScene` + `SceneDelegate` | Mandatory iOS 26 |
| `ObservableObject` + manual invalidation | `@Observable` + `UIObservationTrackingEnabled` | iOS 18 |

## Review Checklist

One gate per domain — the correctness items to verify (details in the matching Core Guideline above):

- [ ] **Lifecycle** — no geometry in `viewDidLoad` (use `viewIsAppearing`); every override calls `super`; child VC = `addChild → addSubview → didMove`; `deinit` confirms dealloc
- [ ] **Auto Layout** — `translatesAutoresizingMaskIntoConstraints = false`; `NSLayoutConstraint.activate([])`; no churn (toggle `isActive`/`.constant`); no `setNeedsLayout()` inside `layoutSubviews` (infinite loop)
- [ ] **Collection views** — diffable + stable identifiers; `reconfigureItems` not `reloadItems`; no duplicate snapshot IDs (`BUG_IN_CLIENT` crash); `CellRegistration` not string dequeue
- [ ] **Navigation** — all 4 appearance slots; on `navigationItem` not `navigationBar`; concurrent-transition guard
- [ ] **Animation** — API fits use case; constraint animation flush → update → animate; `CAAnimation` sets model value first
- [ ] **Memory** — `[weak self]` in escaping closures; `Task`s cancelled in `viewDidDisappear`; delegates `weak`; no strong-self recapture in nested closures
- [ ] **Concurrency** — `Task.isCancelled` checked after `await`; no `DispatchQueue.main.sync` from background; no redundant `@MainActor` on VC subclasses
- [ ] **Images** — downsampled to display size; cell load cancels in `prepareForReuse` + verifies identity; `NSCache` sized by decoded bytes
- [ ] **Interop** — `UIHostingController` retained + full containment; `updateUIView` guards update loops
- [ ] **Keyboard** — `UIKeyboardLayoutGuide`, not notifications
- [ ] **Adaptive/A11y** — `registerForTraitChanges`; Dynamic Type; CGColor re-resolved on trait change; `accessibilityLabel`/`Traits` on custom views
- [ ] **iOS 26+** — `#available` guards with fallbacks; `UIScene` lifecycle; consider `UIObservationTrackingEnabled`

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
